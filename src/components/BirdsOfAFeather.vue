<template>
    <div class="fixed h-full w-full grid place-items-center">
        <h2 class="font-serif font-thin text-6xl text-black/10">Birds of a Feather</h2>
    </div>
    <canvas ref="canvas" class="bird-canvas"></canvas>
  </template>
  
  <script setup lang="ts">
  import { ref, onMounted, onBeforeUnmount } from 'vue';
  
  const canvas = ref<HTMLCanvasElement | null>(null);
  let ctx: CanvasRenderingContext2D | null = null;
  
  interface Bird {
    x: number;
    y: number;
    vx: number;
    vy: number;
  }
  
  const birds: Bird[] = [];
  const numBirds = 500;
  const birdSize = 2;
  const mouse = { x: 0, y: 0, isMoving: false };
  
  function initBirds() {
    for (let i = 0; i < numBirds; i++) {
      birds.push({
        x: Math.random() * canvas.value!.width,
        y: Math.random() * canvas.value!.height,
        vx: Math.random() * 2 - 1,
        vy: Math.random() * 2 - 1,
      });
    }
  }
  
  function updateBirds() {
  birds.forEach(bird => {
    // Separation
    let separationX = 0;
    let separationY = 0;
    let cohesionX = 0;
    let cohesionY = 0;
    let alignmentX = 0;
    let alignmentY = 0;
    let flockmates = 0;

    birds.forEach(otherBird => {
      if (otherBird !== bird) {
        const dx = otherBird.x - bird.x;
        const dy = otherBird.y - bird.y;
        const dist = Math.sqrt(dx * dx + dy * dy);

        if (dist < 25) {
          separationX -= dx;
          separationY -= dy;
        } else if (dist < 50) {
          cohesionX += otherBird.x;
          cohesionY += otherBird.y;
          alignmentX += otherBird.vx;
          alignmentY += otherBird.vy;
          flockmates++;
        }
      }
    });

    // Apply separation
    bird.vx += separationX * 0.05;
    bird.vy += separationY * 0.05;

    // Apply cohesion
    if (flockmates > 0) {
      cohesionX = cohesionX / flockmates - bird.x;
      cohesionY = cohesionY / flockmates - bird.y;
      bird.vx += cohesionX * 0.001; // Increase from 0.0005
        bird.vy += cohesionY * 0.001; // Increase from 0.0005
    }

    // Apply alignment
    if (flockmates > 0) {
      alignmentX = alignmentX / flockmates - bird.vx;
      alignmentY = alignmentY / flockmates - bird.vy;
      bird.vx += alignmentX * 0.05;
      bird.vy += alignmentY * 0.05;
    }

    // Avoid mouse
    if (mouse.isMoving) {
      const dx = bird.x - mouse.x;
      const dy = bird.y - mouse.y;
      const dist = Math.sqrt(dx * dx + dy * dy);
      if (dist < 100) {
        bird.vx += dx / dist * 0.5;
        bird.vy += dy / dist * 0.5;
      }
    }

    // Add some randomness
    bird.vx += (Math.random() - 0.5) * 0.3;
    bird.vy += (Math.random() - 0.5) * 0.3;

    // Limit speed
    const speed = Math.sqrt(bird.vx * bird.vx + bird.vy * bird.vy);
    if (speed > 3) {
      bird.vx = (bird.vx / speed) * 3;
      bird.vy = (bird.vy / speed) * 3;
    }

    // Move bird
    bird.x += bird.vx;
    bird.y += bird.vy;

    // Wrap around edges
    if (bird.x < 0) bird.x = canvas.value!.width;
    if (bird.x > canvas.value!.width) bird.x = 0;
    if (bird.y < 0) bird.y = canvas.value!.height;
    if (bird.y > canvas.value!.height) bird.y = 0;
  });
}
  
  function drawBirds() {
    ctx!.clearRect(0, 0, canvas.value!.width, canvas.value!.height);
    birds.forEach(bird => {
      ctx!.beginPath();
      ctx!.moveTo(bird.x, bird.y);
      ctx!.lineTo(bird.x - bird.vx * birdSize - bird.vy * birdSize, bird.y - bird.vy * birdSize + bird.vx * birdSize);
      ctx!.lineTo(bird.x - bird.vx * birdSize + bird.vy * birdSize, bird.y - bird.vy * birdSize - bird.vx * birdSize);
      ctx!.closePath();
      ctx!.fill();
    });
  }
  
  function animate() {
    updateBirds();
    drawBirds();
    requestAnimationFrame(animate);
  }
  
  onMounted(() => {
    if (canvas.value) {
      canvas.value.width = window.innerWidth;
      canvas.value.height = window.innerHeight;
      ctx = canvas.value.getContext('2d');
      if (ctx) {
        ctx.fillStyle = 'black';
        initBirds();
        animate();
      }
    }
  
    window.addEventListener('mousemove', (event) => {
      mouse.x = event.clientX;
      mouse.y = event.clientY;
      mouse.isMoving = true;
    });
  });
  
  onBeforeUnmount(() => {
    window.removeEventListener('mousemove', (event) => {
      mouse.x = event.clientX;
      mouse.y = event.clientY;
      mouse.isMoving = true;
    });
  });
  </script>
  
  <style scoped>
  .bird-canvas {
    width: 100%;
    height: 100%;
    display: block;
  }
  </style>
  