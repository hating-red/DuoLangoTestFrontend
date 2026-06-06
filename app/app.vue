<template>
  <div class="page">
    <NuxtRouteAnnouncer />

    <main class="recorder">
      <section class="panel">
        <div class="heading">
          <p class="eyebrow">Тренировка произношения</p>
          <h1>Повторяйте текст вслух</h1>
          <p class="lead">
            Слова будут загораться зелёным, когда распознавание подтвердит, что вы произнесли их по порядку.
          </p>
        </div>

        <article class="reading-card" aria-label="Текст для чтения">
          <div class="reading-progress">
            <span>Прогресс</span>
            <strong>{{ highlightedWordsCount }} / {{ targetWordsCount }}</strong>
          </div>

          <p class="target-text">
            <template v-for="part in highlightedTargetParts" :key="part.id">
              <span v-if="part.kind === 'word'" class="target-word" :class="{ spoken: part.isSpoken }">
                {{ part.text }}
              </span>
              <span v-else>{{ part.text }}</span>
            </template>
          </p>
        </article>

        <div class="controls">
          <button class="record-button" :class="{ active: isRecording }" type="button" :disabled="isBusy"
            @click="toggleRecording">
            <span class="record-dot" />
            {{ isRecording ? 'Остановить запись' : 'Начать запись' }}
          </button>

          <div class="status" :class="statusClass">
            <span />
            {{ statusText }}
          </div>
        </div>

        <label class="transcript-label" for="transcript">Расшифровка</label>
        <textarea id="transcript" class="transcript" :value="visibleTranscript" readonly />

        <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
      </section>
    </main>
  </div>
</template>

<script setup lang="ts">
const TARGET_SAMPLE_RATE = 16000
const VOICE_ACTIVITY_THRESHOLD = 0.012
const AUTO_STOP_SILENCE_MS = 7000
const TARGET_TEXT = `Приставка ПРИ
1.Пространственная БЛИЗОСТЬ:
прибрежный, пригородный, привратник, прилюдно
2. Приближение, присоединение:
пришить, приехать, прилететь, прибавить, принести
3. Неполнота действия:
приоткрыть, присесть, причуда, прискорбный
4. Доведение действия до конца:
придумать, прибить
5. Совершение действия в чьих-либо интересах:
приберечь, припрятать, прикопить`

type SpeechMessage =
  | { type: 'partial' | 'final' | 'final_refinement'; text: string }
  | { type: 'status'; message: string }
  | { type: 'error'; message: string }
  | { type: 'done' }

type TargetPart =
  | { id: string; kind: 'word'; text: string; wordIndex: number }
  | { id: string; kind: 'separator'; text: string }

const WORD_PATTERN = /[\p{L}\p{N}]+/gu
const targetParts = parseTargetText(TARGET_TEXT)
const targetWords = targetParts.filter((part) => part.kind === 'word')

const config = useRuntimeConfig()

const isRecording = ref(false)
const isBusy = ref(false)
const finalSegments = ref<string[]>([])
const partialTranscript = ref('')
const statusText = ref('Готово к записи')
const statusKind = ref<'idle' | 'recording' | 'error'>('idle')
const errorMessage = ref('')

let socket: WebSocket | null = null
let socketPromise: Promise<WebSocket> | null = null
let mediaStream: MediaStream | null = null
let audioContext: AudioContext | null = null
let sourceNode: MediaStreamAudioSourceNode | null = null
let processorNode: ScriptProcessorNode | null = null
let hasStartedStreaming = false
let lastSpeechAt = 0

const visibleTranscript = computed(() => {
  const finalText = finalSegments.value.join(' ').trim()
  const partialText = partialTranscript.value.trim()

  return [finalText, partialText].filter(Boolean).join(finalText && partialText ? ' ' : '')
})

