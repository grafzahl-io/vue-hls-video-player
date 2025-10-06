<template>
  <VDefaultVideoPlayer
    v-if="type === 'default'"
    @pause="pause"
    @video-fullscreen-change="onVideoFullScreenChange"
    @video-ended="onVideoEnd"
    :additionHeaders="additionHeaders"
    :introTitle="introTitle"
    :previewImageLink="previewImageLink"
    :showTranscriptBlock="showTranscriptBlock"
    :isFullscreen="isFullscreen"
    :link="link"
    :progress="progress"
    :isMuted="isMuted"
    :autoplay="autoplay"
    v-model="videoElement"
    ref="childRef"
    :hideInitialPlayButton="hideInitialPlayButton"
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
import { ref, toRef } from 'vue'

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change', 'video-fullscreen-action'])

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
  },
  options: {
    type: Object,
    default: {}
  },
  additionHeaders: {
    type: Object,
    default: {}
  }
})

const link = toRef(props, 'link');
const previewImageLink = toRef(props, 'previewImageLink');

defineExpose({ startFullscreen }); 

function pause(currentTime) {
  emit('pause', currentTime)
}
function onVideoFullScreenChange(data) {
  emit('video-fullscreen-change', data)
}
function onVideoEnd(data) {
  emit('video-ended', data);
}
function startFullscreen() {
  childRef.value.startFullscreen();
}
</script>
