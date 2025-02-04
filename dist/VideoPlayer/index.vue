<template>
  <VDefaultVideoPlayer
    v-if="type === 'default'"
    @pause="pause"
    @video-fullscreen-change="onVideoFullScreenChange"
    @video-ended="onVideoEnd"
    @video-fullscreen-action="oVideoFullscreenAction"
    :previewImageLink="previewImageLink"
    :showTranscriptBlock="showTranscriptBlock"
    :isFullscreen="isFullscreen"
    :link="link"
    :progress="progress"
    :isMuted="isMuted"
    :autoplay="autoplay"
    v-model="videoElement"
  >
    <template v-slot:before-media><slot name="before-media"></slot></template>
    <template v-slot:after-media><slot name="after-media"></slot></template>
    <template v-slot:before-transcripts><slot name="before-transcripts"></slot></template>
    <template v-slot:after-transcripts><slot name="after-transcripts"></slot></template>
  </VDefaultVideoPlayer>
  
  <VPreviewVideoPlayer
    v-else-if="type === 'preview'"
    :previewImageLink="previewImageLink"
    :link="link"
  />
</template>

<script setup>
import VDefaultVideoPlayer from './VDefaultVideoPlayer.vue'
import VPreviewVideoPlayer from './VPreviewVideoPlayer.vue'
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
  type: {
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
function onVideoFullScreenChange(data) {
  emit('video-fullscreen-change', data)
}
function onVideoEnd(data) {
  emit('video-ended', data);
}
function oVideoFullscreenAction(data) {
  emit('video-fullscreen-action', data)
}
</script>