const spokenWords = computed(() => extractWords(visibleTranscript.value))
const highlightedWordIndexes = computed(() => getRelaxedMatchedIndexes(targetWords, spokenWords.value))
const highlightedWordsCount = computed(() => highlightedWordIndexes.value.size)
const targetWordsCount = computed(() => targetWords.length)

const highlightedTargetParts = computed(() =>
  targetParts.map((part) => {
    if (part.kind === 'separator') {
      return part
    }

    return {
      ...part,
      isSpoken: highlightedWordIndexes.value.has(part.wordIndex),
    }
  }),
)

const statusClass = computed(() => ({
  recording: statusKind.value === 'recording',
  error: statusKind.value === 'error',
}))

onBeforeUnmount(() => {
  void stopRecording()
})

async function toggleRecording() {
  if (isRecording.value) {
    await stopRecording()
    return
  }

  await startRecording()
}

async function startRecording() {
  isBusy.value = true
  errorMessage.value = ''
  finalSegments.value = []
  partialTranscript.value = ''
  statusText.value = 'Жду речь'
  statusKind.value = 'idle'
  hasStartedStreaming = false
  lastSpeechAt = 0

  try {
    mediaStream = await navigator.mediaDevices.getUserMedia({
      audio: {
        channelCount: 1,
        echoCancellation: true,
        noiseSuppression: true,
        autoGainControl: true,
      },
    })

    audioContext = new AudioContext()
    sourceNode = audioContext.createMediaStreamSource(mediaStream)
    processorNode = audioContext.createScriptProcessor(4096, 1, 1)

    processorNode.onaudioprocess = (event) => {
      void processAudioFrame(event.inputBuffer.getChannelData(0))
    }

    sourceNode.connect(processorNode)
    processorNode.connect(audioContext.destination)

    isRecording.value = true
    statusText.value = 'Жду речь'
    statusKind.value = 'recording'
  } catch (error) {
    errorMessage.value = getErrorMessage(error)
    statusText.value = 'Ошибка'
    statusKind.value = 'error'
    await stopRecording()
  } finally {
    isBusy.value = false
  }
}

async function stopRecording() {
  isBusy.value = true

  processorNode?.disconnect()
  sourceNode?.disconnect()
  processorNode = null
  sourceNode = null

  mediaStream?.getTracks().forEach((track) => track.stop())
  mediaStream = null

  if (audioContext && audioContext.state !== 'closed') {
    await audioContext.close()
  }
  audioContext = null

  if (socket && socket.readyState === WebSocket.OPEN) {
    socket.send(JSON.stringify({ type: 'stop' }))
    socket.close(1000, 'Recording stopped')
  }
  socket = null
  socketPromise = null
  hasStartedStreaming = false
  lastSpeechAt = 0

  isRecording.value = false
  statusText.value = errorMessage.value ? 'Ошибка' : 'Готово к записи'
  statusKind.value = errorMessage.value ? 'error' : 'idle'
  partialTranscript.value = ''
  isBusy.value = false
}

async function processAudioFrame(input: Float32Array) {
  if (!isRecording.value || !audioContext) {
    return
  }

  const volume = getRmsVolume(input)
  const now = performance.now()

  if (volume < VOICE_ACTIVITY_THRESHOLD) {
    if (hasStartedStreaming && now - lastSpeechAt > AUTO_STOP_SILENCE_MS) {
      statusText.value = 'Остановлено по тишине'
      await stopRecording()
    }

    return
  }

  lastSpeechAt = now

  const pcm = convertFloat32ToPcm16(input, audioContext.sampleRate, TARGET_SAMPLE_RATE)
  if (pcm.byteLength === 0) {
    return
  }

  try {
    const ws = await ensureSpeechSocket()

    if (ws.readyState === WebSocket.OPEN) {
      ws.send(pcm)
    }
  } catch (error) {
    errorMessage.value = getErrorMessage(error)
    statusText.value = 'Ошибка соединения'
    statusKind.value = 'error'
    await stopRecording()
  }
}

