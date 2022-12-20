<script setup lang="ts">
import { useEventListener } from 'lazy-js-utils'

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
let keyMap: Record<string, any> = {}
const activeButtonIdMap: Record<string, any> = {}
let keys: string[] = []
let audio: AudioContext
let gainNode: GainNode
const pressed = ref()
const _index = ref()
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

const getFrequency = (note: any) => {
  if (!note)
    return
  let octave, key: keyof typeof tone
  note = note.toLowerCase()

  if (note.length === 3) {
    octave = note.charAt(2)
    key = note.slice(0, 2)
  }
  else {
    octave = note.charAt(1)
    key = note.slice(0, 1)
  }
  octave = +octave + 1
  return tone[key][octave]
}
const initAudio = () => {
  audio = new (window.AudioContext || window.webkitAudioContext)()
  gainNode = audio.createGain()
  gainNode.gain.value = 0.1
  gainNode.connect(audio.destination)
}
const playTone = (id: string) => {
  const oscillator = audio.createOscillator()
  oscillator.type = 'sine'
  oscillator.connect(gainNode)
  oscillator.frequency.value = getFrequency(id) || 0.1
  oscillator.start()
  return { oscillator }
}

const stopTone = (id: string) => {
  const { oscillator } = activeButtonIdMap[id]
  oscillator?.stop()
}
const isActive = (key: string) => (activeButtonIdMap[keyMap[key]] ? ' pressed' : '')

const whiteKeyClass = (octaveIndex: number, index: number) =>
  `white-key-group key tom-${notes[index]}${isActive(keys[index + octaveIndex * 12])}`
const whiteKeyStyle = (octaveIndex: number, index: number) =>
  `left:${60 * octaveIndex * notes.length + 60 * index}px`

const blackKeyClass = (octaveIndex: number, index: number) =>
  `black-key-group key tom-s${notes[index]}${isActive(
    keys[index + (index === 0 || index === 1 ? 7 : 6) + octaveIndex * 12],
  )}`

const isPress = computed(() => isActive(keys[_index.value + pressed.value * 12]))

const blackKeyStyle = (octaveIndex: number, index: number) =>
  `left: ${60 * octaveIndex * notes.length + 60 * index + 45}px`

const handleKeyPressNote = (e: any, index: number, octaveIndex: number) => {
  let key = ''
  if (e.key) {
    key = e.key.toLowerCase()
  }
  else {
    if (e === 'white')
      key = keys[index + octaveIndex * 12]
    else key = keys[index + (index === 0 || index === 1 ? 7 : 6) + octaveIndex * 12]
  }
  const id = keyMap[key]

  if (!activeButtonIdMap[id]) {
    const { oscillator } = playTone(id)
    pressed.value = index
    _index.value = octaveIndex
    activeButtonIdMap[id] = { oscillator }
  }
}
const handleKeyUpNote = (e: any, index: number, octaveIndex: number) => {
  let key = ''
  if (e.key) {
    key = e.key.toLowerCase()
  }
  else {
    if (e === 'white')
      key = keys[index + octaveIndex * 12]
    else key = keys[index + (index === 0 || index === 1 ? 7 : 6) + octaveIndex * 12]
  }
  const id = keyMap[key]

  if (id && activeButtonIdMap[id]) {
    pressed.value = -1
    _index.value = -1
    stopTone(id)
    delete activeButtonIdMap[id]
  }
}
onMounted(() => {
  initKeys()
  initAudio()
})

useEventListener(window, 'keypress', handleKeyPressNote)
useEventListener(window, 'keyup', handleKeyUpNote)
</script>

