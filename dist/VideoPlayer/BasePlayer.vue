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
            :key="subtitle.lang + '-' + currentLang"
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
  <slot name="between-video-and-transcript"></slot> 
  <slot name="before-transcripts"></slot>
  <SubtitleBlock
    :key="currentLang"
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
// import 'media-chrome';

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
  },
  multiLangSources: {
  type: Array,
  default: () => []
  },
  defaultLang: {
    type: String,
    default: 'en'
  }
})

const emit = defineEmits(['pause', 'video-ended', 'video-fullscreen-change', 'pointer-update'])
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
// --- lang switcher ---
const currentLang = ref(props.defaultLang || 'en')
// --- Remember and restore last subtitle language ---
async function selectLang(lang) {

  if (!video.value || !hls) return;
  if (lang === currentLang.value) return;

  const newSource = props.multiLangSources?.find(s => s.lang === lang);
  if (!newSource?.file_url) {
    //Missing HLS source for ${lang}
    return;
  }

  // Remember playback state
  const wasPaused = video.value.paused;
  const time = video.value.currentTime;
  const wasMuted = video.value.muted;
  const rate = video.value.playbackRate;

  currentLang.value = lang;
  // Switching to ${lang}...

  try {
    hls.stopLoad();
    await hls.loadSource(newSource.file_url);
    video.value.muted = true;

    hls.once(Hls.Events.MANIFEST_PARSED, () => {
      // Manifest loaded for ${lang}

      // Restore playback
      video.value.currentTime = Math.max(0, time - 0.1);
      video.value.playbackRate = rate;
      if (!wasPaused) video.value.play();
      setTimeout(() => (video.value.muted = wasMuted), 400);

      // Match subtitle language with audio
      const matchLang = lang.toLowerCase();
      // Matching subtitles for matchLang
      Array.from(video.value.textTracks).forEach(track => {
        const tLang = (track.language || track.srclang || '').toLowerCase();
        console.log('[LangSwitch] Track', tLang, 'current mode:', track.mode);
        track.mode = tLang === matchLang ? 'showing' : 'disabled';
        console.log('[LangSwitch] Track', tLang, '→ new mode:', track.mode);
      });

      currentSubtitleLang.value = lang;
      console.log('[LangSwitch] currentSubtitleLang set to', currentSubtitleLang.value);
      updateLangMenuState();
    });
  } catch (err) {
    console.error("[LangSwitch] Reload failed:", err);
  }
}

console.log('[LangMenu][Check]', document.querySelectorAll('.lang-switcher .lang-menu').length);

function updateLangMenuState() {
  // const menu = document.querySelector('.lang-switcher .lang-menu');
  const mediaTheme = document.querySelector('.video-player-theme-container');
  const controlBar = mediaTheme?.shadowRoot?.querySelector('media-control-bar');
  const langSwitcher = controlBar?.querySelector('.lang-switcher');
  const menu = langSwitcher?.querySelector('.lang-menu');
  if (!menu || !video.value) return;

  const audioCol = menu.querySelector('.audio-col');
  const subCol = menu.querySelector('.sub-col');
  // Updating chevron marks
  console.log('   currentLang:', currentLang.value);
  console.log('   currentSubtitleLang:', currentSubtitleLang.value);
  // --- Audio ---
  audioCol?.querySelectorAll('li').forEach(li => {
    const liLang = li.textContent.trim().toLowerCase();
    li.classList.toggle('active', liLang === currentLang.value.toLowerCase());
  });

  // --- Subtitles ---
  const activeTrack = Array.from(video.value.textTracks).find(t => t.mode === 'showing');
  const activeSubLang = activeTrack
    ? (activeTrack.language || activeTrack.srclang || '').toLowerCase()
    : 'aus';
  console.log('[LangMenu] Active subtitle track:', activeSubLang);
  subCol?.querySelectorAll('li').forEach(li => {
    const liLang = li.textContent.trim().toLowerCase();
    li.classList.toggle('active', liLang === activeSubLang);
  });
}

watch(currentSubtitleLang, () => updateLangMenuState());

/* function selectLang(lang) {
  currentLang.value = lang;
  showLangMenu.value = false;

  const newSource = props.multiLangSources.find(s => s.lang === lang);
  if (newSource) {
    hls.stopLoad();
    hls.detachMedia();
    hls.loadSource(newSource.file_url);
    hls.attachMedia(video.value);
    hls.startLoad();
  }

  // Subtitle sync
  const textTracks = video.value.textTracks;
  for (const track of textTracks) {
    track.mode = track.language === lang ? 'showing' : 'disabled';
  }
} */

