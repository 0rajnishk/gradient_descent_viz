<template>
  <div class="h-dvh w-dvw bg-slate-950 text-slate-100">
    <div class="flex h-full flex-col">
      <header class="border-b border-slate-800 bg-slate-950/80 backdrop-blur">
        <div class="mx-auto flex max-w-[1400px] items-center justify-between gap-3 px-4 py-3">
          <div class="min-w-0">
            <div class="truncate text-lg font-semibold">Gradient Descent Viz (Vite) — M0</div>
            <div class="truncate text-xs text-slate-400">Surface + Vanilla GD + Path (hello viz)</div>
          </div>

          <div class="flex items-center gap-2">
            <button
              class="rounded-md bg-slate-800 px-3 py-2 text-sm font-medium hover:bg-slate-700"
              @click="togglePlay"
            >
              {{ isPlaying ? "Pause" : "Play" }}
            </button>
            <button
              class="rounded-md bg-slate-800 px-3 py-2 text-sm font-medium hover:bg-slate-700"
              @click="reset"
            >
              Reset
            </button>
            <label class="hidden select-none items-center gap-2 text-sm sm:flex">
              <input type="checkbox" v-model="wireframe" />
              <span class="text-slate-300">Wireframe</span>
            </label>
            <div
              v-if="isConverged"
              class="hidden rounded-md border border-emerald-700/50 bg-emerald-950/40 px-2 py-1 text-xs text-emerald-200 sm:block"
            >
              Converged
            </div>
          </div>
        </div>
      </header>

      <main class="mx-auto flex min-h-0 w-full max-w-[1400px] flex-1 flex-col gap-4 px-4 py-4">
        <section class="relative min-h-0 flex-1 overflow-hidden rounded-xl border border-slate-800 bg-slate-900">
          <div ref="threeHost" class="absolute inset-0"></div>

          <div class="pointer-events-none absolute left-3 top-3 flex flex-col gap-1">
            <div class="rounded-md bg-slate-950/70 px-2 py-1 text-xs text-slate-200">
              Steps: <span class="font-semibold">{{ steps }}</span>
            </div>
            <div class="rounded-md bg-slate-950/70 px-2 py-1 text-xs text-slate-200">
              LR: <span class="font-semibold">{{ learningRate }}</span>
            </div>
          </div>
        </section>
      </main>
    </div>
  </div>
</template>

<script>
import { markRaw } from "vue";
import { DEFAULT_START, DEFAULT_SURFACE } from "./app/defaults.js";
import { Viz3D } from "./viz/Viz3D.js";
import { VanillaGradientDescent } from "./gd/optimizers.js";

function log(...args) {
  // Centralized logger so we can easily turn it off later.
  console.log("[M0]", ...args);
}

