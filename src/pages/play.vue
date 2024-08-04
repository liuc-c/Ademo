<script setup lang="ts">
import { watch } from 'vue'
import { useBroadcastChannel } from '@vueuse/core'
import type { Data } from 'clarity-decode'
import { decode } from 'clarity-decode'
import { Visualizer } from 'clarity-visualize'
import type { Envelope } from 'clarity-js/types/data'

const { data } = useBroadcastChannel({ name: 'test_clarity' })
const visualize = new Visualizer()

let sessionId = ''
let events: Data.DecodedEvent[] = []

function dataChange(message: any) {
  if (message) {
    const decoded = decode(message)
    if (decoded.envelope.sequence === 1) {
      reset(decoded.envelope, decoded.dimension?.[0].data[0][0])
    }
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
    sessionId = envelope.sessionId
  }
  events = []

  iframe.style.display = 'block'
  const mobile = isMobileDevice(userAgent)
  visualize.setup(iframe.contentWindow, { version: envelope.version, onresize: resize, mobile })
}

function sort(a: Data.DecodedEvent, b: Data.DecodedEvent): number {
  return a.time - b.time
}

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
</style>
