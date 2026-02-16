<script setup lang="ts">
import { useEventListener } from 'lazy-js-utils'
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'

const tone = {
  'c': [16.35, 32.7, 65.41, 130.81, 261.63, 523.25, 1046.5, 2093.0, 4186.01],
  'c#': [17.32, 34.65, 69.3, 138.59, 277.18, 554.37, 1108.73, 2217.46, 4434.92],
  'd': [18.35, 36.71, 73.42, 146.83, 293.66, 587.33, 1174.66, 2349.32, 4698.64],
  'd#': [19.45, 38.89, 77.78, 155.56, 311.13, 622.25, 1244.51, 2489.02, 4978.03],
  'e': [20.6, 41.2, 82.41, 164.81, 329.63, 659.26, 1318.51, 2637.02],
  'f': [21.83, 43.65, 87.31, 174.61, 349.23, 698.46, 1396.91, 2793.83],
  'f#': [23.12, 46.25, 92.5, 185.0, 369.99, 739.99, 1479.98, 2959.96],
  'g': [24.5, 49.0, 98.0, 196.0, 392.0, 783.99, 1567.98, 3135.96],
  'g#': [25.96, 51.91, 103.83, 207.65, 415.3, 830.61, 1661.22, 3322.44],
  'a': [27.5, 55.0, 110.0, 220.0, 440.0, 880.0, 1760.0, 3520.0],
  'a#': [29.14, 58.27, 116.54, 233.08, 466.16, 932.33, 1864.66, 3729.31],
  'b': [30.87, 61.74, 123.47, 246.94, 493.88, 987.77, 1975.53, 3951.07],
}
const octaves = new Array(2)
const notes = ['c', 'd', 'e', 'f', 'g', 'a', 'b']
const AUTO_PLAY_SOURCE = '__auto__'
const POINTER_SOURCE = '__pointer__'
const customSongStorageKey = 'pinao_custom_songs_v1'
const levelProgressStorageKey = 'pinao_song_levels_v1'

interface SongNote {
  note: string | null
  notes?: string[]
  duration: number
  lyric?: string
}

interface SongItem {
  id: string
  title: string
  artist: string
  notes: SongNote[]
  source: 'builtin' | 'custom'
}

interface PracticeSummary {
  durationLabel: string
  accuracyLabel: string
  rounds: number
  correct: number
  wrong: number
  notesPerMinute: number
  recommendation: string
  testGradeOverview: string
  weakNotesLabel: string
  handFocusLabel: string
  fingeringTip: string
}

interface LevelRule {
  minPassedTests: number
  minAccuracy: number
  minAverageGrade: number
  minA: number
}

interface SongLevelProgress {
  currentLevel: number
  clearedLevel: number
}

interface LevelClearState {
  clearedLevel: number
  unlockedLevel: number | null
  songTitle: string
}

interface NotePerformanceStat {
  correct: number
  wrong: number
}

interface ActiveToneNode {
  noteGain: GainNode
  noteFilter: BiquadFilterNode
  oscillator?: OscillatorNode
  sampleSource?: AudioBufferSourceNode
  key: string
}

interface AutoPlayTonePlanItem {
  note: string
  intensity: number
}

let keyMap: Record<string, string> = {} // Ensure value is string
const activeButtonIdMap: Record<string, ActiveToneNode> = reactive({})
let keys: string[] = []
let audio: AudioContext
let masterGainNode: GainNode
const showPiano = ref(false)
const viewportWidth = ref(1200)
const isAutoPlaying = ref(false)
const autoPlayTimeoutId = ref<ReturnType<typeof setTimeout> | null>(null)
const suspendAudioTimeoutId = ref<ReturnType<typeof setTimeout> | null>(null)
const masterVolume = ref(0.07)
const toneMode = ref<OscillatorType>('sine')
const toneModeOptions: Array<{ value: OscillatorType; label: string }> = [
  { value: 'sine', label: '柔和 (Sine)' },
  { value: 'triangle', label: '清亮 (Triangle)' },
  { value: 'square', label: '电子 (Square)' },
  { value: 'sawtooth', label: '锐利 (Saw)' },
]
const pianoSampleBaseUrls = [
  'https://cdn.jsdelivr.net/npm/tonejs-instrument-piano-mp3@1.1.2',
  'https://fastly.jsdelivr.net/npm/tonejs-instrument-piano-mp3@1.1.2',
  'https://unpkg.com/tonejs-instrument-piano-mp3@1.1.2',
]
const pianoSampleKeys = [
  'A1', 'C2', 'Ds2', 'Fs2', 'A2', 'C3', 'Ds3', 'Fs3',
  'A3', 'C4', 'Ds4', 'Fs4', 'A4', 'C5', 'Ds5', 'Fs5',
  'A5', 'C6', 'Ds6', 'Fs6',
]
const pianoSampleBootstrapKeys = ['C3', 'Fs3', 'A3', 'C4', 'Ds4', 'Fs4', 'A4', 'C5']
const pianoSampleMinReadyCount = 6
const pianoSampleFetchTimeoutMs = 2200
const pianoSampleBuffers: Partial<Record<string, AudioBuffer>> = {}
const pianoSampleLoadState = ref<'idle' | 'loading' | 'ready' | 'error'>('idle')
const pianoSampleLoadedCount = ref(0)
const pianoSampleMirrorInUse = ref('')
let pianoSampleLoadPromise: Promise<void> | null = null
const pianoSampleKeyLoadMap = new Map<string, Promise<boolean>>()
const pianoFallbackWaveCache = new WeakMap<AudioContext, PeriodicWave>()
const speedOptions: Array<{ value: number; label: string }> = [
  { value: 0.6, label: '60%' },
  { value: 0.8, label: '80%' },
  { value: 1, label: '100%' },
]
const timeSignatureOptions: Array<{ value: number; label: string }> = [
  { value: 2, label: '2/4' },
  { value: 3, label: '3/4' },
  { value: 4, label: '4/4' },
]
const countInMeasureOptions: Array<{ value: number; label: string }> = [
  { value: 1, label: '1小节' },
  { value: 2, label: '2小节' },
]
const phraseChunkSizeOptions: Array<{ value: number; label: string }> = [
  { value: 3, label: '3音一组' },
  { value: 4, label: '4音一组' },
  { value: 5, label: '5音一组' },
  { value: 6, label: '6音一组' },
]
const beginnerLesson = [
  'C3', 'D3', 'E3', 'F3', 'G3', 'A3', 'B3', 'C4',
  'B3', 'A3', 'G3', 'F3', 'E3', 'D3', 'C3',
  'C4', 'D4', 'E4', 'F4', 'G4', 'A4', 'B4',
  'A4', 'G4', 'F4', 'E4', 'D4', 'C4',
]
const beginnerMode = ref(true)
const showNoteLabels = ref(true)
const showKeyLabels = ref(true)
const showHandGuide = ref(true)
const handPracticeMode = ref<'both' | 'left' | 'right'>('both')
const practiceSpeed = ref(0.8)
const adaptiveTrainingEnabled = ref(true)
const adaptiveSpeedDelta = ref(0)
const wrongStreak = ref(0)
const correctStreak = ref(0)
const lessonIndex = ref(0)
const lessonCorrect = ref(0)
const lessonWrong = ref(0)
const lessonFeedback = ref<'correct' | 'wrong' | ''>('')
const metronomeEnabled = ref(false)
const metronomeBpm = ref(72)
const beatsPerMeasure = ref(4)
const metronomeBeat = ref(0)
const metronomeDisplayBeat = ref(0)
const metronomeTimerId = ref<ReturnType<typeof setInterval> | null>(null)
const countInEnabled = ref(true)
const countInMeasures = ref(1)
const isCountInActive = ref(false)
const countInCurrentBeat = ref(0)
const countInSessionId = ref(0)
const phraseTypingMode = ref(true)
const phraseTrainerMode = ref<'listen' | 'follow' | 'test'>('follow')
const testLivesMax = 3
const testLives = ref(testLivesMax)
const testMistakesCurrentChunk = ref(0)
const currentChunkTestGrade = ref<'A' | 'B' | 'C' | 'F' | ''>('')
const testGradeStats = reactive({ A: 0, B: 0, C: 0, F: 0 })
const phraseChunkSize = ref(4)
const phraseChunkStart = ref(0)
const phraseChunkInputIndex = ref(0)
const phraseChunkRound = ref(0)
const lessonSourceMode = ref<'basic' | 'song'>('song')
const customSongs = ref<SongItem[]>([])
const selectedSongId = ref('anjing')
const customSongJsonInput = ref('')
const customSongTitleInput = ref('')
const customSongArtistInput = ref('')
const customSongImportStatus = ref('')
const customSongImportStatusType = ref<'success' | 'error' | ''>('')
const customSongFileInputRef = ref<HTMLInputElement | null>(null)
const uiMode = ref<'learn' | 'perform'>('learn')
const practiceLayoutMode = ref<'focus' | 'full'>('focus')
const focusTrainingPhase = ref<'warmup' | 'phrase' | 'test'>('warmup')
const showSongPanelInFocus = ref(false)
const showAdvancedCoachControls = ref(false)
const showSongPanel = ref(true)
const showCoachPanel = ref(true)
const interactionToast = ref('')
const interactionToastType = ref<'success' | 'info' | 'error'>('info')
const interactionToastTimer = ref<ReturnType<typeof setTimeout> | null>(null)
const practiceSessionStartedAt = ref(Date.now())
const showPracticeSummary = ref(false)
const practiceSummary = ref<PracticeSummary | null>(null)
const songLevelProgress = ref<Record<string, SongLevelProgress>>({})
const isLevelChallengeActive = ref(false)
const levelClearState = ref<LevelClearState | null>(null)
const showRetrainSuggestion = ref(false)
const retrainSuggestionText = ref('')
const retrainSuggestionSpeed = ref<number | null>(null)
const notePerformanceMap = reactive<Record<string, NotePerformanceStat>>({})
const handMistakeStats = reactive({ left: 0, right: 0 })

// Song Data with Lyrics
const anJingSong: SongNote[] = [
  // Intro (Instrumental)
  { note: 'G4', duration: 350 }, { note: 'A4', duration: 350 }, { note: 'B4', duration: 350 }, { note: 'C5', duration: 500 }, { note: null, duration: 100 },
  { note: 'B4', duration: 350 }, { note: 'A4', duration: 350 }, { note: 'G4', duration: 700 }, { note: null, duration: 150 },
  { note: 'E4', duration: 350 }, { note: 'F#4', duration: 350 }, { note: 'G4', duration: 700 }, { note: null, duration: 300 },

  // Verse 1
  { note: 'G4', duration: 350, lyric: '只剩下' }, { note: 'A4', duration: 350 }, { note: 'B4', duration: 350, lyric: '钢琴' }, { note: 'C5', duration: 500, lyric: '陪我' }, { note: null, duration: 100 },
  { note: 'B4', duration: 350, lyric: '谈了' }, { note: 'A4', duration: 350, lyric: '一天' }, { note: 'G4', duration: 700 }, { note: null, duration: 150 },
  { note: 'E4', duration: 350, lyric: '睡着' }, { note: 'F#4', duration: 350, lyric: '的大' }, { note: 'G4', duration: 350, lyric: '提琴' }, { note: 'G4', duration: 350 }, { note: 'E4', duration: 700, lyric: '安静' }, { note: null, duration: 300 },

  { note: 'G4', duration: 350, lyric: '旧旧的' }, { note: 'A4', duration: 350 }, { note: 'B4', duration: 350, lyric: '我想' }, { note: 'C5', duration: 500, lyric: '着你' }, { note: null, duration: 100 },
  { note: 'B4', duration: 350, lyric: '的脸' }, { note: 'A4', duration: 350 }, { note: 'G4', duration: 700 }, { note: null, duration: 150 },
  { note: 'E4', duration: 350, lyric: '在我' }, { note: 'F#4', duration: 350, lyric: '的眼' }, { note: 'G4', duration: 700, lyric: '前' }, { note: null, duration: 400 },

  // Pre-Chorus
  { note: 'C5', duration: 350, lyric: '我知' }, { note: 'C5', duration: 350, lyric: '道你' }, { note: 'B4', duration: 350, lyric: '我都' }, { note: 'A4', duration: 500, lyric: '没有' }, { note: null, duration: 100 },
  { note: 'G4', duration: 350, lyric: '错' }, { note: 'A4', duration: 350 }, { note: 'B4', duration: 700, lyric: '只是' }, { note: null, duration: 150 },
  { note: 'A4', duration: 350, lyric: '忘了' }, { note: 'A4', duration: 350, lyric: '怎么' }, { note: 'G4', duration: 350, lyric: '退后' }, { note: 'F#4', duration: 500 }, { note: null, duration: 100 },
  { note: 'E4', duration: 350, lyric: '信誓' }, { note: 'F#4', duration: 350, lyric: '旦旦' }, { note: 'G4', duration: 700, lyric: '给了承诺' }, { note: null, duration: 400 },

  // Chorus
  { note: 'B4', duration: 350, lyric: '我会' }, { note: 'C5', duration: 350, lyric: '学着' }, { note: 'D5', duration: 350, lyric: '放弃' }, { note: 'D5', duration: 500, lyric: '你' }, { note: null, duration: 100 },
  { note: 'C5', duration: 350, lyric: '是因' }, { note: 'B4', duration: 350, lyric: '为我' }, { note: 'A4', duration: 700, lyric: '太爱' }, { note: null, duration: 150 },
  { note: 'G4', duration: 350, lyric: '你' }, { note: 'A4', duration: 350 }, { note: 'B4', duration: 350 }, { note: 'B4', duration: 350 }, { note: 'G4', duration: 700 }, { note: null, duration: 300 },

  { note: 'B4', duration: 350, lyric: '我已' }, { note: 'C5', duration: 350, lyric: '彻底' }, { note: 'D5', duration: 350, lyric: '离开' }, { note: 'D5', duration: 500, lyric: '你' }, { note: null, duration: 100 },
  { note: 'E5', duration: 350, lyric: '爱的' }, { note: 'D5', duration: 350, lyric: '太深' }, { note: 'C5', duration: 700, lyric: '而已' }, { note: null, duration: 150 },
  { note: 'B4', duration: 350, lyric: '我' }, { note: 'C5', duration: 350, lyric: '用力' }, { note: 'B4', duration: 350, lyric: '拉你' }, { note: 'A4', duration: 350, lyric: '手心' }, { note: 'G4', duration: 700 }, { note: null, duration: 400 },
]

const twinkleSong: SongNote[] = [
  { note: 'C4', duration: 400, lyric: '一' }, { note: 'C4', duration: 400, lyric: '闪' }, { note: 'G4', duration: 400, lyric: '一' }, { note: 'G4', duration: 400, lyric: '闪' },
  { note: 'A4', duration: 400, lyric: '亮' }, { note: 'A4', duration: 400 }, { note: 'G4', duration: 700, lyric: '晶晶' },
  { note: 'F4', duration: 400, lyric: '满' }, { note: 'F4', duration: 400, lyric: '天' }, { note: 'E4', duration: 400, lyric: '都' }, { note: 'E4', duration: 400, lyric: '是' },
  { note: 'D4', duration: 400, lyric: '小' }, { note: 'D4', duration: 400 }, { note: 'C4', duration: 700, lyric: '星星' },
]

const joySong: SongNote[] = [
  { note: 'E4', duration: 320, lyric: '欢' }, { note: 'E4', duration: 320, lyric: '乐' }, { note: 'F4', duration: 320 }, { note: 'G4', duration: 320 },
  { note: 'G4', duration: 320, lyric: '女' }, { note: 'F4', duration: 320, lyric: '神' }, { note: 'E4', duration: 320 }, { note: 'D4', duration: 320 },
  { note: 'C4', duration: 320, lyric: '圣' }, { note: 'C4', duration: 320, lyric: '洁' }, { note: 'D4', duration: 320 }, { note: 'E4', duration: 320 },
  { note: 'E4', duration: 480 }, { note: 'D4', duration: 240 }, { note: 'D4', duration: 700 },
]

const happyBirthdaySong: SongNote[] = [
  { note: 'C4', duration: 320 }, { note: 'C4', duration: 220 }, { note: 'D4', duration: 520 }, { note: 'C4', duration: 520 },
  { note: 'F4', duration: 520 }, { note: 'E4', duration: 920 }, { note: null, duration: 180 },
  { note: 'C4', duration: 320 }, { note: 'C4', duration: 220 }, { note: 'D4', duration: 520 }, { note: 'C4', duration: 520 },
  { note: 'G4', duration: 520 }, { note: 'F4', duration: 920 }, { note: null, duration: 180 },
  { note: 'C4', duration: 320 }, { note: 'C4', duration: 220 }, { note: 'C5', duration: 520 }, { note: 'A4', duration: 520 },
  { note: 'F4', duration: 520 }, { note: 'E4', duration: 520 }, { note: 'D4', duration: 900 }, { note: null, duration: 180 },
  { note: 'A#4', duration: 320 }, { note: 'A#4', duration: 220 }, { note: 'A4', duration: 520 }, { note: 'F4', duration: 520 },
  { note: 'G4', duration: 520 }, { note: 'F4', duration: 920 },
]

const furEliseSong: SongNote[] = [
  { note: 'E4', duration: 260 }, { note: 'D#4', duration: 260 }, { note: 'E4', duration: 260 }, { note: 'D#4', duration: 260 },
  { note: 'E4', duration: 260 }, { note: 'B3', duration: 260 }, { note: 'D4', duration: 260 }, { note: 'C4', duration: 260 },
  { note: 'A3', duration: 760 }, { note: null, duration: 200 },
  { note: 'C3', duration: 280 }, { note: 'E3', duration: 280 }, { note: 'A3', duration: 280 }, { note: 'B3', duration: 760 },
  { note: null, duration: 200 }, { note: 'E3', duration: 280 }, { note: 'G#3', duration: 280 }, { note: 'B3', duration: 280 },
  { note: 'C4', duration: 760 }, { note: null, duration: 180 },
]

const canonSong: SongNote[] = [
  { note: 'D4', duration: 360 }, { note: 'A3', duration: 360 }, { note: 'B3', duration: 360 }, { note: 'F#3', duration: 360 },
  { note: 'G3', duration: 360 }, { note: 'D3', duration: 360 }, { note: 'G3', duration: 360 }, { note: 'A3', duration: 560 },
  { note: null, duration: 160 }, { note: 'F#4', duration: 360 }, { note: 'E4', duration: 360 }, { note: 'D4', duration: 360 },
  { note: 'C#4', duration: 360 }, { note: 'B3', duration: 360 }, { note: 'A3', duration: 360 }, { note: 'G3', duration: 560 },
]

const builtInSongs: SongItem[] = [
  { id: 'anjing', title: '安静', artist: '周杰伦', notes: anJingSong, source: 'builtin' },
  { id: 'twinkle', title: '小星星', artist: 'Traditional', notes: twinkleSong, source: 'builtin' },
  { id: 'joy', title: '欢乐颂', artist: 'Beethoven', notes: joySong, source: 'builtin' },
  { id: 'birthday', title: '生日快乐', artist: 'Traditional', notes: happyBirthdaySong, source: 'builtin' },
  { id: 'furelise', title: '致爱丽丝 (片段)', artist: 'Beethoven', notes: furEliseSong, source: 'builtin' },
  { id: 'canon', title: '卡农 (简化)', artist: 'Pachelbel', notes: canonSong, source: 'builtin' },
]

// --- Lyric Display State ---
const currentLyric = ref<string>('')
const noteSemitoneMap: Record<string, number> = {
  C: 0,
  'C#': 1,
  D: 2,
  'D#': 3,
  E: 4,
  F: 5,
  'F#': 6,
  G: 7,
  'G#': 8,
  A: 9,
  'A#': 10,
  B: 11,
}
const midiPitchClassMap = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B']

const createCustomSong = (title: string, artist: string, notes: SongNote[]): SongItem => {
  return {
    id: `custom-${Date.now()}-${Math.floor(Math.random() * 1000)}`,
    title,
    artist,
    notes,
    source: 'custom',
  }
}

const allSongs = computed(() => [...builtInSongs, ...customSongs.value])
const selectedSong = computed(() => {
  const song = allSongs.value.find(item => item.id === selectedSongId.value)
  return song || builtInSongs[0]
})

const levelRules: LevelRule[] = [
  { minPassedTests: 1, minAccuracy: 50, minAverageGrade: 1, minA: 0 },
  { minPassedTests: 3, minAccuracy: 65, minAverageGrade: 1.3, minA: 0 },
  { minPassedTests: 5, minAccuracy: 75, minAverageGrade: 1.6, minA: 0 },
  { minPassedTests: 7, minAccuracy: 82, minAverageGrade: 2, minA: 1 },
  { minPassedTests: 10, minAccuracy: 88, minAverageGrade: 2.3, minA: 2 },
]

const getSongLevelProgress = (songId = selectedSongId.value): SongLevelProgress => {
  if (!songLevelProgress.value[songId]) {
    songLevelProgress.value[songId] = {
      currentLevel: 1,
      clearedLevel: 0,
    }
  }
  return songLevelProgress.value[songId]
}

const toPositiveModulo = (value: number, divisor: number) => {
  if (!divisor)
    return 0
  return ((value % divisor) + divisor) % divisor
}

const getLevelSegment = (totalNotes: number, level: number) => {
  if (!totalNotes)
    return { start: 0, end: 0, length: 0 }
  const safeLevel = Math.max(1, Math.min(levelRules.length, Math.round(level)))
  const levelIndex = safeLevel - 1
  const start = Math.floor((levelIndex * totalNotes) / levelRules.length)
  const rawEnd = Math.floor(((levelIndex + 1) * totalNotes) / levelRules.length)
  if (rawEnd <= start) {
    const fixedStart = Math.max(0, Math.min(totalNotes - 1, start))
    return {
      start: fixedStart,
      end: fixedStart + 1,
      length: 1,
    }
  }
  return {
    start,
    end: rawEnd,
    length: rawEnd - start,
  }
}

