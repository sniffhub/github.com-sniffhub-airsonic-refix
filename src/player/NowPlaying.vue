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
          <!-- Small ambient accent, not a data readout — purely decorative,
               so it's fine that it has no relation to the actual track. -->
          <canvas ref="globeCanvas" class="globe-accent" width="64" height="64" aria-hidden="true" />
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
          <div v-if="modeTransitioning" class="viz-recalibrating">
            RECALIBRATING&hellip;
          </div>
          <div class="viz-readout" aria-hidden="true">
            <div v-for="(line, i) in readoutLines" :key="i">
              {{ line }}
            </div>
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
  import { formatArtists } from '@/shared/utils'

  type VizMode = 'spectrum' | 'lissajous' | 'vfd' | 'dotmatrix' | 'particles'

  const VIZ_MODES: { mode: VizMode, label: string }[] = [
    { mode: 'spectrum', label: 'BARS' },
    { mode: 'lissajous', label: 'XY' },
    { mode: 'vfd', label: 'VFD' },
    { mode: 'dotmatrix', label: 'DOT-MATRIX' },
    { mode: 'particles', label: 'PARTICLES' },
  ]

  // Chunky 5x7 bitmap font for the DOT-MATRIX mode's bottom label — rendered
  // with the same circular-dot technique as the rest of that mode, never a
  // regular text font. '#' = lit cell, '.' = unlit. Unmapped characters fall
  // back to a blank space rather than guessing a glyph.
  const FONT_5X7: Record<string, string[]> = {
    ' ': ['.....', '.....', '.....', '.....', '.....', '.....', '.....'],
    0: ['.###.', '#...#', '#..##', '#.#.#', '##..#', '#...#', '.###.'],
    1: ['..#..', '.##..', '..#..', '..#..', '..#..', '..#..', '.###.'],
    2: ['.###.', '#...#', '....#', '...#.', '..#..', '.#...', '#####'],
    3: ['.###.', '#...#', '....#', '..##.', '....#', '#...#', '.###.'],
    4: ['...#.', '..##.', '.#.#.', '#..#.', '#####', '...#.', '...#.'],
    5: ['#####', '#....', '####.', '....#', '....#', '#...#', '.###.'],
    6: ['..##.', '.#...', '#....', '####.', '#...#', '#...#', '.###.'],
    7: ['#####', '....#', '...#.', '..#..', '.#...', '.#...', '.#...'],
    8: ['.###.', '#...#', '#...#', '.###.', '#...#', '#...#', '.###.'],
    9: ['.###.', '#...#', '#...#', '.####', '....#', '...#.', '.##..'],
    A: ['..#..', '.#.#.', '#...#', '#...#', '#####', '#...#', '#...#'],
    B: ['####.', '#...#', '#...#', '####.', '#...#', '#...#', '####.'],
    C: ['.###.', '#...#', '#....', '#....', '#....', '#...#', '.###.'],
    D: ['####.', '#...#', '#...#', '#...#', '#...#', '#...#', '####.'],
    E: ['#####', '#....', '#....', '####.', '#....', '#....', '#####'],
    F: ['#####', '#....', '#....', '####.', '#....', '#....', '#....'],
    G: ['.###.', '#...#', '#....', '#.###', '#...#', '#...#', '.###.'],
    H: ['#...#', '#...#', '#...#', '#####', '#...#', '#...#', '#...#'],
    I: ['.###.', '..#..', '..#..', '..#..', '..#..', '..#..', '.###.'],
    J: ['..###', '...#.', '...#.', '...#.', '...#.', '#..#.', '.##..'],
    K: ['#...#', '#..#.', '#.#..', '##...', '#.#..', '#..#.', '#...#'],
    L: ['#....', '#....', '#....', '#....', '#....', '#....', '#####'],
    M: ['#...#', '##.##', '#.#.#', '#...#', '#...#', '#...#', '#...#'],
    N: ['#...#', '##..#', '#.#.#', '#..##', '#...#', '#...#', '#...#'],
    O: ['.###.', '#...#', '#...#', '#...#', '#...#', '#...#', '.###.'],
    P: ['####.', '#...#', '#...#', '####.', '#....', '#....', '#....'],
    Q: ['.###.', '#...#', '#...#', '#...#', '#.#.#', '#..#.', '.##.#'],
    R: ['####.', '#...#', '#...#', '####.', '#.#..', '#..#.', '#...#'],
    S: ['.####', '#....', '#....', '.###.', '....#', '....#', '####.'],
    T: ['#####', '..#..', '..#..', '..#..', '..#..', '..#..', '..#..'],
    U: ['#...#', '#...#', '#...#', '#...#', '#...#', '#...#', '.###.'],
    V: ['#...#', '#...#', '#...#', '#...#', '#...#', '.#.#.', '..#..'],
    W: ['#...#', '#...#', '#...#', '#.#.#', '#.#.#', '#.#.#', '.#.#.'],
    X: ['#...#', '#...#', '.#.#.', '..#..', '.#.#.', '#...#', '#...#'],
    Y: ['#...#', '#...#', '.#.#.', '..#..', '..#..', '..#..', '..#..'],
    Z: ['#####', '....#', '...#.', '..#..', '.#...', '#....', '#####'],
    '-': ['.....', '.....', '.....', '#####', '.....', '.....', '.....'],
    ':': ['.....', '..#..', '.....', '.....', '..#..', '.....', '.....'],
    '.': ['.....', '.....', '.....', '.....', '.....', '..#..', '.....'],
    "'": ['..#..', '..#..', '.....', '.....', '.....', '.....', '.....'],
    '/': ['....#', '...#.', '..#..', '.#...', '#....', '.....', '.....'],
    '!': ['..#..', '..#..', '..#..', '..#..', '.....', '..#..', '.....'],
    '?': ['.###.', '#...#', '....#', '..##.', '..#..', '.....', '..#..'],
    '&': ['.##..', '#..#.', '#.#..', '.#...', '#.#.#', '#..#.', '.##.#'],
  }

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

  // PARTICLES mode: the real album art renders as the base layer (cover-fit
  // into the full canvas, no letterboxing), with an audio-reactive particle
  // field as a translucent overlay on top. Grid interval (px, on-screen) at
  // which the art is sampled for overlay motes, and the luminance threshold
  // below which a sampled pixel is skipped entirely (too dark to read).
  const PARTICLE_SAMPLE_STEP = 8
  const PARTICLE_LUMINANCE_MIN = 0.08
  // Audio-reactivity tuning — kept modest so the overlay adds life without
  // swamping the photo; these are the first values to raise if it reads flat.
  const PARTICLE_PULSE_STRENGTH = 0.5 // overall amplitude -> size/brightness boost
  const PARTICLE_MAX_DISPLACEMENT_PX = 3 // per-particle jitter from home position
  // Overlay motes stay translucent so the real photo reads through underneath
  // — this is the "light dust of glowing motes" ceiling, not per-particle alpha.
  const PARTICLE_OVERLAY_OPACITY = 0.45

  // Small rotating wireframe globe accent (decorative only, no audio link):
  // period of one full slow rotation, and a fixed axial tilt so it reads as
  // a sphere rather than a flat spinning disc.
  const GLOBE_ROTATION_PERIOD_MS = 42000
  const GLOBE_TILT_RAD = 0.45

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
        // PARTICLES mode: sampled album-art particles (screen-space x/y,
        // luminance 0-1), built once per track from an offscreen canvas.
        // Non-reactive — rebuilt only when the art image changes, read every
        // frame during draw.
        particlePoints: markRaw([] as { x: number, y: number, lum: number, angle: number }[]),
        particleArtSrc: '',
        // Cached, already-loaded Image for the current track's art — drawn
        // directly as the PARTICLES base layer every frame. Non-reactive.
        particleArtImage: null as HTMLImageElement | null,
        // Pre-rendered radial-gradient glow sprite, built once and reused via
        // drawImage for every particle — avoids per-particle ctx.shadowBlur,
        // which forces a blur recompute on every single draw call and was
        // the dominant cost of PARTICLES mode (shadowBlur, not particle
        // count, was the actual bottleneck).
        particleGlowSprite: null as HTMLCanvasElement | null,
        // VFD peak-hold: per-bar running max (0-1) and remaining hold-frame
        // count before it starts falling. Non-reactive, mutated every frame.
        vfdPeaks: markRaw([] as number[]),
        vfdPeakHold: markRaw([] as number[]),
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
      // Real track info for the DOT-MATRIX bottom marquee — never fabricated
      // placeholder text. Falls back to the mode label when nothing's queued.
      dotMatrixLabel(): string {
        if (!this.track) {
          return 'DOT-MATRIX'
        }
        const title = this.streamTitle || this.track.title
        const artist = this.track.artists.length > 0 ? formatArtists(this.track.artists) : this.track.album
        return artist ? `${title} - ${artist}` : title
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

        // Decorative wireframe globe accent — always spins, independent of
        // viz mode / play state / audio, so it renders every frame here.
        const globeCanvas = this.$refs.globeCanvas as HTMLCanvasElement | undefined
        if (globeCanvas) {
          const gctx = globeCanvas.getContext('2d')
          if (gctx) {
            this.drawGlobe(gctx, globeCanvas.width, globeCanvas.height)
          }
        }

        if (this.vizMode === 'lissajous') {
          // Fading trail instead of a hard clear, for a glowing-phosphor
          // trace look instead of a redrawn-from-scratch frame. Slightly
          // lower alpha than other modes for longer persistence.
          ctx.fillStyle = 'rgba(10, 8, 0, 0.1)'
          ctx.fillRect(0, 0, width, height)
        } else {
          ctx.clearRect(0, 0, width, height)
        }

        // PARTICLES mode (Stage 1): static album-art reconstruction, not yet
        // audio-driven, so it renders regardless of play/pause state.
        if (this.vizMode === 'particles') {
          this.drawParticles(ctx, width, height)
          return
        }

        if (!this.isPlaying) {
          return
        }

        if (this.vizMode === 'spectrum') {
          this.drawSpectrum(ctx, width, height)
        } else if (this.vizMode === 'lissajous') {
          this.drawLissajous(ctx, width, height)
        } else if (this.vizMode === 'vfd') {
          this.drawVfd(ctx, width, height)
        } else {
          this.drawDotMatrix(ctx, width, height)
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
      // PARTICLES mode: sample the current track's album art on an offscreen
      // canvas into a field of grid points (kept as fractions 0-1 of the
      // source image's own dimensions), used to scatter the audio-reactive
      // overlay motes drawn on top of the real photo in drawParticles.
      // Rebuilt only when the art URL changes.
      buildParticles(imgSrc: string, canvasWidth: number, canvasHeight: number) {
        this.particleArtSrc = imgSrc
        this.particlePoints.length = 0
        this.particleArtImage = null
        if (!imgSrc) {
          return
        }

        const img = new Image()
        img.crossOrigin = 'anonymous'
        img.onload = () => {
          // Stale by the time it loads (track changed again) — drop it.
          if (this.particleArtSrc !== imgSrc) {
            return
          }
          this.particleArtImage = markRaw(img)

          const off = document.createElement('canvas')
          off.width = img.naturalWidth
          off.height = img.naturalHeight
          const offCtx = off.getContext('2d')
          if (!offCtx) {
            return
          }
          offCtx.drawImage(img, 0, 0)
          let pixels: ImageData
          try {
            pixels = offCtx.getImageData(0, 0, off.width, off.height)
          } catch (err) {
            // Cross-origin art (no CORS headers) taints the canvas — fail
            // quietly rather than throwing inside an image load handler.
            // The real photo (drawn via drawImage, not getImageData) still
            // renders fine; only the particle overlay is skipped.
            console.warn('PARTICLES: could not read album art pixel data', err)
            return
          }

          // Sample in SCREEN space (canvas pixels), not source-image space,
          // so particle count is bounded by the canvas's own on-screen area
          // (~canvasWidth/step * canvasHeight/step) no matter the art's
          // native resolution. The old image-space step (PARTICLE_SAMPLE_STEP
          // / coverScale) collapsed to 1 whenever the art was smaller than
          // the canvas, sampling every source pixel — e.g. a 300x300 cover
          // produced ~89,000 particles, each drawn twice with shadowBlur
          // every frame, tanking framerate regardless of on-screen size.
          const coverScale = Math.max(canvasWidth / off.width, canvasHeight / off.height)
          const drawWidth = off.width * coverScale
          const drawHeight = off.height * coverScale
          const dx = (canvasWidth - drawWidth) / 2
          const dy = (canvasHeight - drawHeight) / 2

          const points: { x: number, y: number, lum: number, angle: number }[] = []
          for (let sy = 0; sy < canvasHeight; sy += PARTICLE_SAMPLE_STEP) {
            const py = Math.floor((sy - dy) / coverScale)
            if (py < 0 || py >= off.height) {
              continue
            }
            for (let sx = 0; sx < canvasWidth; sx += PARTICLE_SAMPLE_STEP) {
              const px = Math.floor((sx - dx) / coverScale)
              if (px < 0 || px >= off.width) {
                continue
              }
              const idx = (py * off.width + px) * 4
              const r = pixels.data[idx]
              const g = pixels.data[idx + 1]
              const b = pixels.data[idx + 2]
              const lum = (0.299 * r + 0.587 * g + 0.114 * b) / 255
              if (lum < PARTICLE_LUMINANCE_MIN) {
                continue
              }
              // Stable per-particle jitter direction (fixed at build time) so
              // audio-driven displacement reads as organic wobble in a
              // consistent direction, not per-frame noise.
              points.push({ x: sx / canvasWidth, y: sy / canvasHeight, lum, angle: Math.random() * Math.PI * 2 })
            }
          }
          this.particlePoints = markRaw(points)
        }
        img.src = imgSrc
      },
      drawParticles(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const imgSrc = this.track?.image || ''
        if (imgSrc !== this.particleArtSrc) {
          this.buildParticles(imgSrc, width, height)
        }

        const img = this.particleArtImage
        if (!img) {
          // Art hasn't loaded yet (or track has none) — plain backdrop so
          // the canvas isn't left showing stale content from a prior track.
          ctx.fillStyle = 'rgb(10, 8, 0)'
          ctx.fillRect(0, 0, width, height)
          return
        }

        // Base layer: the real album art, cover-fit into the FULL canvas
        // (crop-to-fill, same semantics as the art thumbnail above) — no
        // letterboxing, no square crop, fills the whole width/height.
        const coverScale = Math.max(width / img.naturalWidth, height / img.naturalHeight)
        const drawWidth = img.naturalWidth * coverScale
        const drawHeight = img.naturalHeight * coverScale
        const dx = (width - drawWidth) / 2
        const dy = (height - drawHeight) / 2
        ctx.drawImage(img, dx, dy, drawWidth, drawHeight)

        if (this.particlePoints.length === 0) {
          return
        }

        // Overall amplitude (0-1) from the live frequency data, used to
        // drive PULSE (size/brightness). Zero (no track loaded / paused)
        // collapses the overlay back to its resting size/brightness.
        const data = this.playerStore.getFrequencyData()
        let amp = 0
        if (data.length > 0) {
          let sum = 0
          for (let i = 0; i < data.length; i++) {
            sum += data[i]
          }
          amp = sum / data.length / 255
        }

        const baseDotRadius = Math.max(1, PARTICLE_SAMPLE_STEP * 0.48)
        // PULSE: overall amplitude subtly grows particle size/brightness.
        const dotRadius = baseDotRadius * (1 + amp * PARTICLE_PULSE_STRENGTH)

        const positions: { x: number, y: number, alpha: number }[] = []
        for (const p of this.particlePoints) {
          // DISPLACEMENT: each particle jitters from its home position along
          // a fixed-per-particle direction, scaled by the frequency bin its
          // horizontal position maps to — so different regions of the image
          // react to different parts of the spectrum.
          let jitterX = 0
          let jitterY = 0
          if (data.length > 0) {
            const bin = data[Math.floor(p.x * (data.length - 1))] / 255
            const displace = bin * PARTICLE_MAX_DISPLACEMENT_PX
            jitterX = Math.cos(p.angle) * displace
            jitterY = Math.sin(p.angle) * displace
          }
          positions.push({
            x: p.x * width + jitterX,
            y: p.y * height + jitterY,
            alpha: Math.min(1, (0.4 + p.lum * 0.6 + amp * 0.1) * PARTICLE_OVERLAY_OPACITY),
          })
        }

        // Dark halo pass — a soft, translucent ring behind each particle so
        // the overlay motes read against any photo regardless of its own
        // colors, without going as opaque as the old solid-backdrop version
        // (the photo underneath must stay visible).
        ctx.fillStyle = `rgba(10, 8, 0, ${0.3 * PARTICLE_OVERLAY_OPACITY + 0.1})`
        for (const pos of positions) {
          ctx.beginPath()
          ctx.arc(pos.x, pos.y, dotRadius * 1.7, 0, Math.PI * 2)
          ctx.fill()
        }

        // Bright particle pass, glow baked into a pre-rendered sprite and
        // stamped via drawImage — reads as the same glowing-dust look as
        // ctx.shadowBlur but without recomputing a blur on every draw call.
        if (!this.particleGlowSprite) {
          this.particleGlowSprite = markRaw(this.buildParticleGlowSprite())
        }
        const sprite = this.particleGlowSprite
        const glowDiameter = dotRadius * 3.4
        for (const pos of positions) {
          ctx.globalAlpha = pos.alpha
          ctx.drawImage(sprite, pos.x - glowDiameter / 2, pos.y - glowDiameter / 2, glowDiameter, glowDiameter)
        }
        ctx.globalAlpha = 1
      },
      // Builds the cyan radial-gradient sprite used by the PARTICLES bright
      // pass above — built once and cached, not per frame.
      buildParticleGlowSprite(): HTMLCanvasElement {
        const size = 64
        const c = document.createElement('canvas')
        c.width = size
        c.height = size
        const sctx = c.getContext('2d')!
        const cx = size / 2
        const [r, g, b] = CYAN
        const grad = sctx.createRadialGradient(cx, cx, 0, cx, cx, cx)
        grad.addColorStop(0, `rgba(${r}, ${g}, ${b}, 1)`)
        grad.addColorStop(0.4, `rgba(${r}, ${g}, ${b}, 0.7)`)
        grad.addColorStop(1, `rgba(${r}, ${g}, ${b}, 0)`)
        sctx.fillStyle = grad
        sctx.fillRect(0, 0, size, size)
        return c
      },
      // Small rotating wireframe globe accent — thin amber/cyan latitude
      // (parallel) and longitude (meridian) arcs on a unit sphere, rotated
      // around the vertical axis with a fixed tilt and projected orthographi-
      // cally onto the 2D canvas. Basic trigonometry only, no 3D library —
      // this is a decorative accent, not a real 3D scene.
      drawGlobe(ctx: CanvasRenderingContext2D, width: number, height: number) {
        ctx.clearRect(0, 0, width, height)

        const cx = width / 2
        const cy = height / 2
        const radius = Math.min(width, height) / 2 - 3
        const rotY = this.reducedMotion
          ? 0
          : ((performance.now() % GLOBE_ROTATION_PERIOD_MS) / GLOBE_ROTATION_PERIOD_MS) * Math.PI * 2

        const project = (lat: number, lon: number): { x: number, y: number, z: number } => {
          const x0 = Math.cos(lat) * Math.sin(lon)
          const y0 = Math.sin(lat)
          const z0 = Math.cos(lat) * Math.cos(lon)
          // Spin around the vertical (Y) axis...
          const x1 = x0 * Math.cos(rotY) + z0 * Math.sin(rotY)
          const z1 = -x0 * Math.sin(rotY) + z0 * Math.cos(rotY)
          // ...then a fixed axial tilt, so it reads as a sphere rather than
          // a flat spinning disc.
          const y1 = y0 * Math.cos(GLOBE_TILT_RAD) - z1 * Math.sin(GLOBE_TILT_RAD)
          const z2 = y0 * Math.sin(GLOBE_TILT_RAD) + z1 * Math.cos(GLOBE_TILT_RAD)
          return { x: cx + x1 * radius, y: cy - y1 * radius, z: z2 }
        }

        const SEGMENTS = 16
        const MERIDIANS = 6
        const PARALLELS = 3

        const strokeArc = (points: { x: number, y: number, z: number }[], rgb: [number, number, number]) => {
          const [r, g, b] = rgb
          for (let i = 1; i < points.length; i++) {
            const a = points[i - 1]
            const p = points[i]
            // Depth-based alpha (no real hidden-line removal — just a cheap
            // pseudo-3D read: far-side arcs fade, near-side arcs glow).
            const alpha = 0.12 + 0.5 * ((p.z + radius) / (2 * radius))
            ctx.beginPath()
            ctx.moveTo(a.x, a.y)
            ctx.lineTo(p.x, p.y)
            ctx.strokeStyle = `rgba(${r}, ${g}, ${b}, ${alpha})`
            ctx.lineWidth = 0.75
            ctx.shadowBlur = 2
            ctx.shadowColor = `rgba(${r}, ${g}, ${b}, ${alpha})`
            ctx.stroke()
          }
        }

        for (let m = 0; m < MERIDIANS; m++) {
          const lon = (m / MERIDIANS) * Math.PI * 2
          const points: { x: number, y: number, z: number }[] = []
          for (let s = 0; s <= SEGMENTS; s++) {
            const lat = -Math.PI / 2 + (s / SEGMENTS) * Math.PI
            points.push(project(lat, lon))
          }
          strokeArc(points, m % 2 === 0 ? AMBER : CYAN)
        }

        for (let p = 1; p <= PARALLELS; p++) {
          const lat = -Math.PI / 2 + (p / (PARALLELS + 1)) * Math.PI
          const points: { x: number, y: number, z: number }[] = []
          for (let s = 0; s <= SEGMENTS; s++) {
            const lon = (s / SEGMENTS) * Math.PI * 2
            points.push(project(lat, lon))
          }
          strokeArc(points, p % 2 === 0 ? AMBER : CYAN)
        }

        ctx.shadowBlur = 0
      },
      // Retro car-stereo VFD spectrum analyzer: discrete glowing segments
      // (not a smooth bar) per LED/VFD cell, with a bright peak-hold cap
      // that holds briefly then falls back down like a real VU meter.
      drawVfd(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const data = this.playerStore.getFrequencyData()
        if (data.length === 0) {
          return
        }

        const barCount = 32
        const step = Math.floor(data.length / barCount)
        const barGap = 3
        const barWidth = (width - barGap * (barCount - 1)) / barCount
        const segHeight = 6
        const segGap = 2
        const totalSegments = Math.max(1, Math.floor(height / (segHeight + segGap)))

        if (this.vfdPeaks.length !== barCount) {
          this.vfdPeaks = markRaw(new Array(barCount).fill(0))
          this.vfdPeakHold = markRaw(new Array(barCount).fill(0))
        }

        const HOLD_FRAMES = 24 // ~400ms at 60fps before the peak starts falling
        const FALL_PER_FRAME = 1 / 70 // full-height fall in ~1.1s once it starts

        for (let i = 0; i < barCount; i++) {
          const value = data[i * step] / 255
          const litSegments = Math.round(value * totalSegments)
          const x = i * (barWidth + barGap)

          // Peak-hold: snap up instantly on a new max, hold, then fall.
          if (value >= this.vfdPeaks[i]) {
            this.vfdPeaks[i] = value
            this.vfdPeakHold[i] = HOLD_FRAMES
          } else if (this.vfdPeakHold[i] > 0) {
            this.vfdPeakHold[i]--
          } else {
            this.vfdPeaks[i] = Math.max(value, this.vfdPeaks[i] - FALL_PER_FRAME)
          }
          const peakSegment = Math.max(1, Math.round(this.vfdPeaks[i] * totalSegments))

          for (let s = 0; s < totalSegments; s++) {
            const segY = height - (s + 1) * (segHeight + segGap) + segGap
            const lit = s < litSegments
            const [r, g, b] = lerpColor(s / (totalSegments - 1))

            if (lit) {
              // Punchier bloom than the smooth BARS mode — brighter fill,
              // wider shadow blur, closer to genuine VFD brightness.
              ctx.shadowBlur = 14
              ctx.shadowColor = `rgba(${r}, ${g}, ${b}, 1)`
              ctx.fillStyle = `rgba(${r}, ${g}, ${b}, 1)`
              ctx.fillRect(x, segY, barWidth, segHeight)
            } else {
              // Dim, unlit segment cells — reads as a real segmented
              // display rather than a redrawn shape each frame.
              ctx.shadowBlur = 0
              ctx.fillStyle = `rgba(${r}, ${g}, ${b}, 0.08)`
              ctx.fillRect(x, segY, barWidth, segHeight)
            }
          }

          // Bright single peak-hold cell, drifting down over the lit stack.
          if (this.vfdPeaks[i] > 0.02) {
            const peakY = height - peakSegment * (segHeight + segGap) + segGap
            ctx.shadowBlur = 18
            ctx.shadowColor = 'rgba(255, 250, 230, 1)'
            ctx.fillStyle = 'rgba(255, 250, 230, 1)'
            ctx.fillRect(x, peakY, barWidth, segHeight)
          }
        }
        ctx.shadowBlur = 0
      },
      // Retro car-stereo dot-matrix display: a coarse grid of filled circles
      // (never square pixels — that's the detail that reads as authentic
      // LED/VFD matrix rather than generic pixelation), single cyan-on-black,
      // no amber/gradient. Top rows are an audio-reactive spectrum; the
      // bottom rows are a scrolling marquee of real track info, rendered in
      // the same circular-dot technique via a hand-rolled 5x7 bitmap font.
      drawDotMatrix(ctx: CanvasRenderingContext2D, width: number, height: number) {
        const data = this.playerStore.getFrequencyData()
        if (data.length === 0) {
          return
        }

        const cols = 48
        const rows = 24
        const textRows = 8 // 7px glyph + 1px gap, reserved at the bottom
        const vizRows = rows - textRows
        const cellW = width / cols
        const cellH = height / rows
        const dotRadius = Math.min(cellW, cellH) * 0.38
        const CYAN = '34, 211, 238'

        const step = Math.floor(data.length / cols)
        for (let c = 0; c < cols; c++) {
          const value = data[c * step] / 255
          const lit = Math.round(value * vizRows)
          const cx = c * cellW + cellW / 2
          for (let r = 0; r < vizRows; r++) {
            const rowFromBottom = vizRows - 1 - r
            const on = rowFromBottom < lit
            const cy = r * cellH + cellH / 2
            ctx.beginPath()
            ctx.arc(cx, cy, on ? dotRadius : dotRadius * 0.55, 0, Math.PI * 2)
            if (on) {
              const alpha = Math.min(1, 0.55 + (rowFromBottom / vizRows) * 0.45)
              ctx.shadowBlur = 5
              ctx.shadowColor = `rgba(${CYAN}, 0.9)`
              ctx.fillStyle = `rgba(${CYAN}, ${alpha})`
            } else {
              ctx.shadowBlur = 0
              ctx.fillStyle = `rgba(${CYAN}, 0.06)`
            }
            ctx.fill()
          }
        }
        ctx.shadowBlur = 0

        // Scrolling marquee label, dead rows dimly lit so the whole grid
        // still reads as one continuous matrix rather than two panels.
        const label = this.dotMatrixLabel.toUpperCase()
        const charCols = 6 // 5px glyph + 1px gap
        const textPxWidth = label.length * charCols
        const SCROLL_COLS_PER_SEC = 5
        const scrollable = textPxWidth > cols
        const offset = scrollable
          ? Math.floor((Date.now() / 1000) * SCROLL_COLS_PER_SEC) % (textPxWidth + cols)
          : 0
        const startCol = scrollable ? cols - offset : Math.floor((cols - textPxWidth) / 2)

        for (let r = 0; r < textRows - 1; r++) {
          for (let c = 0; c < cols; c++) {
            const cx = c * cellW + cellW / 2
            const cy = (vizRows + r) * cellH + cellH / 2
            ctx.beginPath()
            ctx.arc(cx, cy, dotRadius * 0.55, 0, Math.PI * 2)
            ctx.fillStyle = `rgba(${CYAN}, 0.06)`
            ctx.fill()
          }
        }
        for (let i = 0; i < label.length; i++) {
          const glyph = FONT_5X7[label[i]] || FONT_5X7[' ']
          const baseCol = startCol + i * charCols
          if (baseCol + 5 < 0 || baseCol >= cols) {
            continue
          }
          for (let gr = 0; gr < 7; gr++) {
            const rowStr = glyph[gr]
            for (let gc = 0; gc < 5; gc++) {
              if (rowStr[gc] !== '#') {
                continue
              }
              const col = baseCol + gc
              if (col < 0 || col >= cols) {
                continue
              }
              const cx = col * cellW + cellW / 2
              const cy = (vizRows + gr) * cellH + cellH / 2
              ctx.beginPath()
              ctx.arc(cx, cy, dotRadius, 0, Math.PI * 2)
              ctx.shadowBlur = 6
              ctx.shadowColor = `rgba(${CYAN}, 1)`
              ctx.fillStyle = `rgba(${CYAN}, 0.95)`
              ctx.fill()
            }
          }
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
    position: relative;
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

  /* Small ambient wireframe-globe accent, corner-badge sized — decorative
     only, never dominant. */
  .globe-accent {
    position: absolute;
    bottom: -12px;
    right: -12px;
    width: 44px;
    height: 44px;
    pointer-events: none;
    z-index: 2;
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
