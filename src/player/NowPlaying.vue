<template>
  <div class="now-playing hud-bracket">
    <template v-if="track">
      <!-- Ambient background — decorative only, never intercepts clicks -->
      <div class="now-playing-bg" aria-hidden="true">
        <div class="drift-shape drift-hex drift-a" />
        <div class="drift-shape drift-hex drift-b" />
        <div class="drift-shape drift-grid drift-c" />
        <div class="drift-shape drift-dot drift-d" />
        <div class="drift-shape drift-dot drift-e" />
        <div class="drift-shape drift-dot drift-f" />
      </div>

      <div class="now-playing-content">
        <router-link :to="{ name: 'queue' }" class="now-playing-back">
          <Icon icon="chevron-left" /> QUEUE
        </router-link>

        <div class="now-playing-art">
          <img v-if="track.image" :src="track.image" :alt="track.title">
          <img v-else src="@/shared/assets/fallback.svg" :alt="track.title">
        </div>

        <div class="now-playing-viz-wrap">
          <canvas ref="canvas" class="now-playing-viz" width="900" height="220" />
          <button
            type="button"
            class="viz-mode-toggle hud-bracket"
            :title="`Visualizer mode: ${vizModeLabel} (click to change)`"
            @click="cycleVizMode"
          >
            {{ vizModeLabel }}
          </button>
        </div>

        <!-- Transport controls sit right under the visualizer, not buried
             at the bottom of the screen, so they're reachable without
             hunting or scrolling. -->
        <div class="now-playing-controls">
          <b-button variant="transparent" size="lg" title="Previous" @click="previous">
            <Icon icon="skip-start" />
          </b-button>
          <b-button variant="transparent" size="lg" class="btn-play" title="Play/Pause (Space)" @click="playPause">
            <Icon :icon="isPlaying ? 'pause' : 'play'" />
          </b-button>
          <b-button variant="transparent" size="lg" title="Next" @click="next">
            <Icon icon="skip-end" />
          </b-button>
        </div>

        <div class="now-playing-info">
          <div class="now-playing-title text-truncate">
            {{ streamTitle || track.title }}
          </div>
          <div class="now-playing-artist text-truncate">
            <template v-if="track.artists.length > 0">
              <span v-for="(artist, index) in track.artists" :key="artist.id">
                <span v-if="index > 0">, </span>
                <router-link :to="{ name: 'artist', params: { id: artist.id } }">{{ artist.name }}</router-link>
              </span>
            </template>
            <template v-else-if="track.album">
              {{ track.album }}
            </template>
          </div>
        </div>

        <ProgressBar class="now-playing-progress" />
      </div>
    </template>
    <div v-else class="now-playing-empty">
      NOTHING PLAYING
    </div>
  </div>
