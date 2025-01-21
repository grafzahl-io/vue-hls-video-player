<template>
  <div class="transcript-container"
    ref="subtitlesContainer">
    <ul v-if="cues.length" class="subtitles">
      <li v-for="cue of cues" :class="{'current-highlight': cursor >= cue.start && cursor <= cue.end}">
        <span class="seconds">{{ secondsToTime(cue.start) }} - {{ secondsToTime(cue.end) }}</span>
        <span class="text">{{ cue.text }}</span>
      </li>
    </ul>
  </div>
</template>

<style lang="css" scoped>
  .subtitles li.current-highlight {
    font-weight: bold;
  }
</style>

<script setup>
import { onMounted, onUpdated, ref } from 'vue'

const props = defineProps({
  subtitle: {
    type: Object,
    default: null
  },
  cursor: {
    type: Number,
    default: 0
  },
})

const emit = defineEmits(['pause', 'test'])
const subtitlesContainer = ref(null)
const cues = ref([])

onMounted(async () => {
  if(props.subtitle) {
    cues.value = await parseVTT(props.subtitle.link)
  }
})

onUpdated(async () => {
  if(props.subtitle) {
    cues.value = await parseVTT(props.subtitle.link)
  }
  // auto scroll text
  if(subtitlesContainer) {
    const activeSubtitle = subtitlesContainer?.value?.querySelectorAll('li.current-highlight')[0];
    if (activeSubtitle) {
      subtitlesContainer.value.scrollTop = activeSubtitle.offsetTop - subtitlesContainer.value.offsetTop;
    }
  }
})


async function parseVTT(fileUrl) {
  const response = await fetch(fileUrl);
  const text = await response.text();
  const cues = [];
  const lines = text.split("\n");
  let cue = null;

  for (let i = 0; i < lines.length; i++) {
    const line = lines[i].trim();
    if (!line) continue;

    if (line.includes("-->")) {
      const [start, end] = line.split(" --> ");
      cue = { start: timeToSeconds(start), end: timeToSeconds(end), text: "" };
    } else if (cue) {
      cue.text += line + " ";
    }
    if (cue && (!lines[i + 1] || lines[i + 1].includes("-->"))) {
      cues.push(cue);
      cue = null;
    }
  }
  return cues;
}

function timeToSeconds(timestamp) {
  const [hours, minutes, seconds] = timestamp.split(":").map(parseFloat);
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