export default {
  name: "App",
  data() {
    return {
      isPlaying: true,
      wireframe: false,
      steps: 0,

      // M0 fixed config
      learningRate: 1e-3,
      gridSize: 51,
      // The raw surface is already fairly tall; keep default Y scale modest so it doesn't look like a skyscraper.
      yScale: 0.5,
      surfaceName: DEFAULT_SURFACE,

      viz: null,
      optimizer: null,
      rafId: null,
      frameCounter: 0,

      // Debug controls
      debug: false,
      tickLogs: 0,
    };
  },
  computed: {
    isConverged() {
      return !!this.optimizer?.isConverged?.();
    },
  },
  watch: {
    wireframe(next) {
      if (this.debug) log("wireframe changed:", next);
    },
    isPlaying(next) {
      if (this.debug) log("isPlaying changed:", next);
    },
  },
  methods: {
    togglePlay() {
      if (this.debug) log("togglePlay clicked");
      this.isPlaying = !this.isPlaying;
    },
    reset() {
      if (this.debug) log("reset clicked");
      this.steps = 0;
      this.optimizer.setFunctionName(this.surfaceName);
      this.optimizer.learning_rate = this.learningRate;
      this.optimizer.setStartingPosition(DEFAULT_START.x, DEFAULT_START.z);
      this.optimizer.resetPositionAndComputeGradient();

      const p = this.optimizer.position();
      this.viz.resetPath();
      this.viz.appendPathPoint(p.x, p.z);
      this.viz.setBallPosition(p.x, p.z);

      if (this.debug) log("reset done; pos=", p, "grad=", { x: this.optimizer.gradX(), z: this.optimizer.gradZ() });
    },
    tick() {
      try {
        // If we've converged, stop auto-playing to avoid the "Play does nothing" confusion.
        if (this.isPlaying && this.optimizer.isConverged()) {
          this.isPlaying = false;
          if (this.debug) log("auto-paused (converged)");
        }

        // M0: simple stepping: 1 step every other frame to keep motion readable.
        const shouldStep = this.isPlaying && this.frameCounter % 2 === 0 && !this.optimizer.isConverged();
        if (shouldStep) {
          const p = this.optimizer.takeGradientStep();
          this.viz.appendPathPoint(p.x, p.z);
          this.viz.setBallPosition(p.x, p.z);
          this.steps += 1;

          if (this.debug && this.tickLogs < 5) {
            this.tickLogs += 1;
            log("step", { steps: this.steps, pos: p, grad: { x: this.optimizer.gradX(), z: this.optimizer.gradZ() } });
          }
        } else if (this.debug && this.tickLogs < 5) {
          this.tickLogs += 1;
          log("tick (no step)", {
            isPlaying: this.isPlaying,
            converged: this.optimizer.isConverged(),
            frameCounter: this.frameCounter,
          });
        }

        // Periodic heartbeat (every ~2s at 60fps)
        if (this.debug && this.frameCounter % 120 === 0) {
          log("heartbeat", {
            isPlaying: this.isPlaying,
            steps: this.steps,
            converged: this.optimizer?.isConverged?.(),
          });
        }

        this.frameCounter = (this.frameCounter + 1) % 1_000_000;
        this.viz.setWireframe(this.wireframe);
        this.viz.render();
      } catch (err) {
        console.error("[M0] tick error:", err);
      } finally {
        this.rafId = requestAnimationFrame(this.tick);
      }
    },
  },
  mounted() {
    // Enable debug logs with `?debug=1`
    try {
      const params = new URLSearchParams(window.location.search);
      this.debug = params.get("debug") === "1";
    } catch {
      this.debug = false;
    }
    if (this.debug) log("mounted (debug enabled)");
    // IMPORTANT: Three.js objects must NOT be made reactive (Vue Proxy breaks WebGLRenderer internals).
    this.viz = markRaw(new Viz3D(this.$refs.threeHost));
    this.viz.setSurface({
      functionName: this.surfaceName,
      gridSize: this.gridSize,
      yScale: this.yScale,
    });

    // Ensure the renderer picks up the final layout size (important for flex/grid layouts).
    this.$nextTick(() => {
      if (this.viz) {
        this.viz.resize();
        const host = this.$refs.threeHost;
        if (this.debug) log("after layout:", { hostW: host?.clientWidth, hostH: host?.clientHeight });
      }
    });

    // Optimizer is pure JS, but still markRaw to avoid accidental Proxy edge-cases.
    this.optimizer = markRaw(
      new VanillaGradientDescent({
      functionName: this.surfaceName,
      learningRate: this.learningRate,
      })
    );
    this.optimizer.setStartingPosition(DEFAULT_START.x, DEFAULT_START.z);
    this.optimizer.resetPositionAndComputeGradient();

    const p = this.optimizer.position();
    this.viz.resetPath();
    this.viz.appendPathPoint(p.x, p.z);
    this.viz.setBallPosition(p.x, p.z);

    if (this.debug) {
      log("init optimizer", {
        start: DEFAULT_START,
        pos: p,
        grad: { x: this.optimizer.gradX(), z: this.optimizer.gradZ() },
        converged: this.optimizer.isConverged(),
      });
    }

    this.tick = this.tick.bind(this);
    this.rafId = requestAnimationFrame(this.tick);
  },
  beforeUnmount() {
    if (this.rafId) cancelAnimationFrame(this.rafId);
    if (this.viz) this.viz.dispose();
  },
};
</script>