</template>
<script lang="ts">
  import { defineComponent } from 'vue'
  import ProgressBar from '@/player/ProgressBar.vue'
  import { usePlayerStore } from '@/player/store'

  type VizMode = 'spectrum' | 'sonar' | 'oscilloscope' | 'lissajous'

  const VIZ_MODES: { mode: VizMode, label: string }[] = [
    { mode: 'spectrum', label: 'BARS' },
    { mode: 'sonar', label: 'SONAR' },
    { mode: 'oscilloscope', label: 'SCOPE' },
    { mode: 'lissajous', label: 'XY' },
  ]

  const AMBER: [number, number, number] = [255, 122, 26]
  const CYAN: [number, number, number] = [34, 211, 238]

  function lerpColor(t: number): [number, number, number] {
    return [
      Math.round(AMBER[0] + (CYAN[0] - AMBER[0]) * t),
      Math.round(AMBER[1] + (CYAN[1] - AMBER[1]) * t),
      Math.round(AMBER[2] + (CYAN[2] - AMBER[2]) * t),
    ]
  }

  export default defineComponent({
    components: {
      ProgressBar,
    },
    setup() {
      return {
        playerStore: usePlayerStore(),
      }
    },
    data() {
      return {
        frame: 0 as number,
        vizModeIndex: 0,
      }
    },
    computed: {
      track() {
        return this.playerStore.track
      },
      isPlaying() {
        return this.playerStore.isPlaying
      },
      streamTitle() {
        return this.playerStore.streamTitle
      },
      vizMode(): VizMode {
        return VIZ_MODES[this.vizModeIndex].mode
      },
      vizModeLabel(): string {
        return VIZ_MODES[this.vizModeIndex].label
      },
    },
    mounted() {
      this.draw()
      window.addEventListener('keydown', this.onKeydown)
    },
    beforeUnmount() {
      cancelAnimationFrame(this.frame)
      window.removeEventListener('keydown', this.onKeydown)
    },
    methods: {
      playPause() {
        return this.playerStore.playPause()
      },
      next() {
        return this.playerStore.next()
      },
      previous() {
        return this.playerStore.previous()
      },
      cycleVizMode() {
        this.vizModeIndex = (this.vizModeIndex + 1) % VIZ_MODES.length
      },
      onKeydown(event: KeyboardEvent) {
        const target = event.target as HTMLElement | null
        const tag = target?.tagName
        if (tag === 'INPUT' || tag === 'TEXTAREA' || target?.isContentEditable) {
          return
        }
        if (event.code === 'Space') {
          event.preventDefault()
          this.playPause()
        }
      },
      draw() {
        this.frame = requestAnimationFrame(this.draw)

        const canvas = this.$refs.canvas as HTMLCanvasElement | undefined
        if (!canvas) {
          return
        }
        const ctx = canvas.getContext('2d')
        if (!ctx) {
          return
        }

        const width = canvas.width
        const height = canvas.height

        if (this.vizMode === 'lissajous') {
          // Fading trail instead of a hard clear, for a glowing-phosphor
          // trace look instead of a redrawn-from-scratch frame.
          ctx.fillStyle = 'rgba(10, 8, 0, 0.15)'
          ctx.fillRect(0, 0, width, height)
        } else {
          ctx.clearRect(0, 0, width, height)
        }

        if (!this.isPlaying) {
          return
        }

        if (this.vizMode === 'spectrum') {
          this.drawSpectrum(ctx, width, height)
        } else if (this.vizMode === 'sonar') {
          this.drawSonar(ctx, width, height)
        } else if (this.vizMode === 'oscilloscope') {
          this.drawOscilloscope(ctx, width, height)
        } else {
          this.drawLissajous(ctx, width, height)
        }
      },
      // Deliberate exception to the single-tone-amber rule used everywhere
      // else in this app (see main.scss:191-198, 368-376) — dual-tone here
      // per explicit request, not an accidental amber/cyan mix.
      drawSpectrum(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const data = this.playerStore.getFrequencyData()
        if (data.length === 0) {
          return
        }

        const barCount = 64
        const step = Math.floor(data.length / barCount)
        const barWidth = width / barCount

        for (let i = 0; i < barCount; i++) {
          const value = data[i * step] / 255
          const barHeight = value * height
          const x = i * barWidth
          const y = height - barHeight

          const [r, g, b] = lerpColor(i / (barCount - 1))

          ctx.shadowBlur = 8
          ctx.shadowColor = `rgba(${r}, ${g}, ${b}, 0.7)`
          ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${0.35 + value * 0.65})`
          ctx.fillRect(x + 1, y, barWidth - 2, barHeight)
        }
      },
      drawSonar(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const data = this.playerStore.getFrequencyData()
        if (data.length === 0) {
          return
        }

        const cx = width / 2
        const cy = height / 2
        const maxRadius = Math.min(width, height) / 2 - 8

        // grid rings + crosshair, amber-dim, matching the RadarSweep widget
        ctx.strokeStyle = 'rgba(255, 122, 26, 0.25)'
        ctx.lineWidth = 1
        for (const frac of [0.25, 0.5, 0.75, 1]) {
          ctx.beginPath()
          ctx.arc(cx, cy, maxRadius * frac, 0, Math.PI * 2)
          ctx.stroke()
        }
        ctx.beginPath()
        ctx.moveTo(cx - maxRadius, cy)
        ctx.lineTo(cx + maxRadius, cy)
        ctx.moveTo(cx, cy - maxRadius)
        ctx.lineTo(cx, cy + maxRadius)
        ctx.stroke()

        // rotating sweep line
        const angle = (Date.now() / 1000) % (Math.PI * 2)
        const sweepX = cx + Math.cos(angle) * maxRadius
        const sweepY = cy + Math.sin(angle) * maxRadius
        const sweepGradient = ctx.createLinearGradient(cx, cy, sweepX, sweepY)
        sweepGradient.addColorStop(0, 'rgba(255, 122, 26, 0)')
        sweepGradient.addColorStop(1, 'rgba(255, 122, 26, 0.9)')
        ctx.strokeStyle = sweepGradient
        ctx.lineWidth = 2
        ctx.beginPath()
        ctx.moveTo(cx, cy)
        ctx.lineTo(sweepX, sweepY)
        ctx.stroke()

        // frequency bins plotted as blips around the circle
        const binCount = 48
        const step = Math.floor(data.length / binCount)
        for (let i = 0; i < binCount; i++) {
          const value = data[i * step] / 255
          if (value < 0.08) {
            continue
          }
          const binAngle = (i / binCount) * Math.PI * 2
          const radius = maxRadius * (0.15 + value * 0.85)
          const x = cx + Math.cos(binAngle) * radius
          const y = cy + Math.sin(binAngle) * radius
          const [r, g, b] = lerpColor(value)

          ctx.beginPath()
          ctx.arc(x, y, 2 + value * 3, 0, Math.PI * 2)
          ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${0.4 + value * 0.6})`
          ctx.shadowBlur = 6
          ctx.shadowColor = `rgba(${r}, ${g}, ${b}, 0.8)`
          ctx.fill()
        }
      },
      drawOscilloscope(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const data = this.playerStore.getTimeDomainData()
        if (data.length === 0) {
          return
        }

        const cx = width / 2
        const cy = height / 2
        const maxRadius = Math.min(width, height) / 2 - 8

        // circular grid, matching the sonar mode / Smith-chart reference look
        ctx.strokeStyle = 'rgba(34, 211, 238, 0.2)'
        ctx.lineWidth = 1
        for (const frac of [0.33, 0.66, 1]) {
          ctx.beginPath()
          ctx.arc(cx, cy, maxRadius * frac, 0, Math.PI * 2)
          ctx.stroke()
        }
        ctx.beginPath()
        ctx.moveTo(cx - maxRadius, cy)
        ctx.lineTo(cx + maxRadius, cy)
        ctx.moveTo(cx, cy - maxRadius)
        ctx.lineTo(cx, cy + maxRadius)
        ctx.stroke()

        // waveform line across the middle
        ctx.beginPath()
        const sliceWidth = width / data.length
        let x = 0
        for (let i = 0; i < data.length; i++) {
          const v = (data[i] - 128) / 128
          const y = cy + v * (height / 2 - 10)
          if (i === 0) {
            ctx.moveTo(x, y)
          } else {
            ctx.lineTo(x, y)
          }
          x += sliceWidth
        }
        ctx.strokeStyle = 'rgba(34, 211, 238, 0.9)'
        ctx.shadowBlur = 10
        ctx.shadowColor = 'rgba(34, 211, 238, 0.8)'
        ctx.lineWidth = 2
        ctx.stroke()
      },
      // True XY/Lissajous oscilloscope mode: X = left channel amplitude,
      // Y = right channel amplitude, plotted against each other rather than
      // against time. This is the classic "oscilloscope music" technique.
      // Honest expectation: with ordinary stereo music (not audio specially
      // composed so left/right trace a deliberate picture, e.g. Jerobeam
      // Fenderson-style tracks), this produces flowing, organic geometric
      // loops and swirls that shift with the stereo image and bass content —
      // not a literal picture. Nothing in this library is mastered for that,
      // so the swirling-loop look is the expected, correct result here.
      drawLissajous(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const { left, right } = this.playerStore.getStereoTimeDomainData()
        if (left.length === 0 || right.length === 0) {
          return
        }

        const cx = width / 2
        const cy = height / 2
        const scale = Math.min(width, height) / 2 - 12

        ctx.beginPath()
        for (let i = 0; i < left.length; i++) {
          const x = cx + ((left[i] - 128) / 128) * scale
          const y = cy + ((right[i] - 128) / 128) * scale
          if (i === 0) {
            ctx.moveTo(x, y)
          } else {
            ctx.lineTo(x, y)
          }
        }
        ctx.strokeStyle = 'rgba(255, 122, 26, 0.85)'
        ctx.shadowBlur = 12
        ctx.shadowColor = 'rgba(255, 122, 26, 0.9)'
        ctx.lineWidth = 1.5
        ctx.stroke()
      },
    },
  })
