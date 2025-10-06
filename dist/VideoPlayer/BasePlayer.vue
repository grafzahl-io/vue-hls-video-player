<template>
  <div class="video-container">
    <div class="media-container" id="hls-player-media-container" :class="{'is-fullscreen': isFullscreen}">
      <div class="media-fullscreen-overlay" v-if="introTitle">
        <div class="intro-title" :class="{ 'auto-hide': autoHideIntroTitle }">
          <h2>{{ introTitle }}</h2>
        </div>
      </div>
      <div class="media-overlay" v-if="initialPlayButton">
        <div class="initial-play" :class="{'hide-playbutton': hideInitialPlayButton}">
          <media-play-button mediapaused="" class="media-button" aria-label="play" tabindex="0" role="button" @click="video.play()">
            <svg slot="icon" viewBox="0 0 32 32">
              <g>
                <path id="icon-play" d="M20.7131 14.6976C21.7208 15.2735 21.7208 16.7265 20.7131 17.3024L12.7442 21.856C11.7442 22.4274 10.5 21.7054 10.5 20.5536L10.5 11.4464C10.5 10.2946 11.7442 9.57257 12.7442 10.144L20.7131 14.6976Z"></path>
              </g>
            </svg>
          </media-play-button>
        </div>
      </div>
      <slot name="before-media"></slot>
      <media-theme-sutro class="video-player-theme-container" :class="{'is-fullscreen': isFullscreen}">

        <video
          class="hls-player"
          slot="media"
          :key="link"
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
            type="application/x-mpegURL"
          />
          <track 
            v-if="subtitles.length"
            v-for="(subtitle, i) in subtitles"
            :src="subtitle.link"
            kind="subtitles"
            :srclang="subtitle.lang"
            @error="onSubtitleError(subtitle.link)"
            :label="subtitle.label" :default="i === 0" />
        </video>
        <media-control-bar>
          <media-fullscreen-button>
            <span slot="tooltip-enter" v-if="options?.ui?.labels?.fullscreen">
              {{ options.ui.labels.fullscreen }}
            </span>
            <span slot="tooltip-exit" v-if="options?.ui?.labels?.exitFullscreen">
              {{ options.ui.labels.exitFullscreen }}
            </span>
          </media-fullscreen-button>
        </media-control-bar>
      </media-theme-sutro>
      <div class="custom-subtitles" v-show="(isFullscreen) || (!showTranscriptBlock)">
        <div class="subtitle-text" ref="subtitlesContainer" style="display: none;"></div>
      </div>
      <slot name="after-media"></slot>
    </div>
  </div>
  <slot name="before-transcripts"></slot>
  <SubtitleBlock
    ref="transcriptRef"
    :subtitle="currentSubtitle"
    :cursor="videoCursor"
    :options="options"
    :showTranscriptBlock="showTranscriptBlock"
    @seek="seekVideo"
    @toggleTranscript="toggleTranscript">
  </SubtitleBlock>
  <slot name="after-transcripts"></slot>
  <slot name="default"></slot>
</template>

<script setup>
import { onMounted, onUpdated, ref, onUnmounted, computed, watch, toRef } from 'vue'
import Hls from 'hls.js'
import 'player.style/sutro';
import SubtitleBlock from './SubtitleBlock.vue';

const props = defineProps({
  introTitle: {
    type: String,
    default: ''
  },
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
  additionHeaders: {
    type: Object,
    default: {}
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
  }
})

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change'])
const video = ref(null)
const subtitlesContainer = ref(null)
const currentSubtitleLang = ref(null)
const videoCursor = ref(0)
const isFullscreen = ref(false);
const orientation = ref(null)
const autoHideIntroTitle = ref(false);
const initialPlayButton = ref(false);
const hideInitialPlayButton = ref(false);
const transcriptRef = ref(null);
const previewImageLink = toRef(props, 'previewImageLink');
const link = toRef(props, 'link');
let currentTime = 0
let hls = null
let buttonElement = null

const videoElement = defineModel()

onMounted(() => {
  prepareVideoPlayer(props.link)
})

onUnmounted(() => {
  if (video.value) {
    video.value.removeEventListener('timeupdate', updateCurrentTime);
  }
  document.removeEventListener('fullscreenchange', onFullscreenChange);
  document.removeEventListener('orientationchange', onOrientationChange);
  window.screen.orientation.removeEventListener("change", onOrientationChange);
});

const mutedAttr = computed(() => {
  // autoplay is only possible when muted
  return (props.autoplay || props.isMuted);
})

