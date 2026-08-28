<template>
  <div
    class="boot-sequence"
    :class="{ glitch: glitching }"
    tabindex="0"
    @click="skip"
    @keydown="skip"
  >
    <div class="boot-lines">
      <div v-for="(line, i) in completedLines" :key="i" class="boot-line">
        {{ line }}
      </div>
      <div v-if="typingLine" class="boot-line">
        {{ typingLine }}<span class="cursor">&#9608;</span>
      </div>
    </div>
    <div class="logo-wrap" :class="{ show: showLogo }">
      <div class="logo">AMBER//OS</div>
      <div class="logo-sub">retro audio terminal</div>
    </div>
    <div v-if="!showLogo" class="skip-hint">press any key to skip</div>
  </div>
</template>

<script lang="ts">
  import { defineComponent } from 'vue'

  const BOOT_LINES = [
    'SYSTEM INITIALIZING...',
    'LOADING AUDIO ENGINE...',
    'CALIBRATING VISUALIZER...',
    'MOUNTING LIBRARY VOLUME...',
    'LIBRARY: 878 TRACKS FOUND',
    'AMBER PHOSPHOR DISPLAY ONLINE',
    'READY.',
  ]

  const CHAR_DELAY = 12 // ms per character while "typing" a line
  const LINE_PAUSE = 90 // ms pause after a line finishes before the next starts
  const GLITCH_AFTER_LINE = 4 // trigger the glitch flash right after this line index completes
  const GLITCH_DURATION = 160 // ms
  const LOGO_GLOW_DURATION = 900 // ms, must match the CSS transition below
  const DONE_HOLD = 400 // ms to hold on the fully-lit logo before emitting done

  export default defineComponent({
    emits: ['done'],
    data() {
      return {
        completedLines: [] as string[],
        typingLine: '',
        glitching: false,
        showLogo: false,
        skipped: false,
        timers: [] as ReturnType<typeof setTimeout>[],
      }
    },
    mounted() {
      const reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches

      if (reducedMotion) {
        this.completedLines = [...BOOT_LINES]
        this.showLogo = true
        this.finish()
        return
      }

      this.runSequence()
    },
    beforeUnmount() {
      this.clearTimers()
    },
    methods: {
      schedule(fn: () => void, delay: number) {
        const id = setTimeout(fn, delay)
        this.timers.push(id)
        return id
      },
      clearTimers() {
        this.timers.forEach(id => clearTimeout(id))
        this.timers = []
      },
      async runSequence() {
        for (let lineIndex = 0; lineIndex < BOOT_LINES.length; lineIndex++) {
          if (this.skipped) return
          await this.typeLine(BOOT_LINES[lineIndex])
          if (this.skipped) return
          this.completedLines.push(BOOT_LINES[lineIndex])
          this.typingLine = ''

          if (lineIndex === GLITCH_AFTER_LINE) {
            await this.runGlitch()
            if (this.skipped) return
          }

          await this.wait(LINE_PAUSE)
        }

        if (this.skipped) return
        this.showLogo = true
        this.schedule(() => this.finish(), LOGO_GLOW_DURATION + DONE_HOLD)
      },
      typeLine(line: string): Promise<void> {
        return new Promise(resolve => {
          let i = 0
          const step = () => {
            if (this.skipped) {
              resolve()
              return
            }
            i += 1
            this.typingLine = line.slice(0, i)
            if (i >= line.length) {
              resolve()
              return
            }
            this.schedule(step, CHAR_DELAY)
          }
          step()
        })
      },
      runGlitch(): Promise<void> {
        return new Promise(resolve => {
          this.glitching = true
          this.schedule(() => {
            this.glitching = false
            resolve()
          }, GLITCH_DURATION)
        })
      },
      wait(delay: number): Promise<void> {
        return new Promise(resolve => this.schedule(resolve, delay))
      },
      skip() {
        if (this.skipped) return
        this.skipped = true
        this.clearTimers()
        this.completedLines = [...BOOT_LINES]
        this.typingLine = ''
        this.glitching = false
        this.showLogo = true
        this.finish()
      },
      finish() {
        this.$emit('done')
      },
    },
  })
</script>

<style scoped>
.boot-sequence {
  position: relative;
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
  cursor: pointer;
  outline: none;
  overflow: hidden;
}

.boot-lines {
  min-height: 0;
}

.boot-line {
  white-space: pre;
}

.boot-line::before {
  content: '> ';
  opacity: 0.7;
}

.cursor {
  display: inline-block;
  margin-left: 2px;
  animation: blink 0.8s steps(1) infinite;
}

@keyframes blink {
  50% { opacity: 0; }
}

.logo-wrap {
  position: absolute;
  inset: 0;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transform: scale(0.96);
  transition: opacity 0.9s ease-out, transform 0.9s ease-out;
  pointer-events: none;
}

.logo-wrap.show {
  opacity: 1;
  transform: scale(1);
}

.logo {
  font-size: 3.5rem;
  letter-spacing: 0.15em;
  color: #ffcc66;
  text-shadow:
    0 0 8px rgba(255, 184, 77, 0.8),
    0 0 24px rgba(255, 153, 51, 0.6),
    0 0 48px rgba(255, 153, 51, 0.35);
}

.logo-sub {
  margin-top: 0.5rem;
  font-size: 1.1rem;
  opacity: 0.7;
  letter-spacing: 0.3em;
  text-transform: uppercase;
}

.skip-hint {
  position: absolute;
  bottom: 1.25rem;
  right: 1.5rem;
  font-size: 0.9rem;
  opacity: 0.4;
}

/* Brief flash/glitch: subtle RGB-split + flicker, not jarring */
.boot-sequence.glitch {
  animation: glitch-flicker 0.16s steps(2, end) 1;
}

.boot-sequence.glitch .boot-lines {
  animation: glitch-shift 0.16s steps(2, end) 1;
}

@keyframes glitch-flicker {
  0% { filter: brightness(1) saturate(1); }
  30% { filter: brightness(1.6) saturate(1.4); }
  60% { filter: brightness(0.7) saturate(1.2); }
  100% { filter: brightness(1) saturate(1); }
}

@keyframes glitch-shift {
  0% { transform: translateX(0); }
  25% { transform: translateX(-3px); }
  50% { transform: translateX(2px); }
  75% { transform: translateX(-1px); }
  100% { transform: translateX(0); }
}

@media (prefers-reduced-motion: reduce) {
  .logo-wrap {
    transition: none;
  }
  .cursor {
    animation: none;
  }
}
</style>
