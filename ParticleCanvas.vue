<script setup lang="ts">
  import { ref, onMounted } from 'vue';

  class Particle {
    private x: number;
    private y: number;
    private vx: number;
    private vy: number;
    private size: number;
    private color: string;
    private ctx: CanvasRenderingContext2D;

    constructor(
      ctx: CanvasRenderingContext2D,
      x: number,
      y: number,
      vx: number,
      vy: number,
      size: number,
      color: string
    ) {
      this.ctx = ctx;
      this.x = x;
      this.y = y;
      this.vx = vx;
      this.vy = vy;
      this.size = size;
      this.color = color;
    }

    public update(mouse: { x: number | null; y: number | null }, canvas: HTMLCanvasElement, repulsionRadius: number) {
      if (this.x < 0 || this.x > canvas.width) this.vx *= -1;
      if (this.y < 0 || this.y > canvas.height) this.vy *= -1;

      if (mouse.x !== null && mouse.y !== null) {
        const dx = this.x - mouse.x;
        const dy = this.y - mouse.y;
        const distSq = dx * dx + dy * dy;

        if (distSq < repulsionRadius * repulsionRadius) {
          const angle = Math.atan2(dy, dx);
          const targetX = mouse.x + Math.cos(angle) * repulsionRadius;
          const targetY = mouse.y + Math.sin(angle) * repulsionRadius;

          this.x = Math.min(Math.max(targetX, 0), canvas.width);
          this.y = Math.min(Math.max(targetY, 0), canvas.height);
        }
      }

      this.x += this.vx;
      this.y += this.vy;

      this.draw();
    }

    private draw() {
      this.ctx.fillStyle = this.color;
      this.ctx.beginPath();
      this.ctx.arc(this.x, this.y, this.size, 0, Math.PI * 2);
      this.ctx.fill();
    }
  }

  class ParticleSystem {
    private canvasRef: HTMLCanvasElement;
    private ctx: CanvasRenderingContext2D;
    private particles: Particle[] = [];
    private density = 25000;
    private lineDistance = 150;
    private repulsionRadius = 200;
    private particleMaxSize = 10;
    // private colorRGB = '255, 255, 224';
    private mouse = { x: null as number | null, y: null as number | null };

    constructor(canvasRef: HTMLCanvasElement) {
      this.canvasRef = canvasRef;
      this.ctx = canvasRef.getContext('2d')!;
      this.resizeCanvas();
      this.adjustParticleCount();
      this.bindEvents();
    }

    public start() {
      this.animate();
    }

    private getRandomArbitrary(min: number, max: number): number {
      return Math.random() * (max - min) + min;
    }

    private generateRandomParticle(): Particle {
      const x = Math.random() * this.canvasRef.width;
      const y = Math.random() * this.canvasRef.height;
      const vx = this.getRandomArbitrary(-1, 1);
      const vy = this.getRandomArbitrary(-1, 1);
      const size = this.getRandomArbitrary(1, this.particleMaxSize);
      const r = Math.floor(Math.random() * 256);
      const g = Math.floor(Math.random() * 256);
      const b = Math.floor(Math.random() * 256);
      const a = 1 - size / this.particleMaxSize;
      const color = `rgba(${r}, ${g}, ${b}, ${a})`;

      return new Particle(this.ctx, x, y, vx, vy, size, color);
    }

    private calculateParticleNum(): number {
      return Math.floor((this.canvasRef.width * this.canvasRef.height) / this.density);
    }

    private adjustParticleCount() {
      const targetCount = this.calculateParticleNum();
      const currentCount = this.particles.length;
      const diff = targetCount - currentCount;

      if (diff > 0) {
        for (let i = 0; i < diff; i++) {
          this.particles.push(this.generateRandomParticle());
        }
      } else if (diff < 0) {
        this.particles.splice(diff);
      }

      // console.log(`Adjusted particle count to: ${this.particles.length}`);
    }

    private connect() {
      const sortedParticles = [...this.particles].sort((a, b) => a.x - b.x);
      for (let i = 0; i < sortedParticles.length; i++) {
        const p1 = sortedParticles[i];
        for (let j = i + 1; j < sortedParticles.length; j++) {
          const p2 = sortedParticles[j];
          if (p2.x - p1.x > this.lineDistance) break;

          const dy = p1.y - p2.y;
          const dx = p1.x - p2.x;
          const distanceSq = dx * dx + dy * dy;

          if (distanceSq < this.lineDistance * this.lineDistance) {
            const distance = Math.sqrt(distanceSq);
            this.ctx.strokeStyle = `rgba(${this.colorRGB}, ${1 - distance / this.lineDistance})`;
            this.ctx.lineWidth = 0.5;
            this.ctx.beginPath();
            this.ctx.moveTo(p1.x, p1.y);
            this.ctx.lineTo(p2.x, p2.y);
            this.ctx.stroke();
          }
        }
      }
    }

    private animate() {
      requestAnimationFrame(() => this.animate());
      this.ctx.clearRect(0, 0, this.canvasRef.width, this.canvasRef.height);
      this.particles.forEach(p => p.update(this.mouse, this.canvasRef, this.repulsionRadius));
      // this.connect(); // optional
    }

    private bindEvents() {
      window.addEventListener('mousemove', (e) => {
        this.mouse.x = e.clientX;
        this.mouse.y = e.clientY;
      });

      window.addEventListener('mouseout', () => {
        this.mouse.x = null;
        this.mouse.y = null;
      });

      window.addEventListener('resize', () => {
        this.resizeCanvas();
        this.adjustParticleCount();
      });
    }

    private resizeCanvas() {
      this.canvasRef.width = window.innerWidth;
      this.canvasRef.height = window.innerHeight;
    }
  }

  const canvasRef = ref<HTMLCanvasElement | null>(null);
  let particleSystem: ParticleSystem;

  onMounted(() => {
    if (!canvasRef.value) return;
    particleSystem = new ParticleSystem(canvasRef.value);
    particleSystem.start();
  });
</script>

<template>
  <canvas ref="canvasRef" class="particle-canvas"></canvas>
</template>

<style>
.particle-canvas {
  position: fixed;
  top: 0;
  left: 0;
  z-index: 0;
  width: 100vw;
  height: 100vh;
}
</style>
