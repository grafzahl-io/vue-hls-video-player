<template>
  <div class="video-container">
    <div class="media-container" id="hls-player-media-container">
      <slot name="before-media"></slot>
      <media-theme-sutro class="video-player-theme-container" :class="{'is-fullscreen': isFullscreen}">
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
          controlslist="nodownload"
          playsinline
          crossorigin
          :muted="mutedAttr"
          :autoplay="autoplay && isMuted"
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
      <div class="custom-subtitles" v-show="(isFullscreen) || (!showTranscriptBlock)">
        <div class="subtitle-text" ref="subtitlesContainer" style="display: none;"></div>
      </div>
      <slot name="after-media"></slot>
    </div>
  </div>
  <slot name="before-transcripts"></slot>
  <SubtitleBlock
    :subtitle="currentSubtitle"
    :cursor="videoCursor" 
    :showTranscriptBlock="showTranscriptBlock"
    @seek="seekVideo"
    @toggleTranscript="toggleTranscript">
  </SubtitleBlock>
  <slot name="after-transcripts"></slot>
  <slot name="default"></slot>
</template>

<script setup>
import { onMounted, onUpdated, ref, onUnmounted, computed, watch } from 'vue'
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
  autoplay: {
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

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change', 'video-fullscreen-action'])
const video = ref(null)
const subtitlesContainer = ref(null)
const currentSubtitleLang = ref(null)
const videoCursor = ref(0)
const isFullscreen = ref(false);
const orientation = ref(null)

const videoElement = defineModel()

onMounted(() => {
  prepareVideoPlayer()
  if (video.value) {

    // pass video element as reference to model
    if (!videoElement.value) {
      videoElement.value = video.value;
    }

    video.value.addEventListener('timeupdate', updateCurrentTime);
    document.addEventListener('fullscreenchange', onFullscreenChange);
    document.addEventListener('orientationchange', onOrientationChange);
    window.screen.orientation.addEventListener("change", onOrientationChange);
    
    /**
     * detect initial display orientation
     */
    if (typeof window.orientation === "undefined") {
      orientation.value = window.screen.orientation.angle === 0 || window.screen.orientation.angle === 180 
        ? "portrait" 
        : "landscape";
    } else {
      orientation.value = window.orientation === 0 || window.orientation === 180 
        ? "portrait" 
        : "landscape"; // for safari
    }

    /**
     * overwrite player.style video fullscreen button
     * to inject own fullscreen logic
     */
    const observer = new MutationObserver((mutationsList, observer) => {
    const mediaTheme = document.querySelector('.video-player-theme-container');

    if (mediaTheme && mediaTheme.shadowRoot) {
      const fullscreenButton = mediaTheme.shadowRoot.querySelector('media-fullscreen-button');
      if (fullscreenButton) {
        fullscreenButton.handleClick = async (event) => {
          event.preventDefault();
          event.stopPropagation();
          event.stopImmediatePropagation();
          enterFullscreen(event)
        }
        observer.disconnect();
      } else {
        console.error('Button not found in Shadow DOM!');
      }
    } else {
      console.error('Shadow Root not found!');
    }
    })
    observer.observe(document.body, { childList: true, subtree: true });
  }
})

onUpdated(() => {

})

onUnmounted(() => {
  if (video.value) {
    video.value.removeEventListener('timeupdate', updateCurrentTime);
    document.removeEventListener('fullscreenchange', onFullscreenChange);
    document.removeEventListener('orientationchange', onOrientationChange);
    window.screen.orientation.addEventListener("change", onOrientationChange);
  }
});

const mutedAttr = computed(() => {
  // autoplay is only possible when muted
  return (props.autoplay || props.isMuted);
})

const currentSubtitle = computed(() => {
  if(props.subtitles) {
    const current = props.subtitles.filter((subt) => {
      return subt.lang === currentSubtitleLang.value
    })
    return current.length ? current[0] : null
  }
  return null
})

watch([props, videoElement], (a) => {
  if(a[0].autoplay && a[1]) {
    // autoplay is only possible when muted
    a[1].muted = true
    setTimeout(() => {
      a[1].play();
    }, 200)
  }
})

function onFullscreenChange() {
  isFullscreen.value = !!document.fullscreenElement
  emit('video-fullscreen-change', document.fullscreenElement)
};

function onOrientationChange(e) { 
  let angle;
  if (e.target && e.target.screen && e.target.screen.orientation) {
    angle = e.target.screen.orientation.angle;
  } else if (typeof window.orientation !== "undefined") {
    // extra sausage for safari
    angle = window.orientation;
  }
  orientation.value = angle === 0 || angle === 180 
    ? "portrait" 
    : "landscape";
}

function enterFullscreen(event) {
  emit('video-fullscreen-action', event)
}

function updateCurrentTime() {
  videoCursor.value = video.value.currentTime;
}

function toggleTranscript() {
  props.showTranscriptBlock = !props.showTranscriptBlock
}

function seekVideo(time) {
  video.value.currentTime = time;
  video.play()
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
          if(subtitlesContainer.value) {
            if (activeCues && activeCues.length > 0) {
              subtitlesContainer.value.textContent = activeCues[0].text
              subtitlesContainer.value.style.display = "block";
            } else {
              subtitlesContainer.value.style.display = "none";
            }
          }
        });
        if (track.mode !== previousModes[index]) {
          if (track.mode === "showing") {
            const activeCues = track.activeCues;
            currentSubtitleLang.value = track.language
            if(subtitlesContainer.value) {
              if (activeCues && activeCues.length > 0) {
                subtitlesContainer.value.style.display = "block";
                subtitlesContainer.value.textContent = activeCues[0].text
              } else {
                subtitlesContainer.value.style.display = "none";
              }
            }
          } else {
            if(subtitlesContainer.value) {
              subtitlesContainer.value.style.display = "none";
            }
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
    max-width: 95%;
    text-align: center;
    background: rgba(0, 0, 0, 0.7);
    border-radius: 6px;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.3);
    transform: translate(-50%) translateY(calc(-100% - 60px));
  }
  .custom-subtitles .subtitle-text {
    color: white;
    font-size: 14px;
    font-family: Arial, sans-serif;
    line-height: 1.5;
    padding: 8px 10px;
  }

  .video-player-theme-container, .hls-player {
    width: 100%;
    height: 100%;
  }

  .video-container {
    position: relative;
    line-height: 0;
  }

  .fullscreen-video .media-container {
  }
  .video-player-theme-container.is-fullscreen {
    width: 100vw;
    height: 100vh;
    object-fit: cover;
  }
  .video-player-theme-container.is-fullscreen .hls-player {
    width: 100vw;
    height: 100vh;
    object-fit: cover;
   }
</style>