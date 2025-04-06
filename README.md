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
  <div id="video-container">
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
  </div>
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
**fullScreenElement**:
1. value: 'hls-player-media-container', type: String

If you need to provide additional UI in fullscreen mode
you can input a parent wrapper of your player to show
in fullscreen

```
fullScreenElement="demo"
```

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

**@video-ended**:
1. Event, called if the video has ended
@video-ended="videoEnded"
```
const videoEnded = (data) => {
  console.log("video ended at time: ", data.currentTime)
  console.log("video element ", data.video)
}
```

**@video-fullscreen-change**:
1. Event, event dispatcher for detecting fullscreen change
@video-fullscreen-change="fullscreenChange"
```
const fullScreenAction = (fullScreenElement) => {
	if(fullScreenElement === null) {
		console.log("fullscreen is off")
	} else {
		console.log("fullscreen is on")
	}
}
```

**@video-fullscreen-action**:
Removed in the 1.1.0 version.
It caused too much problems with mobile devices.
Now you can pass the desired element id to open
fullscreen with prop: fullScreenElement

**showTranscriptBlock**:
1. value: true or false, type: Boolean

pass true, if you want to show the transcript block.
To make transcripts work, your need to provide `.txt` files
in the same path and base-filename like the passed `subtitles` prop.

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
example: 
```
[{
  link: `https://url-to-your/subtitles.vtt`,
  label: 'English',
  lang: 'en'
}]
```

**isMuted**:
1 value: true or false, type: Boolean

it makes the video muted

**autoplay**:
1. value: true or false, type Boolean

it will set the native <video> autoplay property

**options**
1. value: object with settings, type Object

It contains various options to customize
the player.
At the moment the following attribute are supported:
```
  ui: {
    labels: {
      play: 'Play',
      pause: 'Pause',
      fullscreen: 'Toggle fullscreen',
      exitFullscreen: 'Toggle fullscreen',
      mute: 'Mute',
      unmute: 'Unmute',
    }
  }
```

### Last release:
v1.0.14 - v1.1.11
- Fixes
- iOS specific improvements
- New Options to customize the component
- Fix problem with not updating transcript highlighting
- Extending overloaded functions for fullscreen mode (WIP)

v1.0.9 - v1.0.14
- Fixes
- Small styling improvements

v1.0.9
- Fix sizes in fullscreen mode for video
- Hide transcript block completely when hidden

v1.0.8
- Add slots to inject own elements nearby video element
- Add prop for autoplay video

v1.0.7
- Add function to handle own logic for fullscreen

v1.0.6
- Small fixes
- Remove debug log

v1.0.5
- Load transcriptions additionally to subtitles
- Add styled transcription block for better readability
- Improve interaction and dynamic params

v1.0.4
- Make subtitles dynamic
- Add new switch to disable the subtitle block
- Fix some minor issues

v1.0.3
- Removed controls in favour of themable overlay by `player.style`.
- Updated hls library
- Added styled caption overlays. Added separate container to show all captions.
