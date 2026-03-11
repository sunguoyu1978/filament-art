<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { RefreshCw, Download, Settings2 } from 'lucide-react';

const canvasRef = ref<HTMLCanvasElement | null>(null);
const containerRef = ref<HTMLDivElement | null>(null);

// Configuration
const config = ref({
  puffCount: 25,
  filamentsPerPuff: 1200,
  minRadius: 80,
  maxRadius: 240,
  coreSize: 0.4,
  opacity: 0.03,
  colors: [
    { r: 180, g: 120, b: 60 }, // Brownish
    { r: 60, g: 70, b: 90 },   // Dark Blue/Grey
    { r: 210, g: 140, b: 80 }, // Orange/Tan
    { r: 40, g: 45, b: 55 }    // Deep Grey
  ]
});

// Gaussian Random Function
function randomGaussian(mean = 0, std = 1) {
  const u = 1 - Math.random();
  const v = 1 - Math.random();
  const z = Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v);
  return z * std + mean;
}

function draw() {
  const canvas = canvasRef.value;
  if (!canvas) return;
  const ctx = canvas.getContext('2d');
  if (!ctx) return;

  // Clear canvas
  ctx.fillStyle = '#ffffff';
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  // Draw puffs
  for (let i = 0; i < config.value.puffCount; i++) {
    const cx = Math.random() * canvas.width;
    const cy = Math.random() * canvas.height;
    
    // Pick a random color from palette
    const baseColor = config.value.colors[Math.floor(Math.random() * config.value.colors.length)];
    const radius = config.value.minRadius + Math.random() * (config.value.maxRadius - config.value.minRadius);

    // Draw a soft shadow/glow for depth
    const gradient = ctx.createRadialGradient(cx, cy, 0, cx, cy, radius);
    gradient.addColorStop(0, `rgba(${baseColor.r}, ${baseColor.g}, ${baseColor.b}, ${config.value.opacity * 2})`);
    gradient.addColorStop(1, 'rgba(255, 255, 255, 0)');
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(cx, cy, radius, 0, Math.PI * 2);
    ctx.fill();

    for (let j = 0; j < config.value.filamentsPerPuff; j++) {
      const angle = Math.random() * Math.PI * 2;
      
      // Use Gaussian for length to create the "fuzzy" edge effect
      const length = Math.abs(randomGaussian(radius * 0.7, radius * 0.25));
      
      // Add slight color jitter for realism
      const jitter = (Math.random() - 0.5) * 30;
      const r = Math.max(0, Math.min(255, baseColor.r + jitter));
      const g = Math.max(0, Math.min(255, baseColor.g + jitter));
      const b = Math.max(0, Math.min(255, baseColor.b + jitter));

      ctx.beginPath();
      ctx.moveTo(cx, cy);
      
      // Calculate control point for curvature
      const cpAngle = angle + (Math.random() - 0.5) * 0.5;
      const cpLength = length * 0.5;
      const cpx = cx + Math.cos(cpAngle) * cpLength;
      const cpy = cy + Math.sin(cpAngle) * cpLength;

      const tx = cx + Math.cos(angle) * length;
      const ty = cy + Math.sin(angle) * length;
      
      ctx.strokeStyle = `rgba(${r}, ${g}, ${b}, ${config.value.opacity})`;
      ctx.lineWidth = 0.3 + Math.random() * 0.4;
      ctx.quadraticCurveTo(cpx, cpy, tx, ty);
      ctx.stroke();
    }

    // Add a small dense core for 3D depth
    const coreRadius = radius * config.value.coreSize;
    const coreGradient = ctx.createRadialGradient(cx, cy, 0, cx, cy, coreRadius);
    
    // Core is brighter/denser in the center
    coreGradient.addColorStop(0, `rgba(255, 255, 255, ${config.value.opacity * 10})`);
    coreGradient.addColorStop(0.5, `rgba(${baseColor.r}, ${baseColor.g}, ${baseColor.b}, ${config.value.opacity * 5})`);
    coreGradient.addColorStop(1, 'rgba(255, 255, 255, 0)');
    
    ctx.fillStyle = coreGradient;
    ctx.beginPath();
    ctx.arc(cx, cy, coreRadius, 0, Math.PI * 2);
    ctx.fill();
  }
}

function resizeCanvas() {
  if (containerRef.value && canvasRef.value) {
    canvasRef.value.width = containerRef.value.clientWidth;
    canvasRef.value.height = containerRef.value.clientHeight;
    draw();
  }
}

const handleResize = () => {
  resizeCanvas();
};

onMounted(() => {
  window.addEventListener('resize', handleResize);
  resizeCanvas();
});

onUnmounted(() => {
  window.removeEventListener('resize', handleResize);
});

function downloadImage() {
  const canvas = canvasRef.value;
  if (!canvas) return;
  const link = document.createElement('a');
  link.download = 'filament-art.png';
  link.href = canvas.toDataURL();
  link.click();
}
</script>