async function ensureSpeechSocket() {
  if (socket?.readyState === WebSocket.OPEN) {
    return socket
  }

  if (!socketPromise) {
    statusText.value = 'Подключаемся к бэкенду...'
    socketPromise = openSpeechSocket()
      .then((ws) => {
        if (!isRecording.value) {
          ws.close(1000, 'Recording stopped before speech stream opened')
          return ws
        }

        socket = ws
        hasStartedStreaming = true
        statusText.value = 'Идет запись'
        statusKind.value = 'recording'

        return ws
      })
      .finally(() => {
        socketPromise = null
      })
  }

  return socketPromise
}

function openSpeechSocket(): Promise<WebSocket> {
  return new Promise((resolve, reject) => {
    const ws = new WebSocket(buildSocketUrl())
    ws.binaryType = 'arraybuffer'

    ws.onopen = () => resolve(ws)
    ws.onerror = () => reject(new Error('Не удалось подключиться к WebSocket бэкенда'))
    ws.onmessage = (event) => handleSpeechMessage(event.data)
    ws.onclose = (event) => {
      if (isRecording.value && event.code !== 1000) {
        errorMessage.value = 'Соединение с бэкендом было закрыто'
        statusText.value = 'Ошибка соединения'
        statusKind.value = 'error'
        void stopRecording()
      }
    }
  })
}

function buildSocketUrl() {
  if (config.public.speechWsUrl) {
    return config.public.speechWsUrl
  }

  const apiBase = config.public.apiBase || `${window.location.protocol}//${window.location.host}`
  const url = new URL('/audio/stream', apiBase)
  url.protocol = url.protocol === 'https:' ? 'wss:' : 'ws:'
  url.searchParams.set('sampleRate', String(TARGET_SAMPLE_RATE))

  return url.toString()
}

function handleSpeechMessage(rawMessage: unknown) {
  if (typeof rawMessage !== 'string') {
    return
  }

  const message = JSON.parse(rawMessage) as SpeechMessage

  if (message.type === 'partial') {
    partialTranscript.value = message.text
    return
  }

  if (message.type === 'final' || message.type === 'final_refinement') {
    applyFinalText(message.text, message.type === 'final_refinement')
    partialTranscript.value = ''
    return
  }

  if (message.type === 'status') {
    statusText.value = message.message || statusText.value
    return
  }

  if (message.type === 'error') {
    errorMessage.value = message.message
    statusText.value = 'Ошибка SpeechKit'
    statusKind.value = 'error'
  }
}

function applyFinalText(text: string, shouldReplaceLastSegment: boolean) {
  const cleanText = text.trim()

  if (!cleanText) {
    return
  }

  const lastSegment = finalSegments.value.at(-1)
  if (lastSegment === cleanText) {
    return
  }

  if (shouldReplaceLastSegment && finalSegments.value.length > 0) {
    finalSegments.value = [...finalSegments.value.slice(0, -1), cleanText]
    return
  }

  finalSegments.value = [...finalSegments.value, cleanText]
}

function convertFloat32ToPcm16(
  input: Float32Array,
  sourceSampleRate: number,
  targetSampleRate: number,
) {
  const resampled = resampleLinear(input, sourceSampleRate, targetSampleRate)
  const buffer = new ArrayBuffer(resampled.length * 2)
  const view = new DataView(buffer)

  for (let i = 0; i < resampled.length; i += 1) {
    const sample = Math.max(-1, Math.min(1, resampled[i]))
    view.setInt16(i * 2, sample < 0 ? sample * 0x8000 : sample * 0x7fff, true)
  }

  return buffer
}

function resampleLinear(
  input: Float32Array,
  sourceSampleRate: number,
  targetSampleRate: number,
) {
  if (sourceSampleRate === targetSampleRate) {
    return input
  }

  const ratio = sourceSampleRate / targetSampleRate
  const outputLength = Math.floor(input.length / ratio)
  const output = new Float32Array(outputLength)

  for (let i = 0; i < outputLength; i += 1) {
    const position = i * ratio
    const index = Math.floor(position)
    const fraction = position - index
    const current = input[index] || 0
    const next = input[index + 1] || current

    output[i] = current + (next - current) * fraction
  }

  return output
}

