<script setup lang="ts">
//TODO: There needs be some error handling messaging in both components if no mp3 or audio context.
import { ref, useTemplateRef, reactive, watch, onMounted, computed } from 'vue'
import { useEventListener, type MaybeRef } from '@vueuse/core'
import { gsap } from 'gsap'
import {
  PlayIcon,
  PauseIcon,
  ChevronRightIcon,
  ChevronLeftIcon,
} from '@heroicons/vue/24/solid'

import MinimlSpectrumVisualizer from './MinimlSpectrumVisualizer.vue'
import { useStoreRef } from '@/composable/useStoreRef'

const spectrum = useTemplateRef('spectrum')
const audioEl = useTemplateRef('audio-element')
const panelTrack = useTemplateRef('panel-track')

const trackTime = ref<string>("00:00")
const trackDuration = ref<string>("00.00")
const trackIndex = ref<number>(0)
const currentTrack = ref<string>("")
const isPlaying = ref<boolean>(false)

// Add mp3 path here, note if you are testing on Vite with localhost metadata events wont fire (next , previous)
const PATH = "https://audio-tt.s3.amazonaws.com"

// The panel width, if track text wider then GSAP yoyo
const TRACK_WIDTH = 160

//Add tracks here; no plans to make a DOM playlist
const playlist = reactive([
  { artist: "Ernes Guevara", track: "Ernes Guevara - Lost (Original Mix).mp3" },
  {
    artist: "Stockholm Syndrome, Young Squage",
    track: "EMPHI - Stockholm Syndrome (Original Mix).mp3",
  },
  { artist: "EMPHI", track: "EMPHI - Pair of Dice (Original Mix).mp3" },
])

// Check for remaining tracks
const ifTrackNext = computed(() => {
  return trackIndex.value < playlist.length - 1
})

const ifTrackPrev = computed(() => {
  return trackIndex.value > 0
})

// Check for current track
const currTrack = computed(() => {
  return playlist[trackIndex.value].track
})

const togglePlay = () => {
  isPlaying.value = !isPlaying.value
  if (isPlaying.value && audioEl.value) {
    playTrack()
  } else if (audioEl.value) {
    audioEl.value.pause()
  }
}

// For previous and next we need to know if track is playing when we press them
const nextTrack = () => {
  if (ifTrackNext.value && !isPlaying.value) {
    trackIndex.value++
    currentTrack.value = currTrack.value
  } else if (ifTrackNext.value) {
    isPlaying.value = true
    trackIndex.value++
    playTrack()
  }
}

const prevTrack = () => {
  if (ifTrackPrev.value && !isPlaying.value) {
    trackIndex.value--
    currentTrack.value = currTrack.value
  } else if (ifTrackPrev.value) {
    isPlaying.value = true
    trackIndex.value--
    playTrack()
  }
}

const playTrack = () => {
  // Vueuse, easy cancel. oncanplaythrough does not work on mobile, loadedmetadata does???
  const cancelcan = useEventListener(
    audioEl.value as unknown as MaybeRef,
    "loadedmetadata",
    () => {
      audioEl.value?.play()
      cancelcan()
    }
  )
  // Synchronous, so we do this after adding event
  isPlaying.value = true
  audioEl.value!.currentTime = 0
  currentTrack.value = currTrack.value
  audioEl.value?.load()
}

// E from v-on listener
const timeUpdate = () => {
  setTimes()
}

const setTimes = () => {
  const m = ('0' + Math.floor((audioEl.value!.currentTime / 60) % 60)).slice(
    -2
  )
  const s = ('0' + Math.floor(audioEl.value!.currentTime % 60)).slice(-2)
  trackTime.value = `${m}:${s}`
}

// E from v-on listener
const durationUpdate = () => {
  const m = ("0" + Math.floor((audioEl.value!.duration / 60) % 60)).slice(-2)
  const s = ("0" + Math.floor(audioEl.value!.duration % 60)).slice(-2)
  trackDuration.value = `${m}:${s}`
}

const onTrackEnded = () => {
  if (ifTrackNext.value && spectrum.value) {
    trackIndex.value++
    playTrack()
  } else if (audioEl.value && spectrum.value) {
    isPlaying.value = false
    trackIndex.value = 0
    audioEl.value.pause()
    audioEl.value.currentTime = 0
    currentTrack.value = currTrack.value
  }
}

// GSAP, yoyo text left to right if title wider than display
watch(
  [isPlaying, trackIndex],
  () => {
    const { width } = panelTrack.value?.getBoundingClientRect() || {}
    const trackAnim = gsap.timeline()
    if (isPlaying.value && width && width > TRACK_WIDTH) {
      const remWidth = width - TRACK_WIDTH + 10
      trackAnim.fromTo(
        ".panel__box__track",
        { x: 0 },
        {
          duration: width / 100,
          x: -remWidth,
          repeat: -1,
          yoyo: true,
          ease: "sine.inOut",
        }
      )
    } else {
      trackAnim.to(".panel__box__track", {
        x: -3,
        duration: 1,
        ease: "sine.inOut",
        overwrite: "auto",
      })
    }
  },
  { flush: "post" }
)