<template>
  <div class="min-h-screen bg-[#f8f8f8] flex flex-col font-sans text-[#1a1a1a]">
    <!-- Header -->
    <header class="p-6 border-b border-black/5 flex justify-between items-center bg-white/80 backdrop-blur-md sticky top-0 z-10">
      <div>
        <h1 class="text-2xl font-medium tracking-tight italic serif">Filament Art</h1>
        <p class="text-xs text-black/40 uppercase tracking-widest mt-1">Generative Gaussian Distribution</p>
      </div>
      
      <div class="flex gap-3">
        <button 
          @click="draw"
          class="p-2 rounded-full hover:bg-black/5 transition-colors group"
          title="Regenerate"
        >
          <RefreshCw class="w-5 h-5 group-active:rotate-180 transition-transform duration-500" />
        </button>
        <button 
          @click="downloadImage"
          class="p-2 rounded-full hover:bg-black/5 transition-colors"
          title="Download"
        >
          <Download class="w-5 h-5" />
        </button>
      </div>
    </header>

    <!-- Main Content -->
    <main class="flex-1 p-4 md:p-8 flex flex-col lg:flex-row gap-8">
      <!-- Canvas Area -->
      <div 
        ref="containerRef"
        class="flex-1 bg-white rounded-2xl shadow-sm border border-black/5 overflow-hidden relative min-h-[500px]"
      >
        <canvas 
          ref="canvasRef"
          class="w-full h-full block cursor-crosshair"
        ></canvas>
      </div>

      <!-- Controls Sidebar -->
      <aside class="w-full lg:w-80 flex flex-col gap-6">
        <div class="bg-white p-6 rounded-2xl shadow-sm border border-black/5">
          <div class="flex items-center gap-2 mb-6">
            <Settings2 class="w-4 h-4 text-black/40" />
            <h2 class="text-xs font-semibold uppercase tracking-wider text-black/40">Parameters</h2>
          </div>

          <div class="space-y-6">
            <!-- Puff Count -->
            <div class="space-y-2">
              <div class="flex justify-between text-xs">
                <label class="font-medium">Seed Points</label>
                <span class="font-mono text-black/40">{{ config.puffCount }}</span>
              </div>
              <input 
                v-model.number="config.puffCount" 
                type="range" min="1" max="100" 
                class="w-full accent-black"
                @input="draw"
              />
            </div>

            <!-- Filaments -->
            <div class="space-y-2">
              <div class="flex justify-between text-xs">
                <label class="font-medium">Filament Density</label>
                <span class="font-mono text-black/40">{{ config.filamentsPerPuff }}</span>
              </div>
              <input 
                v-model.number="config.filamentsPerPuff" 
                type="range" min="100" max="5000" step="100"
                class="w-full accent-black"
                @input="draw"
              />
            </div>

            <!-- Radius -->
            <div class="space-y-2">
              <div class="flex justify-between text-xs">
                <label class="font-medium">Min Radius</label>
                <span class="font-mono text-black/40">{{ config.minRadius }}px</span>
              </div>
              <input 
                v-model.number="config.minRadius" 
                type="range" min="10" max="300" 
                class="w-full accent-black"
                @input="draw"
              />
            </div>

            <div class="space-y-2">
              <div class="flex justify-between text-xs">
                <label class="font-medium">Max Radius</label>
                <span class="font-mono text-black/40">{{ config.maxRadius }}px</span>
              </div>
              <input 
                v-model.number="config.maxRadius" 
                type="range" min="20" max="600" 
                class="w-full accent-black"
                @input="draw"
              />
            </div>

            <!-- Opacity -->
            <div class="space-y-2">
              <div class="flex justify-between text-xs">
                <label class="font-medium">Core Size</label>
                <span class="font-mono text-black/40">{{ (config.coreSize * 100).toFixed(0) }}%</span>
              </div>
              <input 
                v-model.number="config.coreSize" 
                type="range" min="0.1" max="0.8" step="0.05"
                class="w-full accent-black"
                @input="draw"
              />
            </div>

            <div class="space-y-2">
              <div class="flex justify-between text-xs">
                <label class="font-medium">Ink Opacity</label>
                <span class="font-mono text-black/40">{{ (config.opacity * 100).toFixed(0) }}%</span>
              </div>
              <input 
                v-model.number="config.opacity" 
                type="range" min="0.005" max="0.1" step="0.005"
                class="w-full accent-black"
                @input="draw"
              />
            </div>
          </div>

          <div class="mt-8 pt-6 border-t border-black/5">
            <p class="text-[10px] text-black/30 leading-relaxed">
              * Each filament is drawn using a Gaussian distribution for length, creating a dense core and organic, fuzzy edges.
            </p>
          </div>
        </div>
      </aside>
    </main>

    <!-- Footer -->
    <footer class="p-6 text-center text-[10px] text-black/20 uppercase tracking-[0.2em]">
      &copy; 2026 Filament Generative Engine
    </footer>
  </div>
</template>

<style>
.serif {
  font-family: 'Georgia', serif;
}

/* Custom range input styling for a cleaner look */
input[type=range] {
  -webkit-appearance: none;
  background: transparent;
}

input[type=range]::-webkit-slider-runnable-track {
  width: 100%;
  height: 2px;
  background: #e5e5e5;
  border-radius: 1px;
}

input[type=range]::-webkit-slider-thumb {
  -webkit-appearance: none;
  height: 12px;
  width: 12px;
  border-radius: 50%;
  background: #000;
  cursor: pointer;
  margin-top: -5px;
  transition: transform 0.1s ease;
}

input[type=range]:active::-webkit-slider-thumb {
  transform: scale(1.2);
}
</style>
