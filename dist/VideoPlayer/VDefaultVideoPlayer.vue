<template>
  <BasePlayer
    :introTitle="introTitle"
    :previewImageLink="previewImageLink"
    :link="link"
    :progress="progress"
    :isMuted="isMuted"
    :autoplay="autoplay"
    :isControls="isControls"
    :onVideoEnd="onVideoEnd"
    :isFullscreen="isFullscreen"
    :showTranscriptBlock="showTranscriptBlock"
    :hideInitialPlayButton="hideInitialPlayButton"
    @pause="pause"
    @video-ended="onVideoEnd"
    @video-fullscreen-change="onFullscreenChange"
    v-model="videoElement"
    ref="childRef"
  >
    <template v-slot:before-media><slot name="before-media"></slot></template>
    <template v-slot:after-media><slot name="after-media"></slot></template>
    <template v-slot:before-transcripts><slot name="before-transcripts"></slot></template>
    <template v-slot:after-transcripts><slot name="after-transcripts"></slot></template>
  </BasePlayer>
</template>

<script setup>
import BasePlayer from './BasePlayer.vue'
import { ref, toRef } from 'vue'

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change'])

const videoElement = ref(null);
const childRef = ref(null)

const props = defineProps({
  previewImageLink: {
    type: String,
    default: ''
  },
  link: {
    type: String,
    default: ''
  },
  progress: {
    type: Number,
    default: 0
  },
  isMuted: {
    type: Boolean,
    default: false
  },
  autoplay: {
    type: Boolean,
    default: false
  },
  isControls: {
    type: Boolean,
    default: true
  },
  isFullscreen: {
    type: Boolean,
    default: false
  },
  introTitle: {
    type: String,
    default: ''
  },
  showTranscriptBlock: {
    type: Boolean,
    default: true
  },
  hideInitialPlayButton: {
    type: Boolean,
    default: false
  },
  fullScreenElement: {
    type: String,
    default: 'hls-player-media-container'
  }
})

const link = toRef(props, 'link');
const previewImageLink = toRef(props, 'previewImageLink');

defineExpose({ startFullscreen }); 

function pause(currentTime) {
  emit('pause', currentTime)
}

function onVideoEnd(data) {
  pause()
  emit('video-ended', data);
}

function onFullscreenChange(data) {
  emit('video-fullscreen-change', data);
}

function startFullscreen() {
  childRef.value.startFullscreen();
}
</script>