// --- Frame Pointer Loop ---
let rafId = null
const FPS = 30

function emitPointerUpdate() {
  if (video.value) {
    const t = video.value.currentTime || 0
    const frame = Math.floor(t * FPS)
    emit('pointer-update', { currentTime: t, frame })
  }
  rafId = requestAnimationFrame(emitPointerUpdate)
}

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
  // --- Stop pointer-update loop ---
  if (rafId) cancelAnimationFrame(rafId)
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
    buttonElement.removeAttribute('mediaIsFullscreen');
    const tooltip = buttonElement.shadowRoot?.querySelector('media-tooltip');
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
      buttonElement.setAttribute('mediaIsFullscreen', '');
      const tooltip = buttonElement.shadowRoot?.querySelector('media-tooltip');
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
  if (!video.value) return;

  // Reset previous HLS instance
  if (hls) {
    hls.detachMedia();
    hls.destroy();
  }

  // Preparing video player with link: ${link}
  hls = new Hls({
    fetchSetup: async (context, init) => {
      init = init || {};
      init.headers = new Headers(init.headers || {});
      for (const [key, value] of Object.entries(props.additionHeaders)) {
        init.headers.set(key, value);
      }
      return new Request(context.url, init);
    },
    xhrSetup: async (xhr, url) => {
      const additionHeaders = props.additionHeaders || {};
      for (const [key, value] of Object.entries(additionHeaders)) {
        const val = typeof value === 'function' ? await value(url) : value;
        if (val != null) xhr.setRequestHeader(key, val);
      }
    }
  });

  // Attach HLS
  hls.loadSource(link);
  hls.attachMedia(video.value);
  video.value.textTracks.addEventListener('change', () => {
  Array.from(video.value.textTracks).forEach(track => {
    console.log('[TrackChange] Track', track.language || track.srclang, '→', track.mode);
  });
  });
  // Native subtitle handling – without polling
  Array.from(video.value.textTracks).forEach(track => {
    track.addEventListener("cuechange", e => {
      const cues = e.target.activeCues;
      if (subtitlesContainer.value) {
        if (cues && cues.length > 0) {
          subtitlesContainer.value.style.display = "block";
          subtitlesContainer.value.textContent = cues[0].text;
        } else {
          subtitlesContainer.value.style.display = "none";
        }
      }
    });
  });

  // Automatically attach new tracks (after language switch)
  video.value.textTracks.onaddtrack = (e) => {
    const track = e.track;
    track.addEventListener("cuechange", evt => {
      const cues = evt.target.activeCues;
      if (subtitlesContainer.value) {
        if (cues && cues.length > 0) {
          subtitlesContainer.value.style.display = "block";
          subtitlesContainer.value.textContent = cues[0].text;
        } else {
          subtitlesContainer.value.style.display = "none";
        }
      }
    });
  };

  console.log('[SubtitleInit] Setting up initial subtitles', props.subtitles);
  // Initialize subtitles
  if (props.subtitles?.length > 0) {
    const defaultSub = props.subtitles.find(s => s.lang === props.defaultLang);
    currentSubtitleLang.value = defaultSub ? defaultSub.lang : props.subtitles[0].lang;
    console.log('[SubtitleInit] Default subtitle language set to', currentSubtitleLang.value);
    Array.from(video.value.textTracks).forEach(track => {
      const tLang = (track.language || track.srclang || '').toLowerCase();
      console.log('[SubtitleInit] Track found:', tLang, '->', tLang === currentSubtitleLang.value.toLowerCase() ? 'showing' : 'disabled');
      track.mode = tLang === currentSubtitleLang.value.toLowerCase() ? 'showing' : 'disabled';
    });
  }

  // HLS attached to <video>
  hls.recoverMediaError();

  video.value.muted = props.isMuted;
  video.value.currentTime = props.progress;
  // Chrome-like: update menu whenever track mode changes
  video.value.textTracks.addEventListener('change', updateLangMenuState);
  initVideo(); // Init controls etc.
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
        // controllbar
        const controlBar = mediaTheme.shadowRoot.querySelector('media-control-bar');
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
          // --- Remove default CC button ---
          const ccButton = mediaTheme.shadowRoot.querySelector('media-captions-button');
          if (ccButton) {
            ccButton.remove();
          }

          // --- Amazon Prime Style Language Switcher ---
          if (controlBar && !controlBar.querySelector('.lang-switcher')) {
            const langDiv = document.createElement('div');
            langDiv.className = 'lang-switcher';
            langDiv.innerHTML = `
              <style>
                .lang-switcher {
                  position: relative;
                  display: flex;
                  align-items: center;
                  justify-content: center;
                  margin-left: var(--media-control-spacing, 6px);
                }

                .lang-switcher button {
                  all: unset;
                  display: flex;
                  align-items: center;
                  justify-content: center;
                  cursor: pointer;
                  height: var(--media-button-height, 32px);
                  border-radius: 25%;
                  min-width: var(--media-button-height, 32px);
                  padding: var(--media-button-padding, 0 5px);
                  background: transparent;
                  transition: background 0.15s ease, transform 0.1s ease;
                }

                .lang-switcher button svg {
                  width: var(--media-icon-size, 24px);
                  height: var(--media-icon-size, 24px);
                  color: var(--media-icon-color, white);
                  stroke: currentColor;
                  opacity: var(--media-icon-opacity, 0.9);
                  pointer-events: none;
                  transition: opacity 0.15s ease;
                }

                .lang-switcher button:hover svg {
                  transform: scale(1.1);
                }

                .lang-switcher button:hover {
                  opacity: var(--media-icon-opacity-hover, 1);
                  background: var(--media-control-hover-background, rgba(50 50 70 / .7));
                  transition: backdrop-filter 0.3s, -webkit-backdrop-filter 0.3s;
                  box-shadow: rgba(0, 0, 0, 0.3) 0px 0px 5px;
                  backdrop-filter: blur(10px) invert(15%) brightness(80%) opacity(1);
                  padding: 5px;
                  color: var(--media-text-color, var(--media-primary-color, rgb(238 238 238)));
                }

                .lang-btn[title]:hover::after {
                  content: attr(title);
                  position: fixed;
                  bottom: 122%;
                  right: -100%;
                  white-space: nowrap;
                  background: var(--media-tooltip-background, rgba(0,0,0,0.2));
                  color: var(--media-tooltip-color, #fff);
                  border-radius: 4px;
                  padding: 4px 15px;
                  font-size: var(--media-font-size, 12px);
                  box-shadow: 0 2px 8px rgba(0,0,0,0.4);
                  line-height: calc(1.2 * var(--base));
                }

                .lang-menu {
                  position: absolute;
                  bottom: calc(var(--media-button-height, 32px) + 12px);
                  right: 0;
                  background: var(--media-control-bar-background, rgba(20,20,20,0.4));
                  backdrop-filter: blur(12px);
                  border-radius: 12px;
                  box-shadow: 0 8px 24px rgba(0,0,0,0.5);
                  padding: 10px;
                  min-width: 240px;
                  display: none;
                  animation: fadeUp 0.15s ease-out;
                  color: var(--media-text-color, #fff);
                  font-family: var(--media-font-family, system-ui, sans-serif);
                  font-size: var(--media-font-size, 13px);
                  z-index: 9999;
                }

                @keyframes fadeUp {
                  from { opacity: 0; transform: translateY(8px); }
                  to { opacity: 1; transform: translateY(0); }
                }

                .lang-columns {
                  display: flex;
                  justify-content: space-between;
                  gap: 1rem;
                }

                .lang-col {
                  flex: 1;
                  display: flex;
                  flex-direction: column;
                  margin: 0;
                  padding: 0;
                }

                .lang-col .title {
                  font-weight: 600;
                  font-size: calc(var(--media-font-size, 13px) - 1px);
                  opacity: 0.7;
                  margin-bottom: 4px;
                  border-bottom: 1px solid rgba(255,255,255,0.1);
                  padding-bottom: 2px;
                }

                .lang-col ul {
                  list-style: none;
                  margin: 0;
                  padding: 0;
                }

                .lang-col li {
                  display: flex;
                  align-items: center;
                  justify-content: space-between;
                  cursor: pointer;
                  border-radius: 4px;
                  padding: 2px 12px;
                  transition: background 0.15s ease, color 0.15s ease;
                }
                
                .lang-col li span {
                  line-height: calc(1.2 * var(--base));
                }

                .lang-col li:hover {
                  background: var(--media-control-hover-background, rgba(255,255,255,0.15));
                }

                .lang-col li.active {
                  font-weight: 500;
                  color: var(--media-accent-color, #fff);
                }

                .lang-col li .icon {
                  opacity: 0;
                  transition: opacity 0.15s ease, transform 0.15s ease;
                  transform: translateX(2px);
                }

                .lang-col li.active .icon {
                  opacity: 1;
                  transform: translateX(0);
                }
              </style>

              <button title="Audio & Subtitles" class="lang-btn">
                <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 26 26" fill="none" stroke="currentColor" stroke-width="1.5"
                    stroke-linecap="round" stroke-linejoin="round">
                  <path d="m5 8 6 6"/>
                  <path d="m4 14 6-6 2-3"/>
                  <path d="M2 5h12"/>
                  <path d="M7 2h1"/>
                  <path d="m22 22-5-10-5 10"/>
                  <path d="M14 18h6"/>
                </svg>
              </button>

              <div class="lang-menu">
                <div class="lang-columns">
                  <ul class="lang-col audio-col"><li class="title">Audio</li></ul>
                  <ul class="lang-col sub-col"><li class="title">Subtitles</li></ul>
                </div>
              </div>
            `;

            const menu = langDiv.querySelector('.lang-menu');
            const audioCol = menu.querySelector('.audio-col');
            const subCol = menu.querySelector('.sub-col');
            const button = langDiv.querySelector('button');
            menu.addEventListener('click', e => e.stopPropagation());
            // Chrome Media Player style check icon
            const renderIcon = () => `
              <svg aria-hidden="true" viewBox="0 1 24 24" width="16" height="16" part="checked-indicator indicator">
                <path fill="currentColor" d="m10 15.17 9.193-9.191 1.414 1.414-10.606 10.606-6.364-6.364 1.414-1.414 4.95 4.95Z"></path>
              </svg>
            `;

            // Audio options
            props.multiLangSources.forEach(src => {
              const li = document.createElement('li');
              li.innerHTML = `
                <span>${src.label || src.lang.toUpperCase()}</span>
                <span class="icon">${renderIcon()}</span>
              `;
              if (src.lang === currentLang.value) li.classList.add('active');
              li.addEventListener('click', () => {
                audioCol.querySelectorAll('li').forEach(el => el.classList.remove('active'));
                li.classList.add('active');
                selectLang(src.lang);
                menu.style.display = 'none';
              });
              audioCol.appendChild(li);
            });

            // Subtitle options
            /* const off = document.createElement('li');
            off.innerHTML = `<span>Off</span><span class="icon">${renderIcon()}</span>`;
            off.addEventListener('click', () => {
              Array.from(video.value.textTracks).forEach(track => (track.mode = 'disabled'));
              subCol.querySelectorAll('li').forEach(el => el.classList.remove('active'));
              off.classList.add('active');
              currentSubtitleLang.value = null;
              menu.style.display = 'none';
            });
            subCol.appendChild(off); */

            props.subtitles.forEach(sub => {
              const li = document.createElement('li');
              li.innerHTML = `
                <span>${sub.label || sub.lang.toUpperCase()}</span>
                <span class="icon">${renderIcon()}</span>
              `;
              if (sub.lang === currentSubtitleLang.value) li.classList.add('active');
              li.addEventListener('click', () => {
                Array.from(video.value.textTracks).forEach(track => {
                  const tLang = (track.language || track.srclang || '').toLowerCase();
                  track.mode = tLang === sub.lang.toLowerCase() ? 'showing' : 'disabled';
                });
                subCol.querySelectorAll('li').forEach(el => el.classList.remove('active'));
                li.classList.add('active');
                menu.style.display = 'none';
                currentSubtitleLang.value = sub.lang;
              });
              subCol.appendChild(li);
            });

            button.addEventListener('click', e => {
              e.stopPropagation();
              const isOpen = menu.style.display === 'block';
              menu.style.display = isOpen ? 'none' : 'block';
            });
            document.addEventListener('click', e => {
              if (!menu.contains(e.target) && !button.contains(e.target)) {
                menu.style.display = 'none';
                button.setAttribute('aria-expanded', 'false');
              }
            });


            controlBar.insertBefore(langDiv, fullscreenButton);
            console.log('[LangSwitch] Amazon Prime style menu added');
          }


          observer.disconnect();
        } else {
          console.error('Button not found in Shadow DOM!');
        }
      } else {
        console.error('Shadow Root not found!');
      }
    })
    // --- Start pointer-update loop ---
    if (!rafId) {
      emitPointerUpdate()
    }
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
defineExpose({ startFullscreen, getHls: () => hls, getVideo: () => video.value,
  changeSource: (newLink) => {
    if (!newLink) return;
    hls.stopLoad();
    hls.detachMedia();
    hls.loadSource(newLink);
    hls.attachMedia(video.value);
    hls.startLoad();
  }
 }); 

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