function getRmsVolume(input: Float32Array) {
  let sum = 0

  for (let i = 0; i < input.length; i += 1) {
    sum += input[i] * input[i]
  }

  return Math.sqrt(sum / input.length)
}

function getErrorMessage(error: unknown) {
  if (error instanceof DOMException && error.name === 'NotAllowedError') {
    return 'Браузер не получил доступ к микрофону'
  }

  if (error instanceof Error) {
    return error.message
  }

  return 'Не удалось начать запись'
}

function parseTargetText(text: string): TargetPart[] {
  const parts: TargetPart[] = []
  let lastIndex = 0
  let wordIndex = 0

  for (const match of text.matchAll(WORD_PATTERN)) {
    const textIndex = match.index ?? 0

    if (textIndex > lastIndex) {
      parts.push({
        id: `separator-${lastIndex}`,
        kind: 'separator',
        text: text.slice(lastIndex, textIndex),
      })
    }

    parts.push({
      id: `word-${wordIndex}`,
      kind: 'word',
      text: match[0],
      wordIndex,
    })

    wordIndex += 1
    lastIndex = textIndex + match[0].length
  }

  if (lastIndex < text.length) {
    parts.push({
      id: `separator-${lastIndex}`,
      kind: 'separator',
      text: text.slice(lastIndex),
    })
  }

  return parts
}

function extractWords(text: string) {
  return Array.from(text.matchAll(WORD_PATTERN), (match) => normalizeWord(match[0]))
}

function getRelaxedMatchedIndexes(parts: Extract<TargetPart, { kind: 'word' }>[], words: string[]) {
  const matchedIndexes = new Set<number>()
  let searchFromIndex = 0
  const lookAheadWindow = 5

  for (const word of words) {
    const matchIndex = findNextWordIndex(parts, word, searchFromIndex, lookAheadWindow)

    if (matchIndex === -1) {
      continue
    }

    for (let index = searchFromIndex; index <= matchIndex; index += 1) {
      matchedIndexes.add(index)
    }

    searchFromIndex = matchIndex + 1

    if (searchFromIndex >= parts.length) {
      break
    }
  }

  return matchedIndexes
}

function findNextWordIndex(
  parts: Extract<TargetPart, { kind: 'word' }>[],
  word: string,
  fromIndex: number,
  windowSize: number,
) {
  const toIndex = Math.min(parts.length, fromIndex + windowSize)

  for (let index = fromIndex; index < toIndex; index += 1) {
    if (normalizeWord(parts[index].text) === word) {
      return index
    }
  }

  return -1
}

function normalizeWord(word: string) {
  return word.toLocaleLowerCase('ru-RU').replaceAll('ё', 'е')
}
</script>

<style>
:root {
  color: #172026;
  background: #f5f7f4;
  font-family:
    Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
    sans-serif;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
}

.page {
  min-height: 100vh;
  background:
    linear-gradient(135deg, rgba(93, 190, 112, 0.16), transparent 36%),
    linear-gradient(315deg, rgba(30, 118, 176, 0.12), transparent 42%),
    #f5f7f4;
}

.recorder {
  display: grid;
  min-height: 100vh;
  place-items: center;
  padding: 32px 16px;
}

.panel {
  width: min(920px, 100%);
  padding: clamp(24px, 5vw, 52px);
  border: 1px solid rgba(23, 32, 38, 0.1);
  border-radius: 8px;
  background: rgba(255, 255, 255, 0.9);
  box-shadow: 0 24px 70px rgba(31, 45, 52, 0.12);
}

.heading {
  max-width: 720px;
}

.eyebrow {
  margin: 0 0 12px;
  color: #267f49;
  font-size: 0.78rem;
  font-weight: 800;
  letter-spacing: 0;
  text-transform: uppercase;
}

