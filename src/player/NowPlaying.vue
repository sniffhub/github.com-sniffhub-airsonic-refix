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

        <div class="now-playing-body">
          <!-- Instrument panel stack — decorative accents, occupy the left
               column only, never overlap the readable info on the right.
               Hidden below a width where there's no room for them without
               crowding the real content. -->
          <div class="side-panels" aria-hidden="true">
            <!-- Smith-chart tunnel fly-through, unchanged apart from the
                 planet/jet landmarks moving out into their own panels below. -->
            <div ref="smithTunnelWrap" class="smith-tunnel-wrap hud-bracket">
              <canvas ref="smithTunnelCanvas" class="smith-tunnel-canvas" />
            </div>
            <!-- Two small standalone instrument panels, each its own scene/
                 renderer/render-loop — genuinely independent of the tunnel,
                 not landmarks riding along its path any more. -->
            <div class="mini-panels">
              <div ref="planetPanelWrap" class="mini-panel-wrap hud-bracket">
                <canvas ref="planetPanelCanvas" class="mini-panel-canvas" />
              </div>
              <div ref="abstractPanelWrap" class="mini-panel-wrap hud-bracket">
                <canvas ref="abstractPanelCanvas" class="mini-panel-canvas" />
                <button
                  type="button"
                  class="mini-panel-toggle hud-bracket"
                  :title="`Scene: ${abstractSceneLabel} (click to change)`"
                  @click="cycleAbstractScene"
                >
                  {{ abstractSceneLabel }}
                </button>
              </div>
            </div>
          </div>

          <div class="now-playing-main">
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
        </div>
      </div>
    </template>
    <div v-else class="now-playing-empty">
      NOTHING PLAYING
    </div>
  </div>
