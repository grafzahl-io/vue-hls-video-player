<template>
  <BasePlayer
    :previewImageLink="previewImageLink"
    :link="link"
    :progress="progress"
    :isMuted="isMuted"
    :isControls="isControls"
    :onVideoEnd="onVideoEnd"
    :isFullscreen="isFullscreen"
    :showTranscriptBlock="showTranscriptBlock"
    @pause="pause"
    @video-ended="onVideoEnd"
    @video-fullscreen-change="onFullscreenChange"
    @video-fullscreen-action="oVideoFullscreenAction"
  />
</template>

<script setup>
import BasePlayer from './BasePlayer.vue'

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change'])

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