h1 {
  margin: 0;
  color: #122027;
  font-size: clamp(2rem, 5vw, 4rem);
  line-height: 1.02;
  letter-spacing: 0;
}

.lead {
  margin: 18px 0 0;
  color: #526067;
  font-size: clamp(1rem, 2vw, 1.18rem);
  line-height: 1.65;
}

.reading-card {
  margin-top: 30px;
  border: 1px solid rgba(23, 32, 38, 0.12);
  border-radius: 8px;
  background: #fbfcfb;
  overflow: hidden;
}

.reading-progress {
  display: flex;
  min-height: 52px;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  border-bottom: 1px solid rgba(23, 32, 38, 0.1);
  padding: 0 18px;
  color: #526067;
  font-weight: 800;
}

.reading-progress strong {
  color: #1f8f4d;
}

.target-text {
  margin: 0;
  padding: clamp(18px, 3vw, 26px);
  color: #273238;
  font-size: clamp(1.05rem, 2vw, 1.28rem);
  font-weight: 650;
  line-height: 1.85;
  white-space: pre-wrap;
}

.target-word {
  border-radius: 5px;
  padding: 1px 3px;
  color: #273238;
  transition:
    background-color 140ms ease,
    color 140ms ease;
}

.target-word.spoken {
  background: rgba(31, 143, 77, 0.14);
  color: #168042;
}

.controls {
  display: flex;
  align-items: center;
  gap: 18px;
  margin: 34px 0 26px;
  flex-wrap: wrap;
}

.record-button {
  display: inline-flex;
  min-height: 56px;
  align-items: center;
  justify-content: center;
  gap: 12px;
  border: 0;
  border-radius: 8px;
  padding: 0 24px;
  background: #1f8f4d;
  color: white;
  cursor: pointer;
  font: inherit;
  font-weight: 800;
  transition:
    background 160ms ease,
    transform 160ms ease,
    box-shadow 160ms ease;
  box-shadow: 0 12px 24px rgba(31, 143, 77, 0.24);
}

.record-button:hover:not(:disabled) {
  transform: translateY(-1px);
  background: #167c40;
}

.record-button:disabled {
  cursor: wait;
  opacity: 0.7;
}

.record-button.active {
  background: #c83f3f;
  box-shadow: 0 12px 24px rgba(200, 63, 63, 0.22);
}

.record-dot {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: currentColor;
  box-shadow: 0 0 0 4px rgba(255, 255, 255, 0.22);
}

.status {
  display: inline-flex;
  min-height: 40px;
  align-items: center;
  gap: 10px;
  color: #526067;
  font-weight: 700;
}

.status span {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: #9aa6aa;
}

.status.recording span {
  background: #1f8f4d;
  box-shadow: 0 0 0 6px rgba(31, 143, 77, 0.14);
}

.status.error span {
  background: #c83f3f;
}

.transcript-label {
  display: block;
  margin-bottom: 10px;
  color: #2f3a40;
  font-weight: 800;
}

.transcript {
  width: 100%;
  min-height: 260px;
  resize: vertical;
  border: 1px solid rgba(23, 32, 38, 0.15);
  border-radius: 8px;
  padding: 18px;
  background: #fbfcfb;
  color: #172026;
  font: inherit;
  font-size: 1.06rem;
  line-height: 1.6;
  outline: none;
}

.transcript:focus {
  border-color: #1f8f4d;
  box-shadow: 0 0 0 4px rgba(31, 143, 77, 0.13);
}

.error {
  margin: 14px 0 0;
  color: #a33030;
  font-weight: 700;
}

@media (max-width: 560px) {
  .recorder {
    place-items: stretch;
    padding: 16px;
  }

  .panel {
    padding: 22px;
  }

  .record-button {
    width: 100%;
  }

  .transcript {
    min-height: 220px;
  }
}
</style>
