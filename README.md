# vue-hls-player

## Descriptions

It is a video player for the **m3u8** format
thankfully forked by `LeonidShv/vue-hls-video-player`.
Customized to make the player themable. Added
support for showing subtitles / captions of
the videos.

Requirements:
  only for the **vue 3** and **nuxt 3** projects

### Examples, how to use component
```
npm i vue-hls-player
```

```
<script setup>
  import { VideoPlayer } from 'vue-hls-player';

  function processPause(progress) {
    console.log(progress)
  }

  const subtitles = [
	{
		link: "subtitles-de.vtt",
		label: "Deutsch",
		lang: "de"
	},
	{
		link: "subtitles-en.vtt",
		label: "English",
		lang: "en"
	}
]
</script>

<template>
  <VideoPlayer
      type="default"
      @pause="processPause"
      previewImageLink="poster.webp"
      link="videoLink.m3u8"
      :progress="30"
      :isMuted="false"
      :isControls="true"
			:subtitles="subtitles"
      class="customClassName"
  />

  <VideoPlayer
      type="preview"
      previewImageLink="poster.webp"
      link="videoLink.m3u8"
      class="customClassName"
  />
</template>
```
For **nuxt 3**, try to wrap this component in ClientOnly, images for previewImageLink need to store in public folder
```
<ClientOnly>
  <VideoPlayer
    type="preview"
    previewImageLink="/img/learn.webp"
    link="https://demo.unified-streaming.com/k8s/features/stable/video/tears-of-steel/tears-of-steel.ism/.m3u8"
    class="customClassName"
  />
</ClientOnly>
```
### Props:
**type**: 
1. value: 'default', type: String

default video player, where you can process pauses and setup progress time.

Default props for the **type: default**:
```
:isMuted="false"
```
2. value: 'preview', type: String

you can pause video on hover, without sound (muted), without controls. It does not have access to props: isMuted, isControls, progress, @pause

Default props for the **type: preview**:
```
:isMuted="true"
```

**@pause**: 
1. Event, for processing pauses:
@pause="processPause"
```
function processPause(progress: number) {
  console.log(progress)
}
```
**previewImageLink**: 
1. value: 'poster.webp', type: String

poster image for the video player

**link**: 
1. value: 'videoLink.m3u8', type: String

link on video in format m3u8

**progress**:
1. value: true or false, type: Boolean

it can turn on and off the sound of the video

it can show and hide the video control panel

**subtitles**:
1. value: array of object, for subtitles to append: object has link, lang

subtitles to add as tracks to the video

### Last release:
v1.0.3
1. Removed controls in favour of themable overlay by `player.style`.
2. Updated hls library
3. Added styled caption overlays. Added separate container to show all captions.