const currentSubtitle = computed(() => {
  if(props.subtitles) {
    const current = props.subtitles.filter((subt) => {
      if(currentSubtitleLang.value) {
        return subt.lang === currentSubtitleLang.value
      } else {
        return subt.lang === "en"
      }
    })
    return current.length ? current[0] : null
  }
  return null
})

watch([props, videoElement], (a) => {
  if(a[0].autoplay && a[1] && a[1].paused) {
    // autoplay is only possible when muted
    a[1].muted = true
    setTimeout(() => {
      a[1].play().catch(err => console.warn("Autoplay-Error:", err));
    }, 200)
  }
})

watch(
  () => props.link,
  (newLink, oldLink) => {
  if (newLink !== oldLink) {
    prepareVideoPlayer(newLink);
  }
})

async function startFullscreen() {
  let vpVideoBlock = document.getElementById(props.fullScreenElement);
  if(video.value) {
    currentTime = video.value.currentTime
  }
  if (document.fullscreenElement) {
    if (screen.orientation && screen.orientation.unlock) {
      screen.orientation.unlock();
    }
    await document.exitFullscreen();
    if (/iPhone|iPad|AppleWebKit/i.test(navigator.userAgent)) {
      document.webkitExitFullscreen();
    } 
    isFullscreen.value = false;
    buttonElement.setAttribute('aria-label', "Enter fullscreen mode")
    const tooltip = buttonElement.querySelector('media-tooltip');

    if (tooltip) {
      console.log("Tooltip gefunden:", tooltip);

      // Slots für Tooltip-Enter und Tooltip-Exit finden
      const enterTooltip = tooltip.querySelector('slot[name="tooltip-enter"]');
      const exitTooltip = tooltip.querySelector('slot[name="tooltip-exit"]');

      if (enterTooltip && exitTooltip) {
        enterTooltip.style.display = 'block';
        exitTooltip.style.display = 'none';
      } else {
        console.warn("Tooltip-Slots nicht gefunden!");
      }
    } else {
      console.warn("Kein media-tooltip gefunden!");
    }
  } else {
    isFullscreen.value = true;
    try {
      buttonElement.setAttribute('aria-label', "Exit fullscreen mode")

      const tooltip = buttonElement.querySelector('media-tooltip');

      if (tooltip) {
        console.log("Tooltip gefunden:", tooltip);

        // Slots für Tooltip-Enter und Tooltip-Exit finden
        const enterTooltip = tooltip.querySelector('slot[name="tooltip-enter"]');
        const exitTooltip = tooltip.querySelector('slot[name="tooltip-exit"]');

        if (enterTooltip && exitTooltip) {
          enterTooltip.style.display = 'none';
          exitTooltip.style.display = 'block';
        } else {
          console.warn("Tooltip-Slots nicht gefunden!");
        }
      } else {
        console.warn("Kein media-tooltip gefunden!");
      }

      if (vpVideoBlock.requestFullscreen) {
        await vpVideoBlock.requestFullscreen();
      } else if (/iPhone|iPad|AppleWebKit/i.test(navigator.userAgent)) {
        vpVideoBlock = document.querySelector('video.hls-player');
        await vpVideoBlock.webkitEnterFullScreen(); // iOS snot
        vpVideoBlock.addEventListener("webkitendfullscreen", () => {
          setTimeout(() => {
            vpVideoBlock.style.width = "100%";
            vpVideoBlock.style.height = "auto";
          }, 100);
          isFullscreen.value = false;
          vpVideoBlock.removeEventListener("webkitendfullscreen")
        });
      } else if (vpVideoBlock.mozRequestFullScreen) {
        await vpVideoBlock.mozRequestFullScreen(); // Firefox
      } else if (vpVideoBlock.msRequestFullscreen) {
        await vpVideoBlock.msRequestFullscreen(); // IE/Edge
      }
      if (screen.orientation && screen.orientation.lock) {
        try {
          await screen.orientation.lock("landscape");
        } catch (error) {
          console.warn("Orientation lock failed", error);
        }
      } else {
        console.warn("Orientation lock not supported.");
      }
    } catch (error) {
      console.error("Fullscreen could not be activated", error);
      isFullscreen.value = false;
    }

    video.value.currentTime = currentTime;
  }
}

function onSubtitleError(link) {
  console.error(`Error on loading subtitles: ${link}`);
}

function onFullscreenChange(e) {
  e.preventDefault();
  // hide intro title after x seconds
  if(isFullscreen.value === true) {
    setTimeout(() => {
      autoHideIntroTitle.value = true;
    }, 5000)
  } else {
    autoHideIntroTitle.value = false;
  }
  isFullscreen.value = !!document.fullscreenElement;
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

  let isPortrait = angle === 0 || angle === 180;
  let isLandscape = angle === 90 || angle === -90;

  if (document.fullscreenElement && isPortrait) {
    setTimeout(async () => {
      if (!document.fullscreenElement) {
        startFullscreen();
      }
    }, 700);
  }
}

