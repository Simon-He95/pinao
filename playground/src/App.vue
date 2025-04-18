<script setup lang="ts">
import { useEventListener } from 'lazy-js-utils'
import { computed, onMounted, reactive, ref } from 'vue' // Import necessary functions

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
let keyMap: Record<string, string> = {} // Ensure value is string
const activeButtonIdMap: Record<string, { oscillator: OscillatorNode; key: string }> = reactive({})
let keys: string[] = []
let audio: AudioContext
let gainNode: GainNode
const showPiano = ref(false)
const isAutoPlaying = ref(false)
const autoPlayTimeoutId = ref<ReturnType<typeof setTimeout> | null>(null)

// Song Data with Lyrics
const anJingSong: Array<{ note: string | null; duration: number; lyric?: string }> = [
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

// --- Lyric Display State ---
const currentLyric = ref<string>('')

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
  if (!audio && (window.AudioContext || window.webkitAudioContext)) {
    audio = new (window.AudioContext || window.webkitAudioContext)()
    gainNode = audio.createGain()
    gainNode.gain.value = 0.1
    gainNode.connect(audio.destination)
    // Resume context if it starts suspended (required by some browsers)
    if (audio.state === 'suspended')
      audio.resume()
  }
}

const playTone = (id: string): { oscillator: OscillatorNode } | null => {
  initAudio() // Ensure audio context is ready
  if (!audio)
    return null // Exit if audio context failed to initialize

  // Resume AudioContext if it's suspended
  if (audio.state === 'suspended')
    audio.resume()

  const frequency = getFrequency(id)
  if (frequency <= 0.1)
    return null // Don't play invalid notes

  const oscillator = audio.createOscillator()
  oscillator.type = 'sine'
  oscillator.connect(gainNode)
  oscillator.frequency.value = frequency
  oscillator.start()
  return { oscillator }
}

const stopTone = (id: string) => {
  if (activeButtonIdMap[id] && activeButtonIdMap[id].oscillator) {
    try {
      activeButtonIdMap[id].oscillator.stop()
      activeButtonIdMap[id].oscillator.disconnect()
    }
    catch (e) {
      // Oscillator might already be stopped
      console.warn(`Error stopping oscillator for ${id}:`, e)
    }
  }
}

// --- Refactored Key Press/Release ---
const pressKey = (noteId: string, keyChar = '') => {
  if (!noteId || activeButtonIdMap[noteId])
    return // Don't press if invalid or already active

  const toneResult = playTone(noteId)
  if (toneResult)
    activeButtonIdMap[noteId] = { oscillator: toneResult.oscillator, key: keyChar }
}

const releaseKey = (noteId: string) => {
  if (!noteId || !activeButtonIdMap[noteId])
    return // Don't release if invalid or not active

  stopTone(noteId)
  delete activeButtonIdMap[noteId]
}

// --- Auto Play Logic (Updated for Chords) ---
const interruptAutoPlay = () => {
  if (isAutoPlaying.value) {
    isAutoPlaying.value = false
    if (autoPlayTimeoutId.value) {
      clearTimeout(autoPlayTimeoutId.value)
      autoPlayTimeoutId.value = null
    }
    // Stop ALL currently playing auto-play notes
    // const currentNote = pressedKey.value; // REMOVE THIS LINE
    Object.keys(activeButtonIdMap).forEach((noteId) => {
      if (activeButtonIdMap[noteId]?.key === '') { // Check if it's an auto-play note
        releaseKey(noteId)
      }
    })
    currentLyric.value = '' // Clear lyric on interrupt
  }
}

