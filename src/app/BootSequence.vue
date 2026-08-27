<template>
  <div class="boot-sequence">
    <div v-for="(line, i) in shownLines" :key="i" class="boot-line">
      {{ line }}
    </div>
  </div>
</template>

<script lang="ts">
  import { defineComponent } from 'vue'

  const BOOT_LINES = [
    'SYSTEM INITIALIZING...',
    'LOADING LIBRARY...',
    'AMBER PHOSPHOR DISPLAY ONLINE',
    'READY.',
  ]

  export default defineComponent({
    data() {
      return {
        shownLines: [] as string[],
      }
    },
    mounted() {
      const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches

      if (reducedMotion) {
        this.shownLines = [...BOOT_LINES]
        this.$emit('done')
        return
      }

      let i = 0
      const step = () => {
        if (i >= BOOT_LINES.length) {
          setTimeout(() => this.$emit('done'), 200)
          return
        }
        this.shownLines.push(BOOT_LINES[i])
        i += 1
        setTimeout(step, 300)
      }
      step()
    },
  })
</script>

<style scoped>
.boot-sequence {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  padding: 2rem;
  font-family: 'VT323', 'Courier New', monospace;
  font-size: 1.5rem;
  color: #ffb84d;
  background-image:
    linear-gradient(rgba(10, 8, 0, 0.85), rgba(10, 8, 0, 0.85)),
    url('/bg/terraina62.jpeg');
  background-size: cover;
  background-position: center;
}

.boot-line {
  white-space: pre;
}

.boot-line::before {
  content: '> ';
  opacity: 0.7;
}
</style>