function updateCurrentTime() {
  videoCursor.value = video.value.currentTime;
  if(!video.value.paused) {
    initialPlayButton.value = false;
  }

  // update transcripts
  if(transcriptRef && video) {
    transcriptRef.value.onTimeUpdate(video.value)
  }
}

function toggleTranscript() {
  props.showTranscriptBlock = !props.showTranscriptBlock
}

function seekVideo(time) {
  video.value.currentTime = time;
  video.value.play()
}

function prepareVideoPlayer(link) {
  let initiallyLoaded = true;
  if (video.value) {
    if (hls) {
      hls.detachMedia();
      hls.destroy();
      initiallyLoaded = false;
    }
    hls = new Hls();
    hls.loadSource(link)
    hls.attachMedia(video.value)
    hls.recoverMediaError()

    if(props.additionHeaders) {
      /**
       * inject additional headers
       */
      hls.config.fetchSetup = async (context, init) => {
        init = init || {};
        init.headers = new Headers(init.headers || {});
        // set headers
        for (const [key, value] of Object.entries(props.additionHeaders)) {
          init.headers.set(key, value);
        }
        return new Request(context.url, init);
      };
    }

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
    initVideo();
  }
}

function initVideo() {
  if (video.value) {

    // pass video element as reference to model
    if (!videoElement.value) {
      videoElement.value = video.value;
    }

    video.value.addEventListener('timeupdate', updateCurrentTime);
    document.addEventListener('fullscreenchange', onFullscreenChange);
    document.addEventListener('orientationchange', onOrientationChange);
    window.screen.orientation.addEventListener("change", onOrientationChange);

    if(video.value.paused || video.value.currentTime === 0) {
      initialPlayButton.value = true

      if(props.hideInitialPlayButton) {
        setTimeout(() => {
          hideInitialPlayButton.value = true
        }, 1200)
      }
    }

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
        buttonElement = fullscreenButton
        const playbackRateButton = mediaTheme.shadowRoot.querySelector('media-playback-rate-menu');
        playbackRateButton.setAttribute('rates', '0.25 0.5 0.75 1 1.5 2 3');
        if (fullscreenButton) {
          fullscreenButton.handleClick = async (event) => {
            event.preventDefault();
            event.stopPropagation();
            event.stopImmediatePropagation();
            startFullscreen();
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
defineExpose({ startFullscreen }); 

</script>
<style>
  video::cue {
    display: none!important;
    text-indent: -999%;
    color: transparent;
    background-color: transparent;
    text-shadow: none !important;
    visibility: hidden !important;
    opacity: 0 !important;
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
    margin: 0;
  }

  .video-player-theme-container video {
    object-fit: cover;
  }

  .video-container {
    position: relative;
    line-height: 0;
  }
  
  .media-overlay {
    position: absolute;
    height: calc(100% - 50px);
    top: 0;
    width: 100%;
    z-index: 99;
    background: transparent;
    -webkit-transition: 1.5s ease-in-out;
    -moz-transition: 1.5s ease-in-out;
    transition: 1.5s ease-in-out;
  }
  .media-overlay .initial-play {
    display: flex;
    justify-content: center;
    align-content: center;
    align-items: center;
    height: 100%;
    -webkit-transition: 0.6s ease-in-out;
    -moz-transition: 0.6s ease-in-out;
    transition: 0.6s ease-in-out;
  }
  .media-overlay .initial-play.hide-playbutton {
    opacity: 0;
  }
  .vp-vpvideo-block:hover .media-overlay .initial-play.hide-playbutton {
    opacity: 1;
  }
  .media-overlay .initial-play media-play-button {
    width: 80px;
    height: 80px;
    margin-top: 50px;
    border-radius: 10px;
  }
  .media-overlay .initial-play media-play-button svg {
    width: 100%;
    height: 100%;
  }
  .media-fullscreen-overlay {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    z-index: 9999999;
    padding: 2rem 5rem;
  }
  .is-fullscreen .media-fullscreen-overlay {
    display: block!important;
  }
  .intro-title h2 {
    font-size: 23px;
    color: white;
    opacity: 1;
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

  .auto-hide {
    opacity: 0;
    -webkit-transition: opacity 1.5s ease-in-out;
    -moz-transition: opacity 1.5s ease-in-out;
    transition: opacity 1.5s ease-in-out;
  }
</style>