const playSong = async (song: Array<{ note: string | null; duration: number; lyric?: string }>) => {
  isAutoPlaying.value = true
  currentLyric.value = '' // Clear lyrics initially
  for (let i = 0; i < song.length; i++) {
    if (!isAutoPlaying.value)
      break

    const { note, duration, lyric } = song[i]

    // Update lyric when the note starts
    if (lyric)
      currentLyric.value = lyric

    if (note) {
      pressKey(note, '')
      if (activeButtonIdMap[note]) {
        await new Promise((resolve) => {
          autoPlayTimeoutId.value = setTimeout(() => {
            releaseKey(note)
            resolve(null)
          }, duration)
        })
        autoPlayTimeoutId.value = null
      }
      else {
        await new Promise((resolve) => {
          autoPlayTimeoutId.value = setTimeout(resolve, duration)
        })
        autoPlayTimeoutId.value = null
      }
    }
    else {
      await new Promise((resolve) => {
        autoPlayTimeoutId.value = setTimeout(resolve, duration)
      })
      autoPlayTimeoutId.value = null
      // Optionally clear lyric during rests
      // currentLyric.value = "";
    }
    if (isAutoPlaying.value && i < song.length - 1)
      await new Promise(resolve => setTimeout(resolve, 50))
  }
  if (isAutoPlaying.value) { // Check if it finished naturally
    isAutoPlaying.value = false
    currentLyric.value = '' // Clear lyric at the end
  }
}

// --- Class and Style Bindings (Corrected) ---
const isActive = (noteId: string): boolean => {
  // Check if the noteId exists as a key in the activeButtonIdMap
  return !!activeButtonIdMap[noteId]
}

const whiteKeyClass = (octaveIndex: number, index: number) => {
  // Find the note ID based on octave and index
  const keyChar = keys[index + octaveIndex * 12]
  const noteId = keyMap[keyChar]
  return `white-key-group key${isActive(noteId) ? ' pressed' : ''}`
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
    return `black-key-group key${isActive(noteId) ? ' pressed' : ''}`
  }
  return 'black-key-group key' // Should not happen if logic is correct
}

