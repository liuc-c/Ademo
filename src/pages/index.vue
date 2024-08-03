<script setup lang="ts" generic="T extends any, O extends any">
import { onMounted, ref } from 'vue'
import { useBroadcastChannel } from '@vueuse/core'
import { clarity } from 'clarity-js'
import TheInput from '~/components/TheInput.vue'

const name = ref('')

const { post } = useBroadcastChannel({ name: 'test_clarity' })
// eslint-disable-next-line no-restricted-syntax
const enum Region {
  Header = 1,
  Footer = 2,
  Navigation = 3,
}
onMounted(() => {
  clarity.start({
    delay: 500,
    lean: false,
    drop: [],
    mask: [],
    unmask: [],
    content: true,
    fraud: true,
    track: true,
    throttleDom: true,
    checksum: [],
    regions: [
      [Region.Header, 'header'],
      [Region.Footer, 'footer'],
      [Region.Navigation, 'nav'],
    ],
    upload: (data: string): void => {
      post(data)
    },
    projectId: 'hhhhhhh',
  })
})
</script>

<template>
  <div v-for="item in 20">
    <div text-20>
      title
    </div>

    <div py-4 />

    <TheInput
      v-model="name"
      placeholder="What's your name?"
      autocomplete="false"
    />
  </div>
</template>
