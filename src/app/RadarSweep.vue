<template>
  <div class="radar-widget hud-bracket hud-pulse" :class="{ compact }">
    <div v-if="!compact" class="radar-label">
      <span>AUDIO SCAN</span>
    </div>
    <div class="radar-circle">
      <div class="radar-ring ring-25" />
      <div class="radar-ring ring-50" />
      <div class="radar-ring ring-75" />
      <div class="radar-ring ring-100" />
      <div class="radar-crosshair-h" />
      <div class="radar-crosshair-v" />
      <div class="radar-sweep" />
      <div
        v-for="blip in blips"
        :key="blip.id"
        class="radar-blip"
        :style="{ top: blip.top, left: blip.left, animationDelay: blip.delay }"
      />
    </div>
    <div v-if="!compact" class="radar-readout">
      <span>SIG: LOCKED</span>
      <span>CH: 02/02</span>
      <span>GAIN: {{ gain }}%</span>
    </div>
  </div>
</template>
<script lang="ts">
  import { defineComponent } from 'vue'

  export default defineComponent({
    props: {
      compact: { type: Boolean, default: false },
    },
    data() {
      return {
        gain: 78,
        blips: [
          { id: 1, top: '30%', left: '62%', delay: '0s' },
          { id: 2, top: '68%', left: '40%', delay: '1.3s' },
          { id: 3, top: '45%', left: '25%', delay: '2.6s' },
        ],
      }
    },
  })
</script>
<style scoped>
.radar-widget {
  margin: 1rem 0.5rem;
  padding: 0.75rem;
  position: relative;
}

.radar-widget.compact {
  margin: 0;
  padding: 0.25rem;
  width: 52px;
  flex: 0 0 auto;
}

.radar-label {
  font-size: 0.8rem;
  letter-spacing: 0.12em;
  text-transform: uppercase;
  color: var(--term-amber-dim);
  margin-bottom: 0.5rem;
}

.radar-circle {
  position: relative;
  width: 100%;
  aspect-ratio: 1 / 1;
  border-radius: 50%;
  overflow: hidden;
  background: radial-gradient(circle, rgba(255, 122, 26, 0.06) 0%, transparent 72%);
  border: 1px solid var(--term-amber-dim);
}

.radar-ring {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  border: 1px solid rgba(255, 122, 26, 0.35);
  border-radius: 50%;
}
.ring-25 { width: 25%; height: 25%; }
.ring-50 { width: 50%; height: 50%; }
.ring-75 { width: 75%; height: 75%; }
.ring-100 { width: 100%; height: 100%; }

.radar-crosshair-h, .radar-crosshair-v {
  position: absolute;
  background: rgba(255, 122, 26, 0.25);
}
.radar-crosshair-h { top: 50%; left: 0; width: 100%; height: 1px; }
.radar-crosshair-v { top: 0; left: 50%; width: 1px; height: 100%; }

.radar-sweep {
  position: absolute;
  inset: 0;
  background: conic-gradient(
    from 0deg,
    transparent 0deg,
    rgba(255, 122, 26, 0.45) 18deg,
    transparent 42deg
  );
  animation: radar-spin 4s linear infinite;
}

@keyframes radar-spin {
  to { transform: rotate(360deg); }
}

.radar-blip {
  position: absolute;
  width: 4px;
  height: 4px;
  border-radius: 50%;
  background: var(--term-amber);
  box-shadow: 0 0 4px 1px var(--term-amber);
  animation: radar-blip-pulse 3.9s ease-in-out infinite;
  opacity: 0;
}

@keyframes radar-blip-pulse {
  0%, 92%, 100% { opacity: 0; }
  4%, 10% { opacity: 1; }
}

.radar-readout {
  display: flex;
  flex-direction: column;
  gap: 0.1rem;
  margin-top: 0.5rem;
  font-size: 0.85rem;
  letter-spacing: 0.06em;
  color: var(--term-amber-dim);
}
</style>