onMounted(() => {
  const { addElem } = useStoreRef()
  addElem("audioEl", audioEl)
  currentTrack.value = currTrack.value
})
</script>

<template>
  <div class="player-wrapper">
    <audio :src="`${PATH}/${currentTrack}`" type="audio/mp3" preload="auto" v-on:timeupdate="timeUpdate"
      v-on:durationchange="durationUpdate" v-on:ended="onTrackEnded" ref="audio-element"
      crossorigin="anonymous"></audio>
    <div class="panel">
      <div class="panel__box">
        <div :class="{ 'panel__box__track--paused': !isPlaying }" class="panel__box__track" ref="panel-track">
          {{ currentTrack }}
        </div>
      </div>
    </div>
    <div class="time">
      <div class="time__current" ref="track-time">
        {{ trackTime }}
      </div>
      <div class="time__duration" ref="track-duration">
        {{ trackDuration }}
      </div>
    </div>
    <div class="controls">
      <div class="controls__pause-txt" :class="{ 'controls__pause-txt--show': !isPlaying }">
        -PAUSE-
      </div>
      <PlayIcon @click="togglePlay" class="controls__play" :class="{ 'controls__play--show': !isPlaying }" />
      <PauseIcon @click="togglePlay" class="controls__pause" :class="{ 'controls__pause--show': isPlaying }" />
      <ChevronLeftIcon @click="prevTrack" class="controls__prev" :class="{ 'controls__prev--end': !ifTrackPrev }">
      </ChevronLeftIcon>
      <ChevronRightIcon @click="nextTrack" class="controls__next" :class="{ 'controls__next--end': !ifTrackNext }">
      </ChevronRightIcon>
    </div>
    <div v-if="audioEl">
      <MinimlSpectrumVisualizer ref="spectrum" />
    </div>
  </div>
</template>

<style lang="scss" scoped>
/**
 * Please note that I use a variable Typekit font, update SCSS files as you need *
 */

.currstate {
  color: white;
  position: fixed;
  width: 100px;
  top: 100px;
  left: 0px;
  z-index: 900;
  font-size: 20px;
}

body {
  -webkit-overflow-scrolling: none;
  overflow: hidden;
  overscroll-behavior: none;
}

.player-wrapper {
  position: relative;
  width: 240px;
  height: 70px;
  -webkit-overflow-scrolling: none;
  overflow: hidden;
  overscroll-behavior: none;
}

.player-wrapper * {
  font-family: $sans-ui;
  font-optical-sizing: $sans-ui-optic;
  font-size: 12px;
  font-weight: 250;
}

.panel {
  color: $clr-secondary;
  height: 100%;
  //border: 1px solid white;

  &__box {
    position: absolute;
    left: 40px;
    bottom: 4px;
    height: 14px;
    width: 160px;
    padding: 0 5px 0 5px;
    overflow: hidden;
    user-select: none;

    &__track {
      white-space: nowrap;
      text-align: center;
      width: fit-content;
      margin: auto;
    }

    &__track--paused {
      color: $clr-secondary;
    }
  }
}

.time {
  position: absolute;
  top: 0;
  height: inherit;
  width: inherit;
  color: #f8f9fa;
  user-select: none;

  &__current {
    position: absolute;
    left: 3px;
    bottom: 2px;
    text-align: right;
  }

  &__duration {
    position: absolute;
    right: 2px;
    bottom: 2px;
    text-align: left;
  }
}

.controls {
  position: absolute;
  top: 0;
  height: inherit;
  width: inherit;
  text-align: center;

  &__play,
  &__pause {
    position: absolute;
    display: none;
    width: 34px;
    height: auto;
    right: 0px;
    bottom: 18px;
    color: #f8f9fa;
    cursor: pointer;
    transition: color 0.3s ease-in-out;

    &--show {
      display: block;
    }
  }

  &__play:hover,
  &__play:focus,
  &__pause:hover,
  &__pause:active {
    color: $clr-tertiary;
  }

  &__prev,
  &__next {
    position: absolute;
    width: 30px;
    height: auto;
    right: 54px;
    bottom: 20px;
    color: #f8f9fa;
    cursor: pointer;
    transition: color 0.3s ease-in-out;

    &--end {
      opacity: 0.5;
    }
  }

  &__next {
    right: 35px;
  }

  &__next:hover,
  &__next:focus,
  &__prev:hover,
  &__prev:active {
    color: $clr-tertiary;
  }

  &__pause-txt {
    position: relative;
    top: 30px;
    right: 25px;
    opacity: 0;
    transition: opacity 0.3s ease-in;
    user-select: none;
    color: $clr-secondary;

    &--show {
      opacity: 1;
    }
  }
}
</style>