const normalizeNoteInput = (value: unknown) => {
  if (value === null)
    return null
  if (typeof value !== 'string')
    return null
  const raw = value.trim().toUpperCase()
  if (!raw || raw === 'R' || raw === 'REST' || raw === '-')
    return null
  const match = raw.match(/^([A-G])(#?)(\d)$/)
  if (!match)
    return null
  return `${match[1]}${match[2]}${match[3]}`
}

const normalizeNoteListInput = (value: unknown) => {
  if (!Array.isArray(value))
    return []
  const normalized = value
    .map(item => normalizeNoteInput(item))
    .filter((item): item is string => !!item)
  return Array.from(new Set(normalized))
}

const pickPrimaryNote = (noteIds: string[]) => {
  if (!noteIds.length)
    return null
  return [...noteIds].sort((a, b) => (noteToMidi(b) || 0) - (noteToMidi(a) || 0))[0]
}

const getSongNotePlayableNotes = (songNote: SongNote | null | undefined) => {
  if (!songNote)
    return []
  const candidate = Array.isArray(songNote.notes) && songNote.notes.length
    ? songNote.notes
    : songNote.note
      ? [songNote.note]
      : []
  return Array.from(new Set(candidate.filter(Boolean)))
}

const normalizeSongNote = (raw: any): SongNote | null => {
  if (!raw || typeof raw !== 'object')
    return null
  const note = normalizeNoteInput(raw.note)
  const chordNotes = normalizeNoteListInput(raw.notes)
  const rawNote = typeof raw.note === 'string' ? raw.note.trim().toUpperCase() : raw.note
  if (rawNote !== null && rawNote !== undefined && rawNote !== '' && rawNote !== 'R' && rawNote !== 'REST' && rawNote !== '-' && note === null && !chordNotes.length)
    return null
  const duration = Number(raw.duration)
  if (!Number.isFinite(duration))
    return null
  const safeDuration = Math.round(Math.max(80, Math.min(4000, duration)))
  const lyric = typeof raw.lyric === 'string' ? raw.lyric.slice(0, 20) : undefined
  const resolvedPrimary = note || pickPrimaryNote(chordNotes)
  const mergedChord = Array.from(new Set([...(resolvedPrimary ? [resolvedPrimary] : []), ...chordNotes]))
    .sort((a, b) => (noteToMidi(a) || 0) - (noteToMidi(b) || 0))
  return {
    note: resolvedPrimary,
    notes: mergedChord.length > 1 ? mergedChord : undefined,
    duration: safeDuration,
    lyric,
  }
}

const buildSongFromUnknown = (payload: any, fallbackTitle = '自定义练习曲'): SongItem | null => {
  let title = fallbackTitle
  let artist = 'Unknown'
  let notesRaw: any[] | null = null

  if (Array.isArray(payload)) {
    notesRaw = payload
  }
  else if (payload && typeof payload === 'object' && Array.isArray(payload.notes)) {
    notesRaw = payload.notes
    if (typeof payload.title === 'string' && payload.title.trim())
      title = payload.title.trim()
    if (typeof payload.artist === 'string' && payload.artist.trim())
      artist = payload.artist.trim()
  }

  if (!notesRaw || !notesRaw.length)
    return null

  const notes = notesRaw.map(normalizeSongNote).filter((note): note is SongNote => !!note)
  if (!notes.length)
    return null

  return createCustomSong(title, artist, notes)
}

const buildSongFromSimpleText = (rawText: string, fallbackTitle = '文本练习曲'): SongItem | null => {
  const segments = rawText
    .split(/\n|,/)
    .map(item => item.trim())
    .filter(Boolean)

  if (!segments.length)
    return null

  const notes: SongNote[] = []
  for (const segment of segments) {
    const match = segment.match(/^([A-Ga-g](?:#)?\d|R|REST|-)\s*(?:(?:[:\s])\s*(\d+))?(?:\s*(?:[:|])\s*(.+))?$/)
    if (!match)
      continue
    const note = normalizeNoteInput(match[1])
    const duration = Number(match[2] || 350)
    const safeDuration = Math.round(Math.max(80, Math.min(4000, duration)))
    const lyric = match[3]?.trim().slice(0, 20)
    notes.push({ note, duration: safeDuration, lyric })
  }

  if (!notes.length)
    return null

  return createCustomSong(fallbackTitle, 'Text Import', notes)
}

const readVarLength = (data: Uint8Array, startIndex: number) => {
  let value = 0
  let index = startIndex
  for (let i = 0; i < 4; i++) {
    const byte = data[index]
    if (byte === undefined)
      return null
    value = (value << 7) | (byte & 0x7F)
    index += 1
    if ((byte & 0x80) === 0)
      return { value, nextIndex: index }
  }
  return null
}

const midiToNote = (midi: number) => {
  if (!Number.isInteger(midi) || midi < 0 || midi > 127)
    return null
  const pitchClass = midiPitchClassMap[midi % 12]
  const octave = Math.floor(midi / 12) - 1
  return `${pitchClass}${octave}`
}

const parseMidiToSong = (buffer: ArrayBuffer, fallbackTitle = 'MIDI 练习曲'): SongItem | null => {
  const data = new Uint8Array(buffer)
  if (data.length < 14)
    return null

  const readU16 = (index: number) => ((data[index] << 8) | data[index + 1]) >>> 0
  const readU32 = (index: number) => (
    ((data[index] << 24) >>> 0)
    + ((data[index + 1] << 16) >>> 0)
    + ((data[index + 2] << 8) >>> 0)
    + (data[index + 3] >>> 0)
  ) >>> 0

  const headerId = String.fromCharCode(data[0], data[1], data[2], data[3])
  if (headerId !== 'MThd')
    return null

  const headerLength = readU32(4)
  const trackCount = readU16(10)
  const division = readU16(12)
  if ((division & 0x8000) !== 0)
    return null // SMPTE format is not handled in this learner parser.
  const ticksPerQuarter = division
  if (!ticksPerQuarter)
    return null

  type TempoEvent = { tick: number; microsecondsPerQuarter: number }
  type MidiNoteEvent = { startTick: number; endTick: number; midi: number; velocity: number; trackIndex: number }

  const tempoEvents: TempoEvent[] = [{ tick: 0, microsecondsPerQuarter: 500000 }]
  const noteEvents: MidiNoteEvent[] = []
  let offset = 8 + headerLength

  for (let trackIndex = 0; trackIndex < trackCount; trackIndex++) {
    if (offset + 8 > data.length)
      break
    const chunkId = String.fromCharCode(data[offset], data[offset + 1], data[offset + 2], data[offset + 3])
    const chunkLength = readU32(offset + 4)
    const trackStart = offset + 8
    const trackEnd = trackStart + chunkLength
    offset = trackEnd
    if (chunkId !== 'MTrk' || trackEnd > data.length)
      continue

    let index = trackStart
    let tick = 0
    let runningStatus = 0
    const openNotes = new Map<number, { tick: number; velocity: number }>()

    while (index < trackEnd) {
      const delta = readVarLength(data, index)
      if (!delta)
        break
      tick += delta.value
      index = delta.nextIndex
      if (index >= trackEnd)
        break

      let status = data[index]
      if (status < 0x80) {
        status = runningStatus
      }
      else {
        index += 1
        runningStatus = status
      }

      if (status === 0xFF) {
        const metaType = data[index]
        index += 1
        const metaLength = readVarLength(data, index)
        if (!metaLength)
          break
        index = metaLength.nextIndex
        const payloadEnd = index + metaLength.value
        if (payloadEnd > trackEnd)
          break
        if (metaType === 0x51 && metaLength.value === 3) {
          const tempo = (data[index] << 16) | (data[index + 1] << 8) | data[index + 2]
          if (tempo > 0)
            tempoEvents.push({ tick, microsecondsPerQuarter: tempo })
        }
        index = payloadEnd
        continue
      }

      if (status === 0xF0 || status === 0xF7) {
        const sysexLength = readVarLength(data, index)
        if (!sysexLength)
          break
        index = sysexLength.nextIndex + sysexLength.value
        continue
      }

      const eventType = status & 0xF0
      const isOneDataByteEvent = eventType === 0xC0 || eventType === 0xD0
      if (isOneDataByteEvent) {
        index += 1
        continue
      }

      const param1 = data[index]
      const param2 = data[index + 1]
      index += 2
      if (param1 === undefined || param2 === undefined)
        break

      if (eventType === 0x90 && param2 > 0) {
        openNotes.set(param1, { tick, velocity: param2 })
      }
      else if (eventType === 0x80 || (eventType === 0x90 && param2 === 0)) {
        const opened = openNotes.get(param1)
        if (!opened)
          continue
        const endTick = Math.max(tick, opened.tick + 1)
        noteEvents.push({
          startTick: opened.tick,
          endTick,
          midi: param1,
          velocity: opened.velocity,
          trackIndex,
        })
        openNotes.delete(param1)
      }
    }
  }

  if (!noteEvents.length)
    return null

  const dedupedTempo = Array.from(
    tempoEvents
      .sort((a, b) => a.tick - b.tick)
      .reduce((map, item) => {
        map.set(item.tick, item.microsecondsPerQuarter)
        return map
      }, new Map<number, number>())
      .entries(),
  )
    .map(([tick, microsecondsPerQuarter]) => ({ tick, microsecondsPerQuarter }))
    .sort((a, b) => a.tick - b.tick)

  if (!dedupedTempo.length || dedupedTempo[0].tick !== 0)
    dedupedTempo.unshift({ tick: 0, microsecondsPerQuarter: 500000 })

  const tempoPoints: Array<{ tick: number; microsecondsPerQuarter: number; msAtTick: number }> = []
  let lastTick = 0
  let lastMs = 0
  let currentTempo = dedupedTempo[0].microsecondsPerQuarter
  tempoPoints.push({ tick: 0, microsecondsPerQuarter: currentTempo, msAtTick: 0 })

  for (let i = 1; i < dedupedTempo.length; i++) {
    const current = dedupedTempo[i]
    const deltaTicks = current.tick - lastTick
    if (deltaTicks > 0)
      lastMs += (deltaTicks * currentTempo) / ticksPerQuarter / 1000
    currentTempo = current.microsecondsPerQuarter
    lastTick = current.tick
    tempoPoints.push({ tick: current.tick, microsecondsPerQuarter: currentTempo, msAtTick: lastMs })
  }

  const tickToMs = (targetTick: number) => {
    let point = tempoPoints[0]
    for (let i = 1; i < tempoPoints.length; i++) {
      if (tempoPoints[i].tick > targetTick)
        break
      point = tempoPoints[i]
    }
    const deltaTicks = targetTick - point.tick
    return point.msAtTick + (deltaTicks * point.microsecondsPerQuarter) / ticksPerQuarter / 1000
  }

  const groupedByStart = noteEvents
    .sort((a, b) => a.startTick - b.startTick || b.velocity - a.velocity || b.midi - a.midi)
    .reduce((map, event) => {
      const bucket = map.get(event.startTick) || []
      bucket.push(event)
      map.set(event.startTick, bucket)
      return map
    }, new Map<number, MidiNoteEvent[]>())

  const notes: SongNote[] = []
  let timelineMs = 0

  for (const [startTick, eventsAtTick] of Array.from(groupedByStart.entries()).sort((a, b) => a[0] - b[0])) {
    const prioritized = [...eventsAtTick]
      .sort((a, b) => b.velocity - a.velocity || b.midi - a.midi || a.trackIndex - b.trackIndex)
      .slice(0, 4)
    if (!prioritized.length)
      continue

    const primaryEvent = prioritized[0]
    const primaryNote = midiToNote(primaryEvent.midi)
    const chordNotes = Array.from(
      new Set(
        prioritized
          .map(event => midiToNote(event.midi))
          .filter((noteId): noteId is string => !!noteId),
      ),
    ).sort((a, b) => (noteToMidi(a) || 0) - (noteToMidi(b) || 0))
    const resolvedPrimary = primaryNote || pickPrimaryNote(chordNotes)
    if (!resolvedPrimary)
      continue

    const startMs = tickToMs(startTick)
    const endTick = prioritized.reduce((maxTick, event) => Math.max(maxTick, event.endTick), startTick + 1)
    const endMs = tickToMs(endTick)
    const gap = Math.round(startMs - timelineMs)
    if (gap >= 90)
      notes.push({ note: null, duration: Math.min(4000, gap) })

    const durationMs = Math.round(Math.max(100, Math.min(3000, endMs - startMs)))
    notes.push({
      note: resolvedPrimary,
      notes: chordNotes.length > 1 ? chordNotes : undefined,
      duration: durationMs,
    })
    timelineMs = Math.max(timelineMs, startMs + durationMs)
  }

  if (!notes.length)
    return null

  return createCustomSong(fallbackTitle, 'MIDI Import', notes)
}

const saveCustomSongs = () => {
  try {
    localStorage.setItem(customSongStorageKey, JSON.stringify(customSongs.value))
  }
  catch (error) {
    console.warn('Failed to save custom songs:', error)
  }
}

const loadCustomSongs = () => {
  try {
    const raw = localStorage.getItem(customSongStorageKey)
    if (!raw)
      return
    const parsed = JSON.parse(raw)
    if (!Array.isArray(parsed))
      return
    const hydrated = parsed
      .map((item: any) => buildSongFromUnknown(item, item?.title || '自定义练习曲'))
      .filter((item): item is SongItem => !!item)
      .map((item, index) => ({ ...item, id: `custom-loaded-${index}-${item.id}` }))
    customSongs.value = hydrated
  }
  catch (error) {
    console.warn('Failed to load custom songs:', error)
  }
}

const saveSongLevelProgress = () => {
  try {
    localStorage.setItem(levelProgressStorageKey, JSON.stringify(songLevelProgress.value))
  }
  catch (error) {
    console.warn('Failed to save song level progress:', error)
  }
}

const loadSongLevelProgress = () => {
  try {
    const raw = localStorage.getItem(levelProgressStorageKey)
    if (!raw)
      return
    const parsed = JSON.parse(raw)
    if (!parsed || typeof parsed !== 'object')
      return
    const hydrated: Record<string, SongLevelProgress> = {}
    Object.entries(parsed).forEach(([songId, value]) => {
      const item = value as any
      const currentLevel = Number(item?.currentLevel)
      const clearedLevel = Number(item?.clearedLevel)
      if (!Number.isFinite(currentLevel) || !Number.isFinite(clearedLevel))
        return
      hydrated[songId] = {
        currentLevel: Math.max(1, Math.min(levelRules.length, Math.round(currentLevel))),
        clearedLevel: Math.max(0, Math.min(levelRules.length, Math.round(clearedLevel))),
      }
    })
    songLevelProgress.value = hydrated
  }
  catch (error) {
    console.warn('Failed to load song level progress:', error)
  }
}

const applyImportMeta = (song: SongItem) => {
  const customTitle = customSongTitleInput.value.trim()
  const customArtist = customSongArtistInput.value.trim()
  if (customTitle)
    song.title = customTitle.slice(0, 40)
  if (customArtist)
    song.artist = customArtist.slice(0, 40)
  return song
}

const exportSelectedSongJson = async () => {
  customSongImportStatus.value = ''
  customSongImportStatusType.value = ''
  const payload = {
    title: selectedSong.value.title,
    artist: selectedSong.value.artist,
    notes: selectedSong.value.notes,
  }
  const json = JSON.stringify(payload, null, 2)

  try {
    if (navigator.clipboard?.writeText) {
      await navigator.clipboard.writeText(json)
      customSongImportStatus.value = `已复制《${selectedSong.value.title}》JSON 到剪贴板`
      customSongImportStatusType.value = 'success'
      pushInteractionToast('当前曲谱 JSON 已复制', 'success')
      return
    }
  }
  catch {
    // Fallback to download below.
  }

  try {
    const safeName = selectedSong.value.title.replace(/[^\w-]+/g, '_').slice(0, 40) || 'song'
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const anchor = document.createElement('a')
    anchor.href = url
    anchor.download = `${safeName}.json`
    document.body.appendChild(anchor)
    anchor.click()
    anchor.remove()
    URL.revokeObjectURL(url)
    customSongImportStatus.value = `已下载《${selectedSong.value.title}》曲谱 JSON`
    customSongImportStatusType.value = 'success'
    pushInteractionToast('当前曲谱 JSON 已下载', 'success')
  }
  catch {
    customSongImportStatus.value = '导出失败：浏览器不支持复制或下载'
    customSongImportStatusType.value = 'error'
    pushInteractionToast('导出失败，请检查浏览器权限', 'error')
  }
}

const copySongTemplate = async () => {
  customSongImportStatus.value = ''
  customSongImportStatusType.value = ''
  const template = JSON.stringify({
    title: '我的练习曲',
    artist: 'Me',
    notes: [
      { note: 'C4', duration: 350 },
      { note: 'D4', duration: 350 },
      { note: 'E4', duration: 700, lyric: 'Hi' },
      { note: null, duration: 150 },
    ],
  }, null, 2)
  try {
    await navigator.clipboard.writeText(template)
    customSongImportStatus.value = '模板已复制，可直接粘贴修改后导入'
    customSongImportStatusType.value = 'success'
    pushInteractionToast('导入模板已复制', 'success')
  }
  catch {
    customSongImportStatus.value = '复制模板失败：请检查浏览器剪贴板权限'
    customSongImportStatusType.value = 'error'
    pushInteractionToast('复制模板失败，请检查剪贴板权限', 'error')
  }
}

const importSongFromJsonText = () => {
  customSongImportStatus.value = ''
  customSongImportStatusType.value = ''
  if (!customSongJsonInput.value.trim()) {
    customSongImportStatus.value = '请先粘贴 JSON 内容'
    customSongImportStatusType.value = 'error'
    return
  }

  try {
    let song: SongItem | null = null
    try {
      const parsed = JSON.parse(customSongJsonInput.value)
      song = buildSongFromUnknown(parsed)
    }
    catch {
      song = buildSongFromSimpleText(customSongJsonInput.value)
    }
    if (!song)
      throw new Error('invalid song schema')
    song = applyImportMeta(song)
    customSongs.value.unshift(song)
    selectedSongId.value = song.id
    customSongImportStatus.value = `已导入：${song.title}（${song.notes.length} 音）`
    customSongImportStatusType.value = 'success'
    pushInteractionToast(`导入成功：${song.title}`, 'success')
    customSongJsonInput.value = ''
    saveCustomSongs()
  }
  catch (error) {
    customSongImportStatus.value = '导入失败：请检查 JSON 或简写文本格式'
    customSongImportStatusType.value = 'error'
    pushInteractionToast('导入失败，请检查文本格式', 'error')
  }
}

const openCustomSongFilePicker = () => {
  customSongFileInputRef.value?.click()
}

const importSongFromFile = async (event: Event) => {
  const input = event.target as HTMLInputElement
  const file = input.files?.[0]
  if (!file)
    return

  customSongImportStatus.value = ''
  customSongImportStatusType.value = ''
  try {
    let song: SongItem | null = null
    const filename = file.name.replace(/\.(json|txt|mid|midi)$/i, '')
    if (/\.mid(i)?$/i.test(file.name)) {
      const buffer = await file.arrayBuffer()
      song = parseMidiToSong(buffer, filename || 'MIDI 练习曲')
    }
    else {
      const text = await file.text()
      try {
        const parsed = JSON.parse(text)
        song = buildSongFromUnknown(parsed, filename || '文件练习曲')
      }
      catch {
        song = buildSongFromSimpleText(text, filename || '文本练习曲')
      }
    }
    if (!song)
      throw new Error('invalid file schema')
    song = applyImportMeta(song)
    customSongs.value.unshift(song)
    selectedSongId.value = song.id
    customSongImportStatus.value = `已导入文件：${song.title}（${song.notes.length} 音）`
    customSongImportStatusType.value = 'success'
    pushInteractionToast(`文件导入成功：${song.title}`, 'success')
    saveCustomSongs()
  }
  catch (error) {
    customSongImportStatus.value = '文件导入失败：请检查 JSON/TXT/MIDI 内容'
    customSongImportStatusType.value = 'error'
    pushInteractionToast('文件导入失败，请检查格式', 'error')
  }
  finally {
    input.value = ''
  }
}

const removeSelectedCustomSong = () => {
  const target = selectedSong.value
  if (!target || target.source !== 'custom')
    return
  customSongs.value = customSongs.value.filter(item => item.id !== target.id)
  selectedSongId.value = builtInSongs[0].id
  saveCustomSongs()
  pushInteractionToast(`已删除：${target.title}`, 'info')
}

const playCurrentPhraseChunk = async (silentOrEvent: boolean | Event = false) => {
  const silent = typeof silentOrEvent === 'boolean' ? silentOrEvent : false
  if (!phraseChunkNotes.value.length)
    return
  const phraseSong = phraseChunkNotes.value.map(note => ({ note, duration: 320 }))
  interruptAutoPlay()
  initAudio()
  if (!audio)
    return
  const canStart = await runCountIn()
  if (!canStart)
    return
  playSong(phraseSong)
  if (!silent)
    pushInteractionToast('正在播放当前短句', 'info')
}

function getKeyByNote(noteId: string | null) {
  if (!noteId)
    return ''
  const key = Object.keys(keyMap).find(k => keyMap[k] === noteId)
  return key ? key.toUpperCase() : ''
}

const noteToMidi = (noteId: string | null) => {
  if (!noteId)
    return null
  const match = noteId.match(/^([A-G])(#?)(\d)$/)
  if (!match)
    return null
  const pitchClass = `${match[1]}${match[2]}` as keyof typeof noteSemitoneMap
  const octave = Number(match[3])
  const semitone = noteSemitoneMap[pitchClass]
  if (semitone === undefined)
    return null
  return (octave + 1) * 12 + semitone
}

const pianoSampleKeyToNoteId = (sampleKey: string) => {
  const match = sampleKey.match(/^([A-G])(s?)(\d)$/)
  if (!match)
    return null
  return `${match[1]}${match[2] ? '#' : ''}${match[3]}`
}

const findNearestPianoSample = (noteId: string, onlyLoaded = false) => {
  const targetMidi = noteToMidi(noteId)
  if (targetMidi === null)
    return null
  let nearestKey: string | null = null
  let nearestMidi: number | null = null
  let nearestDiff = Number.POSITIVE_INFINITY
  for (const sampleKey of pianoSampleKeys) {
    if (onlyLoaded && !pianoSampleBuffers[sampleKey])
      continue
    const sampleNoteId = pianoSampleKeyToNoteId(sampleKey)
    const sampleMidi = noteToMidi(sampleNoteId)
    if (sampleMidi === null)
      continue
    const diff = Math.abs(targetMidi - sampleMidi)
    if (diff < nearestDiff) {
      nearestDiff = diff
      nearestKey = sampleKey
      nearestMidi = sampleMidi
    }
  }
  if (!nearestKey || nearestMidi === null)
    return null
  return { targetMidi, sampleKey: nearestKey, sampleMidi: nearestMidi }
}

const syncPianoSampleState = (markError = false) => {
  pianoSampleLoadedCount.value = Object.keys(pianoSampleBuffers).length
  if (pianoSampleLoadedCount.value >= pianoSampleMinReadyCount) {
    pianoSampleLoadState.value = 'ready'
    return
  }
  if (markError)
    pianoSampleLoadState.value = 'error'
}

const fetchArrayBufferWithTimeout = async (url: string, timeoutMs = pianoSampleFetchTimeoutMs) => {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs)
  try {
    const response = await fetch(url, { cache: 'force-cache', signal: controller.signal })
    if (!response.ok)
      throw new Error(`HTTP ${response.status}`)
    return await response.arrayBuffer()
  }
  finally {
    clearTimeout(timeoutId)
  }
}

const fetchPianoSampleFromMirrors = async (sampleKey: string): Promise<{ arrayBuffer: ArrayBuffer; baseUrl: string } | null> => {
  for (const baseUrl of pianoSampleBaseUrls) {
    const url = `${baseUrl}/${sampleKey}.mp3`
    try {
      const arrayBuffer = await fetchArrayBufferWithTimeout(url)
      return { arrayBuffer, baseUrl }
    }
    catch (error) {
      console.warn(`Sample fetch failed for ${sampleKey} from ${baseUrl}:`, error)
    }
  }
  return null
}

const loadSinglePianoSample = async (sampleKey: string): Promise<boolean> => {
  if (pianoSampleBuffers[sampleKey])
    return true
  const pending = pianoSampleKeyLoadMap.get(sampleKey)
  if (pending)
    return pending

  const task = (async () => {
    initAudio()
    if (!audio)
      return false
    const result = await fetchPianoSampleFromMirrors(sampleKey)
    if (!result)
      return false
    try {
      const audioBuffer = await audio.decodeAudioData(result.arrayBuffer.slice(0))
      pianoSampleBuffers[sampleKey] = audioBuffer
      if (!pianoSampleMirrorInUse.value && result.baseUrl)
        pianoSampleMirrorInUse.value = result.baseUrl
      syncPianoSampleState(false)
      return true
    }
    catch (error) {
      console.warn(`Failed to decode sample ${sampleKey}:`, error)
      return false
    }
  })().finally(() => {
    pianoSampleKeyLoadMap.delete(sampleKey)
  })

  pianoSampleKeyLoadMap.set(sampleKey, task)
  return task
}

const loadPianoSamples = async () => {
  if (pianoSampleLoadState.value === 'ready')
    return
  if (pianoSampleLoadPromise)
    return pianoSampleLoadPromise

  pianoSampleLoadState.value = 'loading'
  pianoSampleLoadPromise = (async () => {
    const bootstrapKeys = Array.from(new Set(pianoSampleBootstrapKeys.filter(key => pianoSampleKeys.includes(key))))
    await Promise.allSettled(bootstrapKeys.map(sampleKey => loadSinglePianoSample(sampleKey)))
    syncPianoSampleState(false)

    const remainingKeys = pianoSampleKeys.filter(sampleKey => !bootstrapKeys.includes(sampleKey))
    if (remainingKeys.length > 0) {
      void Promise.allSettled(remainingKeys.map(sampleKey => loadSinglePianoSample(sampleKey)))
        .finally(() => {
          syncPianoSampleState(pianoSampleLoadedCount.value < pianoSampleMinReadyCount)
        })
    }

    if (pianoSampleLoadedCount.value < pianoSampleMinReadyCount)
      pianoSampleLoadState.value = 'loading'
  })()
    .catch((error) => {
      console.warn('Failed to load piano samples:', error)
      syncPianoSampleState(true)
    })
    .finally(() => {
      pianoSampleLoadPromise = null
    })
  return pianoSampleLoadPromise
}

const warmupPianoSamples = async () => {
  if (pianoSampleLoadState.value === 'ready')
    return
  await Promise.race([
    loadPianoSamples(),
    new Promise(resolve => setTimeout(resolve, 3200)),
  ])
}

const anJingChordProgression = [
  { bass: 'G2', tones: ['D3', 'B3'] },
  { bass: 'D2', tones: ['A2', 'F#3'] },
  { bass: 'E2', tones: ['B2', 'G3'] },
  { bass: 'B1', tones: ['F#2', 'D3'] },
  { bass: 'C2', tones: ['G2', 'E3'] },
  { bass: 'G1', tones: ['D2', 'B2'] },
  { bass: 'A1', tones: ['E2', 'C3'] },
  { bass: 'D2', tones: ['A2', 'F#3'] },
]

const buildChordTonePlan = (primaryNote: string, noteIds: string[]): AutoPlayTonePlanItem[] => {
  const uniqueNotes = Array.from(new Set(noteIds)).sort((a, b) => (noteToMidi(b) || 0) - (noteToMidi(a) || 0))
  const plan: AutoPlayTonePlanItem[] = [{ note: primaryNote, intensity: 1 }]
  const supportIntensities = [0.62, 0.5, 0.42, 0.36]
  let supportIndex = 0
  uniqueNotes.forEach((noteId) => {
    if (noteId === primaryNote || supportIndex >= supportIntensities.length)
      return
    plan.push({ note: noteId, intensity: supportIntensities[supportIndex] })
    supportIndex += 1
  })
  return plan
}

const buildAnJingAccompaniment = (melodyNote: string, duration: number, melodyStep: number): AutoPlayTonePlanItem[] => {
  const chord = anJingChordProgression[Math.floor(melodyStep / 4) % anJingChordProgression.length]
  const plan: AutoPlayTonePlanItem[] = [{ note: melodyNote, intensity: 1 }]
  plan.push({ note: chord.bass, intensity: 0.36 })
  if (duration >= 420) {
    plan.push({ note: chord.tones[0], intensity: 0.3 })
    plan.push({ note: chord.tones[1], intensity: 0.24 })
  }
  else {
    const toneIndex = melodyStep % 2
    plan.push({ note: chord.tones[toneIndex], intensity: 0.28 })
  }
  return plan
}

const buildAutoPlayTonePlan = (song: SongNote[], index: number, melodyStep: number): AutoPlayTonePlanItem[] => {
  const current = song[index]
  const playableNotes = getSongNotePlayableNotes(current)
  if (!playableNotes.length)
    return []
  const primaryNote = current.note || pickPrimaryNote(playableNotes)
  if (!primaryNote)
    return []
  const isSelectedSongPlayback = song === selectedSong.value.notes
  if (playableNotes.length > 1) {
    if (isSelectedSongPlayback)
      return buildChordTonePlan(primaryNote, playableNotes)
    return [{ note: primaryNote, intensity: 1 }]
  }
  if (isSelectedSongPlayback && selectedSong.value.id === 'anjing')
    return buildAnJingAccompaniment(primaryNote, current.duration, melodyStep)
  return [{ note: primaryNote, intensity: 1 }]
}

const getHandForNote = (noteId: string | null): 'left' | 'right' => {
  const midi = noteToMidi(noteId)
  if (midi === null)
    return 'left'
  return midi >= 60 ? 'right' : 'left'
}

const getOrCreateNotePerformanceStat = (noteId: string) => {
  if (!notePerformanceMap[noteId])
    notePerformanceMap[noteId] = { correct: 0, wrong: 0 }
  return notePerformanceMap[noteId]
}

const recordExpectedNoteAttempt = (noteId: string | null, isCorrect: boolean) => {
  if (!noteId)
    return
  const stat = getOrCreateNotePerformanceStat(noteId)
  if (isCorrect)
    stat.correct += 1
  else
    stat.wrong += 1

  if (!isCorrect) {
    if (getHandForNote(noteId) === 'left')
      handMistakeStats.left += 1
    else
      handMistakeStats.right += 1
  }
}

const isNoteInPracticeRange = (noteId: string | null) => {
  const midi = noteToMidi(noteId)
  return midi !== null && midi >= 48 && midi <= 71
}

const selectedSongPracticeNotes = computed(() => {
  return selectedSong.value.notes
    .map(item => item.note)
    .filter((note): note is string => !!note && isNoteInPracticeRange(note))
})
const selectedSongRangeLabel = computed(() => {
  const noteIds = selectedSong.value.notes
    .map(item => item.note)
    .filter((note): note is string => !!note)
  if (!noteIds.length)
    return '--'
  const midiValues = noteIds
    .map(note => noteToMidi(note))
    .filter((item): item is number => item !== null)
  if (!midiValues.length)
    return '--'
  const minMidi = Math.min(...midiValues)
  const maxMidi = Math.max(...midiValues)
  return `${midiToNote(minMidi) || '?'} ~ ${midiToNote(maxMidi) || '?'}`
})
const selectedSongDurationLabel = computed(() => {
  const totalMs = selectedSong.value.notes.reduce((sum, item) => sum + item.duration, 0)
  if (!totalMs)
    return '--'
  const totalSeconds = Math.round(totalMs / 1000)
  const min = Math.floor(totalSeconds / 60)
  const sec = String(totalSeconds % 60).padStart(2, '0')
  return `${min}:${sec}`
})
const currentSongLevelProgress = computed(() => getSongLevelProgress())
const currentSongLevel = computed(() => currentSongLevelProgress.value.currentLevel)
const currentSongClearedLevel = computed(() => currentSongLevelProgress.value.clearedLevel)
const currentLevelRule = computed(() => levelRules[Math.min(levelRules.length - 1, Math.max(0, currentSongLevel.value - 1))])

const lessonNotes = computed(() => {
  const baseNotes = lessonSourceMode.value === 'song' && selectedSongPracticeNotes.value.length
    ? selectedSongPracticeNotes.value
    : beginnerLesson
  if (handPracticeMode.value === 'both')
    return baseNotes
  return baseNotes.filter(noteId => getHandForNote(noteId) === handPracticeMode.value)
})

const currentLevelSegment = computed(() => getLevelSegment(lessonNotes.value.length, currentSongLevel.value))
const levelChallengeScope = computed(() => {
  const total = lessonNotes.value.length
  if (!total)
    return { start: 0, length: 0 }
  if (!isLevelChallengeActive.value || lessonSourceMode.value !== 'song') {
    return {
      start: 0,
      length: total,
    }
  }
  const segment = currentLevelSegment.value
  const start = Math.max(0, Math.min(total - 1, segment.start))
  return {
    start,
    length: Math.max(1, Math.min(total - start, segment.length)),
  }
})
const currentLevelSegmentLabel = computed(() => {
  const segment = currentLevelSegment.value
  if (!segment.length)
    return '--'
  return `${segment.start + 1}-${segment.end}`
})
const currentLevelSegmentProgressLabel = computed(() => {
  const scope = levelChallengeScope.value
  if (!scope.length)
    return '--'
  const scopeOffset = toPositiveModulo(phraseChunkStart.value - scope.start, scope.length)
  const current = Math.min(scope.length, scopeOffset + phraseChunkInputIndex.value + 1)
  return `${current}/${scope.length}`
})

const phraseChunkNotes = computed(() => {
  const notesPool = lessonNotes.value
  const scope = levelChallengeScope.value
  if (!notesPool.length || !scope.length)
    return [] as string[]
  const scopeOffset = toPositiveModulo(phraseChunkStart.value - scope.start, scope.length)
  const start = scope.start + scopeOffset
  const remaining = scope.length - scopeOffset
  const chunkSize = Math.min(Math.max(1, phraseChunkSize.value), remaining)
  return notesPool.slice(start, start + chunkSize)
})

const targetLessonNote = computed(() => {
  if (!beginnerMode.value || !lessonNotes.value.length)
    return null
  if (phraseTypingMode.value)
    return phraseChunkNotes.value[phraseChunkInputIndex.value] || null
  return lessonNotes.value[lessonIndex.value % lessonNotes.value.length]
})

const targetLessonKey = computed(() => getKeyByNote(targetLessonNote.value))
const targetLessonHand = computed(() => {
  if (!targetLessonNote.value)
    return '--'
  return getHandForNote(targetLessonNote.value) === 'left' ? '左手' : '右手'
})
const lessonProgress = computed(() => {
  if (beginnerMode.value && phraseTypingMode.value && phraseChunkNotes.value.length)
    return `段 ${phraseChunkRound.value + 1} · ${Math.min(phraseChunkInputIndex.value + 1, phraseChunkNotes.value.length)}/${phraseChunkNotes.value.length}`
  const total = lessonNotes.value.length || 1
  return `${(lessonIndex.value % total) + 1}/${total}`
})
const lessonAccuracy = computed(() => {
  const total = lessonCorrect.value + lessonWrong.value
  if (!total)
    return '--'
  return `${Math.round((lessonCorrect.value / total) * 100)}%`
})
const metronomeBeatDots = computed(() => Array.from({ length: beatsPerMeasure.value }, (_, index) => index))
const metronomeCurrentBeat = computed(() => metronomeDisplayBeat.value || 1)
const effectivePracticeSpeed = computed(() => {
  const baseSpeed = practiceSpeed.value
  const adjustedSpeed = adaptiveTrainingEnabled.value ? baseSpeed + adaptiveSpeedDelta.value : baseSpeed
  return Number(Math.max(0.6, Math.min(1, adjustedSpeed)).toFixed(2))
})
const adaptiveTempoScale = computed(() => {
  if (!practiceSpeed.value)
    return 1
  return effectivePracticeSpeed.value / practiceSpeed.value
})
const effectiveMetronomeBpm = computed(() => {
  return Math.max(40, Math.round(metronomeBpm.value * adaptiveTempoScale.value))
})
const effectivePracticeSpeedLabel = computed(() => `${Math.round(effectivePracticeSpeed.value * 100)}%`)
const pianoSampleStatusLabel = computed(() => {
  const progress = `${Math.min(pianoSampleLoadedCount.value, pianoSampleKeys.length)}/${pianoSampleKeys.length}`
  const mirrorText = pianoSampleMirrorInUse.value ? ` · ${new URL(pianoSampleMirrorInUse.value).host}` : ''
  if (pianoSampleLoadState.value === 'ready')
    return `钢琴采样已启用 ${progress}${mirrorText}`
  if (pianoSampleLoadState.value === 'loading')
    return pianoSampleLoadedCount.value > 0 ? `钢琴采样部分可用 ${progress}` : `钢琴采样加载中 ${progress}`
  if (pianoSampleLoadState.value === 'error')
    return pianoSampleLoadedCount.value > 0 ? `钢琴采样部分可用 ${progress}` : `采样加载失败 ${progress}，当前使用合成音`
  return '点击后自动加载钢琴采样'
})
const autoPlayLegatoRatio = computed(() => {
  return selectedSong.value.id === 'anjing' ? 0.97 : 0.92
})
const countInTotalBeats = computed(() => countInMeasures.value * beatsPerMeasure.value)
const phraseChunkRangeLabel = computed(() => {
  const scope = levelChallengeScope.value
  if (!scope.length || !phraseChunkNotes.value.length)
    return '--'
  const scopeOffset = toPositiveModulo(phraseChunkStart.value - scope.start, scope.length)
  const start = scope.start + scopeOffset + 1
  const end = start + phraseChunkNotes.value.length - 1
  return `${start}-${end}`
})
const phraseChunkProgress = computed(() => {
  if (!phraseChunkNotes.value.length)
    return 0
  return Math.min(100, Math.round((phraseChunkInputIndex.value / phraseChunkNotes.value.length) * 100))
})
const phraseTypingGroupSize = computed(() => {
  if (phraseChunkNotes.value.length <= 4)
    return 2
  return currentSongLevel.value >= 4 ? 3 : 2
})
const phraseVisibleGroupLimit = computed(() => {
  if (!phraseChunkNotes.value.length)
    return 0
  if (phraseTrainerMode.value === 'listen')
    return Number.POSITIVE_INFINITY
  return Math.floor(phraseChunkInputIndex.value / phraseTypingGroupSize.value) + 1
})
const phraseRhythmNodes = computed(() => {
  return phraseChunkNotes.value.map((note, index) => {
    const groupIndex = Math.floor(index / phraseTypingGroupSize.value)
    const visible = groupIndex <= phraseVisibleGroupLimit.value
    const phase = index < phraseChunkInputIndex.value ? 'done' : index === phraseChunkInputIndex.value ? 'active' : 'pending'
    const accent = (index + 1) % phraseTypingGroupSize.value === 0
    return {
      note,
      index,
      visible,
      phase,
      accent,
    }
  })
})
const noteHeatmapRows = computed(() => {
  return Object.entries(notePerformanceMap)
    .map(([note, stat]) => {
      const total = stat.correct + stat.wrong
      return {
        note,
        correct: stat.correct,
        wrong: stat.wrong,
        total,
        accuracy: total ? Math.round((stat.correct / total) * 100) : 100,
      }
    })
    .filter(item => item.total > 0)
    .sort((a, b) => b.wrong - a.wrong || a.accuracy - b.accuracy || b.total - a.total)
})
const weakNotes = computed(() => noteHeatmapRows.value.filter(item => item.wrong > 0).slice(0, 6))
const weakNotesLabel = computed(() => {
  if (!weakNotes.value.length)
    return '暂无明显错音集中'
  return weakNotes.value
    .slice(0, 4)
    .map(item => `${item.note}×${item.wrong}`)
    .join(' · ')
})
const handFocusLabel = computed(() => {
  const leftWrong = handMistakeStats.left
  const rightWrong = handMistakeStats.right
  if (!leftWrong && !rightWrong)
    return '左右手错误分布稳定，继续保持当前触键。'
  if (leftWrong >= rightWrong * 1.3)
    return `左手失误偏多（左${leftWrong} / 右${rightWrong}），建议左手单独慢练 2 轮。`
  if (rightWrong >= leftWrong * 1.3)
    return `右手失误偏多（右${rightWrong} / 左${leftWrong}），建议右手单独慢练 2 轮。`
  return `左右手接近（左${leftWrong} / 右${rightWrong}），重点修正错音集中区。`
})
const fingeringTip = computed(() => {
  const focus = weakNotes.value[0]
  if (!focus)
    return '当前状态稳定，继续按“慢-准-稳”节奏推进。'
  const handLabel = getHandForNote(focus.note) === 'left' ? '左手' : '右手'
  if (focus.note.includes('#'))
    return `${handLabel}黑键 ${focus.note} 失误较多：避免拇指顶黑键，优先用 2/3 指过渡。`
  return `${handLabel}在 ${focus.note} 附近易错：先把速度降到 ${effectivePracticeSpeedLabel.value} 做 2 轮稳定练习。`
})
const phraseFlowSteps = ['选曲', '听句', '跟弹', '测试']
const phraseFlowActiveStep = computed(() => {
  if (!beginnerMode.value || !phraseTypingMode.value)
    return 0
  if (phraseTrainerMode.value === 'listen')
    return 1
  if (phraseTrainerMode.value === 'follow')
    return 2
  return 3
})
const focusPhaseSteps: Array<{ value: 'warmup' | 'phrase' | 'test'; label: string; desc: string }> = [
  { value: 'warmup', label: '热身', desc: '先听再找手感' },
  { value: 'phrase', label: '分句', desc: '按组跟弹稳定' },
  { value: 'test', label: '测试', desc: '隐藏提示检验' },
]
const focusPhaseActiveIndex = computed(() => focusPhaseSteps.findIndex(step => step.value === focusTrainingPhase.value))
const focusPhaseHint = computed(() => {
  if (focusTrainingPhase.value === 'warmup')
    return '热身阶段只保留听句和基础速度，不显示复杂数据。'
  if (focusTrainingPhase.value === 'phrase')
    return '分句阶段专注节奏轨道与当前短句，先稳再快。'
  return '测试阶段开启挑战看板与评级，检验独立完成度。'
})
const pianoGeometry = computed(() => {
  const isCompactPiano = isFocusLayout.value || uiMode.value === 'perform'
  if (!isCompactPiano) {
    return {
      whiteKeyWidth: 58,
      whiteKeyMargin: 2,
      blackKeyWidth: 34,
      whiteKeyHeight: 220,
      blackKeyHeight: 142,
      blackKeyTop: 76,
      pianoTopPad: 76,
      pianoSidePad: 16,
      pianoBottomPad: 16,
      blackLeftInset: 10,
    }
  }
  if (viewportWidth.value <= 520) {
    return {
      whiteKeyWidth: 31,
      whiteKeyMargin: 1,
      blackKeyWidth: 19,
      whiteKeyHeight: 132,
      blackKeyHeight: 82,
      blackKeyTop: 46,
      pianoTopPad: 46,
      pianoSidePad: 8,
      pianoBottomPad: 8,
      blackLeftInset: 6,
    }
  }
  if (viewportWidth.value <= 820) {
    return {
      whiteKeyWidth: 38,
      whiteKeyMargin: 1,
      blackKeyWidth: 24,
      whiteKeyHeight: 156,
      blackKeyHeight: 98,
      blackKeyTop: 54,
      pianoTopPad: 54,
      pianoSidePad: 10,
      pianoBottomPad: 10,
      blackLeftInset: 7,
    }
  }
  return {
    whiteKeyWidth: 46,
    whiteKeyMargin: 1,
    blackKeyWidth: 28,
    whiteKeyHeight: 178,
    blackKeyHeight: 112,
    blackKeyTop: 62,
    pianoTopPad: 62,
    pianoSidePad: 12,
    pianoBottomPad: 12,
    blackLeftInset: 8,
  }
})
const pianoStyleVars = computed(() => {
  const geometry = pianoGeometry.value
  return {
    '--white-key-width': `${geometry.whiteKeyWidth}px`,
    '--white-key-margin': `${geometry.whiteKeyMargin}px`,
    '--black-key-width': `${geometry.blackKeyWidth}px`,
    '--white-key-height': `${geometry.whiteKeyHeight}px`,
    '--black-key-height': `${geometry.blackKeyHeight}px`,
    '--black-key-top': `${geometry.blackKeyTop}px`,
    '--piano-top-pad': `${geometry.pianoTopPad}px`,
    '--piano-side-pad': `${geometry.pianoSidePad}px`,
    '--piano-bottom-pad': `${geometry.pianoBottomPad}px`,
  }
})
const isFocusLayout = computed(() => practiceLayoutMode.value === 'focus')
const showAdvancedCoach = computed(() => !isFocusLayout.value || showAdvancedCoachControls.value)
const isSongPanelVisible = computed(() => {
  if (isFocusLayout.value)
    return showSongPanelInFocus.value
  if (uiMode.value === 'perform')
    return showSongPanel.value
  return true
})
const isSongPanelBodyVisible = computed(() => isFocusLayout.value ? showSongPanelInFocus.value : showSongPanel.value)
const levelProgressPercent = computed(() => {
  return Math.round((currentSongClearedLevel.value / levelRules.length) * 100)
})
const currentLevelLivesMax = computed(() => {
  if (currentSongLevel.value >= 5)
    return 1
  if (currentSongLevel.value >= 3)
    return 2
  return 3
})
const testLivesDots = computed(() => Array.from({ length: currentLevelLivesMax.value }, (_, index) => index))
const passedTestCount = computed(() => testGradeStats.A + testGradeStats.B + testGradeStats.C)
const lessonAccuracyNumber = computed(() => {
  const total = lessonCorrect.value + lessonWrong.value
  return total ? (lessonCorrect.value / total) * 100 : 0
})
const averageTestGradePoint = computed(() => {
  if (!passedTestCount.value)
    return 0
  const gradePoints = testGradeStats.A * 3 + testGradeStats.B * 2 + testGradeStats.C
  return Number((gradePoints / passedTestCount.value).toFixed(2))
})
const currentLevelRuleMet = computed(() => {
  const rule = currentLevelRule.value
  return passedTestCount.value >= rule.minPassedTests
    && lessonAccuracyNumber.value >= rule.minAccuracy
    && averageTestGradePoint.value >= rule.minAverageGrade
    && testGradeStats.A >= rule.minA
})
const currentLevelRuleText = computed(() => {
  const rule = currentLevelRule.value
  return `通过≥${rule.minPassedTests} · 准确率≥${rule.minAccuracy}% · 平均评级≥${rule.minAverageGrade} · A≥${rule.minA}`
})
const testGradeOverviewLabel = computed(() => {
  return `A ${testGradeStats.A} · B ${testGradeStats.B} · C ${testGradeStats.C} · F ${testGradeStats.F}`
})
const levelChallengeStatusLabel = computed(() => {
  if (isLevelChallengeActive.value)
    return `闯关进行中 · 段内进度 ${currentLevelSegmentProgressLabel.value}`
  return '闯关待开始'
})
const retrainSuggestionSpeedLabel = computed(() => {
  if (!retrainSuggestionSpeed.value)
    return '--'
  return `${Math.round(retrainSuggestionSpeed.value * 100)}%`
})

const getPhraseTypingDisplayNote = (note: string, index: number) => {
  const groupIndex = Math.floor(index / phraseTypingGroupSize.value)
  if (groupIndex > phraseVisibleGroupLimit.value)
    return '··'
  return note
}

const getPhraseTypingMaskClass = (index: number) => {
  const groupIndex = Math.floor(index / phraseTypingGroupSize.value)
  return groupIndex > phraseVisibleGroupLimit.value ? 'masked' : ''
}

const clearRetrainSuggestionState = () => {
  showRetrainSuggestion.value = false
  retrainSuggestionText.value = ''
  retrainSuggestionSpeed.value = null
}

const getSteppedDownPracticeSpeed = () => {
  const currentIndex = speedOptions.findIndex(option => option.value === practiceSpeed.value)
  if (currentIndex > 0)
    return speedOptions[currentIndex - 1].value
  return Number(Math.max(0.6, Math.min(1, practiceSpeed.value - 0.2)).toFixed(2))
}

const prepareRetrainSuggestion = () => {
  const suggestedSpeed = getSteppedDownPracticeSpeed()
  retrainSuggestionSpeed.value = suggestedSpeed
  showRetrainSuggestion.value = true
  const modeHint = testMistakesCurrentChunk.value >= 2 ? '先听一句再跟弹' : '先回到跟弹'
  retrainSuggestionText.value = `本段建议 ${modeHint}，并把速度调整到 ${Math.round(suggestedSpeed * 100)}% 后再重测。`
}

const applyRetrainSuggestion = async () => {
  if (retrainSuggestionSpeed.value)
    practiceSpeed.value = retrainSuggestionSpeed.value
  adaptiveSpeedDelta.value = 0
  phraseChunkInputIndex.value = 0
  setPhraseTrainerMode('listen', false)
  await playCurrentPhraseChunk(true)
  if (phraseTrainerMode.value === 'listen')
    setPhraseTrainerMode('follow', false)
  pushInteractionToast(`已按建议复训，目标速度 ${retrainSuggestionSpeedLabel.value}`, 'info')
  clearRetrainSuggestionState()
}

const dismissRetrainSuggestion = () => {
  clearRetrainSuggestionState()
}

const closeLevelClearDialog = () => {
  levelClearState.value = null
}

const enterUnlockedLevel = () => {
  const unlockedLevel = levelClearState.value?.unlockedLevel
  closeLevelClearDialog()
  if (unlockedLevel === null) {
    isLevelChallengeActive.value = false
    setPhraseTrainerMode('follow', false)
    pushInteractionToast('已完成全部关卡，可切歌或自由练习', 'success')
    return
  }
  startCurrentLevelChallenge()
  pushInteractionToast(`进入 Level ${unlockedLevel} 挑战`, 'success')
}

const resetLesson = () => {
  lessonIndex.value = 0
  lessonCorrect.value = 0
  lessonWrong.value = 0
  lessonFeedback.value = ''
  wrongStreak.value = 0
  correctStreak.value = 0
  adaptiveSpeedDelta.value = 0
  phraseChunkStart.value = isLevelChallengeActive.value ? currentLevelSegment.value.start : 0
  phraseChunkInputIndex.value = 0
  phraseChunkRound.value = 0
  focusTrainingPhase.value = 'warmup'
  phraseTrainerMode.value = 'follow'
  testLives.value = currentLevelLivesMax.value
  testMistakesCurrentChunk.value = 0
  currentChunkTestGrade.value = ''
  testGradeStats.A = 0
  testGradeStats.B = 0
  testGradeStats.C = 0
  testGradeStats.F = 0
  Object.keys(notePerformanceMap).forEach((noteId) => {
    delete notePerformanceMap[noteId]
  })
  handMistakeStats.left = 0
  handMistakeStats.right = 0
  practiceSessionStartedAt.value = Date.now()
  showPracticeSummary.value = false
  practiceSummary.value = null
  clearRetrainSuggestionState()
}

const resetCurrentChunkTestState = () => {
  testLives.value = currentLevelLivesMax.value
  testMistakesCurrentChunk.value = 0
  currentChunkTestGrade.value = ''
}

const getChunkTestGrade = (mistakes: number, passed: boolean): 'A' | 'B' | 'C' | 'F' => {
  if (!passed)
    return 'F'
  if (mistakes <= 0)
    return 'A'
  if (mistakes <= 1)
    return 'B'
  return 'C'
}

const finalizeChunkTestResult = (passed: boolean) => {
  const grade = getChunkTestGrade(testMistakesCurrentChunk.value, passed)
  currentChunkTestGrade.value = grade
  testGradeStats[grade] += 1
  if (!passed) {
    prepareRetrainSuggestion()
    setPhraseTrainerMode('follow', false)
    pushInteractionToast('测试未通过，已切回跟弹并给出复训建议', 'error')
  }
  else {
    clearRetrainSuggestionState()
    pushInteractionToast(`测试通过，当前段评级 ${grade}`, 'success')
  }
  testLives.value = currentLevelLivesMax.value
  testMistakesCurrentChunk.value = 0
}

const restartCurrentChunkTest = () => {
  clearRetrainSuggestionState()
  setPhraseTrainerMode('test', false)
  phraseChunkInputIndex.value = 0
  resetCurrentChunkTestState()
  pushInteractionToast('已重置本段测试，重新开始', 'info')
}

const getPracticeRecommendation = (accuracy: number, notesPerMinute: number) => {
  if (accuracy < 0.7)
    return '建议先切换到“听句”并降速到 60%-80%，先保证音准再提速。'
  if (accuracy < 0.85)
    return '建议继续“跟弹”模式，把当前短句连续正确 3 次后再切到测试。'
  if (notesPerMinute < 40)
    return '准确率不错，下一步可以维持正确率并稍微提速 5%-10%。'
  return '当前状态很好，可以切到测试模式并挑战下一段。'
}

const openPracticeSummary = (fromAuto = false) => {
  const elapsedSec = Math.max(1, Math.round((Date.now() - practiceSessionStartedAt.value) / 1000))
  const total = lessonCorrect.value + lessonWrong.value
  const accuracy = total ? lessonCorrect.value / total : 0
  const notesPerMinute = Math.round((lessonCorrect.value / elapsedSec) * 60)
  const minutes = Math.floor(elapsedSec / 60)
  const seconds = String(elapsedSec % 60).padStart(2, '0')
  practiceSummary.value = {
    durationLabel: `${minutes}:${seconds}`,
    accuracyLabel: total ? `${Math.round(accuracy * 100)}%` : '--',
    rounds: phraseChunkRound.value,
    correct: lessonCorrect.value,
    wrong: lessonWrong.value,
    notesPerMinute,
    recommendation: getPracticeRecommendation(accuracy, notesPerMinute),
    testGradeOverview: testGradeOverviewLabel.value,
    weakNotesLabel: weakNotesLabel.value,
    handFocusLabel: handFocusLabel.value,
    fingeringTip: fingeringTip.value,
  }
  showPracticeSummary.value = true
  if (!fromAuto)
    pushInteractionToast('已生成练习复盘', 'info')
}

const closePracticeSummary = () => {
  showPracticeSummary.value = false
}

const clearInteractionToast = () => {
  if (interactionToastTimer.value) {
    clearTimeout(interactionToastTimer.value)
    interactionToastTimer.value = null
  }
}

const pushInteractionToast = (message: string, type: 'success' | 'info' | 'error' = 'info', duration = 2200) => {
  interactionToast.value = message
  interactionToastType.value = type
  clearInteractionToast()
  interactionToastTimer.value = setTimeout(() => {
    interactionToast.value = ''
    interactionToastTimer.value = null
  }, duration)
}

const setPracticeLayoutMode = (mode: 'focus' | 'full') => {
  practiceLayoutMode.value = mode
  if (mode === 'focus') {
    showSongPanelInFocus.value = false
    showAdvancedCoachControls.value = false
    if (phraseTrainerMode.value === 'test')
      focusTrainingPhase.value = 'test'
    else if (phraseTrainerMode.value === 'follow')
      focusTrainingPhase.value = 'phrase'
    else
      focusTrainingPhase.value = 'warmup'
    pushInteractionToast('已切换到专注练习布局', 'info')
    return
  }
  showSongPanel.value = uiMode.value !== 'perform'
  showCoachPanel.value = true
  pushInteractionToast('已切换到完整布局', 'info')
}

const toggleSongWorkbench = () => {
  if (isFocusLayout.value) {
    showSongPanelInFocus.value = !showSongPanelInFocus.value
    pushInteractionToast(showSongPanelInFocus.value ? '已展开选曲区' : '已收起选曲区', 'info')
    return
  }
  showSongPanel.value = !showSongPanel.value
}

const toggleAdvancedCoachPanel = () => {
  showAdvancedCoachControls.value = !showAdvancedCoachControls.value
  pushInteractionToast(showAdvancedCoachControls.value ? '已展开高级练习设置' : '已收起高级练习设置', 'info')
}

const setFocusTrainingPhase = (phase: 'warmup' | 'phrase' | 'test', needToast = true) => {
  focusTrainingPhase.value = phase
  if (phase === 'warmup')
    setPhraseTrainerMode('listen', false, false)
  else if (phase === 'phrase')
    setPhraseTrainerMode('follow', false, false)
  else
    setPhraseTrainerMode('test', false, false)
  if (!needToast)
    return
  if (phase === 'warmup')
    pushInteractionToast('已进入热身阶段', 'info')
  else if (phase === 'phrase')
    pushInteractionToast('已进入分句阶段', 'info')
  else
    pushInteractionToast('已进入测试阶段', 'info')
}

const startFocusWarmup = async () => {
  setFocusTrainingPhase('warmup', false)
  await playCurrentPhraseChunk(true)
  if (isFocusLayout.value && focusTrainingPhase.value === 'warmup')
    await playCurrentPhraseChunk(true)
  if (isFocusLayout.value)
    setFocusTrainingPhase('phrase', false)
  pushInteractionToast('热身完成，进入分句跟弹', 'success')
}

const startFocusPhraseStage = () => {
  setFocusTrainingPhase('phrase')
}

const startFocusTestStage = () => {
  setFocusTrainingPhase('test')
}

const setUiMode = (mode: 'learn' | 'perform') => {
  uiMode.value = mode
  if (mode === 'perform') {
    showSongPanel.value = false
    showCoachPanel.value = false
    pushInteractionToast('已切换到演奏视图', 'info')
  }
  else {
    showSongPanel.value = true
    showCoachPanel.value = true
    pushInteractionToast('已切换到学习视图', 'info')
  }
}

const setPhraseTrainerMode = (mode: 'listen' | 'follow' | 'test', needToast = true, syncFocusPhase = true) => {
  const wasMode = phraseTrainerMode.value
  phraseTrainerMode.value = mode
  if (syncFocusPhase && isFocusLayout.value) {
    if (mode === 'test')
      focusTrainingPhase.value = 'test'
    else if (mode === 'follow')
      focusTrainingPhase.value = 'phrase'
    else
      focusTrainingPhase.value = 'warmup'
  }
  if (mode === 'test' && wasMode !== 'test') {
    clearRetrainSuggestionState()
    resetCurrentChunkTestState()
  }
  if (!needToast)
    return
  if (mode === 'listen')
    pushInteractionToast('已切到听句模式', 'info')
  else if (mode === 'follow')
    pushInteractionToast('已切到跟弹模式', 'info')
  else
    pushInteractionToast('已切到测试模式（不显示目标键）', 'info')
}

const startPhraseListen = async () => {
  setPhraseTrainerMode('listen')
  await playCurrentPhraseChunk(true)
}

const startPhraseFollow = () => {
  setPhraseTrainerMode('follow')
}

const startPhraseTest = () => {
  clearRetrainSuggestionState()
  setPhraseTrainerMode('test')
}

const resetCurrentSongLevelProgress = () => {
  songLevelProgress.value[selectedSongId.value] = {
    currentLevel: 1,
    clearedLevel: 0,
  }
  isLevelChallengeActive.value = false
  closeLevelClearDialog()
  resetLesson()
  pushInteractionToast(`已重置《${selectedSong.value.title}》闯关进度`, 'info')
}

const startCurrentLevelChallenge = () => {
  beginnerMode.value = true
  phraseTypingMode.value = true
  isLevelChallengeActive.value = true
  lessonSourceMode.value = 'song'
  if (showCoachPanel.value === false)
    showCoachPanel.value = true
  closeLevelClearDialog()
  resetLesson()
  phraseChunkStart.value = currentLevelSegment.value.start
  setPhraseTrainerMode('test', false)
  resetCurrentChunkTestState()
  pushInteractionToast(`开始 Level ${currentSongLevel.value} 闯关 · 段落 ${currentLevelSegmentLabel.value}`, 'info')
}

const tryCompleteCurrentLevel = () => {
  if (!isLevelChallengeActive.value || !currentLevelRuleMet.value)
    return false

  const progress = getSongLevelProgress()
  const clearedLevel = progress.currentLevel
  const unlockedLevel = clearedLevel < levelRules.length ? clearedLevel + 1 : null
  progress.clearedLevel = Math.max(progress.clearedLevel, clearedLevel)
  if (unlockedLevel !== null)
    progress.currentLevel = unlockedLevel

  levelClearState.value = {
    clearedLevel,
    unlockedLevel,
    songTitle: selectedSong.value.title,
  }
  isLevelChallengeActive.value = false
  setPhraseTrainerMode('follow', false)
  clearRetrainSuggestionState()
  pushInteractionToast(unlockedLevel === null ? '全关卡通关，已完成本曲挑战' : `Level ${clearedLevel} 通关`, 'success', 2600)
  return true
}

const quickStopAll = (needToast = true) => {
  interruptAutoPlay()
  stopMetronome()
  cancelCountIn()
  releaseAllNotes()
  if (needToast)
    pushInteractionToast('已停止当前播放', 'info')
}

const isTypingTarget = (target: EventTarget | null) => {
  if (!(target instanceof HTMLElement))
    return false
  const tagName = target.tagName
  return tagName === 'INPUT' || tagName === 'TEXTAREA' || tagName === 'SELECT' || target.isContentEditable
}

const getPhraseNoteStateClass = (index: number) => {
  if (index < phraseChunkInputIndex.value)
    return 'done'
  if (index === phraseChunkInputIndex.value)
    return 'active'
  return 'pending'
}

const handleLearningInput = (noteId: string, source: string) => {
  if (!beginnerMode.value || source === AUTO_PLAY_SOURCE)
    return
  if (phraseTypingMode.value && phraseChunkNotes.value.length) {
    const expectedNote = phraseChunkNotes.value[phraseChunkInputIndex.value]
    if (!expectedNote)
      return

    if (noteId === expectedNote) {
      recordExpectedNoteAttempt(expectedNote, true)
      lessonCorrect.value += 1
      lessonFeedback.value = 'correct'
      wrongStreak.value = 0
      correctStreak.value += 1
      if (adaptiveTrainingEnabled.value && correctStreak.value >= 3 && adaptiveSpeedDelta.value < 0) {
        adaptiveSpeedDelta.value = Number(Math.min(0, adaptiveSpeedDelta.value + 0.05).toFixed(2))
        correctStreak.value = 0
      }

      phraseChunkInputIndex.value += 1
      if (phraseChunkInputIndex.value >= phraseChunkNotes.value.length) {
        const chunkCompletedInTest = phraseTrainerMode.value === 'test'
        if (chunkCompletedInTest)
          finalizeChunkTestResult(true)
        const scope = levelChallengeScope.value
        const scopeLength = scope.length || 1
        const scopeOffset = toPositiveModulo(phraseChunkStart.value - scope.start, scopeLength)
        const nextOffset = scopeOffset + phraseChunkNotes.value.length
        phraseChunkStart.value = nextOffset >= scopeLength ? scope.start : scope.start + nextOffset
        phraseChunkInputIndex.value = 0
        phraseChunkRound.value += 1
        if (phraseTrainerMode.value === 'listen')
          setPhraseTrainerMode('follow', false)
        if (isFocusLayout.value && focusTrainingPhase.value === 'warmup')
          focusTrainingPhase.value = 'phrase'
        if (phraseChunkRound.value > 0 && phraseChunkRound.value % 4 === 0)
          openPracticeSummary(true)
        if (chunkCompletedInTest)
          tryCompleteCurrentLevel()
      }
    }
    else {
      recordExpectedNoteAttempt(expectedNote, false)
      lessonWrong.value += 1
      lessonFeedback.value = 'wrong'
      correctStreak.value = 0
      wrongStreak.value += 1
      if (phraseTrainerMode.value === 'test') {
        testMistakesCurrentChunk.value += 1
        testLives.value = Math.max(0, testLives.value - 1)
        phraseChunkInputIndex.value = 0
        if (testLives.value <= 0) {
          finalizeChunkTestResult(false)
        }
        else {
          pushInteractionToast(`测试失误，还剩 ${testLives.value} 次机会`, 'error')
        }
      }
      if (adaptiveTrainingEnabled.value && wrongStreak.value >= 2) {
        adaptiveSpeedDelta.value = Number(Math.max(-0.25, adaptiveSpeedDelta.value - 0.05).toFixed(2))
        wrongStreak.value = 0
      }
    }
    return
  }

  const expectedNote = targetLessonNote.value
  if (!expectedNote)
    return

  if (noteId === expectedNote) {
    recordExpectedNoteAttempt(expectedNote, true)
    lessonCorrect.value += 1
    const total = lessonNotes.value.length || 1
    lessonIndex.value = (lessonIndex.value + 1) % total
    lessonFeedback.value = 'correct'
    wrongStreak.value = 0
    correctStreak.value += 1
    if (adaptiveTrainingEnabled.value && correctStreak.value >= 3 && adaptiveSpeedDelta.value < 0) {
      adaptiveSpeedDelta.value = Number(Math.min(0, adaptiveSpeedDelta.value + 0.05).toFixed(2))
      correctStreak.value = 0
    }
  }
  else {
    recordExpectedNoteAttempt(expectedNote, false)
    lessonWrong.value += 1
    lessonFeedback.value = 'wrong'
    correctStreak.value = 0
    wrongStreak.value += 1
    if (adaptiveTrainingEnabled.value && wrongStreak.value >= 2) {
      adaptiveSpeedDelta.value = Number(Math.max(-0.25, adaptiveSpeedDelta.value - 0.05).toFixed(2))
      wrongStreak.value = 0
    }
  }
}

// --- Core Logic Functions ---
const initKeys = () => {
  keyMap = {
    a: 'C3',
    s: 'D3',
    d: 'E3',
    f: 'F3',
    h: 'G3',
    j: 'A3',
    k: 'B3',
    q: 'C#3',
    w: 'D#3',
    e: 'F#3',
    r: 'G#3',
    t: 'A#3',
    z: 'C4',
    x: 'D4',
    c: 'E4',
    v: 'F4',
    b: 'G4',
    n: 'A4',
    m: 'B4',
    y: 'C#4',
    u: 'D#4',
    i: 'F#4',
    o: 'G#4',
    p: 'A#4',
  }
  keys = Object.keys(keyMap)
}

const getFrequency = (note: string | null): number => {
  if (!note)
    return 0 // Return 0 for rests/null notes
  let octave, key: keyof typeof tone
  note = note.toLowerCase()

  if (note.length === 3) {
    octave = note.charAt(2)
    key = note.slice(0, 2) as keyof typeof tone
  }
  else {
    octave = note.charAt(1)
    key = note.slice(0, 1) as keyof typeof tone
  }
  octave = +octave + 1
  if (tone[key] && tone[key][octave] !== undefined)
    return tone[key][octave]

  console.warn(`Invalid note or octave: ${key}${octave - 1}`)
  return 0.1
}

const initAudio = () => {
  const AudioContextCtor = window.AudioContext
    || (window as Window & { webkitAudioContext?: typeof AudioContext }).webkitAudioContext
  if (!audio && AudioContextCtor) {
    audio = new AudioContextCtor()
    masterGainNode = audio.createGain()
    masterGainNode.gain.value = masterVolume.value
    masterGainNode.connect(audio.destination)
    // Resume context if it starts suspended (required by some browsers)
    if (audio.state === 'suspended')
      audio.resume()
  }
}

const applyMasterVolume = () => {
  if (!audio || !masterGainNode)
    return
  const now = audio.currentTime
  const safeVolume = Math.max(0, Math.min(masterVolume.value, 0.16))
  masterGainNode.gain.cancelScheduledValues(now)
  masterGainNode.gain.setTargetAtTime(safeVolume, now, 0.02)
}

const clearSuspendAudioTask = () => {
  if (suspendAudioTimeoutId.value) {
    clearTimeout(suspendAudioTimeoutId.value)
    suspendAudioTimeoutId.value = null
  }
}

const scheduleSuspendAudio = () => {
  if (!audio)
    return
  clearSuspendAudioTask()
  if (Object.keys(activeButtonIdMap).length > 0 || isAutoPlaying.value || metronomeEnabled.value || isCountInActive.value)
    return

  suspendAudioTimeoutId.value = setTimeout(() => {
    if (audio && audio.state === 'running')
      audio.suspend()
    suspendAudioTimeoutId.value = null
  }, 500)
}

const playClickTick = (accent: boolean) => {
  initAudio()
  if (!audio)
    return

  clearSuspendAudioTask()
  if (audio.state === 'suspended')
    audio.resume()

  const oscillator = audio.createOscillator()
  const gain = audio.createGain()
  const now = audio.currentTime

  oscillator.type = 'square'
  oscillator.frequency.value = accent ? 1320 : 980
  oscillator.connect(gain)
  gain.connect(masterGainNode)

  gain.gain.setValueAtTime(0.0001, now)
  gain.gain.exponentialRampToValueAtTime(accent ? 0.32 : 0.22, now + 0.003)
  gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.045)

  oscillator.start(now)
  oscillator.stop(now + 0.05)
  oscillator.onended = () => {
    oscillator.disconnect()
    gain.disconnect()
  }
}

const cancelCountIn = () => {
  countInSessionId.value += 1
  isCountInActive.value = false
  countInCurrentBeat.value = 0
}

const runCountIn = async () => {
  if (!countInEnabled.value || countInTotalBeats.value <= 0)
    return true

  const sessionId = countInSessionId.value + 1
  countInSessionId.value = sessionId
  isCountInActive.value = true
  countInCurrentBeat.value = 0
  const intervalMs = Math.max(120, Math.round(60000 / effectiveMetronomeBpm.value))

  for (let beat = 0; beat < countInTotalBeats.value; beat++) {
    if (countInSessionId.value !== sessionId) {
      isCountInActive.value = false
      countInCurrentBeat.value = 0
      return false
    }
    const beatInMeasure = beat % beatsPerMeasure.value
    countInCurrentBeat.value = beat + 1
    playClickTick(beatInMeasure === 0)
    await new Promise(resolve => setTimeout(resolve, intervalMs))
  }

  isCountInActive.value = false
  countInCurrentBeat.value = 0
  return true
}

const stopMetronome = () => {
  if (metronomeTimerId.value) {
    clearInterval(metronomeTimerId.value)
    metronomeTimerId.value = null
  }
  metronomeBeat.value = 0
  metronomeDisplayBeat.value = 0
  scheduleSuspendAudio()
}

const playMetronomeTick = () => {
  const beatIndex = metronomeBeat.value % beatsPerMeasure.value
  const accent = beatIndex === 0
  metronomeDisplayBeat.value = beatIndex + 1
  playClickTick(accent)
  metronomeBeat.value = (beatIndex + 1) % beatsPerMeasure.value
}

const startMetronome = () => {
  stopMetronome()
  const intervalMs = Math.max(120, Math.round(60000 / effectiveMetronomeBpm.value))
  playMetronomeTick()
  metronomeTimerId.value = setInterval(playMetronomeTick, intervalMs)
}

const getPianoFallbackWave = (context: AudioContext) => {
  const cached = pianoFallbackWaveCache.get(context)
  if (cached)
    return cached
  const harmonics = 18
  const real = new Float32Array(harmonics + 1)
  const imag = new Float32Array(harmonics + 1)
  for (let i = 1; i <= harmonics; i++) {
    const strength = (1 / i) * Math.exp(-i / 8)
    real[i] = strength * 0.2
    imag[i] = strength
  }
  const wave = context.createPeriodicWave(real, imag, { disableNormalization: false })
  pianoFallbackWaveCache.set(context, wave)
  return wave
}

const playTone = (id: string, source = AUTO_PLAY_SOURCE, intensity = 1): Omit<ActiveToneNode, 'key'> | null => {
  initAudio() // Ensure audio context is ready
  if (!audio)
    return null // Exit if audio context failed to initialize

  clearSuspendAudioTask()
  // Resume AudioContext if it's suspended
  if (audio.state === 'suspended')
    audio.resume()

  const frequency = getFrequency(id)
  if (frequency <= 0.1)
    return null // Don't play invalid notes

  if (pianoSampleLoadState.value === 'idle' || (pianoSampleLoadState.value === 'error' && pianoSampleLoadedCount.value === 0))
    void loadPianoSamples()

  const nearestSample = findNearestPianoSample(id, true)
  const sampleBuffer = nearestSample ? pianoSampleBuffers[nearestSample.sampleKey] : undefined
  if (nearestSample && sampleBuffer) {
    const sampleSource = audio.createBufferSource()
    const noteFilter = audio.createBiquadFilter()
    const noteGain = audio.createGain()
    const now = audio.currentTime
    const isAutoNote = source === AUTO_PLAY_SOURCE
    const safeIntensity = Math.max(0.08, Math.min(1, intensity))
    const peakGain = (isAutoNote ? 0.74 : 0.9) * safeIntensity
    const sustainGain = (isAutoNote ? 0.2 : 0.26) * safeIntensity
    const attack = isAutoNote ? 0.006 : 0.01
    const decay = isAutoNote ? 0.3 : 0.24

    sampleSource.buffer = sampleBuffer
    sampleSource.playbackRate.value = 2 ** ((nearestSample.targetMidi - nearestSample.sampleMidi) / 12)
    sampleSource.connect(noteFilter)
    noteFilter.type = 'lowpass'
    noteFilter.frequency.setValueAtTime(isAutoNote ? 3600 : 4200, now)
    noteFilter.Q.setValueAtTime(0.7, now)
    noteFilter.connect(noteGain)
    noteGain.connect(masterGainNode)
    noteGain.gain.setValueAtTime(0.0001, now)
    noteGain.gain.exponentialRampToValueAtTime(peakGain, now + attack)
    noteGain.gain.exponentialRampToValueAtTime(sustainGain, now + attack + decay)
    sampleSource.start(now)
    return { sampleSource, noteGain, noteFilter }
  }
  const nearestPendingSample = findNearestPianoSample(id, false)
  if (nearestPendingSample && !pianoSampleBuffers[nearestPendingSample.sampleKey])
    void loadSinglePianoSample(nearestPendingSample.sampleKey)

  const oscillator = audio.createOscillator()
  const noteFilter = audio.createBiquadFilter()
  const noteGain = audio.createGain()
  const now = audio.currentTime
  const isAutoNote = source === AUTO_PLAY_SOURCE
  const safeIntensity = Math.max(0.08, Math.min(1, intensity))
  const peakGain = (isAutoNote ? 0.78 : 0.95) * safeIntensity
  const sustainGain = (isAutoNote ? 0.16 : 0.22) * safeIntensity
  const attack = isAutoNote ? 0.008 : 0.012
  const decay = isAutoNote ? 0.24 : 0.2

  if (toneMode.value === 'sine')
    oscillator.setPeriodicWave(getPianoFallbackWave(audio))
  else
    oscillator.type = toneMode.value
  oscillator.connect(noteFilter)
  noteFilter.type = 'lowpass'
  noteFilter.frequency.setValueAtTime(isAutoNote ? 1900 : 2400, now)
  noteFilter.Q.setValueAtTime(1.05, now)
  noteFilter.connect(noteGain)
  noteGain.connect(masterGainNode)
  noteGain.gain.setValueAtTime(0.0001, now)
  noteGain.gain.exponentialRampToValueAtTime(peakGain * 0.92, now + attack * 0.8)
  noteGain.gain.exponentialRampToValueAtTime(sustainGain * 1.05, now + attack + decay * 1.2)
  oscillator.frequency.value = frequency
  oscillator.start()
  return { oscillator, noteGain, noteFilter }
}

const stopTone = (id: string) => {
  if (activeButtonIdMap[id]?.noteGain && activeButtonIdMap[id]?.noteFilter) {
    try {
      const { oscillator, sampleSource, noteGain, noteFilter, key } = activeButtonIdMap[id]
      if (!audio) {
        if (oscillator) {
          oscillator.stop()
          oscillator.disconnect()
        }
        if (sampleSource) {
          sampleSource.stop()
          sampleSource.disconnect()
        }
        noteFilter.disconnect()
        noteGain.disconnect()
        return
      }

      const now = audio.currentTime
      const release = key === AUTO_PLAY_SOURCE ? 0.085 : 0.06

      noteGain.gain.cancelScheduledValues(now)
      noteGain.gain.setValueAtTime(Math.max(noteGain.gain.value, 0.0001), now)
      noteGain.gain.exponentialRampToValueAtTime(0.0001, now + release)
      if (oscillator)
        oscillator.stop(now + release + 0.01)
      if (sampleSource)
        sampleSource.stop(now + release + 0.01)
      const cleanup = () => {
        oscillator?.disconnect()
        sampleSource?.disconnect()
        noteFilter.disconnect()
        noteGain.disconnect()
      }
      if (oscillator)
        oscillator.onended = cleanup
      else if (sampleSource)
        sampleSource.onended = cleanup
      else
        cleanup()
    }
    catch (e) {
      // Source might already be stopped
      console.warn(`Error stopping source for ${id}:`, e)
    }
  }
}

const releaseAllNotes = () => {
  Object.keys(activeButtonIdMap).forEach(noteId => releaseKey(noteId))
}

const releasePointerNotes = () => {
  Object.keys(activeButtonIdMap).forEach((noteId) => {
    if (activeButtonIdMap[noteId]?.key === POINTER_SOURCE)
      releaseKey(noteId)
  })
}

// --- Refactored Key Press/Release ---
const pressKey = (noteId: string, keyChar = AUTO_PLAY_SOURCE, intensity = 1) => {
  if (!noteId || activeButtonIdMap[noteId])
    return // Don't press if invalid or already active

  const toneResult = playTone(noteId, keyChar, intensity)
  if (toneResult) {
    activeButtonIdMap[noteId] = { ...toneResult, key: keyChar }
    handleLearningInput(noteId, keyChar)
  }
}

const releaseKey = (noteId: string) => {
  if (!noteId || !activeButtonIdMap[noteId])
    return // Don't release if invalid or not active

  stopTone(noteId)
  delete activeButtonIdMap[noteId]
  scheduleSuspendAudio()
}

// --- Auto Play Logic (Updated for Chords) ---
const interruptAutoPlay = () => {
  cancelCountIn()
  if (isAutoPlaying.value) {
    isAutoPlaying.value = false
    if (autoPlayTimeoutId.value) {
      clearTimeout(autoPlayTimeoutId.value)
      autoPlayTimeoutId.value = null
    }
    // Stop ALL currently playing auto-play notes
    // const currentNote = pressedKey.value; // REMOVE THIS LINE
    Object.keys(activeButtonIdMap).forEach((noteId) => {
      if (activeButtonIdMap[noteId]?.key === AUTO_PLAY_SOURCE) {
        releaseKey(noteId)
      }
    })
    currentLyric.value = '' // Clear lyric on interrupt
  }
}

const playSong = async (song: SongNote[]) => {
  isAutoPlaying.value = true
  currentLyric.value = '' // Clear lyrics initially
  const speed = effectivePracticeSpeed.value
  let melodyStep = 0
  for (let i = 0; i < song.length; i++) {
    if (!isAutoPlaying.value)
      break

    const { duration, lyric } = song[i]
    const actualDuration = Math.round(duration / speed)
    const playableNotes = getSongNotePlayableNotes(song[i])

    // Update lyric when the note starts
    if (lyric)
      currentLyric.value = lyric

    if (playableNotes.length) {
      const tonePlan = buildAutoPlayTonePlan(song, i, melodyStep)
      const uniqueTonePlan = tonePlan.reduce((map, item) => {
        const current = map.get(item.note)
        if (!current || item.intensity > current.intensity)
          map.set(item.note, item)
        return map
      }, new Map<string, AutoPlayTonePlanItem>())

      const planItems = Array.from(uniqueTonePlan.values())
      planItems.forEach(item => pressKey(item.note, AUTO_PLAY_SOURCE, item.intensity))
      const activeAutoNotes = planItems
        .map(item => item.note)
        .filter(noteId => activeButtonIdMap[noteId]?.key === AUTO_PLAY_SOURCE)

      if (activeAutoNotes.length) {
        const releaseAfter = Math.max(80, Math.min(actualDuration, Math.round(actualDuration * autoPlayLegatoRatio.value)))
        await new Promise((resolve) => {
          autoPlayTimeoutId.value = setTimeout(() => {
            activeAutoNotes.forEach(noteId => releaseKey(noteId))
            resolve(null)
          }, releaseAfter)
        })
        autoPlayTimeoutId.value = null
        const restAfterRelease = Math.max(0, actualDuration - releaseAfter)
        if (restAfterRelease > 0) {
          await new Promise((resolve) => {
            autoPlayTimeoutId.value = setTimeout(resolve, restAfterRelease)
          })
          autoPlayTimeoutId.value = null
        }
      }
      else {
        await new Promise((resolve) => {
          autoPlayTimeoutId.value = setTimeout(resolve, actualDuration)
        })
        autoPlayTimeoutId.value = null
      }
      melodyStep += 1
    }
    else {
      await new Promise((resolve) => {
        autoPlayTimeoutId.value = setTimeout(resolve, actualDuration)
      })
      autoPlayTimeoutId.value = null
      // Optionally clear lyric during rests
      // currentLyric.value = "";
    }
  }
  if (isAutoPlaying.value) { // Check if it finished naturally
    isAutoPlaying.value = false
    currentLyric.value = '' // Clear lyric at the end
  }
  scheduleSuspendAudio()
}

// --- Class and Style Bindings (Corrected) ---
const isActive = (noteId: string): boolean => {
  // Check if the noteId exists as a key in the activeButtonIdMap
  return !!activeButtonIdMap[noteId]
}

const isGuideTarget = (noteId: string) => {
  if (phraseTypingMode.value && phraseTrainerMode.value === 'test')
    return false
  return !!(beginnerMode.value && targetLessonNote.value && noteId === targetLessonNote.value)
}

const getHandClass = (noteId: string) => {
  if (!showHandGuide.value)
    return ''
  return getHandForNote(noteId) === 'left' ? ' hand-left' : ' hand-right'
}

const whiteKeyClass = (octaveIndex: number, index: number) => {
  // Find the note ID based on octave and index
  const keyChar = keys[index + octaveIndex * 12]
  const noteId = keyMap[keyChar]
  return `white-key-group key${isActive(noteId) ? ' pressed' : ''}${isGuideTarget(noteId) ? ' guide-target' : ''}${getHandClass(noteId)}`
}
const whiteKeyStyle = (octaveIndex: number, index: number) => {
  return {}
}

const blackKeyClass = (octaveIndex: number, index: number) => {
  // Find the note ID based on octave and index for black keys
  let blackKeyIndexInKeys = -1
  if (index === 0)
    blackKeyIndexInKeys = 7 // C#
  else if (index === 1)
    blackKeyIndexInKeys = 8 // D#
  else if (index === 3)
    blackKeyIndexInKeys = 9 // F#
  else if (index === 4)
    blackKeyIndexInKeys = 10// G#
  else if (index === 5)
    blackKeyIndexInKeys = 11// A#

  if (blackKeyIndexInKeys !== -1) {
    const keyChar = keys[blackKeyIndexInKeys + octaveIndex * 12]
    const noteId = keyMap[keyChar]
    return `black-key-group key${isActive(noteId) ? ' pressed' : ''}${isGuideTarget(noteId) ? ' guide-target' : ''}${getHandClass(noteId)}`
  }
  return 'black-key-group key' // Should not happen if logic is correct
}

const blackKeyStyle = (octaveIndex: number, index: number) => {
  const geometry = pianoGeometry.value
  const whiteKeyWidth = geometry.whiteKeyWidth
  const whiteKeyMargin = geometry.whiteKeyMargin
  const blackKeyWidth = geometry.blackKeyWidth
  const totalWhiteKeyWidth = whiteKeyWidth + whiteKeyMargin

  const baseOffset = totalWhiteKeyWidth * (index + 1) - (blackKeyWidth / 2)
  const octaveOffset = totalWhiteKeyWidth * notes.length * octaveIndex
  const pianoPaddingLeft = geometry.blackLeftInset

  return `left: ${pianoPaddingLeft + octaveOffset + baseOffset}px`
}

// --- Event Handlers ---
const handleInteractionStart = (type: 'white' | 'black', index: number, octaveIndex: number) => {
  interruptAutoPlay() // Interrupt auto-play on user interaction

  let keyChar = ''
  let noteId = ''

  if (type === 'white') {
    keyChar = keys[index + octaveIndex * 12]
  }
  else {
    let blackKeyIndexInKeys = -1
    if (index === 0)
      blackKeyIndexInKeys = 7
    else if (index === 1)
      blackKeyIndexInKeys = 8
    else if (index === 3)
      blackKeyIndexInKeys = 9
    else if (index === 4)
      blackKeyIndexInKeys = 10
    else if (index === 5)
      blackKeyIndexInKeys = 11

    if (blackKeyIndexInKeys !== -1)
      keyChar = keys[blackKeyIndexInKeys + octaveIndex * 12]
  }
  noteId = keyMap[keyChar]
  if (noteId)
    pressKey(noteId, POINTER_SOURCE)
}

const handleInteractionEnd = (type: 'white' | 'black', index: number, octaveIndex: number) => {
  let keyChar = ''
  let noteId = ''
  if (type === 'white') {
    keyChar = keys[index + octaveIndex * 12]
  }
  else {
    let blackKeyIndexInKeys = -1
    if (index === 0)
      blackKeyIndexInKeys = 7
    else if (index === 1)
      blackKeyIndexInKeys = 8
    else if (index === 3)
      blackKeyIndexInKeys = 9
    else if (index === 4)
      blackKeyIndexInKeys = 10
    else if (index === 5)
      blackKeyIndexInKeys = 11

    if (blackKeyIndexInKeys !== -1)
      keyChar = keys[blackKeyIndexInKeys + octaveIndex * 12]
  }
  noteId = keyMap[keyChar]

  // Only release if this key is actually active (prevents releasing due to mouseleave after keyup)
  if (noteId && activeButtonIdMap[noteId])
    releaseKey(noteId)
}

const toggleSongPlayback = async () => {
  if (isAutoPlaying.value || isCountInActive.value) {
    interruptAutoPlay()
    cancelCountIn()
    pushInteractionToast('已停止自动演奏', 'info')
    return
  }
  await playSelectedSongWithCountIn()
}

const handleKeyboardDown = (e: KeyboardEvent) => {
  if (isTypingTarget(e.target))
    return
  if (e.repeat)
    return // Ignore repeats
  if (e.code === 'Space') {
    e.preventDefault()
    void toggleSongPlayback()
    return
  }
  if (e.key === 'Escape') {
    e.preventDefault()
    quickStopAll()
    return
  }
  if (phraseTypingMode.value && beginnerMode.value && e.key === '1') {
    e.preventDefault()
    void startPhraseListen()
    return
  }
  if (phraseTypingMode.value && beginnerMode.value && e.key === '2') {
    e.preventDefault()
    startPhraseFollow()
    return
  }
  if (phraseTypingMode.value && beginnerMode.value && e.key === '3') {
    e.preventDefault()
    startPhraseTest()
    return
  }
  const keyChar = e.key.toLowerCase()
  const noteId = keyMap[keyChar]
  if (noteId) {
    e.preventDefault()
    interruptAutoPlay() // Interrupt on key press
    pressKey(noteId, keyChar)
  }
}

const handleKeyboardUp = (e: KeyboardEvent) => {
  if (isTypingTarget(e.target))
    return
  const keyChar = e.key.toLowerCase()
  const noteId = keyMap[keyChar]
  // Find the noteId that corresponds to this keyChar, as multiple keys might map to the same note
  // However, our current keyMap is one-to-one, so direct lookup is fine.
  // If a note was pressed by this key, release it.
  if (noteId && activeButtonIdMap[noteId] && activeButtonIdMap[noteId].key === keyChar) {
    e.preventDefault()
    releaseKey(noteId)
  }
}

const handleWindowBlur = () => {
  quickStopAll(false)
}

const updateViewportWidth = () => {
  viewportWidth.value = window.innerWidth
}

const handleVisibilityChange = () => {
  if (document.hidden) {
    quickStopAll(false)
  }
}

// --- Lifecycle Hook ---
onMounted(() => {
  initKeys()
  updateViewportWidth()
  loadCustomSongs()
  loadSongLevelProgress()
  if (!allSongs.value.some(item => item.id === selectedSongId.value))
    selectedSongId.value = builtInSongs[0].id
  getSongLevelProgress(selectedSongId.value)
})

// --- Event Listeners ---
useEventListener(window, 'keydown', handleKeyboardDown)
useEventListener(window, 'keyup', handleKeyboardUp)
useEventListener(window, 'blur', handleWindowBlur)
useEventListener(window, 'resize', updateViewportWidth)
useEventListener(window, 'mouseup', releasePointerNotes)
useEventListener(window, 'touchend', releasePointerNotes)
useEventListener(document, 'visibilitychange', handleVisibilityChange)
watch(masterVolume, applyMasterVolume)
watch([beginnerMode, handPracticeMode], ([enabled]) => {
  if (!enabled) {
    lessonFeedback.value = ''
    isLevelChallengeActive.value = false
    closeLevelClearDialog()
    clearRetrainSuggestionState()
    return
  }
  if (isLevelChallengeActive.value)
    return
  resetLesson()
})
watch(selectedSongId, () => {
  if (isAutoPlaying.value)
    interruptAutoPlay()
  isLevelChallengeActive.value = false
  closeLevelClearDialog()
  clearRetrainSuggestionState()
  getSongLevelProgress(selectedSongId.value)
  resetLesson()
})
watch(lessonSourceMode, (mode) => {
  if (isAutoPlaying.value)
    interruptAutoPlay()
  if (isLevelChallengeActive.value && mode === 'song')
    return
  if (mode !== 'song' && isLevelChallengeActive.value) {
    isLevelChallengeActive.value = false
    closeLevelClearDialog()
    clearRetrainSuggestionState()
  }
  resetLesson()
})
watch([phraseTypingMode, phraseChunkSize], () => {
  phraseChunkStart.value = isLevelChallengeActive.value ? currentLevelSegment.value.start : 0
  phraseChunkInputIndex.value = 0
  phraseChunkRound.value = 0
  if (!phraseTypingMode.value) {
    isLevelChallengeActive.value = false
    closeLevelClearDialog()
    clearRetrainSuggestionState()
    phraseTrainerMode.value = 'follow'
    focusTrainingPhase.value = 'warmup'
    resetCurrentChunkTestState()
  }
})
watch(
  [() => levelChallengeScope.value.start, () => levelChallengeScope.value.length],
  ([scopeStart, scopeLength]) => {
    if (!scopeLength) {
      phraseChunkStart.value = 0
      phraseChunkInputIndex.value = 0
      return
    }
    const scopeOffset = toPositiveModulo(phraseChunkStart.value - scopeStart, scopeLength)
    const normalizedStart = scopeStart + scopeOffset
    if (phraseChunkStart.value !== normalizedStart)
      phraseChunkStart.value = normalizedStart
    if (phraseChunkInputIndex.value >= phraseChunkNotes.value.length)
      phraseChunkInputIndex.value = 0
  },
)
watch(customSongs, saveCustomSongs, { deep: true })
watch(songLevelProgress, saveSongLevelProgress, { deep: true })
watch(practiceSpeed, () => {
  adaptiveSpeedDelta.value = 0
  wrongStreak.value = 0
  correctStreak.value = 0
})
watch(adaptiveTrainingEnabled, (enabled) => {
  if (enabled)
    return
  adaptiveSpeedDelta.value = 0
  wrongStreak.value = 0
  correctStreak.value = 0
})
watch([metronomeEnabled, metronomeBpm, beatsPerMeasure, effectivePracticeSpeed], ([enabled]) => {
  if (!enabled) {
    stopMetronome()
    return
  }
  startMetronome()
})

const playSelectedSongWithCountIn = async () => {
  interruptAutoPlay()
  initAudio()
  if (!audio) {
    pushInteractionToast('音频初始化失败，请刷新后重试', 'error')
    return
  }
  await warmupPianoSamples()
  if (pianoSampleLoadedCount.value === 0)
    pushInteractionToast('钢琴采样未就绪，当前会回退为合成音色', 'info', 2800)
  else if (pianoSampleLoadState.value !== 'ready')
    pushInteractionToast(`钢琴采样部分可用（${pianoSampleLoadedCount.value}/${pianoSampleKeys.length}）`, 'info', 2200)
  const canStart = await runCountIn()
  if (!canStart) {
    pushInteractionToast('已取消起拍', 'info')
    return
  }
  playSong(selectedSong.value.notes)
  pushInteractionToast(`开始演奏《${selectedSong.value.title}》`, 'info')
}

const handleStartClick = () => {
  showPiano.value = true
  initAudio()
  void loadPianoSamples()
  // Start playing after a short delay to allow UI rendering and audio setup
  setTimeout(async () => {
    await playSelectedSongWithCountIn()
  }, 500)
}

onBeforeUnmount(() => {
  quickStopAll(false)
  clearInteractionToast()
  clearSuspendAudioTask()
  if (audio && audio.state !== 'closed')
    audio.close()
})
</script>

<template>
  <div class="piano-container" :class="{ 'focus-layout': isFocusLayout && showPiano, 'perform-layout': uiMode === 'perform' && !isFocusLayout && showPiano }">
    <div class="bg-orb bg-orb-left" />
    <div class="bg-orb bg-orb-right" />

    <!-- Start Screen -->
    <div v-if="!showPiano" class="start-screen">
      <p class="start-eyebrow">
        PINAO STUDIO
      </p>
      <h1>在线钢琴</h1>
      <p>点击开始，体验选曲演奏与分段练习</p>
      <button class="start-button" @click="handleStartClick">
        开始弹奏
      </button>
    </div>

    <!-- Piano and Indicator (shown after start) -->
    <template v-if="showPiano">
      <div v-if="!isFocusLayout" class="top-panel">
        <h1 class="main-title">
          在线钢琴
        </h1>
        <div class="top-right">
          <div class="mode-tag" :class="{ active: isAutoPlaying }">
            {{ isAutoPlaying ? `自动演奏《${selectedSong.title}》中` : '自由弹奏模式' }}
          </div>
          <div class="control-panel">
            <label class="control-item volume-item">
              <span>音量 {{ Math.round(masterVolume * 100) }}%</span>
              <input
                v-model.number="masterVolume"
                class="volume-slider"
                type="range"
                min="0"
                max="0.16"
                step="0.005"
                @input="applyMasterVolume"
              >
            </label>
            <label class="control-item tone-item">
              <span>音色</span>
              <select v-model="toneMode" class="tone-select">
                <option v-for="option in toneModeOptions" :key="option.value" :value="option.value">
                  {{ option.label }}
                </option>
              </select>
              <small class="tone-status">{{ pianoSampleStatusLabel }}</small>
            </label>
          </div>
        </div>
      </div>

      <div class="interaction-bar">
        <div class="layout-switch">
          <button
            class="layout-switch-btn"
            :class="{ active: isFocusLayout }"
            type="button"
            @click="setPracticeLayoutMode('focus')"
          >
            专注练习
          </button>
          <button
            class="layout-switch-btn"
            :class="{ active: !isFocusLayout }"
            type="button"
            @click="setPracticeLayoutMode('full')"
          >
            完整布局
          </button>
        </div>
        <div v-if="!isFocusLayout" class="view-switch">
          <button
            class="view-switch-btn"
            :class="{ active: uiMode === 'learn' }"
            type="button"
            @click="setUiMode('learn')"
          >
            学习视图
          </button>
          <button
            class="view-switch-btn"
            :class="{ active: uiMode === 'perform' }"
            type="button"
            @click="setUiMode('perform')"
          >
            演奏视图
          </button>
        </div>
        <div class="quick-actions">
          <button class="quick-action" type="button" @click="toggleSongPlayback">
            {{ isAutoPlaying || isCountInActive ? '停止自动演奏' : '播放当前曲目' }} (Space)
          </button>
          <button v-if="uiMode !== 'perform'" class="quick-action" type="button" :disabled="!phraseChunkNotes.length" @click="playCurrentPhraseChunk">
            播放当前短句
          </button>
          <button v-if="uiMode === 'perform' && !isFocusLayout" class="quick-action" type="button" @click="toggleSongWorkbench">
            {{ isSongPanelVisible ? '收起选曲面板' : '选曲面板' }}
          </button>
          <button v-if="isFocusLayout" class="quick-action" type="button" @click="toggleSongWorkbench">
            {{ isSongPanelVisible ? '收起选曲区' : '选曲与关卡' }}
          </button>
          <button v-if="isFocusLayout" class="quick-action" type="button" @click="toggleAdvancedCoachPanel">
            {{ showAdvancedCoach ? '收起高级设置' : '展开高级设置' }}
          </button>
          <button class="quick-action danger" type="button" @click="quickStopAll()">
            紧急停止 (Esc)
          </button>
        </div>
      </div>

      <Transition name="toast-fade">
        <div v-if="interactionToast" class="interaction-toast" :class="interactionToastType">
          {{ interactionToast }}
        </div>
      </Transition>

      <div v-show="isSongPanelVisible" class="song-panel" :class="{ focus: isFocusLayout, perform: uiMode === 'perform' && !isFocusLayout }">
        <div class="panel-head">
          <div class="panel-title-wrap">
            <p class="panel-title">
              {{ isFocusLayout ? '选曲与关卡' : '曲目与导入' }}
            </p>
            <p class="panel-subtitle">
              {{ isFocusLayout ? '专注练习下仅保留选曲和关卡' : '选曲、导出、上传自定义曲谱' }}
            </p>
          </div>
          <button class="panel-toggle" type="button" @click="toggleSongWorkbench">
            {{ isSongPanelBodyVisible ? '收起' : '展开' }}
          </button>
        </div>
        <Transition name="panel-drop">
          <div v-show="isSongPanelBodyVisible" class="panel-body">
            <div class="song-row">
              <label class="song-select">
                <span>曲目</span>
                <select v-model="selectedSongId">
                  <option v-for="song in allSongs" :key="song.id" :value="song.id">
                    {{ song.title }} - {{ song.artist }} {{ song.source === 'custom' ? '(自定义)' : '' }}
                  </option>
                </select>
              </label>
              <button class="song-action" type="button" @click="playSelectedSongWithCountIn">
                播放选中
              </button>
              <button class="song-action" type="button" @click="exportSelectedSongJson">
                导出当前 JSON
              </button>
              <button
                v-if="selectedSong.source === 'custom'"
                class="song-action song-action-danger"
                type="button"
                @click="removeSelectedCustomSong"
              >
                删除自定义
              </button>
            </div>
            <div class="song-row song-row-meta">
              <label class="song-select">
                <span>学习素材</span>
                <select v-model="lessonSourceMode">
                  <option value="basic">
                    基础音阶
                  </option>
                  <option value="song">
                    当前歌曲可弹音符
                  </option>
                </select>
              </label>
              <span class="song-meta">
                当前曲目音符 {{ selectedSong.notes.length }} 个 · 可练习 {{ selectedSongPracticeNotes.length }} 个 · 音域 {{ selectedSongRangeLabel }} · 时长 {{ selectedSongDurationLabel }}
              </span>
            </div>
            <div class="level-card">
              <div class="level-card-head">
                <span class="level-title">闯关进度</span>
                <span class="level-badge">Level {{ currentSongLevel }} / {{ levelRules.length }}</span>
              </div>
              <div class="level-track">
                <span class="level-track-fill" :style="{ width: `${levelProgressPercent}%` }" />
              </div>
              <p class="level-rule">
                当前关要求：{{ currentLevelRuleText }}
              </p>
              <p class="level-segment">
                本关固定段落：{{ currentLevelSegmentLabel }} · 共 {{ currentLevelSegment.length }} 音
              </p>
              <p class="level-status" :class="{ pass: currentLevelRuleMet, active: isLevelChallengeActive }">
                当前状态：{{ levelChallengeStatusLabel }} · 已通过 {{ passedTestCount }} 次测试
              </p>
              <div class="level-actions">
                <button class="song-action" type="button" @click="startCurrentLevelChallenge">
                  {{ isLevelChallengeActive ? '重开本关挑战' : '开始本关挑战' }}
                </button>
                <button class="song-action" type="button" @click="resetCurrentSongLevelProgress">
                  重置本曲关卡
                </button>
              </div>
            </div>
            <details v-if="!isFocusLayout" class="song-import">
              <summary>导入自定义曲谱（JSON / TXT / MIDI）</summary>
              <p class="song-import-tip">
                支持 JSON：`{ "title":"...", "artist":"...", "notes":[{"note":"C4","notes":["C4","E4","G4"],"duration":350}] }`，支持简写文本：`C4 350, D4 350, E4 700`，也支持上传 `.mid/.midi` 自动提取多音轨和弦。
              </p>
              <div class="song-import-meta">
                <input
                  v-model="customSongTitleInput"
                  class="song-import-meta-input"
                  type="text"
                  maxlength="40"
                  placeholder="可选：覆盖导入标题"
                >
                <input
                  v-model="customSongArtistInput"
                  class="song-import-meta-input"
                  type="text"
                  maxlength="40"
                  placeholder="可选：覆盖导入作者"
                >
              </div>
              <textarea
                v-model="customSongJsonInput"
                class="song-import-input"
                rows="5"
                placeholder="粘贴 JSON 或简写文本曲谱"
              />
              <div class="song-import-actions">
                <button class="song-action" type="button" @click="importSongFromJsonText">
                  从文本导入
                </button>
                <button class="song-action" type="button" @click="openCustomSongFilePicker">
                  上传 .json / .txt / .mid
                </button>
                <button class="song-action" type="button" @click="copySongTemplate">
                  复制导入模板
                </button>
                <input
                  ref="customSongFileInputRef"
                  type="file"
                  accept=".json,.txt,.mid,.midi,application/json,text/plain,audio/midi,audio/x-midi"
                  class="song-import-file"
                  @change="importSongFromFile"
                >
              </div>
              <p v-if="customSongImportStatus" class="song-import-status" :class="customSongImportStatusType">
                {{ customSongImportStatus }}
              </p>
            </details>
            <p v-else class="song-focus-tip">
              专注练习中已隐藏导入模块。切换到“完整布局”可上传/编辑自定义曲谱。
            </p>
          </div>
        </Transition>
      </div>

      <div class="coach-panel" :class="{ focus: isFocusLayout }">
        <div class="panel-head">
          <div class="panel-title-wrap">
            <p class="panel-title">
              新手引导
            </p>
            <p class="panel-subtitle">
              {{ isFocusLayout ? '专注当前短句，只保留必要训练信息' : '分段跟弹、节拍控制和双手引导' }}
            </p>
          </div>
          <div class="panel-head-actions">
            <button v-if="isFocusLayout" class="coach-reset" type="button" @click="toggleAdvancedCoachPanel">
              {{ showAdvancedCoach ? '收起高级' : '高级设置' }}
            </button>
            <button class="coach-reset" type="button" @click="resetLesson">
              重置练习
            </button>
            <button class="panel-toggle" type="button" @click="showCoachPanel = !showCoachPanel">
              {{ showCoachPanel ? '收起' : '展开' }}
            </button>
          </div>
        </div>
        <Transition name="panel-drop">
          <div v-show="showCoachPanel" class="panel-body">
            <div v-if="beginnerMode && phraseTypingMode && !isFocusLayout" class="learning-flow">
              <span
                v-for="(step, index) in phraseFlowSteps"
                :key="step"
                class="learning-flow-step"
                :class="{ active: phraseFlowActiveStep === index, done: phraseFlowActiveStep > index }"
              >
                {{ index + 1 }}. {{ step }}
              </span>
            </div>
            <div v-if="beginnerMode && phraseTypingMode && !isFocusLayout" class="phrase-trainer-switch">
              <button
                class="phrase-mode-btn"
                :class="{ active: phraseTrainerMode === 'listen' }"
                type="button"
                @click="startPhraseListen"
              >
                1 听句
              </button>
              <button
                class="phrase-mode-btn"
                :class="{ active: phraseTrainerMode === 'follow' }"
                type="button"
                @click="startPhraseFollow"
              >
                2 跟弹
              </button>
              <button
                class="phrase-mode-btn"
                :class="{ active: phraseTrainerMode === 'test' }"
                type="button"
                @click="startPhraseTest"
              >
                3 测试
              </button>
            </div>
            <div v-if="beginnerMode && phraseTypingMode && isFocusLayout" class="focus-phase-board">
              <span class="focus-phase-title">专注流程</span>
              <div class="focus-phase-steps">
                <button
                  v-for="(step, index) in focusPhaseSteps"
                  :key="`focus-${step.value}`"
                  class="focus-phase-step"
                  :class="{ active: focusTrainingPhase === step.value, done: focusPhaseActiveIndex > index }"
                  type="button"
                  @click="setFocusTrainingPhase(step.value)"
                >
                  <span>{{ index + 1 }} · {{ step.label }}</span>
                  <small>{{ step.desc }}</small>
                </button>
              </div>
              <p class="focus-phase-hint">
                {{ focusPhaseHint }}
              </p>
            </div>
            <div class="coach-row coach-row-main">
              <label class="coach-switch">
                <input v-model="beginnerMode" type="checkbox">
                <span>学习模式</span>
              </label>
              <label class="coach-switch">
                <input v-model="phraseTypingMode" type="checkbox">
                <span>拼音式分段</span>
              </label>
              <label class="coach-select">
                <span>每段</span>
                <select v-model.number="phraseChunkSize" :disabled="!phraseTypingMode">
                  <option v-for="option in phraseChunkSizeOptions" :key="option.value" :value="option.value">
                    {{ option.label }}
                  </option>
                </select>
              </label>
              <label class="coach-select">
                <span>速度</span>
                <select v-model.number="practiceSpeed">
                  <option v-for="option in speedOptions" :key="option.value" :value="option.value">
                    {{ option.label }}
                  </option>
                </select>
              </label>
              <template v-if="showAdvancedCoach">
                <label class="coach-switch">
                  <input v-model="showHandGuide" type="checkbox">
                  <span>左右手引导</span>
                </label>
                <label class="coach-switch">
                  <input v-model="showNoteLabels" type="checkbox">
                  <span>显示音名</span>
                </label>
                <label class="coach-switch">
                  <input v-model="showKeyLabels" type="checkbox">
                  <span>显示键位</span>
                </label>
                <label class="coach-select">
                  <span>分练</span>
                  <select v-model="handPracticeMode">
                    <option value="both">
                      双手
                    </option>
                    <option value="left">
                      左手
                    </option>
                    <option value="right">
                      右手
                    </option>
                  </select>
                </label>
                <label class="coach-switch">
                  <input v-model="adaptiveTrainingEnabled" type="checkbox">
                  <span>自适应速度</span>
                </label>
                <div class="coach-adaptive" :class="{ active: adaptiveTrainingEnabled }">
                  当前训练速度 {{ effectivePracticeSpeedLabel }}
                </div>
                <label class="coach-switch">
                  <input v-model="countInEnabled" type="checkbox">
                  <span>起拍 Count-in</span>
                </label>
                <label class="coach-select">
                  <span>起拍长度</span>
                  <select v-model.number="countInMeasures">
                    <option v-for="option in countInMeasureOptions" :key="option.value" :value="option.value">
                      {{ option.label }}
                    </option>
                  </select>
                </label>
                <label class="coach-switch">
                  <input v-model="metronomeEnabled" type="checkbox">
                  <span>节拍器</span>
                </label>
                <label class="coach-select">
                  <span>拍号</span>
                  <select v-model.number="beatsPerMeasure">
                    <option v-for="option in timeSignatureOptions" :key="option.value" :value="option.value">
                      {{ option.label }}
                    </option>
                  </select>
                </label>
                <label class="coach-bpm">
                  <span>BPM {{ metronomeBpm }}</span>
                  <input
                    v-model.number="metronomeBpm"
                    type="range"
                    min="40"
                    max="180"
                    step="1"
                    :disabled="!metronomeEnabled"
                  >
                </label>
                <div class="metronome-visual" :class="{ active: metronomeEnabled, counting: isCountInActive }">
                  <span class="metronome-visual-label">
                    <template v-if="isCountInActive">
                      Count-in {{ countInCurrentBeat }}/{{ countInTotalBeats }}
                    </template>
                    <template v-else>
                      第 {{ metronomeCurrentBeat }} 拍 / 共 {{ beatsPerMeasure }} 拍 · 实际 {{ effectiveMetronomeBpm }} BPM
                    </template>
                  </span>
                  <div class="metronome-dots">
                    <span
                      v-for="dot in metronomeBeatDots"
                      :key="dot"
                      class="metronome-dot"
                      :class="{ active: metronomeEnabled && metronomeCurrentBeat === dot + 1, counting: isCountInActive }"
                    />
                  </div>
                </div>
              </template>
              <div v-else-if="isFocusLayout" class="focus-layout-tip">
                专注布局已隐藏高级设置，点击“高级设置”可展开节拍器/分手练习等选项。
              </div>
              <div class="coach-stat" :class="lessonFeedback">
                <span v-if="beginnerMode && phraseTypingMode && phraseTrainerMode === 'test'">测试模式：隐藏目标键，专注听感与手感</span>
                <span v-else-if="beginnerMode">下一键 {{ targetLessonNote || '--' }} ({{ targetLessonKey || '-' }}) · {{ targetLessonHand }}</span>
                <span v-else>学习模式已关闭，可自由弹奏</span>
              </div>
              <div class="coach-score-wrap">
                <div class="coach-score">
                  进度 {{ lessonProgress }} · 对 {{ lessonCorrect }} · 错 {{ lessonWrong }} · 准确率 {{ lessonAccuracy }} · 目标速度 {{ Math.round(practiceSpeed * 100) }}%
                </div>
                <button class="coach-mini-action" type="button" @click="openPracticeSummary()">
                  查看复盘
                </button>
              </div>
            </div>
            <div v-if="beginnerMode && phraseTypingMode" class="phrase-typing-strip">
              <div class="phrase-typing-head">
                <span class="phrase-typing-label">短句 {{ phraseChunkRangeLabel }}</span>
                <button
                  v-if="!isFocusLayout"
                  class="phrase-play-btn"
                  type="button"
                  @click="startPhraseListen"
                >
                  播放当前短句
                </button>
                <button
                  v-else-if="focusTrainingPhase === 'warmup'"
                  class="phrase-play-btn"
                  type="button"
                  @click="void startFocusWarmup()"
                >
                  开始热身（听2遍）
                </button>
                <button
                  v-else-if="focusTrainingPhase === 'phrase'"
                  class="phrase-play-btn"
                  type="button"
                  @click="startFocusTestStage"
                >
                  进入测试
                </button>
                <button
                  v-else
                  class="phrase-play-btn"
                  type="button"
                  @click="startFocusPhraseStage"
                >
                  回到分句
                </button>
              </div>
              <div class="phrase-progress">
                <span class="phrase-progress-label">完成度 {{ phraseChunkProgress }}%</span>
                <div class="phrase-progress-track">
                  <span class="phrase-progress-fill" :style="{ width: `${phraseChunkProgress}%` }" />
                </div>
              </div>
              <div v-if="!isFocusLayout || focusTrainingPhase !== 'warmup'" class="phrase-rhythm-lane">
                <span class="phrase-rhythm-label">拼句轨道（每 {{ phraseTypingGroupSize }} 音一组）</span>
                <div class="phrase-rhythm-track">
                  <span
                    v-for="node in phraseRhythmNodes"
                    :key="`rhythm-${node.index}-${phraseChunkStart}`"
                    class="phrase-rhythm-node"
                    :class="[node.phase, { masked: !node.visible, accent: node.accent }]"
                  />
                </div>
              </div>
              <div v-if="isFocusLayout" class="focus-phase-tip">
                {{ focusPhaseHint }}
              </div>
              <div v-if="phraseTrainerMode === 'test' && (!isFocusLayout || focusTrainingPhase === 'test')" class="test-strip">
                <div class="test-lives">
                  <span class="test-label">测试机会</span>
                  <span
                    v-for="dot in testLivesDots"
                    :key="`life-${dot}`"
                    class="test-life-dot"
                    :class="{ active: dot < testLives }"
                  />
                  <span class="test-grade">
                    最近评级 {{ currentChunkTestGrade || '--' }}
                  </span>
                </div>
                <button class="coach-mini-action" type="button" @click="restartCurrentChunkTest">
                  重测本段
                </button>
              </div>
              <div v-if="showRetrainSuggestion && (!isFocusLayout || focusTrainingPhase !== 'warmup')" class="retrain-banner">
                <span class="retrain-banner-text">{{ retrainSuggestionText }}</span>
                <div class="retrain-banner-actions">
                  <button class="coach-mini-action" type="button" @click="void applyRetrainSuggestion()">
                    按建议复训
                  </button>
                  <button class="coach-mini-action ghost" type="button" @click="dismissRetrainSuggestion">
                    先保持当前速度
                  </button>
                </div>
              </div>
              <div v-if="!isFocusLayout || focusTrainingPhase === 'test'" class="grade-board">
                <span class="grade-board-label">测试看板</span>
                <span class="grade-chip a">A {{ testGradeStats.A }}</span>
                <span class="grade-chip b">B {{ testGradeStats.B }}</span>
                <span class="grade-chip c">C {{ testGradeStats.C }}</span>
                <span class="grade-chip f">F {{ testGradeStats.F }}</span>
              </div>
              <div v-if="!isFocusLayout || focusTrainingPhase === 'test'" class="challenge-board">
                <span class="challenge-board-title">关卡 {{ currentSongLevel }}/{{ levelRules.length }}</span>
                <span class="challenge-board-rule">{{ currentLevelRuleText }}</span>
                <span class="challenge-board-segment">段落 {{ currentLevelSegmentLabel }} · {{ levelChallengeStatusLabel }}</span>
                <span class="challenge-board-rule" :class="{ pass: currentLevelRuleMet }">{{ currentLevelRuleMet ? '本关条件已满足，可等待结算' : '继续累计测试成绩以满足本关条件' }}</span>
              </div>
              <div v-if="!isFocusLayout || focusTrainingPhase !== 'warmup'" class="insight-board">
                <span class="insight-board-title">错音热力图与指法建议</span>
                <div class="heat-chip-row">
                  <span v-if="!weakNotes.length" class="heat-chip empty">当前轮次暂无明显错音热点</span>
                  <template v-else>
                    <span
                      v-for="item in weakNotes"
                      :key="`heat-${item.note}`"
                      class="heat-chip"
                      :class="item.wrong >= 3 ? 'hot' : item.wrong === 2 ? 'warm' : 'cool'"
                    >
                      {{ item.note }} 错{{ item.wrong }} · 准{{ item.accuracy }}%
                    </span>
                  </template>
                </div>
                <span class="insight-line">{{ handFocusLabel }}</span>
                <span class="insight-line">{{ fingeringTip }}</span>
              </div>
              <div v-if="!isFocusLayout || focusTrainingPhase !== 'warmup'" class="phrase-typing-notes">
                <span
                  v-for="node in phraseRhythmNodes"
                  :key="`${node.note}-${node.index}-${phraseChunkStart}`"
                  class="phrase-typing-note"
                  :class="[getPhraseNoteStateClass(node.index), getPhraseTypingMaskClass(node.index)]"
                >
                  {{ getPhraseTypingDisplayNote(node.note, node.index) }}
                </span>
              </div>
            </div>
          </div>
        </Transition>
      </div>

      <div class="piano-stage" :class="{ focus: isFocusLayout, perform: uiMode === 'perform' && !isFocusLayout }">
        <div class="piano" :style="pianoStyleVars">
          <template v-for="(_, octaveIndex) in octaves" :key="octaveIndex">
            <!-- White Keys -->
            <template v-for="(_, index) in notes" :key="`w-${octaveIndex}-${index}`">
              <div
                :class="whiteKeyClass(octaveIndex, index)"
                :style="whiteKeyStyle(octaveIndex, index)"
                @mousedown="handleInteractionStart('white', index, octaveIndex)"
                @mouseup="handleInteractionEnd('white', index, octaveIndex)"
                @touchstart.prevent="handleInteractionStart('white', index, octaveIndex)"
                @touchend.prevent="handleInteractionEnd('white', index, octaveIndex)"
              >
                <span v-if="showNoteLabels" class="f-notes">{{ notes[index].toUpperCase() }}{{ octaveIndex + 3 }}</span>
                <span v-if="showKeyLabels" class="f-keymap">{{ keys[index + octaveIndex * 12]?.toUpperCase() }}</span>
              </div>
            </template>
            <!-- Black Keys -->
            <template v-for="(_, index) in notes" :key="`b-${octaveIndex}-${index}`">
              <div
                v-if="index !== 2 && index !== 6"
                :class="blackKeyClass(octaveIndex, index)"
                :style="blackKeyStyle(octaveIndex, index)"
                @mousedown="handleInteractionStart('black', index, octaveIndex)"
                @mouseup="handleInteractionEnd('black', index, octaveIndex)"
                @touchstart.prevent="handleInteractionStart('black', index, octaveIndex)"
                @touchend.prevent="handleInteractionEnd('black', index, octaveIndex)"
              >
                <span v-if="showNoteLabels" class="f-notes">{{ notes[index].toUpperCase() }}#{{ octaveIndex + 3 }}</span>
                <span v-if="showKeyLabels" class="f-keymap">{{ keys[index + (index === 0 || index === 1 ? 7 : 6) + octaveIndex * 12]?.toUpperCase() }}</span>
              </div>
            </template>
          </template>
        </div>
      </div>

      <!-- Lyric Display -->
      <div v-if="!isFocusLayout && uiMode !== 'perform'" class="lyric-display">
        {{ currentLyric || ' ' }}
      </div>

      <div v-if="isAutoPlaying" class="autoplay-indicator" :class="{ focus: isFocusLayout }">
        Playing "{{ selectedSong.title }}"... Click or press a key to stop.
      </div>
      <div v-else-if="isCountInActive" class="autoplay-indicator" :class="{ focus: isFocusLayout }">
        Count-in 准备中... {{ countInCurrentBeat }}/{{ countInTotalBeats }}
      </div>
      <div v-else class="autoplay-indicator" :class="{ focus: isFocusLayout }">
        自由弹奏中...
      </div>
      <div v-if="!isFocusLayout && uiMode !== 'perform'" class="shortcut-hint">
        {{ isFocusLayout
          ? '专注布局快捷键：Space 播放/停止 · Esc 紧急停止 · 1/2/3 切换听句跟弹测试'
          : '快捷键：Space 播放/停止自动演奏 · Esc 紧急停止 · 1/2/3 听句/跟弹/测试 · A~P 键盘弹奏' }}
      </div>

      <Transition name="summary-fade">
        <div v-if="showPracticeSummary && practiceSummary" class="summary-mask" @click.self="closePracticeSummary">
          <div class="summary-card">
            <h3>本轮练习复盘</h3>
            <p class="summary-subtitle">
              用数据看状态，再决定是提速还是回到听句
            </p>
            <div class="summary-grid">
              <div class="summary-item">
                <span class="summary-label">练习时长</span>
                <strong>{{ practiceSummary.durationLabel }}</strong>
              </div>
              <div class="summary-item">
                <span class="summary-label">准确率</span>
                <strong>{{ practiceSummary.accuracyLabel }}</strong>
              </div>
              <div class="summary-item">
                <span class="summary-label">已完成段数</span>
                <strong>{{ practiceSummary.rounds }}</strong>
              </div>
              <div class="summary-item">
                <span class="summary-label">速度（音/分）</span>
                <strong>{{ practiceSummary.notesPerMinute }}</strong>
              </div>
            </div>
            <p class="summary-grade-overview">
              测试评级分布：{{ practiceSummary.testGradeOverview }}
            </p>
            <p class="summary-level-overview">
              当前闯关：Level {{ currentSongLevel }} / {{ levelRules.length }} · 段落 {{ currentLevelSegmentLabel }} · {{ isLevelChallengeActive ? '进行中' : '待开始' }}
            </p>
            <p class="summary-insight">
              错音热点：{{ practiceSummary.weakNotesLabel }}
            </p>
            <p class="summary-insight">
              手部提示：{{ practiceSummary.handFocusLabel }}
            </p>
            <p class="summary-note">
              {{ practiceSummary.recommendation }}
            </p>
            <p class="summary-note soft">
              {{ practiceSummary.fingeringTip }}
            </p>
            <div class="summary-actions">
              <button class="summary-btn ghost" type="button" @click="closePracticeSummary">
                继续练习
              </button>
              <button class="summary-btn" type="button" @click="startPhraseTest(); closePracticeSummary()">
                进入测试
              </button>
            </div>
          </div>
        </div>
      </Transition>
      <Transition name="summary-fade">
        <div v-if="levelClearState" class="summary-mask" @click.self="closeLevelClearDialog">
          <div class="summary-card level-clear-card">
            <h3>Level {{ levelClearState.clearedLevel }} 通关</h3>
            <p class="summary-subtitle">
              曲目《{{ levelClearState.songTitle }}》已完成当前关卡，是否进入下一关？
            </p>
            <p class="summary-level-overview">
              下一目标：{{ levelClearState.unlockedLevel === null ? '已完成全部关卡' : 'Level ' + levelClearState.unlockedLevel }}
            </p>
            <div class="summary-actions">
              <button class="summary-btn ghost" type="button" @click="closeLevelClearDialog">
                先留在当前页面
              </button>
              <button class="summary-btn" type="button" @click="enterUnlockedLevel">
                {{ levelClearState.unlockedLevel === null ? '回到自由练习' : '进入下一关' }}
              </button>
            </div>
          </div>
        </div>
      </Transition>
    </template>
  </div>
</template>

<style lang="scss" scoped>
:global(body) {
  margin: 0;
}

.piano-container {
  --panel-bg: rgba(17, 25, 40, 0.74);
  --panel-border: rgba(255, 255, 255, 0.16);
  --accent: #31c2ff;
  --accent-soft: #85dbff;
  --text-main: #e9f2ff;
  --text-muted: #97abc7;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  width: 100%;
  min-height: 100vh;
  background:
    radial-gradient(circle at 18% 16%, rgba(61, 146, 255, 0.26), transparent 36%),
    radial-gradient(circle at 82% 84%, rgba(49, 194, 255, 0.2), transparent 32%),
    linear-gradient(145deg, #09121f 0%, #111d31 42%, #050b15 100%);
  padding: 20px;
  box-sizing: border-box;
  overflow: hidden;
  font-family: 'Avenir Next', 'PingFang SC', 'Hiragino Sans GB', 'Microsoft Yahei', sans-serif;
  position: relative;
}

.piano-container.focus-layout {
  min-height: 100dvh;
  height: 100dvh;
  justify-content: flex-start;
  padding: 10px 12px;
  overflow: hidden;
}

.piano-container.perform-layout {
  min-height: 100dvh;
  height: 100dvh;
  justify-content: flex-start;
  padding: 10px 12px;
  overflow: hidden;
}

.bg-orb {
  position: absolute;
  width: 360px;
  height: 360px;
  border-radius: 50%;
  filter: blur(32px);
  pointer-events: none;
  opacity: 0.46;
}

.bg-orb-left {
  left: -120px;
  top: -120px;
  background: radial-gradient(circle, rgba(57, 173, 255, 0.55) 0%, rgba(57, 173, 255, 0) 72%);
}

.bg-orb-right {
  right: -140px;
  bottom: -140px;
  background: radial-gradient(circle, rgba(0, 255, 194, 0.35) 0%, rgba(0, 255, 194, 0) 70%);
}

.piano {
  position: relative;
  display: flex;
  border: 1px solid #25180e;
  background:
    linear-gradient(180deg, #7a5336 0%, #5f412b 40%, #3a2719 100%);
  padding: var(--piano-top-pad, 76px) var(--piano-side-pad, 16px) var(--piano-bottom-pad, 16px);
  border-radius: 18px;
  box-shadow:
    0 30px 60px rgba(2, 6, 14, 0.55),
    inset 0 1px 0 rgba(255, 240, 220, 0.25),
    inset 0 -8px 16px rgba(20, 9, 2, 0.42);
  height: auto;
  width: auto;
  user-select: none;
  margin: 0 auto;
  overflow: hidden;
}

.piano::before {
  content: '';
  position: absolute;
  top: 14px;
  left: 12px;
  right: 12px;
  height: 46px;
  border-radius: 10px;
  background: linear-gradient(180deg, #1e120a 0%, #0f0905 100%);
  box-shadow:
    inset 0 2px 2px rgba(255, 219, 180, 0.16),
    inset 0 -2px 3px rgba(0, 0, 0, 0.8);
}

.piano::after {
  content: 'PINAO · LEARNER EDITION';
  position: absolute;
  top: 30px;
  left: 50%;
  transform: translateX(-50%);
  color: #d7b289;
  font-size: 10px;
  letter-spacing: 0.17em;
  font-weight: 700;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.7);
}

.key {
  position: relative;
  cursor: pointer;
  transition: background 0.16s ease, transform 0.11s ease-out, box-shadow 0.16s ease;
  box-sizing: border-box;
}

.white-key-group {
  width: var(--white-key-width, 58px);
  height: var(--white-key-height, 220px);
  background:
    linear-gradient(180deg, #fffdf8 0%, #f7f2e7 68%, #e5dcca 100%);
  border: 1px solid #b9ac94;
  border-top: none;
  border-radius: 0 0 12px 12px;
  box-shadow:
    inset 0 -6px 10px rgba(90, 70, 36, 0.2),
    0 8px 12px rgba(4, 11, 22, 0.25);
  margin-right: var(--white-key-margin, 2px);
  z-index: 1;

  &:last-child {
    margin-right: 0;
  }

  &.pressed {
    background:
      linear-gradient(180deg, #f2ebdc 0%, #e9decb 70%, #d6c8af 100%);
    transform: translateY(5px);
    box-shadow:
      inset 0 -2px 4px rgba(45, 30, 10, 0.3),
      0 2px 3px rgba(5, 10, 22, 0.24),
      0 0 14px rgba(236, 190, 115, 0.5);
    border-color: #ab9c84;
  }
}

.black-key-group {
  position: absolute;
  width: var(--black-key-width, 34px);
  height: var(--black-key-height, 142px);
  background: linear-gradient(180deg, #353535 0%, #0d0d10 100%);
  border: 1px solid #000;
  border-top: none;
  border-radius: 0 0 9px 9px;
  box-shadow:
    inset 0 -5px 7px rgba(255, 255, 255, 0.12),
    0 7px 11px rgba(1, 3, 7, 0.58);
  z-index: 2;
  top: var(--black-key-top, 76px);

  &.pressed {
    background: linear-gradient(180deg, #252525 0%, #050507 100%);
    transform: translateY(4px);
    box-shadow:
      inset 0 -2px 4px rgba(255, 255, 255, 0.08),
      0 3px 4px rgba(0, 0, 0, 0.5),
      0 0 16px rgba(236, 190, 115, 0.62);
    border-color: #000;
  }
}

.white-key-group.hand-left::before,
.white-key-group.hand-right::before {
  content: '';
  position: absolute;
  left: 2px;
  right: 2px;
  top: 0;
  height: 5px;
  border-radius: 0 0 6px 6px;
  opacity: 0.6;
}

.white-key-group.hand-left::before {
  background: linear-gradient(180deg, rgba(86, 157, 255, 0.8) 0%, rgba(86, 157, 255, 0) 100%);
}

.white-key-group.hand-right::before {
  background: linear-gradient(180deg, rgba(255, 151, 88, 0.8) 0%, rgba(255, 151, 88, 0) 100%);
}

.black-key-group.hand-left::after,
.black-key-group.hand-right::after {
  content: '';
  position: absolute;
  top: 4px;
  left: 8px;
  right: 8px;
  height: 4px;
  border-radius: 999px;
  opacity: 0.72;
}

.black-key-group.hand-left::after {
  background: rgba(108, 178, 255, 0.86);
}

.black-key-group.hand-right::after {
  background: rgba(255, 170, 112, 0.9);
}

.f-notes, .f-keymap {
  position: absolute;
  width: 100%;
  text-align: center;
  font-weight: 600;
  font-size: 11px;
  color: #4b5d73;
  pointer-events: none;
  left: 0;
  bottom: 12px;
  transition: color 0.12s ease;
  letter-spacing: 0.03em;
}

.f-keymap {
  bottom: 28px;
  color: #6f839b;
  font-size: 10px;
}

.black-key-group .f-notes {
  color: #d7e8ff;
  bottom: 10px;
}
.black-key-group .f-keymap {
  color: #9fc0e0;
  bottom: 25px;
}

.white-key-group.pressed .f-notes,
.white-key-group.pressed .f-keymap {
  color: #8a6225;
}

.black-key-group.pressed .f-notes,
.black-key-group.pressed .f-keymap {
  color: #ffd58f;
}

.white-key-group.guide-target:not(.pressed) {
  border-color: #d6a64f;
  box-shadow:
    inset 0 -6px 10px rgba(90, 70, 36, 0.2),
    0 0 0 2px rgba(214, 166, 79, 0.3),
    0 0 16px rgba(214, 166, 79, 0.55);
  animation: guidePulse 1.2s ease-in-out infinite;
}

.black-key-group.guide-target:not(.pressed) {
  box-shadow:
    inset 0 -5px 7px rgba(255, 255, 255, 0.12),
    0 7px 11px rgba(1, 3, 7, 0.58),
    0 0 0 2px rgba(214, 166, 79, 0.32),
    0 0 14px rgba(214, 166, 79, 0.6);
  animation: guidePulse 1.2s ease-in-out infinite;
}

@keyframes guidePulse {
  0%, 100% {
    filter: brightness(1);
  }
  50% {
    filter: brightness(1.08);
  }
}

.autoplay-indicator {
  margin-top: 16px;
  color: var(--text-muted);
  font-size: 13px;
  letter-spacing: 0.03em;
  transition: color 0.2s ease;
}

.autoplay-indicator.focus {
  margin-top: 6px;
  font-size: 12px;
}

.piano-container.perform-layout .autoplay-indicator {
  margin-top: 6px;
  font-size: 12px;
}

.shortcut-hint {
  margin-top: 8px;
  color: #8ea7c3;
  font-size: 12px;
  letter-spacing: 0.02em;
  text-align: center;
}

.summary-mask {
  position: fixed;
  inset: 0;
  z-index: 60;
  background: rgba(2, 8, 16, 0.62);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px;
  box-sizing: border-box;
}

.summary-card {
  width: min(540px, 100%);
  border-radius: 16px;
  border: 1px solid rgba(154, 196, 236, 0.38);
  background: linear-gradient(180deg, rgba(13, 26, 42, 0.96) 0%, rgba(11, 20, 33, 0.96) 100%);
  box-shadow: 0 20px 34px rgba(1, 7, 14, 0.54);
  padding: 16px;
}

.level-clear-card {
  border-color: rgba(238, 191, 110, 0.55);
  background: linear-gradient(180deg, rgba(44, 30, 15, 0.95) 0%, rgba(20, 14, 8, 0.95) 100%);
}

.summary-card h3 {
  margin: 0;
  color: #e7f4ff;
  font-size: 18px;
}

.summary-subtitle {
  margin: 6px 0 0;
  color: #a9c2dd;
  font-size: 12px;
}

.summary-grid {
  margin-top: 12px;
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
}

.summary-item {
  border-radius: 10px;
  border: 1px solid rgba(152, 190, 227, 0.28);
  background: rgba(19, 35, 53, 0.62);
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.summary-label {
  color: #96b3d1;
  font-size: 11px;
}

.summary-item strong {
  color: #e8f4ff;
  font-size: 16px;
}

.summary-grade-overview {
  margin: 10px 0 0;
  color: #cae2fb;
  font-size: 12px;
}

.summary-level-overview {
  margin: 6px 0 0;
  color: #b9d4ef;
  font-size: 12px;
}

.summary-insight {
  margin: 6px 0 0;
  color: #c6ddf5;
  font-size: 12px;
}

.summary-note {
  margin: 12px 0 0;
  padding: 10px;
  border-radius: 10px;
  border: 1px solid rgba(238, 197, 125, 0.32);
  background: rgba(82, 58, 28, 0.34);
  color: #ffe8c8;
  font-size: 13px;
  line-height: 1.6;
}

.summary-note.soft {
  margin-top: 8px;
  border-color: rgba(138, 190, 232, 0.32);
  background: rgba(26, 45, 67, 0.36);
  color: #d8ecff;
}

.summary-actions {
  margin-top: 12px;
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

.summary-btn {
  height: 32px;
  border-radius: 999px;
  border: 1px solid rgba(240, 192, 112, 0.6);
  background: rgba(81, 56, 26, 0.76);
  color: #ffeccc;
  font-size: 12px;
  padding: 0 14px;
  cursor: pointer;
}

.summary-btn.ghost {
  border-color: rgba(148, 188, 226, 0.42);
  background: rgba(22, 39, 57, 0.78);
  color: #d7ebff;
}

.summary-fade-enter-active,
.summary-fade-leave-active {
  transition: opacity 0.2s ease;
}

.summary-fade-enter-from,
.summary-fade-leave-to {
  opacity: 0;
}

.top-panel {
  position: relative;
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  width: min(920px, 100%);
  margin-bottom: 18px;
  gap: 14px;
}

.top-right {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-left: auto;
}

.interaction-bar {
  position: relative;
  z-index: 1;
  width: min(920px, 100%);
  margin-bottom: 12px;
  padding: 10px;
  border-radius: 14px;
  border: 1px solid rgba(123, 169, 215, 0.32);
  background: rgba(10, 21, 35, 0.72);
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  flex-wrap: wrap;
}

.piano-container.focus-layout .interaction-bar {
  margin-bottom: 8px;
  padding: 8px 10px;
  gap: 8px;
}

.piano-container.perform-layout .top-panel {
  margin-bottom: 8px;
}

.piano-container.perform-layout .main-title {
  font-size: clamp(1.35rem, 3.6vw, 2rem);
}

.piano-container.perform-layout .interaction-bar {
  margin-bottom: 8px;
  padding: 8px 10px;
  gap: 8px;
}

.layout-switch {
  display: inline-flex;
  border-radius: 999px;
  border: 1px solid rgba(232, 194, 128, 0.44);
  background: rgba(44, 31, 20, 0.72);
  padding: 3px;
  gap: 4px;
}

.layout-switch-btn {
  height: 30px;
  border-radius: 999px;
  border: none;
  background: transparent;
  color: #dec9a6;
  font-size: 12px;
  padding: 0 12px;
  cursor: pointer;
}

.layout-switch-btn.active {
  background: linear-gradient(180deg, rgba(192, 138, 67, 0.9) 0%, rgba(150, 104, 46, 0.96) 100%);
  color: #fff0d8;
  box-shadow: 0 3px 8px rgba(63, 37, 12, 0.44);
}

.view-switch {
  display: inline-flex;
  border-radius: 999px;
  border: 1px solid rgba(132, 176, 224, 0.42);
  background: rgba(17, 31, 48, 0.76);
  padding: 3px;
  gap: 4px;
}

.view-switch-btn {
  height: 30px;
  border-radius: 999px;
  border: none;
  background: transparent;
  color: #c5dcf3;
  font-size: 12px;
  padding: 0 14px;
  cursor: pointer;
}

.view-switch-btn.active {
  background: linear-gradient(180deg, rgba(83, 139, 193, 0.92) 0%, rgba(53, 99, 147, 0.96) 100%);
  color: #e8f5ff;
  box-shadow: 0 3px 8px rgba(20, 44, 69, 0.46);
}

.quick-actions {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 8px;
  margin-left: auto;
}

.piano-container.focus-layout .quick-actions {
  margin-left: 0;
  width: 100%;
  justify-content: flex-start;
}

.piano-container.perform-layout .quick-actions {
  margin-left: 0;
}

.quick-action {
  height: 30px;
  border-radius: 999px;
  border: 1px solid rgba(134, 202, 242, 0.46);
  background: rgba(20, 42, 63, 0.82);
  color: #dff1ff;
  font-size: 12px;
  padding: 0 12px;
  cursor: pointer;
}

.quick-action.danger {
  border-color: rgba(255, 142, 142, 0.55);
  color: #ffe4e4;
  background: rgba(81, 32, 32, 0.74);
}

.quick-action:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.interaction-toast {
  position: fixed;
  top: 18px;
  right: 18px;
  z-index: 50;
  min-width: 180px;
  max-width: min(440px, calc(100vw - 24px));
  padding: 10px 14px;
  border-radius: 10px;
  border: 1px solid rgba(130, 177, 220, 0.52);
  background: rgba(13, 28, 44, 0.95);
  color: #d8ecff;
  font-size: 12px;
  box-shadow: 0 12px 24px rgba(1, 6, 13, 0.48);
}

.interaction-toast.success {
  border-color: rgba(123, 227, 167, 0.6);
  color: #d5ffe8;
}

.interaction-toast.error {
  border-color: rgba(248, 148, 148, 0.65);
  color: #ffdcdc;
}

.toast-fade-enter-active,
.toast-fade-leave-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.toast-fade-enter-from,
.toast-fade-leave-to {
  opacity: 0;
  transform: translateY(-6px);
}

.mode-tag {
  border: 1px solid rgba(127, 170, 216, 0.4);
  color: #b6d4f6;
  background: rgba(16, 28, 44, 0.66);
  border-radius: 999px;
  padding: 8px 14px;
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.05em;
  white-space: nowrap;
}

.mode-tag.active {
  color: #d9f6ff;
  border-color: rgba(133, 219, 255, 0.56);
  box-shadow: 0 0 0 1px rgba(133, 219, 255, 0.2), 0 0 18px rgba(49, 194, 255, 0.35);
}

.control-panel {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  border-radius: 14px;
  background: rgba(11, 21, 35, 0.7);
  border: 1px solid rgba(148, 181, 219, 0.26);
}

.control-item {
  display: flex;
  align-items: center;
  gap: 8px;
}

.control-item span {
  color: #b8d5f3;
  font-size: 12px;
  font-weight: 600;
  letter-spacing: 0.03em;
  white-space: nowrap;
}

.volume-item {
  width: 188px;
}

.volume-slider {
  width: 100%;
  margin: 0;
  accent-color: #4ed0ff;
  cursor: pointer;
}

.tone-select {
  height: 30px;
  min-width: 128px;
  border-radius: 8px;
  border: 1px solid rgba(156, 197, 238, 0.38);
  background: rgba(18, 33, 53, 0.9);
  color: #deedff;
  font-size: 12px;
  padding: 0 8px;
  outline: none;
}

.tone-select:focus-visible {
  border-color: rgba(126, 226, 255, 0.9);
  box-shadow: 0 0 0 2px rgba(126, 226, 255, 0.24);
}

.tone-item {
  align-items: flex-start;
}

.tone-status {
  max-width: 230px;
  color: #8fb7de;
  font-size: 11px;
  line-height: 1.2;
}

.song-panel {
  position: relative;
  z-index: 1;
  width: min(920px, 100%);
  margin-bottom: 12px;
  border-radius: 16px;
  padding: 12px 14px;
  background: rgba(9, 19, 31, 0.82);
  border: 1px solid rgba(124, 161, 203, 0.32);
  box-shadow: 0 10px 22px rgba(5, 10, 16, 0.35);
}

.song-panel.focus {
  background: rgba(11, 23, 37, 0.9);
  border-color: rgba(141, 181, 225, 0.42);
}

.piano-container.focus-layout .song-panel.focus {
  position: fixed;
  top: 74px;
  left: 50%;
  transform: translateX(-50%);
  width: min(920px, calc(100vw - 24px));
  max-height: 70dvh;
  overflow: auto;
  z-index: 30;
}

.song-panel.perform {
  position: fixed;
  top: 74px;
  left: 50%;
  transform: translateX(-50%);
  width: min(920px, calc(100vw - 24px));
  max-height: 66dvh;
  overflow: auto;
  z-index: 30;
}

.panel-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
}

.panel-head-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.panel-title-wrap {
  min-width: 0;
}

.panel-title {
  margin: 0;
  color: #deedf9;
  font-size: 13px;
  font-weight: 700;
  letter-spacing: 0.06em;
}

.panel-subtitle {
  margin: 4px 0 0;
  color: #a8bfd8;
  font-size: 12px;
}

.panel-toggle {
  height: 30px;
  border: 1px solid rgba(154, 196, 236, 0.44);
  border-radius: 999px;
  background: rgba(20, 38, 58, 0.86);
  color: #d9edff;
  font-size: 12px;
  padding: 0 12px;
  cursor: pointer;
}

.panel-body {
  margin-top: 10px;
}

.panel-drop-enter-active,
.panel-drop-leave-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.panel-drop-enter-from,
.panel-drop-leave-to {
  opacity: 0;
  transform: translateY(-5px);
}

.song-row {
  display: flex;
  align-items: center;
  gap: 10px;
}

.song-row + .song-row {
  margin-top: 10px;
}

.song-row-meta {
  justify-content: space-between;
}

.song-select {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  color: #d5e2ef;
  font-size: 12px;
}

.song-select select {
  height: 30px;
  min-width: 210px;
  border-radius: 8px;
  border: 1px solid rgba(176, 198, 221, 0.44);
  background: rgba(21, 34, 50, 0.82);
  color: #e0edfb;
  padding: 0 8px;
}

.song-action {
  height: 30px;
  border: 1px solid rgba(126, 226, 255, 0.42);
  border-radius: 8px;
  background: linear-gradient(180deg, rgba(33, 64, 94, 0.9) 0%, rgba(18, 35, 55, 0.94) 100%);
  color: #ddf1ff;
  font-size: 12px;
  padding: 0 12px;
  cursor: pointer;
}

.song-action:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.song-action-danger {
  border-color: rgba(255, 142, 142, 0.55);
  background: linear-gradient(180deg, rgba(97, 45, 45, 0.9) 0%, rgba(68, 25, 25, 0.92) 100%);
  color: #ffe3e3;
}

.song-meta {
  color: #a9c1dc;
  font-size: 12px;
}

.level-card {
  margin-top: 10px;
  border-radius: 10px;
  border: 1px solid rgba(158, 194, 227, 0.32);
  background: rgba(16, 30, 45, 0.6);
  padding: 10px;
}

.level-card-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.level-title {
  color: #deecf9;
  font-size: 12px;
  font-weight: 600;
}

.level-badge {
  height: 22px;
  border-radius: 999px;
  border: 1px solid rgba(240, 192, 112, 0.62);
  background: rgba(82, 58, 29, 0.65);
  color: #ffedce;
  font-size: 11px;
  padding: 0 8px;
  display: inline-flex;
  align-items: center;
}

.level-track {
  margin-top: 8px;
  height: 7px;
  border-radius: 999px;
  background: rgba(121, 161, 196, 0.27);
  overflow: hidden;
}

.level-track-fill {
  display: block;
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, #7bd2ff 0%, #f1bf72 100%);
  transition: width 0.2s ease;
}

.level-rule {
  margin: 8px 0 0;
  color: #afcae5;
  font-size: 12px;
}

.level-segment {
  margin: 6px 0 0;
  color: #9bc0e6;
  font-size: 12px;
}

.level-status {
  margin: 6px 0 0;
  color: #a3bfdc;
  font-size: 12px;
}

.level-status.pass {
  color: #ccf5dc;
}

.level-status.active {
  color: #f8dfb7;
}

.level-actions {
  margin-top: 8px;
  display: flex;
  gap: 8px;
}

.song-import {
  margin-top: 10px;
  border: 1px dashed rgba(120, 163, 208, 0.35);
  border-radius: 10px;
  padding: 8px 10px;
  background: rgba(14, 27, 42, 0.42);
}

.song-import summary {
  cursor: pointer;
  color: #cae0fb;
  font-size: 12px;
  font-weight: 600;
}

.song-import-tip {
  margin: 8px 0 0;
  color: #9eb7d1;
  font-size: 12px;
  line-height: 1.5;
}

.song-import-meta {
  margin-top: 8px;
  display: flex;
  gap: 8px;
}

.song-import-meta-input {
  flex: 1;
  min-width: 0;
  height: 32px;
  border-radius: 8px;
  border: 1px solid rgba(147, 180, 214, 0.4);
  background: rgba(12, 22, 34, 0.88);
  color: #e7f1ff;
  padding: 0 10px;
  font-size: 12px;
  box-sizing: border-box;
}

.song-import-input {
  margin-top: 8px;
  width: 100%;
  box-sizing: border-box;
  border-radius: 10px;
  border: 1px solid rgba(147, 180, 214, 0.4);
  background: rgba(12, 22, 34, 0.88);
  color: #e7f1ff;
  padding: 10px;
  font-size: 12px;
  resize: vertical;
}

.song-import-actions {
  margin-top: 8px;
  display: flex;
  gap: 8px;
}

.song-import-file {
  display: none;
}

.song-import-status {
  margin: 8px 0 0;
  font-size: 12px;
  color: #b5ccdf;
}

.song-import-status.success {
  color: #baf4d2;
}

.song-import-status.error {
  color: #ffc5c5;
}

.song-focus-tip {
  margin: 10px 0 0;
  color: #9fb7d2;
  font-size: 12px;
}

.coach-panel {
  position: relative;
  z-index: 1;
  width: min(920px, 100%);
  margin-bottom: 14px;
  border-radius: 16px;
  padding: 12px 14px;
  background: rgba(14, 24, 37, 0.84);
  border: 1px solid rgba(177, 152, 109, 0.34);
  box-shadow: 0 10px 24px rgba(7, 10, 15, 0.38);
}

.coach-panel.focus {
  border-color: rgba(229, 188, 117, 0.48);
  box-shadow: 0 12px 26px rgba(14, 10, 4, 0.4);
}

.piano-container.focus-layout .coach-panel.focus {
  margin-bottom: 8px;
  padding: 10px;
}

.coach-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.coach-row + .coach-row {
  margin-top: 10px;
}

.learning-flow {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  gap: 6px;
  margin-bottom: 10px;
}

.learning-flow-step {
  height: 28px;
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  border: 1px solid rgba(165, 195, 224, 0.28);
  color: #afc8e4;
  background: rgba(18, 31, 46, 0.58);
  padding: 0 10px;
  font-size: 12px;
}

.learning-flow-step.active {
  border-color: rgba(236, 196, 127, 0.7);
  color: #ffebcf;
  background: rgba(87, 61, 26, 0.66);
}

.learning-flow-step.done {
  border-color: rgba(119, 223, 161, 0.62);
  color: #d8ffe8;
  background: rgba(28, 60, 47, 0.66);
}

.phrase-trainer-switch {
  display: inline-flex;
  gap: 8px;
  margin-bottom: 10px;
}

.phrase-mode-btn {
  height: 28px;
  border-radius: 999px;
  border: 1px solid rgba(145, 185, 219, 0.38);
  background: rgba(21, 34, 50, 0.7);
  color: #d2e4f8;
  font-size: 12px;
  padding: 0 12px;
  cursor: pointer;
}

.phrase-mode-btn.active {
  border-color: rgba(240, 192, 112, 0.75);
  color: #fff1d8;
  background: rgba(79, 54, 24, 0.72);
  box-shadow: 0 0 12px rgba(240, 192, 112, 0.24);
}

.focus-phase-board {
  margin-bottom: 10px;
  border-radius: 10px;
  border: 1px solid rgba(226, 185, 112, 0.4);
  background: rgba(67, 47, 22, 0.34);
  padding: 8px;
}

.focus-phase-title {
  color: #f6deba;
  font-size: 12px;
  font-weight: 700;
}

.focus-phase-steps {
  margin-top: 6px;
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 6px;
}

.focus-phase-step {
  border-radius: 8px;
  border: 1px solid rgba(176, 198, 221, 0.3);
  background: rgba(23, 37, 55, 0.64);
  color: #d5e7fa;
  font-size: 12px;
  padding: 6px;
  display: flex;
  flex-direction: column;
  gap: 2px;
  cursor: pointer;
}

.focus-phase-step small {
  color: #97b5d3;
  font-size: 10px;
}

.focus-phase-step.active {
  border-color: rgba(240, 192, 112, 0.78);
  background: rgba(82, 58, 29, 0.72);
  color: #fff0d2;
}

.focus-phase-step.done {
  border-color: rgba(111, 225, 146, 0.56);
  background: rgba(28, 60, 47, 0.62);
  color: #d8ffe8;
}

.focus-phase-hint {
  margin: 8px 0 0;
  color: #d4e7fb;
  font-size: 12px;
}

.coach-title-wrap {
  min-width: 0;
}

.coach-title {
  margin: 0;
  color: #f4e0bf;
  font-size: 13px;
  font-weight: 700;
  letter-spacing: 0.08em;
}

.coach-subtitle {
  margin: 4px 0 0;
  color: #b8c7d7;
  font-size: 12px;
}

.coach-reset {
  height: 30px;
  border: 1px solid rgba(231, 195, 134, 0.52);
  border-radius: 999px;
  padding: 0 14px;
  background: linear-gradient(180deg, #3e2f1d 0%, #2a1e12 100%);
  color: #f7d9a9;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
}

.coach-row-main {
  flex-wrap: wrap;
  justify-content: flex-start;
}

.coach-switch {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  color: #d5e2ef;
  font-size: 12px;
}

.coach-switch input {
  accent-color: #dfb062;
  cursor: pointer;
}

.coach-select {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  color: #d5e2ef;
  font-size: 12px;
}

.coach-select select {
  height: 28px;
  border-radius: 8px;
  border: 1px solid rgba(176, 198, 221, 0.44);
  background: rgba(21, 34, 50, 0.78);
  color: #e0edfb;
  padding: 0 8px;
}

.coach-select select:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.coach-bpm {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  color: #d5e2ef;
  font-size: 12px;
}

.coach-bpm input {
  width: 120px;
  margin: 0;
  accent-color: #dfb062;
  cursor: pointer;
}

.coach-bpm input:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.coach-adaptive {
  height: 28px;
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  border: 1px solid rgba(176, 198, 221, 0.3);
  padding: 0 10px;
  color: #c8daef;
  background: rgba(24, 37, 53, 0.65);
  font-size: 12px;
}

.coach-adaptive.active {
  border-color: rgba(111, 225, 146, 0.55);
  color: #cdf6da;
}

.metronome-visual {
  height: 28px;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  border-radius: 999px;
  padding: 0 10px;
  border: 1px solid rgba(176, 198, 221, 0.3);
  background: rgba(19, 32, 47, 0.7);
}

.metronome-visual.active {
  border-color: rgba(224, 176, 98, 0.58);
  box-shadow: 0 0 0 1px rgba(224, 176, 98, 0.24);
}

.metronome-visual.counting {
  border-color: rgba(125, 219, 255, 0.68);
  box-shadow: 0 0 0 1px rgba(125, 219, 255, 0.24);
}

.metronome-visual-label {
  color: #dfebf8;
  font-size: 12px;
  white-space: nowrap;
}

.metronome-dots {
  display: inline-flex;
  align-items: center;
  gap: 5px;
}

.metronome-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: rgba(176, 198, 221, 0.36);
  transition: transform 0.08s ease, background-color 0.08s ease, box-shadow 0.08s ease;
}

.metronome-dot.active {
  background: #f9c878;
  box-shadow: 0 0 10px rgba(249, 200, 120, 0.7);
  transform: scale(1.24);
}

.metronome-dot.counting.active {
  background: #7bd2ff;
  box-shadow: 0 0 10px rgba(123, 210, 255, 0.7);
}

.coach-stat {
  height: 28px;
  display: inline-flex;
  align-items: center;
  border-radius: 999px;
  border: 1px solid rgba(176, 198, 221, 0.3);
  padding: 0 10px;
  color: #c8daef;
  background: rgba(24, 37, 53, 0.65);
  font-size: 12px;
}

.coach-stat.correct {
  border-color: rgba(111, 225, 146, 0.6);
  color: #c8f8d8;
}

.coach-stat.wrong {
  border-color: rgba(248, 114, 114, 0.68);
  color: #ffd1d1;
}

.coach-score {
  color: #d8c6a7;
  font-size: 12px;
  letter-spacing: 0.04em;
}

.coach-score-wrap {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.focus-layout-tip {
  border-radius: 999px;
  border: 1px dashed rgba(228, 187, 113, 0.48);
  color: #efd6ae;
  background: rgba(71, 51, 25, 0.44);
  padding: 0 10px;
  height: 28px;
  display: inline-flex;
  align-items: center;
  font-size: 12px;
}

.coach-mini-action {
  height: 26px;
  border-radius: 999px;
  border: 1px solid rgba(143, 198, 244, 0.45);
  background: rgba(20, 42, 63, 0.82);
  color: #dff1ff;
  font-size: 12px;
  padding: 0 10px;
  cursor: pointer;
}

.coach-mini-action.ghost {
  border-color: rgba(153, 180, 207, 0.4);
  background: rgba(20, 34, 48, 0.82);
  color: #c9dff5;
}

.phrase-typing-strip {
  margin-top: 10px;
  border-radius: 12px;
  padding: 10px;
  border: 1px solid rgba(125, 219, 255, 0.28);
  background: rgba(15, 30, 46, 0.56);
}

.phrase-typing-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}

.phrase-typing-label {
  display: inline-block;
  color: #9ac1e8;
  font-size: 12px;
  margin-bottom: 8px;
}

.phrase-play-btn {
  height: 26px;
  border-radius: 999px;
  border: 1px solid rgba(125, 219, 255, 0.55);
  background: rgba(20, 45, 68, 0.74);
  color: #d9f3ff;
  font-size: 12px;
  padding: 0 10px;
  cursor: pointer;
  margin-bottom: 8px;
}

.phrase-progress {
  margin-bottom: 8px;
}

.phrase-progress-label {
  display: inline-block;
  color: #a8caea;
  font-size: 11px;
  margin-bottom: 5px;
}

.phrase-progress-track {
  height: 6px;
  width: 100%;
  border-radius: 999px;
  background: rgba(108, 145, 181, 0.28);
  overflow: hidden;
}

.phrase-progress-fill {
  display: block;
  height: 100%;
  background: linear-gradient(90deg, #7fd4ff 0%, #f1bf72 100%);
  border-radius: 999px;
  transition: width 0.16s ease;
}

.phrase-rhythm-lane {
  margin-bottom: 8px;
}

.phrase-rhythm-label {
  display: inline-block;
  color: #9ec3e8;
  font-size: 11px;
  margin-bottom: 5px;
}

.phrase-rhythm-track {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(20px, 1fr));
  gap: 6px;
}

.phrase-rhythm-node {
  height: 7px;
  border-radius: 999px;
  border: 1px solid rgba(142, 177, 208, 0.28);
  background: rgba(95, 128, 159, 0.34);
  transition: background-color 0.12s ease, box-shadow 0.12s ease, transform 0.12s ease;
}

.phrase-rhythm-node.done {
  background: rgba(108, 217, 148, 0.7);
  border-color: rgba(122, 228, 161, 0.62);
}

.phrase-rhythm-node.active {
  background: rgba(244, 198, 121, 0.9);
  border-color: rgba(247, 210, 143, 0.8);
  box-shadow: 0 0 8px rgba(244, 198, 121, 0.52);
  transform: translateY(-1px);
}

.phrase-rhythm-node.pending {
  opacity: 0.82;
}

.phrase-rhythm-node.masked {
  opacity: 0.32;
}

.phrase-rhythm-node.accent {
  box-shadow: 0 0 0 1px rgba(129, 188, 237, 0.3);
}

.focus-phase-tip {
  margin-bottom: 8px;
  border-radius: 8px;
  border: 1px dashed rgba(235, 196, 123, 0.52);
  background: rgba(69, 48, 21, 0.36);
  color: #f4dcb7;
  font-size: 12px;
  line-height: 1.45;
  padding: 7px 8px;
}

.test-strip {
  margin-bottom: 8px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
  flex-wrap: wrap;
}

.test-lives {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.test-label {
  color: #9ec0e1;
  font-size: 11px;
}

.test-life-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: rgba(150, 175, 203, 0.35);
}

.test-life-dot.active {
  background: #f2c277;
  box-shadow: 0 0 8px rgba(242, 194, 119, 0.72);
}

.test-grade {
  color: #e8f4ff;
  font-size: 12px;
  margin-left: 4px;
}

.retrain-banner {
  margin-bottom: 8px;
  border-radius: 9px;
  border: 1px solid rgba(238, 191, 110, 0.44);
  background: rgba(82, 59, 28, 0.46);
  padding: 8px;
}

.retrain-banner-text {
  display: block;
  color: #ffe8c7;
  font-size: 12px;
  line-height: 1.5;
}

.retrain-banner-actions {
  margin-top: 8px;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.grade-board {
  margin-bottom: 8px;
  display: inline-flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}

.grade-board-label {
  color: #9dc0e2;
  font-size: 11px;
}

.grade-chip {
  height: 22px;
  border-radius: 999px;
  border: 1px solid rgba(152, 190, 227, 0.36);
  background: rgba(21, 34, 50, 0.72);
  color: #d8ebff;
  padding: 0 8px;
  display: inline-flex;
  align-items: center;
  font-size: 11px;
}

.grade-chip.a {
  border-color: rgba(125, 223, 162, 0.6);
  color: #d8ffe7;
}

.grade-chip.b {
  border-color: rgba(137, 212, 255, 0.62);
  color: #dcf3ff;
}

.grade-chip.c {
  border-color: rgba(240, 192, 112, 0.68);
  color: #ffeccc;
}

.grade-chip.f {
  border-color: rgba(246, 131, 131, 0.7);
  color: #ffdcdc;
}

.challenge-board {
  margin-bottom: 8px;
  border-radius: 9px;
  border: 1px solid rgba(156, 191, 224, 0.3);
  background: rgba(19, 34, 50, 0.62);
  padding: 8px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.challenge-board-title {
  color: #e3f1ff;
  font-size: 12px;
  font-weight: 600;
}

.challenge-board-rule {
  color: #a8c6e3;
  font-size: 11px;
}

.challenge-board-rule.pass {
  color: #c8f8d8;
}

.challenge-board-segment {
  color: #f8dfb7;
  font-size: 11px;
}

.insight-board {
  margin-bottom: 8px;
  border-radius: 9px;
  border: 1px solid rgba(136, 184, 228, 0.32);
  background: rgba(18, 34, 52, 0.58);
  padding: 8px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}

.insight-board-title {
  color: #dff0ff;
  font-size: 12px;
  font-weight: 600;
}

.heat-chip-row {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.heat-chip {
  height: 22px;
  border-radius: 999px;
  border: 1px solid rgba(137, 180, 220, 0.32);
  background: rgba(21, 34, 50, 0.72);
  color: #d3e8ff;
  padding: 0 8px;
  display: inline-flex;
  align-items: center;
  font-size: 11px;
}

.heat-chip.cool {
  border-color: rgba(127, 212, 255, 0.55);
  color: #d8f2ff;
}

.heat-chip.warm {
  border-color: rgba(240, 192, 112, 0.65);
  color: #ffe9c7;
}

.heat-chip.hot {
  border-color: rgba(246, 131, 131, 0.75);
  color: #ffd5d5;
}

.heat-chip.empty {
  border-style: dashed;
  color: #aac4df;
}

.insight-line {
  color: #b9d4ef;
  font-size: 12px;
  line-height: 1.45;
}

.phrase-typing-notes {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.phrase-typing-note {
  min-width: 44px;
  text-align: center;
  border-radius: 8px;
  padding: 5px 8px;
  font-size: 12px;
  color: #d8e8f9;
  border: 1px solid rgba(156, 189, 222, 0.3);
  background: rgba(20, 33, 48, 0.75);
  transition: transform 0.12s ease, background-color 0.12s ease, box-shadow 0.12s ease;
}

.phrase-typing-note.done {
  color: #d8ffe8;
  border-color: rgba(111, 225, 146, 0.56);
  background: rgba(28, 60, 47, 0.66);
}

.phrase-typing-note.active {
  color: #fff3d8;
  border-color: rgba(240, 192, 112, 0.74);
  background: rgba(75, 52, 25, 0.72);
  box-shadow: 0 0 10px rgba(240, 192, 112, 0.42);
  transform: translateY(-1px);
}

.phrase-typing-note.pending {
  opacity: 0.88;
}

.phrase-typing-note.masked {
  opacity: 0.34;
  color: #8fa6c0;
  border-style: dashed;
}

.piano-stage {
  position: relative;
  z-index: 1;
  width: min(940px, 100%);
  padding: 18px;
  border-radius: 24px;
  background: linear-gradient(180deg, rgba(21, 14, 10, 0.92) 0%, rgba(10, 8, 6, 0.94) 100%);
  border: 1px solid rgba(132, 101, 71, 0.54);
  backdrop-filter: blur(10px);
  overflow-x: auto;
  overflow-y: hidden;
  box-shadow: 0 24px 44px rgba(2, 6, 12, 0.46);
}

.piano-stage.focus {
  width: min(900px, 100%);
  padding: 10px;
  border-radius: 16px;
  overflow: hidden;
  box-sizing: border-box;
}

.piano-stage.perform {
  width: min(920px, 100%);
  padding: 10px;
  border-radius: 16px;
  overflow: hidden;
  box-sizing: border-box;
}

.start-screen {
  position: relative;
  z-index: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  width: min(520px, 100%);
  color: var(--text-main);
  padding: 38px 28px;
  border-radius: 28px;
  background: var(--panel-bg);
  border: 1px solid var(--panel-border);
  backdrop-filter: blur(10px);
  box-shadow: 0 28px 48px rgba(5, 11, 22, 0.45);
}

.start-eyebrow {
  margin: 0;
  letter-spacing: 0.22em;
  color: #8ebce4;
  font-size: 11px;
  font-weight: 600;
}

.start-screen h1 {
  margin: 12px 0 8px;
  font-size: clamp(2.2rem, 7vw, 3.5rem);
  line-height: 1.05;
  letter-spacing: 0.04em;
}

.start-screen p {
  margin: 0;
  color: #b5cbe6;
  font-size: 15px;
  line-height: 1.7;
}

.start-button {
  margin-top: 24px;
  padding: 12px 30px;
  font-size: 15px;
  letter-spacing: 0.06em;
  color: #072338;
  font-weight: 700;
  background: linear-gradient(120deg, #7ee2ff 0%, #42c4ff 56%, #27aff2 100%);
  border: 1px solid rgba(255, 255, 255, 0.5);
  border-radius: 999px;
  cursor: pointer;
  transition: transform 0.2s ease, box-shadow 0.2s ease, filter 0.2s ease;
  box-shadow: 0 10px 24px rgba(20, 120, 196, 0.4);
}

.start-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 16px 28px rgba(20, 120, 196, 0.5);
  filter: brightness(1.04);
}

.start-button:active {
  transform: translateY(0);
}

.lyric-display {
  position: relative;
  z-index: 1;
  margin-top: 24px;
  min-height: 2.1em;
  font-size: clamp(1.1rem, 2.7vw, 1.6rem);
  color: #dff1ff;
  text-align: center;
  min-width: 300px;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-shadow: 0 6px 20px rgba(0, 0, 0, 0.4);
}

.main-title {
  margin: 0;
  font-size: clamp(1.7rem, 4.6vw, 2.7rem);
  color: var(--text-main);
  text-shadow: 0 5px 16px rgba(0, 0, 0, 0.44);
  font-weight: 700;
  text-align: left;
  letter-spacing: 0.03em;
}

@media (max-width: 820px) {
  .piano-container {
    padding: 16px 12px 18px;
  }

  .piano-container.focus-layout {
    padding: 8px 8px 10px;
  }

  .piano-container.perform-layout {
    padding: 8px 8px 10px;
  }

  .top-panel {
    flex-direction: column;
    align-items: flex-start;
    margin-bottom: 14px;
    width: min(920px, 100%);
  }

  .top-right {
    width: 100%;
    margin-left: 0;
    flex-direction: column;
    align-items: stretch;
  }

  .interaction-bar {
    width: min(920px, 100%);
    flex-direction: column;
    align-items: stretch;
    gap: 10px;
    padding: 10px;
  }

  .layout-switch {
    width: 100%;
    box-sizing: border-box;
  }

  .layout-switch-btn {
    flex: 1;
  }

  .view-switch {
    width: 100%;
    box-sizing: border-box;
  }

  .view-switch-btn {
    flex: 1;
  }

  .quick-actions {
    width: 100%;
    flex-direction: column;
    align-items: stretch;
    margin-left: 0;
  }

  .quick-action {
    width: 100%;
  }

  .song-panel {
    width: min(920px, 100%);
    padding: 10px;
    margin-bottom: 10px;
  }

  .piano-container.focus-layout .song-panel.focus {
    top: 62px;
    width: calc(100vw - 16px);
    max-height: 76dvh;
  }

  .song-panel.perform {
    top: 62px;
    width: calc(100vw - 16px);
    max-height: 76dvh;
  }

  .song-row {
    flex-direction: column;
    align-items: flex-start;
  }

  .song-row-meta {
    align-items: flex-start;
  }

  .song-select {
    width: 100%;
  }

  .song-select select {
    min-width: 0;
    flex: 1;
  }

  .song-import-actions {
    width: 100%;
    flex-direction: column;
  }

  .song-import-meta {
    flex-direction: column;
  }

  .song-action {
    width: 100%;
  }

  .level-actions {
    flex-direction: column;
  }

  .panel-head {
    flex-direction: column;
    align-items: flex-start;
  }

  .panel-head-actions {
    width: 100%;
    flex-direction: column;
    align-items: stretch;
  }

  .panel-toggle {
    width: 100%;
  }

  .coach-panel {
    width: min(920px, 100%);
    padding: 10px;
    margin-bottom: 10px;
  }

  .coach-row {
    flex-direction: column;
    align-items: flex-start;
  }

  .coach-row-main {
    gap: 8px;
  }

  .learning-flow {
    width: 100%;
  }

  .phrase-trainer-switch {
    width: 100%;
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 6px;
  }

  .phrase-mode-btn {
    width: 100%;
    padding: 0 6px;
  }

  .focus-phase-steps {
    grid-template-columns: 1fr;
  }

  .phrase-typing-strip {
    margin-top: 8px;
  }

  .phrase-typing-note {
    min-width: 38px;
    padding: 4px 6px;
  }

  .phrase-typing-head {
    flex-direction: column;
    align-items: flex-start;
  }

  .phrase-play-btn {
    width: 100%;
    margin-bottom: 6px;
  }

  .test-strip {
    width: 100%;
    flex-direction: column;
    align-items: stretch;
  }

  .test-lives {
    width: 100%;
    justify-content: flex-start;
    flex-wrap: wrap;
  }

  .grade-board {
    width: 100%;
  }

  .phrase-rhythm-track {
    gap: 4px;
  }

  .insight-board {
    width: 100%;
    box-sizing: border-box;
  }

  .heat-chip-row {
    width: 100%;
  }

  .retrain-banner-actions {
    width: 100%;
    flex-direction: column;
    align-items: stretch;
  }

  .challenge-board {
    width: 100%;
  }

  .coach-bpm {
    width: 100%;
  }

  .coach-bpm input {
    flex: 1;
    width: auto;
  }

  .coach-adaptive {
    width: 100%;
    box-sizing: border-box;
    justify-content: center;
  }

  .coach-score-wrap {
    width: 100%;
  }

  .coach-mini-action {
    width: 100%;
  }

  .metronome-visual {
    width: 100%;
    justify-content: space-between;
    box-sizing: border-box;
  }

  .control-panel {
    width: 100%;
    box-sizing: border-box;
    flex-wrap: wrap;
    gap: 10px;
  }

  .volume-item {
    width: 100%;
  }

  .tone-item {
    width: 100%;
  }

  .tone-select {
    flex: 1;
    min-width: 0;
  }

  .tone-status {
    max-width: none;
  }

  .mode-tag {
    font-size: 11px;
    padding: 7px 12px;
  }

  .piano-stage {
    border-radius: 18px;
    padding: 12px 10px;
    -webkit-overflow-scrolling: touch;
  }

  .piano-stage.focus {
    padding: 8px 6px;
  }

  .piano-stage.perform {
    padding: 8px 6px;
  }

  .lyric-display {
    min-width: 100%;
    margin-top: 18px;
  }

  .autoplay-indicator {
    margin-top: 10px;
    text-align: center;
  }

  .shortcut-hint {
    width: 100%;
    text-align: left;
    line-height: 1.5;
  }

  .summary-card {
    padding: 12px;
  }

  .summary-grid {
    grid-template-columns: 1fr;
  }

  .summary-actions {
    flex-direction: column;
  }

  .summary-btn {
    width: 100%;
  }
}
</style>
