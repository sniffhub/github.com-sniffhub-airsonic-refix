<template>
  <div class="now-playing hud-bracket">
    <template v-if="track">
      <!-- Ambient background — decorative only, never intercepts clicks.
           Each parallax layer gets its own mouse-driven translate; the
           drift-* keyframe animation stays on the inner .drift-shape so
           both transforms can coexist (a single element can't carry both
           a CSS animation and a JS-driven inline transform at once). -->
      <div class="now-playing-bg" aria-hidden="true">
        <div class="parallax-layer" :style="parallaxStyle(0.4)">
          <div class="drift-shape drift-hex drift-a" />
          <div class="drift-shape drift-dot drift-d" />
        </div>
        <div class="parallax-layer" :style="parallaxStyle(0.8)">
          <div class="drift-shape drift-hex drift-b" />
          <div class="drift-shape drift-dot drift-e" />
        </div>
        <div class="parallax-layer" :style="parallaxStyle(1.3)">
          <div class="drift-shape drift-grid drift-c" />
          <div class="drift-shape drift-dot drift-f" />
        </div>
      </div>

      <!-- Slow vertical scan sweep — distinct from the static CRT scanline
           texture; this one actually moves, occasionally, across everything. -->
      <div class="scan-sweep" aria-hidden="true" />

      <div class="now-playing-content">
        <router-link :to="{ name: 'queue' }" class="now-playing-back">
          <Icon icon="chevron-left" /> QUEUE
        </router-link>

        <div class="now-playing-art">
          <img v-if="track.image" :src="track.image" :alt="track.title">
          <img v-else src="@/shared/assets/fallback.svg" :alt="track.title">
        </div>

        <div class="now-playing-viz-wrap" :class="{ 'mode-glitch': modeTransitioning }">
          <canvas ref="canvas" class="now-playing-viz" width="900" height="220" />
          <button
            type="button"
            class="viz-mode-toggle hud-bracket"
            :title="`Visualizer mode: ${vizModeLabel} (click to change)`"
            @click="cycleVizMode"
          >
            {{ vizModeLabel }}
          </button>
          <div v-if="modeTransitioning" class="viz-recalibrating">RECALIBRATING&hellip;</div>
          <div class="viz-readout" aria-hidden="true">
            <div v-for="(line, i) in readoutLines" :key="i">{{ line }}</div>
          </div>
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
  import { defineComponent, markRaw } from 'vue'
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

  // How long a Lissajous trail point stays visible before it's pruned, and
  // how it ages through the amber→cyan gradient over that lifetime.
  const LISSAJOUS_TRAIL_MS = 900
  const MODE_GLITCH_MS = 180 // matches BootSequence's glitch-flicker timing

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
        // Non-reactive: mutated every animation frame, so it's kept out of
        // Vue's reactivity system with markRaw to avoid proxy overhead.
        lissajousHistory: markRaw([] as { x: number, y: number, t: number }[]),
        modeTransitioning: false,
        modeTransitionTimer: 0 as ReturnType<typeof setTimeout> | 0,
        parallaxX: 0,
        parallaxY: 0,
        reducedMotion: false,
        // Real facts about the running audio graph (sampleRate/fftSize),
        // fetched once the audio pipeline exists — never fabricated.
        audioInfo: { sampleRate: 0, fftSize: 0 },
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
      // Corner data-readout HUD — only genuinely available values, no
      // invented numbers (there's no BPM/bitrate field on Track).
      readoutLines(): string[] {
        const lines = [`T+${this.formatClock(this.playerStore.currentTime)} / ${this.formatClock(this.playerStore.duration)}`]
        if (this.audioInfo.sampleRate > 0) {
          lines.push(`FFT ${this.audioInfo.fftSize} @ ${this.audioInfo.sampleRate}Hz`)
        }
        if (this.playerStore.queue && this.playerStore.queueIndex > -1) {
          lines.push(`TRK ${this.playerStore.queueIndex + 1}/${this.playerStore.queue.length}`)
        }
        const rg = this.track?.replayGain
        if (rg && Number.isFinite(rg.trackGain)) {
          lines.push(`RG ${rg.trackGain >= 0 ? '+' : ''}${rg.trackGain.toFixed(1)}dB`)
        }
        return lines
      },
    },
    mounted() {
      this.reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches
      this.audioInfo = this.playerStore.getAudioInfo()
      this.draw()
      window.addEventListener('keydown', this.onKeydown)
      if (!this.reducedMotion) {
        window.addEventListener('mousemove', this.onMousemove)
      }
    },
    beforeUnmount() {
      cancelAnimationFrame(this.frame)
      if (this.modeTransitionTimer) {
        clearTimeout(this.modeTransitionTimer)
      }
      window.removeEventListener('keydown', this.onKeydown)
      window.removeEventListener('mousemove', this.onMousemove)
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
        this.lissajousHistory.length = 0
        if (this.reducedMotion) {
          return
        }
        this.modeTransitioning = true
        if (this.modeTransitionTimer) {
          clearTimeout(this.modeTransitionTimer)
        }
        this.modeTransitionTimer = setTimeout(() => {
          this.modeTransitioning = false
        }, MODE_GLITCH_MS)
      },
      formatClock(seconds: number): string {
        if (!Number.isFinite(seconds) || seconds < 0) {
          return '--:--'
        }
        const m = Math.floor(seconds / 60)
        const s = Math.floor(seconds % 60)
        return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`
      },
      onMousemove(event: MouseEvent) {
        this.parallaxX = (event.clientX / window.innerWidth - 0.5) * 2
        this.parallaxY = (event.clientY / window.innerHeight - 0.5) * 2
      },
      parallaxStyle(depth: number) {
        if (this.reducedMotion) {
          return {}
        }
        const x = (this.parallaxX * depth * 18).toFixed(1)
        const y = (this.parallaxY * depth * 18).toFixed(1)
        return { transform: `translate(${x}px, ${y}px)` }
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
          // trace look instead of a redrawn-from-scratch frame. Slightly
          // lower alpha than other modes for longer persistence.
          ctx.fillStyle = 'rgba(10, 8, 0, 0.1)'
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
        const now = Date.now()

        // Append this frame's points to the running history (downsampled —
        // full fftSize resolution isn't needed for a persistence trail and
        // would make the per-frame redraw below far too expensive), tagged
        // with their birth time so age drives the color.
        for (let i = 0; i < left.length; i += 4) {
          this.lissajousHistory.push({
            x: cx + ((left[i] - 128) / 128) * scale,
            y: cy + ((right[i] - 128) / 128) * scale,
            t: now,
          })
        }
        // Prune anything older than the trail lifetime.
        const cutoff = now - LISSAJOUS_TRAIL_MS
        let start = 0
        while (start < this.lissajousHistory.length && this.lissajousHistory[start].t < cutoff) {
          start++
        }
        if (start > 0) {
          this.lissajousHistory.splice(0, start)
        }

        // Redrawing one stroke per point pair would mean thousands of
        // stroke() calls a frame. Instead, bucket points into a handful of
        // age bands and draw each band as a single path — cheap, and still
        // reads as a smooth amber→cyan gradient along the trace as it ages.
        const history = this.lissajousHistory
        const BUCKETS = 6
        ctx.lineWidth = 1.5
        for (let b = 0; b < BUCKETS; b++) {
          const ageLo = b / BUCKETS
          const ageHi = (b + 1) / BUCKETS
          const [r, g, b2] = lerpColor((ageLo + ageHi) / 2)
          const alpha = 0.85 * (1 - ageLo)
          ctx.beginPath()
          let drawing = false
          for (let i = 0; i < history.length; i++) {
            const age = (now - history[i].t) / LISSAJOUS_TRAIL_MS
            if (age >= ageLo && age < ageHi) {
              if (drawing) ctx.lineTo(history[i].x, history[i].y)
              else { ctx.moveTo(history[i].x, history[i].y); drawing = true }
            } else {
              drawing = false
            }
          }
          ctx.strokeStyle = `rgba(${r}, ${g}, ${b2}, ${alpha})`
          ctx.shadowBlur = 10 * (1 - ageLo) + 2
          ctx.shadowColor = `rgba(${r}, ${g}, ${b2}, ${alpha})`
          ctx.stroke()
        }

        // Leading-edge glow particle at the newest point.
        const head = history[history.length - 1]
        if (head) {
          ctx.beginPath()
          ctx.arc(head.x, head.y, 2.5, 0, Math.PI * 2)
          ctx.fillStyle = 'rgba(255, 200, 140, 0.95)'
          ctx.shadowBlur = 16
          ctx.shadowColor = 'rgba(255, 160, 60, 1)'
          ctx.fill()
        }
        ctx.shadowBlur = 0
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

  .parallax-layer {
    position: absolute;
    inset: 0;
    transition: transform 0.4s ease-out;
    will-change: transform;
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
    .parallax-layer {
      transition: none;
    }
  }

  /* Slow vertical scan sweep: sits above everything, occasional and subtle,
     distinct from the app's static CRT scanline texture — this one moves. */
  .scan-sweep {
    position: absolute;
    inset: 0;
    z-index: 5;
    pointer-events: none;
    overflow: hidden;
  }
  .scan-sweep::before {
    content: '';
    position: absolute;
    left: 0;
    right: 0;
    height: 40px;
    top: -40px;
    background: linear-gradient(
      to bottom,
      rgba(255, 184, 77, 0) 0%,
      rgba(255, 184, 77, 0.06) 50%,
      rgba(255, 184, 77, 0) 100%
    );
    animation: scan-sweep-move 9s linear infinite;
  }
  @keyframes scan-sweep-move {
    0% { transform: translateY(0); }
    22% { transform: translateY(calc(100vh + 40px)); }
    100% { transform: translateY(calc(100vh + 40px)); }
  }
  @media (prefers-reduced-motion: reduce) {
    .scan-sweep {
      display: none;
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

  /* "Recalibrating" glitch transition on visualizer mode switch — mirrors
     BootSequence's glitch-flicker/glitch-shift timing (0.16s steps(2)). */
  .now-playing-viz-wrap.mode-glitch {
    animation: viz-glitch-flicker 0.18s steps(2, end) 1;
  }
  .now-playing-viz-wrap.mode-glitch .now-playing-viz {
    animation: viz-glitch-shift 0.18s steps(2, end) 1;
  }
  @keyframes viz-glitch-flicker {
    0% { filter: brightness(1) saturate(1); }
    30% { filter: brightness(1.6) saturate(1.4); }
    60% { filter: brightness(0.7) saturate(1.2); }
    100% { filter: brightness(1) saturate(1); }
  }
  @keyframes viz-glitch-shift {
    0% { transform: translateX(0); }
    25% { transform: translateX(-3px); }
    50% { transform: translateX(2px); }
    75% { transform: translateX(-1px); }
    100% { transform: translateX(0); }
  }
  .viz-recalibrating {
    position: absolute;
    inset: 0;
    z-index: 3;
    display: flex;
    align-items: center;
    justify-content: center;
    pointer-events: none;
    font-family: 'VT323', 'Courier New', monospace;
    font-size: 1.1rem;
    letter-spacing: 0.15em;
    text-transform: uppercase;
    color: var(--term-amber);
    text-shadow: 0 0 8px rgba(255, 184, 77, 0.8);
    background: rgba(10, 8, 0, 0.25);
  }

  .viz-readout {
    position: absolute;
    left: 8px;
    bottom: 8px;
    z-index: 2;
    pointer-events: none;
    font-family: 'VT323', 'Courier New', monospace;
    font-size: 0.78rem;
    line-height: 1.3;
    letter-spacing: 0.04em;
    color: var(--term-amber-dim);
    opacity: 0.65;
    text-shadow: 0 0 4px rgba(255, 122, 26, 0.4);
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