<template>
  <div id="app" class="piano">
    <template v-for="(octave, octaveIndex) in octaves" :key="octaveIndex">
      <template v-for="(note, index) in notes" :key="index">
        <div
          :class="whiteKeyClass(octaveIndex, index)"
          :style="whiteKeyStyle(octaveIndex, index)"
          :active="isPress"
          @mouseover="handleKeyPressNote('white', index, octaveIndex)"
          @mouseleave="handleKeyUpNote('white', index, octaveIndex)"
        >
          <div class="tec y90-left size-lr" />
          <div class="tec y90-right size-lr" />
          <div class="tec x90-top keyTop size-t" />
          <div class="tec x90-front">
            <span class="f-notes" />
          </div>
        </div>
        <div
          v-if="index !== 2 && index !== 6"
          :class="blackKeyClass(octaveIndex, index)"
          :style="blackKeyStyle(octaveIndex, index)"
          @mouseover="handleKeyPressNote('black', index, octaveIndex)"
          @mouseleave="handleKeyUpNote('black', index, octaveIndex)"
        >
          <div class="tec y90-left size-lr" />
          <div class="tec y90-right size-lr" />
          <div class="tec x90-top keyTop size-t" />
          <div class="tec x90-front">
            <span class="f-notes" />
          </div>
        </div>
      </template>
    </template>
  </div>
</template>

<style lang="scss" scoped>
@mixin createKey(
  $colorLeft,
  $colorRight,
  $colorTop,
  $colorFront,
  $borderColor,
  $keyWidth,
  $keyHeight,
  $keyDeep
) {
  transform-style: preserve-3d;
  width: $keyWidth;
  height: $keyHeight;
  position: absolute;
  transform-origin: 0% 0% (-$keyDeep);
  backface-visibility: hidden;
  transition: 0.2s;

  .size-lr {
    width: $keyDeep;
    height: $keyHeight;
  }
  .size-t {
    width: $keyWidth;
    height: $keyDeep;
  }
  .tec {
    position: absolute;
    top: 0;
    left: 0;
    transform-origin: 0% 0%;
    border: 1px solid $borderColor;
  }
  .y90-left {
    transform: rotateY(90deg);
    background: $colorLeft;
    background-image: linear-gradient(to bottom right, $colorTop, $colorLeft);
  }
  .y90-right {
    transform: rotateY(90deg);
    left: $keyWidth;
    background: $colorRight;
    background-image: linear-gradient(to bottom right, $colorTop, $colorRight);
  }
  .x90-top {
    transform: rotateX(-90deg);
    background: $colorTop;
    background-image: linear-gradient(to bottom right, $colorFront, $colorTop);
  }
  .x90-front {
    transform: rotateX(0deg);
    width: $keyWidth;
    height: $keyHeight;
    background: $colorFront;
    background-image: linear-gradient(to top bottom, $colorTop, $colorFront);
    position: relative;
  }
  .x90-front2 {
    transform: rotateY(0deg);
    margin-left: $keyWidth;
    width: $keyWidth;
    height: $keyHeight;
    background: $colorFront;
    background-image: linear-gradient(to bottom right, $colorRight, $colorFront);
  }
}
html,
body {
  width: 100%;
  height: 100%;
}
#app {
  width: 800px;
  height: 100vh;
  margin: 0 auto;
  padding: 0;
}

.piano {
  transform-style: preserve-3d;
  perspective: 1500px;
  position: absolute;
  width: 1260px;
  height: 70px;
  top: 50%;
  left: 50%;
  transform: translateX(-50%) translateZ(10px) rotateY(0deg) rotateX(-50deg);
}
.white-key-group {
  z-index: 1;
  @include createKey(#999, #999, #fff, #eee, #ccc, 60px, 70px, 400px);
  &.pressed {
    transform: rotateX(-10deg);
  }
}
.black-key-group {
  margin-top: -42px;
  transform: translateZ(-150px);
  z-index: 100;
  @include createKey(#222, #222, #111, #333, #222, 30px, 40px, 250px);

  .f-notes {
    color: #eee;
    top: 5%;
  }
  .f-keymap {
    color: #eee;
    bottom: 5%;
  }

  &.pressed {
    transform: rotateX(-10deg) translateZ(-150px);
    margin-top: -20px;
  }
}

.active {
  box-shadow: 0px 0px 150px 10px #72ecfc;
}
.f-notes {
  text-align: center;
  font-weight: bold;
  font-size: 16px;
  color: #222;
  position: absolute;
  top: 10%;
  width: 100%;
  text-align: center;
}
.f-keymap {
  text-align: center;
  font-weight: bold;
  font-size: 12px;
  color: #222;
  position: absolute;
  bottom: 10%;
  width: 100%;
  text-align: center;
}
</style>