</template>
<script lang="ts">
  import { defineComponent, markRaw } from 'vue'
  import * as THREE from 'three'
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

  // Ambient color drift (Smith tunnel, planet, and crystal panels only) — a
  // slow, small, purely time-driven sine wander layered on top of each
  // element's fixed lerpColor(t) hue, so those panels read as "quietly
  // alive" rather than frozen on one static color forever. Never tied to
  // audio data — elapsed time only.
  const COLOR_DRIFT_PERIOD_S = 25
  const COLOR_DRIFT_AMOUNT = 0.16
  // How strongly the tunnel's shared vertex-color gradient gets tinted by
  // the drift (tunnel stations bake their gradient into vertex colors, so
  // the drift there is a multiplicative tint layered on top rather than a
  // direct material color swap like the other two panels get).
  const TUNNEL_TINT_STRENGTH = 0.3

  function colorDriftPhase(elapsedSeconds: number): number {
    return Math.sin(elapsedSeconds * (Math.PI * 2 / COLOR_DRIFT_PERIOD_S))
  }

  // Re-centers every material found under `object` on its own base hue
  // (`baseT`) plus the current drift offset — used for the planet panel,
  // the crystal scene, and the tunnel's core landmark/starfield, all of
  // which store their visible color directly on `material.color`.
  function applyColorDrift(object: THREE.Object3D, baseT: number, elapsedSeconds: number) {
    const t = THREE.MathUtils.clamp(baseT + colorDriftPhase(elapsedSeconds) * COLOR_DRIFT_AMOUNT, 0, 1)
    const [r, g, b] = lerpColor(t)
    const seen = new Set<THREE.Material>()
    object.traverse((obj) => {
      const material = (obj as THREE.Mesh | THREE.LineSegments | THREE.LineLoop | THREE.Points).material as
        (THREE.Material & { color?: THREE.Color }) | undefined
      if (material?.color && !seen.has(material)) {
        seen.add(material)
        material.color.setRGB(r / 255, g / 255, b / 255)
      }
    })
  }

  // Tunnel stations bake their resistance-ring gradient into vertex colors
  // (see buildSmithStationGeometry) and leave `material.color` white, since
  // Three.js multiplies vertex color by material color — so drift there is
  // a gentle multiplicative wash toward amber/cyan rather than a direct hue
  // swap, applied uniformly across every station so the whole tunnel breathes
  // in sync.
  function tunnelDriftTint(elapsedSeconds: number): [number, number, number] {
    const t = 0.5 + 0.5 * colorDriftPhase(elapsedSeconds)
    const [r, g, b] = lerpColor(t)
    const maxC = Math.max(r, g, b, 1)
    return [
      1 - TUNNEL_TINT_STRENGTH * (1 - r / maxC),
      1 - TUNNEL_TINT_STRENGTH * (1 - g / maxC),
      1 - TUNNEL_TINT_STRENGTH * (1 - b / maxC),
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

  // Smith-chart tunnel fly-through (left-column accent, Now Playing screen
  // only). Cross-section reproduces the Smith chart's constant-resistance
  // circle family — circles of radius 1/(1+r) centered at (r/(1+r), 0),
  // all tangent at x=radius — plus a fan of radial spokes from the origin,
  // styled amber->cyan instead of the reference image's literal green.
  const SMITH_TUNNEL_RESISTANCES = [0, 0.3, 0.7, 1.5, 3]
  const SMITH_TUNNEL_CIRCLE_SEGMENTS = 40
  const SMITH_TUNNEL_SPOKE_COUNT = 8
  const SMITH_TUNNEL_SCALE = 3 // world units per chart unit
  const SMITH_TUNNEL_STATION_SPACING = 5
  // Raised from 10: more stations in flight at once so several rings are
  // always visible receding into the distance, not just a couple.
  const SMITH_TUNNEL_STATION_COUNT = 20
  const SMITH_TUNNEL_SPEED = 3 // world units/sec camera travel
  const SMITH_TUNNEL_LOOP_LENGTH = SMITH_TUNNEL_STATION_COUNT * SMITH_TUNNEL_STATION_SPACING
  // Depth cues applied per-station every frame, on top of natural
  // perspective: rings within this many spacings of the camera scale up
  // (exaggerated parallax pop) and brighten (opacity ramp), fading back to
  // normal size/dimness for anything farther out.
  const SMITH_TUNNEL_NEAR_ZONE = SMITH_TUNNEL_STATION_SPACING * 3
  const SMITH_TUNNEL_NEAR_SCALE_BOOST = 0.9
  const SMITH_TUNNEL_NEAR_OPACITY = 1.0
  const SMITH_TUNNEL_FAR_OPACITY = 0.25

  // Stage 2: the DELPHI "core" barrel landmark passed along the tunnel's
  // path, plus a far parallax starfield layer. (The orbital "planet" used
  // to ride along here too; it now lives in its own standalone instrument
  // panel — see MINI_PANEL_SPECS below. The wireframe "jet" that also rode
  // along here was removed entirely, not moved.)
  const SMITH_CORE_Z = -12

  const SMITH_STARFIELD_COUNT = 160
  const SMITH_STARFIELD_RANGE = SMITH_TUNNEL_LOOP_LENGTH * 1.4
  const SMITH_STARFIELD_RADIUS_MIN = 4
  const SMITH_STARFIELD_RADIUS_MAX = 10

  function pushLine(positions: number[], colors: number[], x0: number, y0: number, x1: number, y1: number, color: [number, number, number]) {
    positions.push(x0, y0, 0, x1, y1, 0)
    const [r, g, b] = color
    colors.push(r / 255, g / 255, b / 255, r / 255, g / 255, b / 255)
  }

  // One tunnel "station" cross-section, in local XY (Z=0) — reused as shared
  // geometry across every repeated station instance, so building the tunnel
  // costs one geometry regardless of how many stations are in view.
  function buildSmithStationGeometry(): THREE.BufferGeometry {
    const positions: number[] = []
    const colors: number[] = []

    SMITH_TUNNEL_RESISTANCES.forEach((r, idx) => {
      const cx = (r / (1 + r)) * SMITH_TUNNEL_SCALE
      const rad = (1 / (1 + r)) * SMITH_TUNNEL_SCALE
      const color = lerpColor(idx / (SMITH_TUNNEL_RESISTANCES.length - 1))
      for (let seg = 0; seg < SMITH_TUNNEL_CIRCLE_SEGMENTS; seg++) {
        const a0 = (seg / SMITH_TUNNEL_CIRCLE_SEGMENTS) * Math.PI * 2
        const a1 = ((seg + 1) / SMITH_TUNNEL_CIRCLE_SEGMENTS) * Math.PI * 2
        pushLine(
          positions, colors,
          cx + Math.cos(a0) * rad, Math.sin(a0) * rad,
          cx + Math.cos(a1) * rad, Math.sin(a1) * rad,
          color,
        )
      }
    })

    for (let i = 0; i < SMITH_TUNNEL_SPOKE_COUNT; i++) {
      const angle = (i / SMITH_TUNNEL_SPOKE_COUNT) * Math.PI * 2
      pushLine(
        positions, colors,
        0, 0,
        Math.cos(angle) * SMITH_TUNNEL_SCALE, Math.sin(angle) * SMITH_TUNNEL_SCALE,
        lerpColor(0.5),
      )
    }

    const geometry = new THREE.BufferGeometry()
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    geometry.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))
    return geometry
  }

  // DELPHI-style "core": three ringed cross-sections, built from Three.js
  // primitives (wireframed) rather than hand-rolled lines — more reliable
  // than manual geometry for genuinely round rings. (The connecting barrel
  // cylinder was removed — just the ring set now.)
  function buildCoreObject(): THREE.Group {
    const group = new THREE.Group()
    const [r, g, b] = lerpColor(0.35)
    const material = new THREE.LineBasicMaterial({
      color: new THREE.Color(r / 255, g / 255, b / 255),
      transparent: true,
      opacity: 0.8,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })

    for (const zOff of [-1.1, 0, 1.1]) {
      const ring = new THREE.TorusGeometry(0.62, 0.015, 6, 28)
      const mesh = new THREE.LineSegments(new THREE.WireframeGeometry(ring), material)
      mesh.rotation.x = Math.PI / 2
      mesh.position.z = zOff
      group.add(mesh)
      ring.dispose()
    }

    return group
  }

  // Orbital "planet": wireframe sphere with two tilted elliptical rings.
  function buildPlanetObject(): THREE.Group {
    const group = new THREE.Group()
    const [r, g, b] = lerpColor(0.6)
    const material = new THREE.LineBasicMaterial({
      color: new THREE.Color(r / 255, g / 255, b / 255),
      transparent: true,
      opacity: 0.75,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })

    const sphere = new THREE.SphereGeometry(0.7, 12, 8)
    group.add(new THREE.LineSegments(new THREE.WireframeGeometry(sphere), material))
    sphere.dispose()

    const orbitTilts = [0.25, -0.35]
    const orbitRadii = [1.4, 1.9]
    orbitTilts.forEach((tilt, idx) => {
      const curve = new THREE.EllipseCurve(0, 0, orbitRadii[idx], orbitRadii[idx] * 0.55, 0, Math.PI * 2, false, 0)
      const points = curve.getPoints(48).map(p => new THREE.Vector3(p.x, p.y, 0))
      const geometry = new THREE.BufferGeometry().setFromPoints(points)
      const line = new THREE.LineLoop(geometry, material)
      line.rotation.x = Math.PI / 2 + tilt
      line.rotation.z = idx * 0.6
      group.add(line)
    })

    return group
  }

  // Four abstract wireframe scenes cycled in this panel (see
  // ABSTRACT_SCENE_SPECS below). Same amber/cyan additive-line terminal
  // aesthetic as the rest of the HUD.

  // 1. CRYSTAL — a faceted low-poly gem. An octahedron already reads as a
  // classic faceted diamond; stretching it vertically (applied via scale
  // below, not baked into the geometry) is what turns it into a gem
  // silhouette rather than a plain 8-sided ball.
  const CRYSTAL_SCALE: [number, number, number] = [0.85, 1.3, 0.85]

  function buildCrystalObject(): THREE.Group {
    const group = new THREE.Group()
    const [r, g, b] = lerpColor(0.25)
    const color = new THREE.Color(r / 255, g / 255, b / 255)

    const geometry = new THREE.OctahedronGeometry(0.9, 0)

    const fillMaterial = new THREE.MeshBasicMaterial({
      color,
      transparent: true,
      opacity: 0.08,
      side: THREE.DoubleSide,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
    const fillMesh = new THREE.Mesh(geometry, fillMaterial)
    fillMesh.scale.set(...CRYSTAL_SCALE)
    group.add(fillMesh)

    const lineMaterial = new THREE.LineBasicMaterial({
      color,
      transparent: true,
      opacity: 0.85,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
    const wireframe = new THREE.LineSegments(new THREE.WireframeGeometry(geometry), lineMaterial)
    wireframe.scale.set(...CRYSTAL_SCALE)
    group.add(wireframe)

    const haloMaterial = new THREE.LineBasicMaterial({
      color,
      transparent: true,
      opacity: 0.2,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      depthTest: false,
    })
    const halo = new THREE.LineSegments(wireframe.geometry, haloMaterial)
    halo.scale.set(CRYSTAL_SCALE[0] * 1.18, CRYSTAL_SCALE[1] * 1.18, CRYSTAL_SCALE[2] * 1.18)
    halo.renderOrder = -1
    group.add(halo)

    return group
  }

  // 2. AUDIO-REACTIVE SPHERE — an icosahedron-style wireframe sphere driven
  // by the live frequency data (same AnalyserNode the BARS/VFD/DOT-MATRIX
  // modes already read via playerStore.getFrequencyData()). Built as a
  // manual line-segment "soup" (each triangle's 3 edges, vertices not
  // shared) rather than THREE.WireframeGeometry so the position buffer can
  // be rewritten in place every frame — WireframeGeometry is a one-time
  // snapshot with no mechanism to re-derive itself from a deformed source.
  const AUDIO_SPHERE_RADIUS = 0.72
  const AUDIO_SPHERE_DETAIL = 1
  const AUDIO_SPHERE_BANDS = 8

  function buildIcosahedronEdgeSoup(radius: number, detail: number): Float32Array {
    const source = new THREE.IcosahedronGeometry(radius, detail).toNonIndexed()
    const pos = source.getAttribute('position') as THREE.BufferAttribute
    const positions: number[] = []
    for (let i = 0; i < pos.count; i += 3) {
      const a: [number, number, number] = [pos.getX(i), pos.getY(i), pos.getZ(i)]
      const b: [number, number, number] = [pos.getX(i + 1), pos.getY(i + 1), pos.getZ(i + 1)]
      const c: [number, number, number] = [pos.getX(i + 2), pos.getY(i + 2), pos.getZ(i + 2)]
      positions.push(...a, ...b, ...b, ...c, ...c, ...a)
    }
    source.dispose()
    return new Float32Array(positions)
  }

  function buildAudioSphereObject(): THREE.Group {
    const group = new THREE.Group()
    const [r, g, b] = lerpColor(0.5)
    const color = new THREE.Color(r / 255, g / 255, b / 255)

    const basePositions = buildIcosahedronEdgeSoup(AUDIO_SPHERE_RADIUS, AUDIO_SPHERE_DETAIL)
    const geometry = new THREE.BufferGeometry()
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(basePositions.slice(), 3))
    // Pristine base radii to distort from each frame — distorting in place
    // and reading the result back would compound the effect frame over frame.
    geometry.userData.basePositions = basePositions

    const material = new THREE.LineBasicMaterial({
      color,
      transparent: true,
      opacity: 0.85,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
    const wireframe = new THREE.LineSegments(geometry, material)
    wireframe.name = 'sphereWire'
    group.add(wireframe)

    // Glow/halo + ambient particles: same depth-read technique as the
    // researched holographic jet panel (enlarged additive duplicate layers
    // behind the crisp geometry, plus a few floating points around it).
    // The halo shares the exact same (mutated-in-place) geometry, so it
    // automatically follows the live distortion without extra per-frame work.
    const haloLayers = [
      { scale: 1.14, opacity: 0.22 },
      { scale: 1.3, opacity: 0.1 },
    ]
    haloLayers.forEach(({ scale, opacity }) => {
      const haloMaterial = new THREE.LineBasicMaterial({
        color,
        transparent: true,
        opacity,
        blending: THREE.AdditiveBlending,
        depthWrite: false,
        depthTest: false,
      })
      const halo = new THREE.LineSegments(geometry, haloMaterial)
      halo.scale.setScalar(scale)
      halo.renderOrder = -1
      group.add(halo)
    })

    const particles = buildOrbitParticles(2.2, 1.1, lerpColor(0.65))
    particles.name = 'sphereParticles'
    group.add(particles)

    return group
  }

  function averageBand(freqData: Uint8Array, band: number, bands: number): number {
    const start = Math.floor((band / bands) * freqData.length)
    const end = Math.max(start + 1, Math.floor(((band + 1) / bands) * freqData.length))
    let sum = 0
    for (let i = start; i < end; i++) {
      sum += freqData[i]
    }
    return sum / (end - start) / 255
  }

  function animateAudioSphere(group: THREE.Group, dt: number, elapsed: number, freqData: Uint8Array) {
    group.rotation.y += dt * 0.3
    group.rotation.x += dt * 0.08

    const wireframe = group.getObjectByName('sphereWire') as THREE.LineSegments | undefined
    if (wireframe && freqData.length > 0) {
      const geometry = wireframe.geometry as THREE.BufferGeometry
      const base = geometry.userData.basePositions as Float32Array
      const posAttr = geometry.getAttribute('position') as THREE.BufferAttribute

      const bandAmps: number[] = []
      let overall = 0
      for (let band = 0; band < AUDIO_SPHERE_BANDS; band++) {
        bandAmps.push(averageBand(freqData, band, AUDIO_SPHERE_BANDS))
      }
      for (let i = 0; i < freqData.length; i++) {
        overall += freqData[i]
      }
      overall = overall / freqData.length / 255

      const vertexCount = base.length / 3
      for (let i = 0; i < vertexCount; i++) {
        const bx = base[i * 3]
        const by = base[i * 3 + 1]
        const bz = base[i * 3 + 2]
        const len = Math.sqrt(bx * bx + by * by + bz * bz) || 1
        const dirX = bx / len
        const dirY = by / len
        const dirZ = bz / len
        // Bucket by azimuth so different bands visibly displace different
        // sides of the sphere, rather than a single uniform pulse.
        const azimuth = (Math.atan2(dirZ, dirX) + Math.PI) / (Math.PI * 2)
        const band = Math.min(AUDIO_SPHERE_BANDS - 1, Math.floor(azimuth * AUDIO_SPHERE_BANDS))
        const newLen = len * (1 + overall * 0.12 + bandAmps[band] * 0.32)
        posAttr.setXYZ(i, dirX * newLen, dirY * newLen, dirZ * newLen)
      }
      posAttr.needsUpdate = true
    }

    const particles = group.getObjectByName('sphereParticles') as THREE.Points | undefined
    if (particles) {
      animateOrbitParticles(particles, dt, 2.2, 1.1)
    }
  }

  // Small drifting point field used around the audio sphere for a depth
  // reference, same wraparound-drift technique used elsewhere in this file.
  function buildOrbitParticles(xBound: number, yBound: number, colorRgb: [number, number, number]): THREE.Points {
    const count = 20
    const positions = new Float32Array(count * 3)
    const velocities = new Float32Array(count * 3)
    for (let i = 0; i < count; i++) {
      positions[i * 3] = (Math.random() * 2 - 1) * xBound
      positions[i * 3 + 1] = (Math.random() * 2 - 1) * yBound
      positions[i * 3 + 2] = -0.4 - Math.random() * 2
      velocities[i * 3] = (Math.random() * 2 - 1) * 0.04
      velocities[i * 3 + 1] = (Math.random() * 2 - 1) * 0.02
      velocities[i * 3 + 2] = 0
    }
    const geometry = new THREE.BufferGeometry()
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    const [r, g, b] = colorRgb
    const material = new THREE.PointsMaterial({
      color: new THREE.Color(r / 255, g / 255, b / 255),
      size: 0.03,
      transparent: true,
      opacity: 0.5,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      sizeAttenuation: true,
    })
    const points = new THREE.Points(geometry, material)
    points.userData.velocities = velocities
    points.userData.count = count
    return points
  }

  function animateOrbitParticles(points: THREE.Points, dt: number, xBound: number, yBound: number) {
    const geometry = points.geometry as THREE.BufferGeometry
    const positionAttr = geometry.getAttribute('position') as THREE.BufferAttribute
    const velocities = points.userData.velocities as Float32Array
    const count = points.userData.count as number
    for (let i = 0; i < count; i++) {
      let x = positionAttr.getX(i) + velocities[i * 3] * dt
      let y = positionAttr.getY(i) + velocities[i * 3 + 1] * dt
      if (x > xBound) x -= xBound * 2
      if (x < -xBound) x += xBound * 2
      if (y > yBound) y -= yBound * 2
      if (y < -yBound) y += yBound * 2
      positionAttr.setXYZ(i, x, y, positionAttr.getZ(i))
    }
    positionAttr.needsUpdate = true
  }

  // 3. WIREFRAME CUBE — deliberately the plainest of the four: a clean
  // rotating box, no fill/halo, as a geometric contrast to the other three.
  function buildCubeObject(): THREE.Group {
    const group = new THREE.Group()
    const [r, g, b] = lerpColor(0.4)
    const color = new THREE.Color(r / 255, g / 255, b / 255)
    const geometry = new THREE.BoxGeometry(1.05, 1.05, 1.05)
    const material = new THREE.LineBasicMaterial({
      color,
      transparent: true,
      opacity: 0.85,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })
    const wireframe = new THREE.LineSegments(new THREE.WireframeGeometry(geometry), material)
    group.add(wireframe)
    geometry.dispose()
    return group
  }

  // 4. WIREFRAME TERRAIN — a scrolling grid landscape, in the same spirit
  // as the Smith-chart tunnel (continuous, looping travel) but a receding
  // ground plane rather than a tube. Only Y (height) is rewritten per
  // frame; X/Z stay fixed, so "scrolling" is really the height field's
  // sample phase sliding through Z rather than geometry actually moving —
  // cheaper than shifting/recycling real vertex positions and produces an
  // identical infinite-scroll look.
  const TERRAIN_WIDTH = 4
  const TERRAIN_DEPTH = 6
  const TERRAIN_SEGMENTS_X = 14
  const TERRAIN_SEGMENTS_Z = 20
  const TERRAIN_SCROLL_SPEED = 0.6

  function terrainHeightAt(x: number, z: number): number {
    return Math.sin(x * 0.9 + z * 0.35) * 0.24 + Math.sin(x * 0.35 - z * 0.6) * 0.16
  }

  function buildTerrainObject(): THREE.Group {
    const group = new THREE.Group()
    const [r, g, b] = lerpColor(0.72)
    const color = new THREE.Color(r / 255, g / 255, b / 255)
    const material = new THREE.LineBasicMaterial({
      color,
      transparent: true,
      opacity: 0.75,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
    })

    const xs: number[] = []
    for (let i = 0; i <= TERRAIN_SEGMENTS_X; i++) {
      xs.push(-TERRAIN_WIDTH / 2 + (i / TERRAIN_SEGMENTS_X) * TERRAIN_WIDTH)
    }
    const zs: number[] = []
    for (let j = 0; j <= TERRAIN_SEGMENTS_Z; j++) {
      zs.push(-(j / TERRAIN_SEGMENTS_Z) * TERRAIN_DEPTH)
    }

    // A grid of unfilled line segments (row lines + column lines), not a
    // solid mesh — `basePairs` records each vertex-instance's static (x, z)
    // so the per-frame update below only ever has to solve for height.
    const positions: number[] = []
    const basePairs: number[] = []
    const pushSeg = (x0: number, z0: number, x1: number, z1: number) => {
      positions.push(x0, 0, z0, x1, 0, z1)
      basePairs.push(x0, z0, x1, z1)
    }
    zs.forEach((z) => {
      for (let i = 0; i < xs.length - 1; i++) {
        pushSeg(xs[i], z, xs[i + 1], z)
      }
    })
    xs.forEach((x) => {
      for (let j = 0; j < zs.length - 1; j++) {
        pushSeg(x, zs[j], x, zs[j + 1])
      }
    })

    const geometry = new THREE.BufferGeometry()
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    geometry.userData.basePairs = new Float32Array(basePairs)
    const mesh = new THREE.LineSegments(geometry, material)
    mesh.name = 'terrainMesh'
    group.add(mesh)

    return group
  }

  function animateTerrain(group: THREE.Group, elapsed: number) {
    const mesh = group.getObjectByName('terrainMesh') as THREE.LineSegments | undefined
    if (!mesh) {
      return
    }
    const geometry = mesh.geometry as THREE.BufferGeometry
    const basePairs = geometry.userData.basePairs as Float32Array
    const posAttr = geometry.getAttribute('position') as THREE.BufferAttribute
    const phase = elapsed * TERRAIN_SCROLL_SPEED
    const vertexCount = basePairs.length / 2
    for (let i = 0; i < vertexCount; i++) {
      const x = basePairs[i * 2]
      const z = basePairs[i * 2 + 1]
      posAttr.setY(i, terrainHeightAt(x, z + phase))
    }
    posAttr.needsUpdate = true
  }

  // Frees the geometries/materials of an object and everything under it —
  // shared by the mini-panel and abstract-panel dispose paths below (and
  // the tunnel's landmark cleanup), since both build a fresh Three.js
  // object per panel/scene and need it fully released on unmount or swap.
  function disposeObjectTree(object: THREE.Object3D) {
    object.traverse((obj) => {
      const mesh = obj as THREE.LineSegments | THREE.LineLoop | THREE.Mesh | THREE.Points
      mesh.geometry?.dispose()
      const material = mesh.material as THREE.Material | THREE.Material[] | undefined
      if (Array.isArray(material)) {
        material.forEach(m => m.dispose())
      } else {
        material?.dispose()
      }
    })
  }

  interface AbstractSceneSpec {
    label: string
    buildObject: () => THREE.Group
    spin: (group: THREE.Group, dt: number, elapsed: number, freqData: Uint8Array) => void
    // Only the terrain scene needs a non-default camera (raised, tilted
    // down over the landscape) — everything else uses the panel's default
    // front-on framing.
    cameraSetup?: (camera: THREE.PerspectiveCamera) => void
    // Ambient color drift is opt-in per scene: only CRYSTAL gets it (see
    // COLOR_DRIFT_AMOUNT above) — SPHERE/CUBE/TERRAIN are left untouched,
    // so this is undefined for them.
    baseColorT?: number
  }
  interface AbstractPanelState {
    renderer: THREE.WebGLRenderer | null
    scene: THREE.Scene | null
    camera: THREE.PerspectiveCamera | null
    object: THREE.Group | null
    resizeObserver: ResizeObserver | null
    frame: number
    lastTime: number
    elapsed: number
  }

  const ABSTRACT_SCENE_SPECS: AbstractSceneSpec[] = [
    {
      label: 'CRYSTAL',
      buildObject: buildCrystalObject,
      spin: (group, dt) => {
        group.rotation.y += dt * 0.35
      },
      baseColorT: 0.25,
    },
    {
      label: 'SPHERE',
      buildObject: buildAudioSphereObject,
      spin: animateAudioSphere,
    },
    {
      label: 'CUBE',
      buildObject: buildCubeObject,
      spin: (group, dt) => {
        group.rotation.y += dt * 0.5
        group.rotation.x += dt * 0.3
      },
    },
    {
      label: 'TERRAIN',
      buildObject: buildTerrainObject,
      spin: (group, dt, elapsed) => animateTerrain(group, elapsed),
      cameraSetup: (camera) => {
        camera.position.set(0, 1.05, 1.5)
        camera.rotation.x = -0.55
      },
    },
  ]

  // Far parallax layer: a sparse dim starfield well outside the tunnel
  // radius, spread across a longer stretch than the tunnel loop. Because it
  // sits much further from the camera's path, the same camera travel moves
  // it much less across the screen than the tunnel rings or landmarks —
  // the explicit "near moves fast, far moves slow" depth cue.
  function respawnStar(positions: Float32Array, i: number, baseZ: number) {
    const angle = Math.random() * Math.PI * 2
    const radius = SMITH_STARFIELD_RADIUS_MIN + Math.random() * (SMITH_STARFIELD_RADIUS_MAX - SMITH_STARFIELD_RADIUS_MIN)
    positions[i * 3] = Math.cos(angle) * radius
    positions[i * 3 + 1] = Math.sin(angle) * radius
    positions[i * 3 + 2] = baseZ + Math.random() * SMITH_STARFIELD_RANGE * 0.15
  }

  function buildStarfield(): { points: THREE.Points, geometry: THREE.BufferGeometry, material: THREE.PointsMaterial } {
    const positions = new Float32Array(SMITH_STARFIELD_COUNT * 3)
    for (let i = 0; i < SMITH_STARFIELD_COUNT; i++) {
      respawnStar(positions, i, -Math.random() * SMITH_STARFIELD_RANGE)
    }
    const geometry = new THREE.BufferGeometry()
    geometry.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
    const [r, g, b] = lerpColor(0.5)
    const material = new THREE.PointsMaterial({
      color: new THREE.Color(r / 255, g / 255, b / 255),
      size: 0.05,
      transparent: true,
      opacity: 0.35,
      blending: THREE.AdditiveBlending,
      depthWrite: false,
      sizeAttenuation: true,
    })
    const points = new THREE.Points(geometry, material)
    return { points, geometry, material }
  }

  // Standalone instrument panels — each a genuinely separate Three.js scene/
  // camera/renderer/render-loop, not a landmark riding along the tunnel's
  // path. Real 3D rotation on the object itself, camera fixed, so each
  // panel reads as its own contained diorama.
  interface MiniPanelSpec {
    wrapRef: string
    canvasRef: string
    buildObject: () => THREE.Group
    spin: (group: THREE.Group, dt: number, elapsed: number) => void
    // Base lerpColor(t) hue this panel's object was built with — the ambient
    // color drift re-centers on this each frame rather than replacing it.
    baseColorT: number
  }
  interface MiniPanelState {
    spec: MiniPanelSpec
    renderer: THREE.WebGLRenderer | null
    scene: THREE.Scene | null
    camera: THREE.PerspectiveCamera | null
    object: THREE.Group | null
    resizeObserver: ResizeObserver | null
    frame: number
    lastTime: number
    elapsed: number
  }

  const MINI_PANEL_SPECS: MiniPanelSpec[] = [
    {
      wrapRef: 'planetPanelWrap',
      canvasRef: 'planetPanelCanvas',
      buildObject: buildPlanetObject,
      // Continuous tumble on two axes so the tilted orbit rings genuinely
      // read as 3D rather than a flat spinning disc.
      spin: (group, dt) => {
        group.rotation.y += dt * 0.5
        group.rotation.x += dt * 0.12
      },
      baseColorT: 0.6,
    },
  ]

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
        // z is a persistent pseudo-depth (0.2-1) assigned per particle at
        // build time — nearer particles (z closer to 1) are drawn larger,
        // jitter more, and shift more with mouse parallax than farther ones.
        particlePoints: markRaw([] as { x: number, y: number, lum: number, angle: number, z: number }[]),
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
        // BARS mode depth rows: rolling history of recent bar-height frames,
        // sampled a few frames back to draw smaller/dimmer "receding" rows
        // behind the main bars — a bar forest instead of a single flat row.
        barsHistory: markRaw([] as Float32Array[]),
        modeTransitioning: false,
        modeTransitionTimer: 0 as ReturnType<typeof setTimeout> | 0,
        parallaxX: 0,
        parallaxY: 0,
        reducedMotion: false,
        // Real facts about the running audio graph (sampleRate/fftSize),
        // fetched once the audio pipeline exists — never fabricated.
        audioInfo: { sampleRate: 0, fftSize: 0 },
        // Smith-chart tunnel (Three.js) — its own independent render loop
        // and renderer, entirely separate from the 2D canvas draw() loop
        // above. Non-reactive: markRaw avoids wrapping Three.js internals
        // in Vue's reactivity proxy, which is both wasteful and unsafe for
        // objects Three.js mutates internally.
        smithTunnelRenderer: null as THREE.WebGLRenderer | null,
        smithTunnelScene: null as THREE.Scene | null,
        smithTunnelCamera: null as THREE.PerspectiveCamera | null,
        smithTunnelGeometry: null as THREE.BufferGeometry | null,
        smithTunnelMaterial: null as THREE.LineBasicMaterial | null,
        smithTunnelStations: markRaw([] as THREE.LineSegments[]),
        // Per-station material clones (one per station instance) so each
        // ring's opacity can ramp independently with its own depth, on top
        // of the shared geometry/color they all clone from.
        smithTunnelStationMaterials: markRaw([] as THREE.LineBasicMaterial[]),
        smithTunnelResizeObserver: null as ResizeObserver | null,
        smithTunnelFrame: 0 as number,
        smithTunnelLastTime: 0,
        // Stage 2: landmark objects (core/planet/jet) passed along the
        // tunnel, plus a far parallax starfield layer. Same non-reactive
        // rationale as the rest of the tunnel state above.
        smithLandmarks: markRaw([] as { group: THREE.Group }[]),
        smithStarPoints: null as THREE.Points | null,
        smithStarGeometry: null as THREE.BufferGeometry | null,
        smithStarMaterial: null as THREE.PointsMaterial | null,
        // Standalone planet instrument panel — own scene per entry, same
        // non-reactive rationale as the tunnel state above.
        miniPanels: markRaw(MINI_PANEL_SPECS.map(spec => ({
          spec,
          renderer: null,
          scene: null,
          camera: null,
          object: null,
          resizeObserver: null,
          frame: 0,
          lastTime: 0,
          elapsed: 0,
        } as MiniPanelState))),
        // Index into ABSTRACT_SCENE_SPECS for the abstract-shape panel that
        // replaced the old jet panel — reactive, since the toggle button's
        // label depends on it.
        abstractSceneIndex: 0,
        // Non-reactive render state for that panel, same shape/rationale as
        // MiniPanelState above but kept separate since scenes are swapped
        // out (rebuilt) on cycle rather than being fixed for the panel's
        // lifetime like the planet panel is.
        abstractPanel: markRaw({
          renderer: null,
          scene: null,
          camera: null,
          object: null,
          resizeObserver: null,
          frame: 0,
          lastTime: 0,
          elapsed: 0,
        } as AbstractPanelState),
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
      abstractSceneLabel(): string {
        return ABSTRACT_SCENE_SPECS[this.abstractSceneIndex].label
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
    watch: {
      // Covers the fresh-page-load case: track starts null while the player
      // store hydrates, so the v-if="track" block (and the canvas refs the
      // tunnel/mini panels attach to) doesn't exist in the DOM at mounted()
      // time. Once track actually becomes truthy, nextTick waits for that
      // block to render before we go looking for the refs. The init methods
      // themselves guard against being run twice.
      track(newTrack: unknown) {
        if (newTrack) {
          this.$nextTick(() => {
            this.initSmithTunnel()
            this.initMiniPanels()
            this.initAbstractPanel()
          })
        }
      },
    },
    mounted() {
      this.reducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches
      this.audioInfo = this.playerStore.getAudioInfo()
      this.draw()
      // track can still be null here (store hydrates async on a fresh page
      // load), in which case the v-if="track" block — and the canvas refs
      // the panels need — isn't in the DOM yet. The `track` watcher below
      // (immediate: true) is what actually fires initSmithTunnel/
      // initMiniPanels in that case; this call only covers the case where
      // track is already set by mount time (e.g. SPA-internal navigation).
      if (this.track) {
        this.initSmithTunnel()
        this.initMiniPanels()
        this.initAbstractPanel()
      }
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
      this.disposeSmithTunnel()
      this.disposeMiniPanels()
      this.disposeAbstractPanel()
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
      initSmithTunnel() {
        if (this.smithTunnelRenderer) {
          return
        }
        const canvas = this.$refs.smithTunnelCanvas as HTMLCanvasElement | undefined
        const wrap = this.$refs.smithTunnelWrap as HTMLElement | undefined
        if (!canvas || !wrap) {
          return
        }

        const renderer = markRaw(new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true }))
        renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2))
        const scene = markRaw(new THREE.Scene())
        scene.fog = new THREE.Fog(0x050200, SMITH_TUNNEL_STATION_SPACING * 1.5, SMITH_TUNNEL_STATION_COUNT * SMITH_TUNNEL_STATION_SPACING * 0.85)

        const camera = markRaw(new THREE.PerspectiveCamera(
          62, 1, 0.1, SMITH_TUNNEL_LOOP_LENGTH * 1.6 + 20,
        ))
        camera.position.set(0, 0, SMITH_TUNNEL_STATION_SPACING)

        const geometry = markRaw(buildSmithStationGeometry())
        const material = markRaw(new THREE.LineBasicMaterial({
          vertexColors: true,
          transparent: true,
          opacity: 0.85,
          blending: THREE.AdditiveBlending,
          depthWrite: false,
        }))

        // `material` itself is kept only as a clone template (never added to
        // the scene) — each station gets its own clone so its opacity can be
        // driven independently by its own depth every frame.
        const stations: THREE.LineSegments[] = []
        const stationMaterials: THREE.LineBasicMaterial[] = []
        for (let i = 0; i < SMITH_TUNNEL_STATION_COUNT; i++) {
          const stationMaterial = markRaw(material.clone())
          const station = markRaw(new THREE.LineSegments(geometry, stationMaterial))
          station.position.z = -i * SMITH_TUNNEL_STATION_SPACING
          scene.add(station)
          stations.push(station)
          stationMaterials.push(stationMaterial)
        }

        this.smithTunnelRenderer = renderer
        this.smithTunnelScene = scene
        this.smithTunnelCamera = camera
        this.smithTunnelGeometry = geometry
        this.smithTunnelMaterial = material
        this.smithTunnelStations = markRaw(stations)
        this.smithTunnelStationMaterials = markRaw(stationMaterials)

        // Landmark objects the camera passes at different points in the
        // loop, each placed once and recycled via the same wrap-around
        // technique as the stations above (own z slot, own recycle check).
        const core = markRaw(buildCoreObject())
        core.position.set(0, 0, SMITH_CORE_Z)
        scene.add(core)

        this.smithLandmarks = markRaw([{ group: core }])

        const { points: starPoints, geometry: starGeometry, material: starMaterial } = buildStarfield()
        scene.add(starPoints)
        this.smithStarPoints = markRaw(starPoints)
        this.smithStarGeometry = markRaw(starGeometry)
        this.smithStarMaterial = markRaw(starMaterial)

        this.resizeSmithTunnel()
        this.smithTunnelResizeObserver = new ResizeObserver(() => this.resizeSmithTunnel())
        this.smithTunnelResizeObserver.observe(wrap)

        this.smithTunnelLastTime = performance.now()
        this.smithTunnelFrame = requestAnimationFrame(this.drawSmithTunnel)
      },
      resizeSmithTunnel() {
        const wrap = this.$refs.smithTunnelWrap as HTMLElement | undefined
        const renderer = this.smithTunnelRenderer
        const camera = this.smithTunnelCamera
        if (!wrap || !renderer || !camera) {
          return
        }
        const width = wrap.clientWidth
        const height = wrap.clientHeight
        if (width === 0 || height === 0) {
          return
        }
        renderer.setSize(width, height, false)
        camera.aspect = width / height
        camera.updateProjectionMatrix()
      },
      drawSmithTunnel(now: number) {
        this.smithTunnelFrame = requestAnimationFrame(this.drawSmithTunnel)
        const renderer = this.smithTunnelRenderer
        const scene = this.smithTunnelScene
        const camera = this.smithTunnelCamera
        if (!renderer || !scene || !camera) {
          return
        }
        const dt = Math.min((now - this.smithTunnelLastTime) / 1000, 0.1)
        this.smithTunnelLastTime = now

        if (!this.reducedMotion) {
          camera.position.z -= SMITH_TUNNEL_SPEED * dt
          const wrapAt = SMITH_TUNNEL_LOOP_LENGTH
          const tunnelElapsed = now / 1000
          const [tintR, tintG, tintB] = tunnelDriftTint(tunnelElapsed)
          this.smithTunnelStations.forEach((station, idx) => {
            if (station.position.z > camera.position.z + SMITH_TUNNEL_STATION_SPACING) {
              station.position.z -= wrapAt
            }

            // Depth cue, recomputed every frame off the station's actual
            // distance ahead of the camera: near rings scale up and brighten
            // beyond what perspective alone gives, far ones settle back to
            // normal size and fog-level dimness — reinforced parallax, and
            // it survives the recycle above with no visible pop since a
            // freshly-wrapped station is immediately "far" again.
            const depthAhead = camera.position.z - station.position.z
            const proximity = THREE.MathUtils.clamp(1 - depthAhead / SMITH_TUNNEL_NEAR_ZONE, 0, 1)
            const eased = proximity * proximity
            station.scale.setScalar(1 + eased * SMITH_TUNNEL_NEAR_SCALE_BOOST)
            const stationMaterial = this.smithTunnelStationMaterials[idx]
            if (stationMaterial) {
              stationMaterial.opacity = SMITH_TUNNEL_FAR_OPACITY +
                (SMITH_TUNNEL_NEAR_OPACITY - SMITH_TUNNEL_FAR_OPACITY) * eased
              stationMaterial.color.setRGB(tintR, tintG, tintB)
            }
          })

          // Landmarks: recycle the same way as stations, plus a slow self
          // spin so each reads as a real object rather than a flat cutout
          // as the camera passes it.
          this.smithLandmarks.forEach(({ group }) => {
            group.rotation.y += dt * 0.35
            if (group.position.z > camera.position.z + SMITH_TUNNEL_STATION_SPACING) {
              group.position.z -= wrapAt
            }
            applyColorDrift(group, 0.35, tunnelElapsed)
          })

          // Starfield: sits well outside the tunnel radius and spans a much
          // longer stretch, so it drifts across the view far slower than
          // the tunnel/landmarks for the same camera travel — the explicit
          // near-fast/far-slow parallax cue.
          const starGeometry = this.smithStarGeometry
          if (starGeometry) {
            const attr = starGeometry.getAttribute('position') as THREE.BufferAttribute
            let dirty = false
            for (let i = 0; i < SMITH_STARFIELD_COUNT; i++) {
              if (attr.getZ(i) > camera.position.z + 2) {
                respawnStar(attr.array as Float32Array, i, camera.position.z - SMITH_STARFIELD_RANGE)
                dirty = true
              }
            }
            if (dirty) {
              attr.needsUpdate = true
            }
          }
          if (this.smithStarPoints) {
            applyColorDrift(this.smithStarPoints, 0.5, tunnelElapsed)
          }
        }

        if (renderer.domElement.clientWidth === 0) {
          return
        }
        renderer.render(scene, camera)
      },
      disposeSmithTunnel() {
        cancelAnimationFrame(this.smithTunnelFrame)
        this.smithTunnelResizeObserver?.disconnect()
        this.smithTunnelResizeObserver = null
        this.smithTunnelGeometry?.dispose()
        this.smithTunnelMaterial?.dispose()
        this.smithTunnelStationMaterials.forEach(m => m.dispose())
        this.smithTunnelStationMaterials = markRaw([])
        this.smithLandmarks.forEach(({ group }) => {
          group.traverse((obj) => {
            const mesh = obj as THREE.LineSegments | THREE.LineLoop
            mesh.geometry?.dispose()
            const material = mesh.material as THREE.Material | THREE.Material[] | undefined
            if (Array.isArray(material)) {
              material.forEach(m => m.dispose())
            } else {
              material?.dispose()
            }
          })
        })
        this.smithLandmarks = markRaw([])
        this.smithStarGeometry?.dispose()
        this.smithStarMaterial?.dispose()
        this.smithStarPoints = null
        this.smithStarGeometry = null
        this.smithStarMaterial = null
        this.smithTunnelRenderer?.dispose()
        this.smithTunnelStations = markRaw([])
      },
      initMiniPanels() {
        this.miniPanels.forEach((_, idx) => this.initMiniPanel(idx))
      },
      initMiniPanel(idx: number) {
        const panel = this.miniPanels[idx]
        if (panel.renderer) {
          return
        }
        const canvas = this.$refs[panel.spec.canvasRef] as HTMLCanvasElement | undefined
        const wrap = this.$refs[panel.spec.wrapRef] as HTMLElement | undefined
        if (!canvas || !wrap) {
          return
        }

        const renderer = markRaw(new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true }))
        renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2))
        const scene = markRaw(new THREE.Scene())
        const camera = markRaw(new THREE.PerspectiveCamera(45, 1, 0.1, 20))
        camera.position.set(0, 0, 3.4)

        const object = markRaw(panel.spec.buildObject())
        scene.add(object)

        panel.renderer = renderer
        panel.scene = scene
        panel.camera = camera
        panel.object = object

        this.resizeMiniPanel(idx)
        panel.resizeObserver = new ResizeObserver(() => this.resizeMiniPanel(idx))
        panel.resizeObserver.observe(wrap)

        panel.lastTime = performance.now()
        panel.frame = requestAnimationFrame(now => this.drawMiniPanel(idx, now))
      },
      resizeMiniPanel(idx: number) {
        const panel = this.miniPanels[idx]
        const wrap = this.$refs[panel.spec.wrapRef] as HTMLElement | undefined
        const renderer = panel.renderer
        const camera = panel.camera
        if (!wrap || !renderer || !camera) {
          return
        }
        const width = wrap.clientWidth
        const height = wrap.clientHeight
        if (width === 0 || height === 0) {
          return
        }
        renderer.setSize(width, height, false)
        camera.aspect = width / height
        camera.updateProjectionMatrix()
      },
      drawMiniPanel(idx: number, now: number) {
        const panel = this.miniPanels[idx]
        panel.frame = requestAnimationFrame(t => this.drawMiniPanel(idx, t))
        const { renderer, scene, camera, object } = panel
        if (!renderer || !scene || !camera || !object) {
          return
        }
        const dt = Math.min((now - panel.lastTime) / 1000, 0.1)
        panel.lastTime = now
        panel.elapsed += dt

        if (!this.reducedMotion) {
          panel.spec.spin(object, dt, panel.elapsed)
          applyColorDrift(object, panel.spec.baseColorT, panel.elapsed)
        }

        if (renderer.domElement.clientWidth === 0) {
          return
        }
        renderer.render(scene, camera)
      },
      disposeMiniPanels() {
        this.miniPanels.forEach((panel) => {
          cancelAnimationFrame(panel.frame)
          panel.resizeObserver?.disconnect()
          panel.resizeObserver = null
          if (panel.object) {
            disposeObjectTree(panel.object)
          }
          panel.object = null
          panel.scene = null
          panel.camera = null
          panel.renderer?.dispose()
          panel.renderer = null
        })
      },
      // Abstract-shape mini panel (replaces the old jet panel): same
      // standalone-scene architecture as the planet panel above, but the
      // scene object is swapped out on cycleAbstractScene() rather than
      // fixed for the panel's lifetime, so init/dispose of just the current
      // object is split out into setAbstractScene() and reused by both.
      initAbstractPanel() {
        const panel = this.abstractPanel
        if (panel.renderer) {
          return
        }
        const canvas = this.$refs.abstractPanelCanvas as HTMLCanvasElement | undefined
        const wrap = this.$refs.abstractPanelWrap as HTMLElement | undefined
        if (!canvas || !wrap) {
          return
        }

        const renderer = markRaw(new THREE.WebGLRenderer({ canvas, alpha: true, antialias: true }))
        renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2))
        const scene = markRaw(new THREE.Scene())
        const camera = markRaw(new THREE.PerspectiveCamera(45, 1, 0.1, 20))

        panel.renderer = renderer
        panel.scene = scene
        panel.camera = camera

        this.setAbstractScene(this.abstractSceneIndex)

        this.resizeAbstractPanel()
        panel.resizeObserver = new ResizeObserver(() => this.resizeAbstractPanel())
        panel.resizeObserver.observe(wrap)

        panel.lastTime = performance.now()
        panel.frame = requestAnimationFrame(now => this.drawAbstractPanel(now))
      },
      setAbstractScene(index: number) {
        const panel = this.abstractPanel
        const { scene, camera } = panel
        if (!scene || !camera) {
          return
        }
        if (panel.object) {
          scene.remove(panel.object)
          disposeObjectTree(panel.object)
          panel.object = null
        }
        camera.position.set(0, 0, 3.4)
        camera.rotation.set(0, 0, 0)
        const spec = ABSTRACT_SCENE_SPECS[index]
        spec.cameraSetup?.(camera)
        const object = markRaw(spec.buildObject())
        scene.add(object)
        panel.object = object
        panel.elapsed = 0
      },
      cycleAbstractScene() {
        this.abstractSceneIndex = (this.abstractSceneIndex + 1) % ABSTRACT_SCENE_SPECS.length
        this.setAbstractScene(this.abstractSceneIndex)
      },
      resizeAbstractPanel() {
        const panel = this.abstractPanel
        const wrap = this.$refs.abstractPanelWrap as HTMLElement | undefined
        const { renderer, camera } = panel
        if (!wrap || !renderer || !camera) {
          return
        }
        const width = wrap.clientWidth
        const height = wrap.clientHeight
        if (width === 0 || height === 0) {
          return
        }
        renderer.setSize(width, height, false)
        camera.aspect = width / height
        camera.updateProjectionMatrix()
      },
      drawAbstractPanel(now: number) {
        const panel = this.abstractPanel
        panel.frame = requestAnimationFrame(t => this.drawAbstractPanel(t))
        const { renderer, scene, camera, object } = panel
        if (!renderer || !scene || !camera || !object) {
          return
        }
        const dt = Math.min((now - panel.lastTime) / 1000, 0.1)
        panel.lastTime = now
        panel.elapsed += dt

        if (!this.reducedMotion) {
          const spec = ABSTRACT_SCENE_SPECS[this.abstractSceneIndex]
          spec.spin(object, dt, panel.elapsed, this.playerStore.getFrequencyData())
          if (spec.baseColorT !== undefined) {
            applyColorDrift(object, spec.baseColorT, panel.elapsed)
          }
        }

        if (renderer.domElement.clientWidth === 0) {
          return
        }
        renderer.render(scene, camera)
      },
      disposeAbstractPanel() {
        const panel = this.abstractPanel
        cancelAnimationFrame(panel.frame)
        panel.resizeObserver?.disconnect()
        panel.resizeObserver = null
        if (panel.object) {
          disposeObjectTree(panel.object)
        }
        panel.object = null
        panel.scene = null
        panel.camera = null
        panel.renderer?.dispose()
        panel.renderer = null
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

        const current = new Float32Array(barCount)
        for (let i = 0; i < barCount; i++) {
          current[i] = data[i * step] / 255
        }

        // Bar forest: draw 2 older frames from history behind the live row,
        // smaller/dimmer/offset — each row is a real past state of the
        // spectrum, not a decorative copy, so the "receding" rows still
        // move with the music instead of just sitting there.
        this.barsHistory.push(current)
        const maxHistory = 20
        if (this.barsHistory.length > maxHistory) {
          this.barsHistory.shift()
        }
        const depthRows = [
          { framesBack: 14, scale: 0.55, yLift: 26, alphaMul: 0.28, xShift: -3 },
          { framesBack: 7, scale: 0.78, yLift: 13, alphaMul: 0.5, xShift: 2 },
        ]
        for (const row of depthRows) {
          const idx = this.barsHistory.length - 1 - row.framesBack
          if (idx < 0) {
            continue
          }
          const frame = this.barsHistory[idx]
          for (let i = 0; i < barCount; i++) {
            const value = frame[i]
            const barHeight = value * height * row.scale
            const x = i * barWidth * row.scale + row.xShift + (width - width * row.scale) / 2
            const y = height - barHeight - row.yLift
            const [r, g, b] = lerpColor(i / (barCount - 1))
            ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${(0.25 + value * 0.5) * row.alphaMul})`
            ctx.fillRect(x + 1, y, barWidth * row.scale - 2, barHeight)
          }
        }

        for (let i = 0; i < barCount; i++) {
          const value = current[i]
          const barHeight = value * height
          const x = i * barWidth
          const y = height - barHeight

          const [r, g, b] = lerpColor(i / (barCount - 1))

          ctx.shadowBlur = 8
          ctx.shadowColor = `rgba(${r}, ${g}, ${b}, 0.7)`
          ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${0.35 + value * 0.65})`
          ctx.fillRect(x + 1, y, barWidth - 2, barHeight)
        }
        ctx.shadowBlur = 0
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

          const points: { x: number, y: number, lum: number, angle: number, z: number }[] = []
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
              // Persistent pseudo-depth (0.2-1) fixed at build time — drives
              // per-particle size/jitter/parallax scale in drawParticles so
              // nearer particles read as physically closer to the viewer,
              // not just brighter or bigger for no reason.
              points.push({ x: sx / canvasWidth, y: sy / canvasHeight, lum, angle: Math.random() * Math.PI * 2, z: 0.2 + Math.random() * 0.8 })
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

        const positions: { x: number, y: number, alpha: number, radiusScale: number }[] = []
        for (const p of this.particlePoints) {
          // DISPLACEMENT: each particle jitters from its home position along
          // a fixed-per-particle direction, scaled by the frequency bin its
          // horizontal position maps to — so different regions of the image
          // react to different parts of the spectrum. DEPTH: nearer
          // particles (higher z) jitter and drift further, matching how
          // closer objects appear to move more for the same real motion.
          let jitterX = 0
          let jitterY = 0
          if (data.length > 0) {
            const bin = data[Math.floor(p.x * (data.length - 1))] / 255
            const displace = bin * PARTICLE_MAX_DISPLACEMENT_PX * (0.5 + p.z * 0.9)
            jitterX = Math.cos(p.angle) * displace
            jitterY = Math.sin(p.angle) * displace
          }
          // DEPTH PARALLAX: nearer particles shift more with mouse movement
          // than farther ones, reusing the same parallaxX/Y signal the rest
          // of the screen already tracks.
          const parallaxPx = this.reducedMotion ? 0 : p.z * 6
          positions.push({
            x: p.x * width + jitterX + this.parallaxX * parallaxPx,
            y: p.y * height + jitterY + this.parallaxY * parallaxPx,
            alpha: Math.min(1, (0.4 + p.lum * 0.6 + amp * 0.1) * PARTICLE_OVERLAY_OPACITY),
            radiusScale: 0.6 + p.z * 0.8,
          })
        }

        // Dark halo pass — a soft, translucent ring behind each particle so
        // the overlay motes read against any photo regardless of its own
        // colors, without going as opaque as the old solid-backdrop version
        // (the photo underneath must stay visible).
        ctx.fillStyle = `rgba(10, 8, 0, ${0.3 * PARTICLE_OVERLAY_OPACITY + 0.1})`
        for (const pos of positions) {
          ctx.beginPath()
          ctx.arc(pos.x, pos.y, dotRadius * pos.radiusScale * 1.7, 0, Math.PI * 2)
          ctx.fill()
        }

        // Bright particle pass, glow baked into a pre-rendered sprite and
        // stamped via drawImage — reads as the same glowing-dust look as
        // ctx.shadowBlur but without recomputing a blur on every draw call.
        if (!this.particleGlowSprite) {
          this.particleGlowSprite = markRaw(this.buildParticleGlowSprite())
        }
        const sprite = this.particleGlowSprite
        for (const pos of positions) {
          const glowDiameter = dotRadius * pos.radiusScale * 3.4
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
      // Renders one wireframe sphere — thin amber/cyan latitude (parallel)
      // and longitude (meridian) arcs on a unit sphere, rotated around the
      // vertical axis with a fixed tilt and projected orthographically onto
      // the 2D canvas. Basic trigonometry only, no 3D library. Parameterized
      // so drawGlobe() can call it twice — a foreground sphere and a smaller,
      // dimmer, slower-rotating background sphere — for real depth parallax
      // rather than a single flat spinning disc.
      drawWireframeSphere(
        ctx: CanvasRenderingContext2D,
        cx: number,
        cy: number,
        radius: number,
        rotY: number,
        alphaMul: number,
      ) {
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
            const alpha = (0.12 + 0.5 * ((p.z + radius) / (2 * radius))) * alphaMul
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
      },
      // Small rotating wireframe globe accent, now with a second, smaller,
      // dimmer sphere at a different depth rotating at a different speed —
      // genuine depth parallax between two real 3D objects, not decoration.
      drawGlobe(ctx: CanvasRenderingContext2D, width: number, height: number) {
        ctx.clearRect(0, 0, width, height)

        const cx = width / 2
        const cy = height / 2
        const radius = Math.min(width, height) / 2 - 3
        const now = performance.now()
        const rotY = this.reducedMotion
          ? 0
          : ((now % GLOBE_ROTATION_PERIOD_MS) / GLOBE_ROTATION_PERIOD_MS) * Math.PI * 2
        // Background sphere: smaller, dimmer, offset, and rotating at ~40%
        // of the foreground speed — reads as further away because it moves
        // slower across the frame for the same elapsed time (parallax).
        const bgPeriod = GLOBE_ROTATION_PERIOD_MS / 0.4
        const rotYBg = this.reducedMotion
          ? 0
          : ((now % bgPeriod) / bgPeriod) * Math.PI * 2

        this.drawWireframeSphere(ctx, cx - radius * 0.22, cy + radius * 0.18, radius * 0.5, rotYBg, 0.45)
        this.drawWireframeSphere(ctx, cx, cy, radius, rotY, 1)

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

        // Faint deeper grid receding behind the live segments — a static,
        // slightly smaller/offset copy of the unlit-cell grid. Cheap (no
        // shadowBlur, fixed alpha) but reads as a second, further-back
        // panel rather than one flat display.
        ctx.shadowBlur = 0
        const bgScale = 0.94
        const bgDx = width * (1 - bgScale) * 0.5 + 5
        const bgDy = -4
        for (let i = 0; i < barCount; i++) {
          const x = i * (barWidth + barGap) * bgScale + bgDx
          for (let s = 0; s < totalSegments; s++) {
            const segY = height - (s + 1) * (segHeight + segGap) + segGap
            const [r, g, b] = lerpColor(s / (totalSegments - 1))
            ctx.fillStyle = `rgba(${r}, ${g}, ${b}, 0.05)`
            ctx.fillRect(x, segY * bgScale + bgDy, barWidth * bgScale, segHeight * bgScale)
          }
        }

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

        // Faint deeper dot grid receding behind the live matrix — a static,
        // slightly smaller/offset copy of the unlit dots, sampled at half
        // density (every other column) to keep the extra arc()+fill() calls
        // cheap. Fixed low alpha, no shadowBlur.
        const bgScale = 0.92
        const bgDx = cellW * 0.6
        const bgDy = -cellH * 0.4
        ctx.fillStyle = `rgba(${CYAN}, 0.045)`
        for (let c = 0; c < cols; c += 2) {
          const bx = c * cellW * bgScale + cellW / 2 + bgDx
          for (let r = 0; r < vizRows; r++) {
            const by = r * cellH * bgScale + cellH / 2 + bgDy
            ctx.beginPath()
            ctx.arc(bx, by, dotRadius * bgScale * 0.55, 0, Math.PI * 2)
            ctx.fill()
          }
        }

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

  .now-playing-body {
    display: flex;
    align-items: stretch;
    justify-content: center;
    gap: 1.5rem;
    width: 100%;
  }

  .now-playing-main {
    display: flex;
    flex-direction: column;
    align-items: center;
    flex: 1 1 auto;
    min-width: 0;
  }

  /* Instrument panel column — decorative, left side only. Hidden below
     ~820px where there's no room for it without crowding the readable
     content on the right; below ~1080px the tunnel keeps its full height
     and the two mini panels drop under it instead of squeezing in beside it. */
  .side-panels {
    flex: 0 0 42%;
    max-width: 480px;
    display: none;
    flex-direction: column;
    gap: 0.6rem;
  }
  @media (min-width: 820px) {
    .side-panels {
      display: flex;
    }
  }

  .smith-tunnel-wrap {
    position: relative;
    flex: 1 1 auto;
    min-height: 200px;
    border: 1px solid var(--term-amber-dim);
    overflow: hidden;
  }
  .smith-tunnel-canvas {
    display: block;
    width: 100%;
    height: 100%;
  }

  /* Two small standalone panels — planet + the abstract-shape cycle — sit
     under the tunnel, side by side, same bracket-corner/hairline-border
     language as the tunnel and the rest of the HUD. */
  .mini-panels {
    display: flex;
    flex: 0 0 auto;
    gap: 0.6rem;
  }
  .mini-panel-wrap {
    position: relative;
    flex: 1 1 50%;
    height: 110px;
    border: 1px solid var(--term-amber-dim);
    overflow: hidden;
  }
  .mini-panel-canvas {
    display: block;
    width: 100%;
    height: 100%;
  }

  /* Scaled-down .viz-mode-toggle for the abstract-shape mini panel — same
     look, smaller footprint to fit the 110px-tall panel. */
  .mini-panel-toggle {
    /* Pinned to the wrap's top edge, not bottom: the wrap's bottom edge can
       sit close to the viewport floor, where the fixed global player bar
       (Player.vue, position: fixed; bottom: 0; z-index: 100) renders on
       top of it — a bottom-anchored toggle's hit-box overlapped that bar's
       full-width seek slider there, so clicks meant for the toggle landed
       on the slider instead and seeked playback. */
    position: absolute;
    top: 4px;
    right: 4px;
    z-index: 2;
    background: rgba(10, 8, 0, 0.6);
    border: 1px solid var(--term-amber-dim);
    color: var(--term-amber-dim);
    font-size: 0.6rem;
    letter-spacing: 0.06em;
    text-transform: uppercase;
    padding: 0.15rem 0.4rem;
    cursor: pointer;
  }
  .mini-panel-toggle:hover {
    color: var(--term-amber);
    border-color: var(--term-amber);
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
    /* vue-slider-component sets an inline `width: auto` on its own root
       element, which beats this width rule entirely. In this flex column
       (.now-playing-main, align-items: center) that leaves the item
       shrink-to-fit with no in-flow content to size against (the slider's
       internals are all position: absolute), collapsing it to 0 width.
       align-self: stretch forces the cross-axis size regardless of the
       inline width, restoring the intended full-width layout. */
    align-self: stretch;
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
