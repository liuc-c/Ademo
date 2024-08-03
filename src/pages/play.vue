<script setup lang="ts">
import { useBroadcastChannel } from '@vueuse/core'
import { watch } from 'vue'

import type { Data } from 'clarity-decode'
import { decode } from 'clarity-decode'
import { Visualizer } from 'clarity-visualize'
import type { Envelope } from 'clarity-js/types/data'

const { data } = useBroadcastChannel({ name: 'test_clarity' })

const visualize = new Visualizer()

let sessionId = ''
let events: Data.DecodedEvent[] = []
let eJson: string[][] = [] // Encoded JSON for the whole session
let pJson: string[] = [] // Encoded JSON for the page
let dJson: Data.DecodedPayload[] = [] // Decoded JSON for the page

function dataChange(message: any) {
  // Handle responses from the background page, if any
  if (message) {
    const decoded = decode(message)
    if (decoded.envelope.sequence === 1 || visualize.state === null) {
      reset(decoded.envelope, decoded.dimension?.[0].data[0][0])
    }
    pJson.push(JSON.parse(message))
    dJson.push(copy(decoded)) // Save a copy of JSON
    if (visualize.state !== null) {
      const merged = visualize.merge([decoded])
      events = events.concat(merged.events).sort(sort)
      if (merged.dom) {
        visualize.dom(merged.dom)
      }
    }
  }
}
watch(data, () => {
  if (data.value)
    dataChange(data.value)
})

function replay(): void {
  // Execute only if there are events to render
  if (events.length > 0) {
    let event = events[0] as any
    const end = event.time + 16 // 60FPS => 16ms / frame
    let index = 0
    while (event && event.time < end) {
      event = event[++index]
    }
    visualize.render(events.splice(0, index))
  }
  requestAnimationFrame(replay)
}

function resize(width: number, height: number): void {
  const margin = 10
  const px = 'px'
  const iframe = document.getElementById('clarity') as HTMLIFrameElement
  const container = iframe.ownerDocument.documentElement
  const offsetTop = iframe.offsetTop
  const availableWidth = container.clientWidth - (2 * margin)
  const availableHeight = container.clientHeight - offsetTop - (2 * margin)
  const scale = Math.min(Math.min(availableWidth / width, 1), Math.min(availableHeight / height, 1))
  iframe.style.position = 'absolute'
  iframe.style.width = width + px
  iframe.style.height = height + px
  iframe.style.transformOrigin = '0 0 0'
  iframe.style.transform = `scale(${scale})`
  iframe.style.border = '1px solid #cccccc'
  iframe.style.overflow = 'hidden'
  iframe.style.left = ((container.clientWidth - (width * scale)) / 2) + px
}

function reset(envelope: Envelope, userAgent: string | undefined): void {
  if (console) { console.clear() }
  let iframe = document.getElementById('clarity') as HTMLIFrameElement
  const metadata = document.getElementById('header') as HTMLDivElement
  if (iframe && iframe.parentElement) {
    iframe.parentElement.removeChild(iframe)
  }
  iframe = document.createElement('iframe')
  iframe.id = 'clarity'
  iframe.title = 'Microsoft Clarity Developer Tools'
  iframe.setAttribute('scrolling', 'no')
  const targetDom = document.querySelector('#app > main') as HTMLElement
  targetDom.appendChild(iframe)
  console.log('Clearing out previous session... moving on to next one.')
  if (sessionId !== envelope.sessionId) {
    eJson = []
    sessionId = envelope.sessionId
  }
  else { eJson.push(pJson) }
  events = []
  pJson = []
  dJson = []

  iframe.style.display = 'block'
  metadata.style.display = 'block'
  const mobile = isMobileDevice(userAgent)
  visualize.setup(iframe.contentWindow, { version: envelope.version, onresize: resize, metadata, mobile })
}

function sort(a: Data.DecodedEvent, b: Data.DecodedEvent): number {
  return a.time - b.time
}

function copy(input: any): any {
  return JSON.parse(JSON.stringify(input))
}

// Call replay on every animation frame to emulate near real-time playback
requestAnimationFrame(replay)

function isMobileDevice(userAgent: string | undefined): boolean {
  if (!userAgent) {
    return false
  }
  return /android|webos|iphone|ipad|ipod|blackberry|windows phone|opera mini|iemobile|mobile|silk|fennec|bada|tizen|symbian|nokia|palmsource|meego|sailfish|kindle|playbook|bb10|rim/i.test(userAgent)
}
</script>

<template>
  <div id="header" />
  <div>
    <iframe id="clarity" title="Clarity Inspector" scrolling="no" />
  </div>
</template>

<style scoped>
body {
  margin:0px;
  padding:0px;
  text-align: center;
}
#clarity {
  height: 100vh;
  width: 100vw;
  overflow: hidden;
  border: 0;
}

#info {
  margin: 30px 30px 10px;
  font-size: 200%;
}

#header {
  height: 100px;
  overflow: hidden;
}

#header div {
  font-size: 8px;
}

#header div span {
  border: 2px solid #CCC;
  margin: 5px;
  padding: 3px;
  background: #CCC;
}

#header div span.visible, #header div span.clicked {
  background: lightgreen;
  border-color: lightgreen;
}

#header div span.clicked {
  border-color: green;
}

#download {
  height: 20px;
}

.loader {
  border: 2px solid #f3f3f3;
  border-top: 2px solid black;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  animation: spin 2s linear infinite;
  text-align: center;
  margin: 20px auto;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

ul {
  list-style: none;
  margin: 15px auto 10px;
  padding: 0;
}

ul li {
  display: inline-block;
  width: 50px;
  height: 30px;
  margin-left: 10px;
  font-size: 6px;
}

ul li h2 {
  width: 100%;
  text-align: center;
  height: 20px;
  margin: 0;
  padding: 10px 0;
  font-size: 14px;
  font-weight: normal;
  margin-bottom: 5px;
  background: green;
  color: white;
}

ul li h2 span {
  font-size: 8px;
}

#download {
  display: none;
  margin-bottom: 10px;
}
</style>
