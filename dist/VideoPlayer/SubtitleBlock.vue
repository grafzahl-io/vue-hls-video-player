<template>
  <div class="transcript-container" ref="subtitlesContainer">
    <ul v-if="txtCues.length" class="subtitles">
      <li
        v-for="(txtCue, index) in txtCues"
        :key="index"
        :class="{ 'current-highlight': isTxtCueActive(txtCue) }"
        @click="seekTo(txtCue.start)"
      >
        <span class="seconds">{{ secondsToTime(txtCue.start) }} - {{ secondsToTime(txtCue.end) }}</span>
        <span class="text">
          <span class="narrator">{{ txtCue.dialog[0].speaker }}</span>
          <span
            v-for="(word, wordIndex) in txtCue.dialog[0].text.split('')"
            :key="wordIndex"
            :class="{ 'active-word': isWordActive(txtCue, word, wordIndex, index) && isTxtCueActive(txtCue) }"
          >
            {{ word }}
          </span>
        </span>
      </li>
    </ul>
  </div>
</template>

<style lang="css" scoped>
.transcript-container {
  max-height: 400px;
  overflow-y: scroll;
  border: 1px solid #ccc;
  padding: 10px;
  font-family: Arial, sans-serif;
}

.subtitles {
  list-style: none;
  padding: 0;
  margin: 0;
}

.subtitles li {
  padding: 5px;
  cursor: pointer;
}

.subtitles li.current-highlight {
  background-color: #f7f7f7;
}

.subtitles li .active-word {
  font-weight: bold;
  background-color: yellow;
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
});

const emit = defineEmits(['seek']);
const subtitlesContainer = ref(null);
const vttCues = ref([]);
const txtCues = ref([]);
const currentCue = ref(null);
const usedHighlighted = ref({});
const currentTxtCue = ref(null)


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
      // usedHighlighted.value = {}
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