<template>
  <div class="transcript-container" ref="subtitlesContainer" v-if="showTranscriptBlock">
    <div class="transcript-toggle">
      <button data-headlessui-state="open" @click="toggleTranscript()">
        <div class="icon">
          <svg v-if="!toggleTranscriptBlock" xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-chevron-right-icon duration-200 h-4 w-4 text-gray-900 stroke-1"><path d="m9 18 6-6-6-6"></path></svg>
          <svg v-if="toggleTranscriptBlock" xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-chevron-right-icon rotate-90 transform duration-200 h-4 w-4 text-gray-900 stroke-1"><path d="m9 18 6-6-6-6"></path></svg>
        </div>
        Transcript
      </button>
    </div>
    <ul v-if="txtCues.length && toggleTranscriptBlock" class="subtitles">
      <li
        v-for="(txtCue, index) in txtCues"
        :key="index"
        :class="{ 'current-highlight': isTxtCueActive(txtCue) }"
        @click="seekTo(txtCue.start)"
      >
        <div class="play-icon">
          <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-square-play lucide transform duration-200 h-4 w-4 text-gray-900 stroke-1"><rect width="18" height="18" x="3" y="3" rx="2"/><path d="m9 8 6 4-6 4Z"/></svg>
        </div>
        <div class="content">
          <span class="meta">
            <span class="seconds">{{ secondsToTime(txtCue.start) }} - {{ secondsToTime(txtCue.end) }}</span>
            <span class="narrator">{{ txtCue.dialog[0].speaker }}</span>
          </span>
          <span class="text">
            <span
              v-for="(word, wordIndex) in txtCue.dialog[0].text.split('')"
              :key="wordIndex"
              :class="{ 'active-word': isWordActive(txtCue, word, wordIndex, index) && isTxtCueActive(txtCue) }"
            >
              {{ word }}
            </span>
          </span>
        </div>
      </li>
    </ul>
  </div>
</template>

<style lang="css" scoped>
.transcript-toggle {
  font-weight: bold;
}
.transcript-toggle button {
  display: flex;
}
.transcript-toggle button .icon {
  padding: 3px;
}
.transcript-container {
  max-height: 400px;
  overflow-y: scroll;
  border-radius: 8px;
  border: 1px solid var(--outline-gray-1);
  padding: 10px;
  font-family: Arial, sans-serif;
  margin-top: 20px;
}

.subtitles {
  list-style: none;
  padding: 0;
  margin: 0;
}

.subtitles li {
  padding: 5px;
  cursor: pointer;
  display: flex;
}

.subtitles li .play-icon {
  min-width: 24px;
  padding-top: 4px;
}

.subtitles li .content {
  max-width: 800px;
}

.subtitles li .content .meta {
  display: flex;
  padding-bottom: 4px;
}
.subtitles li .content .meta .seconds {
  font-weight: 500;
}

.subtitles li .content .meta .narrator {
  padding-left: 10px;
  font-weight: 500;
}

.subtitles li.current-highlight {
}

.subtitles li .active-word {
  background-color: var(--outline-gray-1);
}

.subtitles li .seconds {
  display: block;
}

.subtitles li .text .narrator {
  display: block;
  color: #333;
}
</style>

<script setup>
import { onMounted, watch, ref } from 'vue';

const props = defineProps({
  subtitle: {
    type: Object,
    default: null,
  },
  cursor: {
    type: Number,
    default: 0,
  },
  showTranscriptBlock: {
    type: Boolean,
    default: true
  },
  toggleTranscriptBlock: {
    type: Boolean,
    default: true
  }
});

const emit = defineEmits(['seek', 'toggleTranscript']);
const subtitlesContainer = ref(null);
const vttCues = ref([]);
const txtCues = ref([]);
const currentCue = ref(null);


const loadCues = async () => {
  if (props.subtitle) {
    const vttPath = props.subtitle.link;
    const txtPath = vttPath.replace(/\.vtt$/, '.txt');

    vttCues.value = await parseVTT(vttPath);
    txtCues.value = await parseTXT(txtPath);
  }
};

onMounted(loadCues);

watch(
  () => props.subtitle,
  async (newSubtitle) => {
    if (newSubtitle) await loadCues();
  }
);