</script>
<style scoped>
  .now-playing {
    position: relative;
    min-height: calc(100vh - 140px);
    overflow: hidden;
  }

  /* --- Ambient background: decorative only, z-index below content, never
     intercepts clicks. GPU-cheap CSS transforms only, no canvas. --- */
  .now-playing-bg {
    position: absolute;
    inset: 0;
    z-index: 0;
    pointer-events: none;
    overflow: hidden;
  }

  .drift-shape {
    position: absolute;
    opacity: 0.12;
  }

  .drift-hex {
    width: 120px;
    height: 104px;
    border: 1px solid var(--term-amber-dim);
    clip-path: polygon(25% 0%, 75% 0%, 100% 50%, 75% 100%, 25% 100%, 0% 50%);
  }
  .drift-a { top: 8%; left: 6%; animation: drift-a 42s ease-in-out infinite; }
  .drift-b {
    top: 62%;
    right: 8%;
    width: 80px;
    height: 70px;
    border-color: var(--term-cyan-dim, var(--term-amber-dim));
    animation: drift-b 36s ease-in-out infinite;
  }

  .drift-grid {
    top: 22%;
    right: 12%;
    width: 90px;
    height: 90px;
    background-image:
      linear-gradient(var(--term-amber-dim) 1px, transparent 1px),
      linear-gradient(90deg, var(--term-amber-dim) 1px, transparent 1px);
    background-size: 22px 22px;
    animation: drift-c 50s ease-in-out infinite;
  }

  .drift-dot {
    width: 4px;
    height: 4px;
    border-radius: 50%;
    background: var(--term-cyan, var(--term-amber));
    opacity: 0.25;
  }
  .drift-d { top: 30%; left: 20%; animation: drift-d 28s ease-in-out infinite; }
  .drift-e { top: 75%; left: 40%; animation: drift-e 33s ease-in-out infinite; }
  .drift-f { top: 15%; right: 30%; background: var(--term-amber); animation: drift-f 39s ease-in-out infinite; }

  @keyframes drift-a {
    0%, 100% { transform: translate(0, 0) rotate(0deg); }
    50% { transform: translate(24px, 18px) rotate(8deg); }
  }
  @keyframes drift-b {
    0%, 100% { transform: translate(0, 0) rotate(0deg); }
    50% { transform: translate(-20px, -16px) rotate(-6deg); }
  }
  @keyframes drift-c {
    0%, 100% { transform: translate(0, 0); }
    50% { transform: translate(-16px, 20px); }
  }
  @keyframes drift-d {
    0%, 100% { transform: translate(0, 0); }
    50% { transform: translate(16px, -22px); }
  }
  @keyframes drift-e {
    0%, 100% { transform: translate(0, 0); }
    50% { transform: translate(-18px, -14px); }
  }
  @keyframes drift-f {
    0%, 100% { transform: translate(0, 0); }
    50% { transform: translate(14px, 16px); }
  }

  @media (prefers-reduced-motion: reduce) {
    .drift-shape {
      animation: none !important;
    }
  }

  /* --- Real content: always above the ambient background --- */
  .now-playing-content {
    position: relative;
    z-index: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 1.5rem 1rem 1rem;
  }

  .now-playing-back {
    align-self: flex-start;
    color: var(--term-amber-dim);
    text-transform: uppercase;
    letter-spacing: 0.08em;
    font-size: 0.85rem;
    margin-bottom: 1rem;
  }
  .now-playing-back:hover {
    color: var(--term-amber);
  }

  .now-playing-art {
    width: min(200px, 40vw);
    aspect-ratio: 1 / 1;
    border: 1px solid var(--term-amber-dim);
    padding: 4px;
  }
  .now-playing-art img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
  }

  .now-playing-viz-wrap {
    position: relative;
    width: 100%;
    max-width: 900px;
    margin-top: 1rem;
  }

  .now-playing-viz {
    width: 100%;
    height: 220px;
    display: block;
  }

  .viz-mode-toggle {
    position: absolute;
    top: 8px;
    right: 8px;
    z-index: 2;
    background: rgba(10, 8, 0, 0.6);
    border: 1px solid var(--term-amber-dim);
    color: var(--term-amber-dim);
    font-size: 0.72rem;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    padding: 0.3rem 0.6rem;
    cursor: pointer;
  }
  .viz-mode-toggle:hover {
    color: var(--term-amber);
    border-color: var(--term-amber);
  }

  /* Transport controls sit directly under the visualizer — large, central,
     reachable without hunting or scrolling. */
  .now-playing-controls {
    position: relative;
    z-index: 1;
    display: flex;
    align-items: center;
    gap: 1.5rem;
    margin-top: 1.25rem;
  }
  .btn-play {
    --bs-btn-font-size: 2.5rem;
  }

  .now-playing-info {
    position: relative;
    z-index: 1;
    text-align: center;
    margin-top: 1.25rem;
    max-width: 90%;
  }
  .now-playing-title {
    font-size: 1.4rem;
    text-transform: uppercase;
    letter-spacing: 0.04em;
  }
  .now-playing-artist {
    color: var(--term-amber-dim);
    margin-top: 0.25rem;
  }
  .now-playing-artist a {
    color: inherit;
  }

  .now-playing-progress {
    position: relative;
    z-index: 1;
    width: 100%;
    max-width: 600px;
    margin-top: 1rem;
  }

  .now-playing-empty {
    position: relative;
    z-index: 1;
    margin: auto;
    color: var(--term-amber-dim);
    text-transform: uppercase;
    letter-spacing: 0.08em;
  }
</style>
