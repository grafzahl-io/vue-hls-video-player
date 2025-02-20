<template>
  <BasePlayer
    :previewImageLink="previewImageLink"
    :link="link"
    :progress="progress"
    :isMuted="isMuted"
    :autoplay="autoplay"
    :isControls="isControls"
    :onVideoEnd="onVideoEnd"
    :isFullscreen="isFullscreen"
    :showTranscriptBlock="showTranscriptBlock"
    @pause="pause"
    @video-ended="onVideoEnd"
    @video-fullscreen-change="onFullscreenChange"
    @video-fullscreen-action="oVideoFullscreenAction"
    v-model="videoElement"
  >
    <template v-slot:before-media><slot name="before-media"></slot></template>
    <template v-slot:after-media><slot name="after-media"></slot></template>
    <template v-slot:before-transcripts><slot name="before-transcripts"></slot></template>
    <template v-slot:after-transcripts><slot name="after-transcripts"></slot></template>
  </BasePlayer>
</template>

<script setup>
import BasePlayer from './BasePlayer.vue'
import { ref } from 'vue'

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change', 'video-fullscreen-action'])

const videoElement = ref(null);

defineProps({
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
  showTranscriptBlock: {
    type: Boolean,
    default: true
  }
})

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

function oVideoFullscreenAction(data) {
  emit('video-fullscreen-action', data)
}
</script>
