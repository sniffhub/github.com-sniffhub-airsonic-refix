<template>
  <div style="width: 100vw; height: 100vh;">
    <CRTEffect
      theme="amber"
      fill
      enable-glow
      glow-color="rgba(255, 153, 51, 0.3)"
      edge-glow-color="rgba(255, 153, 51, 0.2)"
      enable-flicker
      flicker-intensity="low"
      enable-vignette
      :vignette-intensity="0.3"
    >
      <BootSequence v-if="!booted" @done="booted = true" />
      <template v-else>
        <component :is="layout">
          <keep-alive max="3">
            <router-view :key="$route.fullPath" />
          </keep-alive>
        </component>
        <ErrorToast />
      </template>
    </CRTEffect>
  </div>
</template>
<script lang="ts">
  import { defineComponent } from 'vue'
  import CRTEffect from 'vault66-crt-effect/vue'
  import 'vault66-crt-effect/style.css'
  import ErrorToast from './ErrorToast.vue'
  import BootSequence from './BootSequence.vue'
  import Default from '@/app/layout/Default.vue'
  import Fullscreen from '@/app/layout/Fullscreen.vue'

  export default defineComponent({
    components: {
      CRTEffect,
      ErrorToast,
      BootSequence,
      Default,
      Fullscreen,
    },
    data() {
      return {
        booted: false,
      }
    },
    computed: {
      layout(): string {
        return (this as any).$route.meta.layout || 'Default'
      }
    }
  })
</script>
