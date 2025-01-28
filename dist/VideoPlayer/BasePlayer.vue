<template>
  <div class="video-container test">
    <media-theme-sutro class="video-player-theme-container">
      <video
        class="hls-player"
        slot="media"
        @pause="pause"
        @keyup="changeSpeed"
        @ended="onVideoEnd"
        @seek="seekVideo"
        ref="video"
        :poster="previewImageLink"
        :controls="false"
        :title="title"
        :isFullscreen="isFullscreen"
        controlslist="nodownload"
        playsinline
        crossorigin
      >
        <source
          :src="link"
          type="application/x-mpegURL"
        />
        <track 
          v-if="subtitles.length"
          v-for="(subtitle, i) in subtitles"
          :src="subtitle.link"
          kind="subtitles"
          :srclang="subtitle.lang"
          :label="subtitle.label" :default="i === 0" />
      </video>
      <div ref="subtitlesContainer" class="custom-subtitles" style="display: none;"></div>
    </media-theme-sutro>
  </div>
  <SubtitleBlock
    v-if="showTranscriptBlock"
    :subtitle="currentSubtitle"
    :cursor="videoCursor" 
    @seek="seekVideo" >
  </SubtitleBlock>
</template>

<script setup>
import { onMounted, onUpdated, ref, onUnmounted, computed } from 'vue'
import Hls from 'hls.js'
import 'player.style/sutro';
import SubtitleBlock from './SubtitleBlock.vue';

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
  title: {
    type: String,
    default: ''
  },
  isMuted: {
    type: Boolean,
    default: false
  },
  /**
   * array of object, for
   * subtitles to append:
   * object has link, lang ()
   */
  subtitles: {
    type: Array,
    default: []
  },
  /**
   * true, if showing separate
   * block with transcripts
   */
  showTranscriptBlock: {
    type: Boolean,
    default: true
  },
  isFullscreen: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change'])
const video = ref(null)
const subtitlesContainer = ref(null)
const currentSubtitleLang = ref(null)
const videoCursor = ref(0)
const isFullscreen = ref(false); 

onMounted(() => {
console.log("mounted current - - changed")
  prepareVideoPlayer()
  if (video.value) {
    checkFullscreen();
    video.value.addEventListener('timeupdate', updateCurrentTime);
    document.addEventListener('fullscreenchange', onFullscreenChange);
  }
})

onUpdated(() => {
})

onUnmounted(() => {
  if (video.value) {
    video.value.removeEventListener('timeupdate', updateCurrentTime);
    document.removeEventListener('fullscreenchange', onFullscreenChange);
  }
});

const currentSubtitle = computed(() => {
  if(props.subtitles) {
    const current = props.subtitles.filter((subt) => {
      return subt.lang === currentSubtitleLang.value
    })
    return current.length ? current[0] : null
  }
  return null
})

function checkFullscreen() {
  isFullscreen.value = document.fullscreenElement === video.value;
};

function onFullscreenChange() {
  checkFullscreen();
  emit('video-fullscreen-change', isFullscreen)
};


function updateCurrentTime() {
  videoCursor.value = video.value.currentTime;
}

function seekVideo(time) {
  video.value.currentTime = time;
}

function prepareVideoPlayer() {
  let hls = new Hls()
  let stream = props.link
  hls.loadSource(stream)

  if (video.value) {
    hls.attachMedia(video.value)
    video.value.muted = props.isMuted
    video.value.currentTime = props.progress

    const textTracks = video.value.textTracks;
    let previousModes = Array.from(textTracks).map((track) => track.mode);
    function checkTrackModeChanges() {
      Array.from(textTracks).forEach((track, index) => {
        track.addEventListener("cuechange", () => {
          const activeCues = track.activeCues;
          currentSubtitleLang.value = track.language
          if (activeCues && activeCues.length > 0) {
            subtitlesContainer.value.textContent = activeCues[0].text
            subtitlesContainer.value.style.display = "block";
          } else {
            subtitlesContainer.value.style.display = "none";
          }
        });
        if (track.mode !== previousModes[index]) {
          if (track.mode === "showing") {
            const activeCues = track.activeCues;
            currentSubtitleLang.value = track.language
            if (activeCues && activeCues.length > 0) {
              subtitlesContainer.value.style.display = "block";
              subtitlesContainer.value.textContent = activeCues[0].text
            } else {
              subtitlesContainer.value.style.display = "none";
            }
          } else {
            subtitlesContainer.value.style.display = "none";
          }
          previousModes[index] = track.mode;
        }
      });
    }
    setInterval(checkTrackModeChanges, 100);
  }
}

function pause() {
  const currentTime = video?.value?.currentTime || 0

  emit('pause', currentTime)
}

function onVideoEnd() {
  const currentTime = video?.value?.currentTime || 0
  pause()
  emit('video-ended', { currentTime: currentTime, video });
}

function changeSpeed(e) {
  if (e.key === 'w' && video && video.value) {
    video.value.playbackRate = video.value.playbackRate + 0.25
  } else if (e.key === 's' && video.value) {
    video.value.playbackRate = video.value.playbackRate - 0.25
  }
}
</script>
<style>
  video::cue {
    display: none!important;
    text-indent: -999%;
    color: transparent;
    background-color: transparent;
  }

  .custom-subtitles {
    position: absolute;
    left: 50%;
    width: auto;
    max-width: 90%;
    text-align: center;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    font-size: 16px;
    font-family: Arial, sans-serif;
    line-height: 1.5;
    padding: 10px 20px;
    border-radius: 10px;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
    margin-top: -120px;
    transform: translateX(-50%) translateY(-100%);
  }

  .video-player-theme-container, .hls-player {
    width: 100%;
  }
</style>