watch(
  () => props.cursor,
  (currentTime) => {
    highlightActiveCue(currentTime);
    checkCurrentCue(currentTime)
  }
);

/**
 * highlgiht the current transcript part
 * and keep it scrolled up
 * @param currentTime 
 */
function highlightActiveCue(currentTime) {
  if (subtitlesContainer.value) {
    const activeSubtitle = subtitlesContainer.value.querySelectorAll('li.current-highlight')[0];
    if (activeSubtitle) {
      subtitlesContainer.value.scrollTop =
        activeSubtitle.offsetTop - subtitlesContainer.value.offsetTop;
    }
  }
}

function toggleTranscript() {
  props.toggleTranscriptBlock = !props.toggleTranscriptBlock
}

function seekTo(time) {
  emit('seek', time);
}

/**
 * returns true, if the given txt
 * is currently played
 * @param txtCue 
 */
function isTxtCueActive(txtCue) {
  return props.cursor >= txtCue.start && props.cursor <= txtCue.end;
}

/**
 * true if the word part can be highlighed
 * as currently active
 * @param txtCue 
 * @param word 
 * @param wordIndex 
 * @param txtIndex 
 */
function isWordActive(txtCue, word, wordIndex, txtIndex) {
  if(!currentCue.value || !word || !txtCue) {
    return false
  }
  const startPos = txtCue.dialog[0].text.indexOf(currentCue.value)
  const endPos = startPos + currentCue.value.length
  if(wordIndex >= startPos && wordIndex < endPos) {
    return true;
  }
  return false;
}

function checkCurrentCue(currentCursor) {
  Array.from(vttCues.value).forEach((a, index) => {
    if(currentCursor >= a.start && currentCursor <= a.end) {
      currentCue.value = a.text
    }
  });
}

/**
 * get subtitles from the
 * vtt files
 * @param fileUrl
 */
async function parseVTT(fileUrl) {
  const response = await fetch(fileUrl);
  const text = await response.text();
  const cues = [];
  const lines = text.split('\n');
  let cue = null;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim();
    if (!line) continue;

    if (line.includes('-->')) {
      const [start, end] = line.split(' --> ');
      cue = { start: timeToSeconds(start), end: timeToSeconds(end), text: '' };
    } else if (cue) {
      cue.text += line + ' ';
    }

    if (cue && (!lines[i + 1] || lines[i + 1].includes('-->'))) {
      cues.push(cue);
      cue = null;
    }
  }
  return cues;
}

/**
 * get the raw transcriptions
 * from the txt files
 * @param fileUrl 
 */
async function parseTXT(fileUrl) {
  const response = await fetch(fileUrl);
  const text = await response.text();
  const cues = [];
  const lines = text.split('\n');
  let cue = null;
  let dialog = null;
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim();
    if (!line) continue;

    /**
     * extract every transcript part by time
     */
    if (line.match(/^\d{2}:\d{2}:\d{2}:\d{2} - \d{2}:\d{2}:\d{2}:\d{2}/)) {
      const [start, end] = line.split(' - ');
      cue = {
        start: timeToSeconds(start),
        end: timeToSeconds(end),
        dialog: []
      };
      dialog = {
        text: '',
        speaker: ''
      }
    } else if (cue && dialog.text == '' && dialog.speaker == '') {
      dialog.speaker = line;
    } else if (cue && dialog.speaker !== '') {
      dialog.text += line + ' ';
    }

    if (cue && (!lines[i + 1] || lines[i + 1].match(/^\d{2}:\d{2}:\d{2}:\d{2} - \d{2}:\d{2}:\d{2}:\d{2}/))) {
      cue.dialog.push(dialog);
      cues.push(cue);
      cue = null;
    }
  }
  return cues;
}

// Hilfsfunktionen
function timeToSeconds(timestamp) {
  if(!timestamp) {
    return 0
  }
  const [hours, minutes, seconds] = timestamp.split(':').map(parseFloat);
  return hours * 3600 + minutes * 60 + seconds;
}

function secondsToTime(seconds) {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.floor((seconds % 3600) / 60);
  const secs = Math.floor(seconds % 60);
  const pad = (num) => String(num).padStart(2, '0');
  return `${pad(hours)}:${pad(minutes)}:${pad(secs)}`;
}
</script>