const blackKeyStyle = (octaveIndex: number, index: number) => {
  const whiteKeyWidth = 58
  const whiteKeyMargin = 2
  const blackKeyWidth = 34
  const totalWhiteKeyWidth = whiteKeyWidth + whiteKeyMargin

  const baseOffset = totalWhiteKeyWidth * (index + 1) - (blackKeyWidth / 2)
  const octaveOffset = totalWhiteKeyWidth * notes.length * octaveIndex
  const pianoPaddingLeft = 10

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
    pressKey(noteId, keyChar)
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

const handleKeyboardDown = (e: KeyboardEvent) => {
  if (e.repeat)
    return // Ignore repeats
  const keyChar = e.key.toLowerCase()
  const noteId = keyMap[keyChar]
  if (noteId) {
    interruptAutoPlay() // Interrupt on key press
    pressKey(noteId, keyChar)
  }
}

const handleKeyboardUp = (e: KeyboardEvent) => {
  const keyChar = e.key.toLowerCase()
  const noteId = keyMap[keyChar]
  // Find the noteId that corresponds to this keyChar, as multiple keys might map to the same note
  // However, our current keyMap is one-to-one, so direct lookup is fine.
  // If a note was pressed by this key, release it.
  if (noteId && activeButtonIdMap[noteId] && activeButtonIdMap[noteId].key === keyChar)
    releaseKey(noteId)
}

// --- Lifecycle Hook ---
onMounted(() => {
  initKeys()
  // Start playing after a short delay to allow UI rendering and audio setup
  setTimeout(() => {
    // Ensure audio is initialized before starting playback
    initAudio()
    if (audio) { // Check if audio context is available
      playSong(anJingSong)
    }
    else {
      console.error('AudioContext could not be initialized. Auto-play cancelled.')
    }
  }, 500)
})

// --- Event Listeners ---
useEventListener(window, 'keydown', handleKeyboardDown)
useEventListener(window, 'keyup', handleKeyboardUp)

// Helper to get note name and octave for display (remains the same)
const getNoteDisplay = (noteId: string | null) => {
  if (!noteId)
    return { name: '', key: '' }
  const key = Object.keys(keyMap).find(k => keyMap[k] === noteId)
  return { name: noteId, key: key ? key.toUpperCase() : '' }
}

const handleStartClick = () => {
  showPiano.value = true
  // Start playing after a short delay to allow UI rendering and audio setup
  setTimeout(() => {
    // Ensure audio is initialized before starting playback
    initAudio()
    if (audio) { // Check if audio context is available
      playSong(anJingSong)
    }
    else {
      console.error('AudioContext could not be initialized. Auto-play cancelled.')
    }
  }, 500)
}
</script>

<template>
  <div class="piano-container">
    <!-- Start Screen -->
    <div v-if="!showPiano" class="start-screen">
      <h1>在线钢琴</h1>
      <p>点击按钮开始体验并聆听《安静》</p>
      <button class="start-button" @click="handleStartClick">
        开始弹奏
      </button>
    </div>

    <!-- Piano and Indicator (shown after start) -->
    <template v-if="showPiano">
      <!-- Add the main title here -->
      <h1 class="main-title">
        在线钢琴
      </h1>

      <div class="piano">
        <!-- ... (Piano keys template remains the same) ... -->
        <template v-for="(octave, octaveIndex) in octaves" :key="octaveIndex">
          <!-- White Keys -->
          <template v-for="(note, index) in notes" :key="`w-${octaveIndex}-${index}`">
            <div
              :class="whiteKeyClass(octaveIndex, index)"
              :style="whiteKeyStyle(octaveIndex, index)"
              @mousedown="handleInteractionStart('white', index, octaveIndex)"
              @mouseup="handleInteractionEnd('white', index, octaveIndex)"
              @touchstart.prevent="handleInteractionStart('white', index, octaveIndex)"
              @touchend.prevent="handleInteractionEnd('white', index, octaveIndex)"
            >
              <span class="f-notes">{{ notes[index].toUpperCase() }}{{ octaveIndex + 3 }}</span>
              <span class="f-keymap">{{ keys[index + octaveIndex * 12]?.toUpperCase() }}</span>
            </div>
          </template>
          <!-- Black Keys -->
          <template v-for="(note, index) in notes" :key="`b-${octaveIndex}-${index}`">
            <div
              v-if="index !== 2 && index !== 6"
              :class="blackKeyClass(octaveIndex, index)"
              :style="blackKeyStyle(octaveIndex, index)"
              @mousedown="handleInteractionStart('black', index, octaveIndex)"
              @mouseup="handleInteractionEnd('black', index, octaveIndex)"
              @touchstart.prevent="handleInteractionStart('black', index, octaveIndex)"
              @touchend.prevent="handleInteractionEnd('black', index, octaveIndex)"
            >
              <span class="f-notes">{{ notes[index].toUpperCase() }}#{{ octaveIndex + 3 }}</span>
              <span class="f-keymap">{{ keys[index + (index === 0 || index === 1 ? 7 : 6) + octaveIndex * 12]?.toUpperCase() }}</span>
            </div>
          </template>
        </template>
      </div>

      <!-- Lyric Display -->
      <div class="lyric-display">
        {{ currentLyric }}
      </div>

      <div v-if="isAutoPlaying" class="autoplay-indicator">
        Playing "安静"... Click or press a key to stop.
      </div>
      <div v-else class="autoplay-indicator">
        自由弹奏中...
      </div>
    </template>
  </div>
</template>

<style lang="scss" scoped>
/* Add a container for centering */
.piano-container {
  display: flex;
  flex-direction: column; // Stack piano and indicator
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100vh;
  background: #333; /* Darker background for contrast */
  padding: 20px;
  box-sizing: border-box;
  overflow: hidden; /* Hide overflow for effects */
}

.piano {
  position: relative;
  display: flex;
  border: 1px solid #222;
  background: linear-gradient(to bottom, #5a5a5a, #3f3f3f);
  padding: 15px 10px 10px 10px;
  border-radius: 8px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5),
              inset 0 2px 2px rgba(255, 255, 255, 0.1); /* Added inset highlight */
  height: auto;
  width: auto;
  user-select: none;
  margin-bottom: 20px; // Space for indicator
}

.key {
  position: relative;
  cursor: pointer;
  /* Enhanced transition for smoother and slightly longer effect */
  transition: background 0.15s ease, transform 0.1s ease-out, box-shadow 0.15s ease;
  box-sizing: border-box;
}

.white-key-group {
  width: 58px;
  height: 220px;
  background: linear-gradient(to bottom, #ffffff 95%, #e8e8e8 100%);
  border: 1px solid #b0b0b0;
  border-top: none;
  border-radius: 0 0 7px 7px;
  box-shadow: inset 0 -3px 5px rgba(0, 0, 0, 0.1),
              0 3px 4px rgba(0, 0, 0, 0.25);
  margin-right: 2px;
  z-index: 1;

  &:last-child {
    margin-right: 0;
  }

  &.pressed {
    background: linear-gradient(to bottom, #f0f0f0 95%, #dcdcdc 100%);
    /* Slightly more pronounced press with a subtle tilt */
    transform: translateY(4px) rotateX(2deg);
    box-shadow: inset 0 -1px 2px rgba(0, 0, 0, 0.2),
                0 1px 1px rgba(0, 0, 0, 0.1),
                /* Added glow effect */
                0 0 15px 2px rgba(70, 190, 255, 0.6);
    border-color: #999;
  }
}

.black-key-group {
  position: absolute;
  width: 34px;
  height: 135px; /* Slightly taller */
  background: linear-gradient(to bottom, #4a4a4a 0%, #1c1c1c 100%);
  border: 1px solid #000;
  border-top: none;
  border-radius: 0 0 6px 6px;
  box-shadow: inset 0 -4px 5px rgba(255, 255, 255, 0.15),
              0 6px 10px rgba(0, 0, 0, 0.6);
  z-index: 2;
  top: 15px;

  &.pressed {
    background: linear-gradient(to bottom, #333 0%, #0a0a0a 100%);
    /* Slightly more pronounced press with a subtle tilt */
    transform: translateY(4px) rotateX(1deg);
    box-shadow: inset 0 -2px 3px rgba(255, 255, 255, 0.1),
                0 3px 4px rgba(0, 0, 0, 0.5),
                /* Added glow effect */
                0 0 18px 3px rgba(70, 190, 255, 0.7);
    border-color: #000;
  }
}

/* Text styling */
.f-notes, .f-keymap {
  position: absolute;
  width: 100%;
  text-align: center;
  font-weight: 500; /* Slightly bolder */
  font-size: 11px;
  color: #555;
  pointer-events: none;
  left: 0;
  bottom: 12px;
  transition: color 0.1s ease;
}

.f-keymap {
  bottom: 28px;
  color: #888;
  font-size: 10px; /* Smaller keymap text */
}

/* Specific text colors for black keys */
.black-key-group .f-notes {
  color: #ccc;
  bottom: 10px;
}
.black-key-group .f-keymap {
   color: #ddd;
   bottom: 25px;
}

/* Change text color on press */
.white-key-group.pressed .f-notes,
.white-key-group.pressed .f-keymap {
  color: #007acc; /* Accent color on press */
}

.black-key-group.pressed .f-notes,
.black-key-group.pressed .f-keymap {
  color: #6acfff; /* Lighter accent color on press */
}

/* Autoplay indicator style */
.autoplay-indicator {
  margin-top: 15px;
  color: #ccc;
  font-size: 14px;
  font-style: italic;
}

/* Start screen styles */
.start-screen {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  color: #fff;
}

.start-button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 16px;
  color: #fff;
  background-color: #007acc;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.start-button:hover {
  background-color: #005f99;
}

/* Lyric display styles */
.lyric-display {
  margin-top: 25px; /* Space above lyrics */
  height: 2em; /* Fixed height to prevent layout shifts */
  font-size: 1.4em;
  color: #e0e0e0;
  text-align: center;
  min-width: 300px; /* Ensure some minimum width */
  font-weight: bold;
  text-shadow: 1px 1px 3px rgba(0, 0, 0, 0.7);
}

/* Main Title Style */
.main-title {
  font-size: 2.8em;
  color: #fff;
  text-shadow: 0 3px 6px rgba(0,0,0,0.6);
  margin-bottom: 25px; /* Space below title */
  font-weight: bold;
  text-align: center;
}
</style>
