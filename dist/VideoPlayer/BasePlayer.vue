<template>
  <div class="video-container">
    <media-theme-sutro>
      <video
        slot="media"
        @pause="pause"
        @ended="pause"
        @keyup="changeSpeed"
        ref="video"
        :poster="previewImageLink"
        :controls="false"
        :title="title"
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
    </media-theme-sutro>
    <div ref="subtitlesContainer" class="custom-subtitles"></div>
  </div>
</template>

<script setup>
import { onMounted, onUpdated, ref } from 'vue'
import Hls from 'hls.js'
import 'player.style/sutro';

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
  }
})

const emit = defineEmits(['pause', 'test'])
const video = ref(null)
const subtitlesContainer = ref(null)

onMounted(() => {
  prepareVideoPlayer()
})

onUpdated(() => {
  prepareVideoPlayer()
})

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
          if (activeCues && activeCues.length > 0) {
            subtitlesContainer.value.textContent = activeCues[0].text
            subtitlesContainer.value.style.display = "block";
          } else {
            subtitlesContainer.value.style.display = "none";
          }
        });
        if (track.mode !== previousModes[index]) {
          console.log(`Track mode changed: ${track.mode}`);
          if (track.mode === "showing") {
            const activeCues = track.activeCues;
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

function changeSpeed(e) {
  if (e.key === 'w' && video.value) {
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
</style>