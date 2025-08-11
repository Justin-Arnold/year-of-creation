<template>
  <div class="game-container">
    <div class="fixed inset-0 flex flex-col items-center justify-center bg-gradient-to-b from-blue-300 to-green-200">
      
      <div v-if="gameState === 'menu'" class="text-center z-10 relative">
        <h1 class="text-6xl font-bold text-gray-800 mb-4">Box Runner</h1>
        <p class="text-xl text-gray-600 mb-2">One button. Infinite possibilities.</p>
        <p class="text-lg text-gray-500 mb-6">Click/tap to jump over obstacles!</p>
        <p v-if="bestScore > 0" class="text-lg text-orange-600 mb-4">Best Score: {{ bestScore }}</p>
        <button 
          @click="startGame" 
          class="bg-black text-white px-8 py-4 text-xl font-bold hover:bg-gray-800 transition-colors"
        >
          START GAME
        </button>
      </div>

      <div v-if="gameState === 'playing'" class="fixed top-4 left-4 z-10">
        <div class="text-2xl font-bold text-black">Score: {{ score }}</div>
        <div v-if="bestScore > 0" class="text-lg text-gray-700">Best: {{ bestScore }}</div>
      </div>

      <div v-if="gameState === 'gameOver'" class="text-center z-10 relative">
        <h2 class="text-4xl font-bold text-gray-800 mb-4">Game Over!</h2>
        <p class="text-2xl text-gray-700 mb-2">Score: {{ score }}</p>
        <p v-if="isNewRecord" class="text-xl text-green-600 mb-4 animate-pulse">🎉 NEW BEST SCORE! 🎉</p>
        <p v-else-if="score < bestScore" class="text-lg text-red-500 mb-4">{{ getCommentary() }}</p>
        <p v-else class="text-lg text-blue-500 mb-4">{{ getCommentary() }}</p>
        <button 
          @click="startGame" 
          class="bg-black text-white px-6 py-3 text-lg font-bold hover:bg-gray-800 transition-colors mr-4"
        >
          TRY AGAIN
        </button>
        <button 
          @click="goToMenu" 
          class="bg-gray-600 text-white px-6 py-3 text-lg font-bold hover:bg-gray-500 transition-colors"
        >
          MENU
        </button>
      </div>

      <canvas 
        ref="canvas" 
        @click="jump" 
        @keydown.space.prevent="jump"
        tabindex="0"
        class="absolute inset-0 cursor-pointer outline-none"
      ></canvas>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue';

const canvas = ref<HTMLCanvasElement | null>(null);
let ctx: CanvasRenderingContext2D | null = null;
let animationId: number;

const gameState = ref<'menu' | 'playing' | 'gameOver'>('menu');
const score = ref(0);
const bestScore = ref(0);
const isNewRecord = ref(false);

interface Player {
  x: number;
  y: number;
  width: number;
  height: number;
  velocityY: number;
  isGrounded: boolean;
  isJumping: false;
}

interface Obstacle {
  x: number;
  y: number;
  width: number;
  height: number;
  passed: boolean;
}

const player: Player = {
  x: 100,
  y: 0,
  width: 30,
  height: 30,
  velocityY: 0,
  isGrounded: true,
  isJumping: false
};

const obstacles: Obstacle[] = [];
const gravity = 0.8;
const jumpPower = -15;
const gameSpeed = 4;
const groundY = ref(0);

const humorousComments = [
  "That box had places to be! 📦",
  "Gravity: 1, Box: 0",
  "The obstacles weren't even trying...",
  "Physics lesson: What goes up... hits stuff.",
  "Even a square peg can't fit in a round hole... or over it.",
  "Box.exe has stopped working",
  "That's what happens when you skip leg day!",
  "The ground missed you!",
  "Obstacle: 'Am I a joke to you?' Box: 'Yes.'",
  "404: Jump not found"
];

const encouragingComments = [
  "Not bad for a box! 📦",
  "You're getting the hang of it!",
  "Solid attempt!",
  "Keep jumping, brave box!",
  "Practice makes perfect!",
  "The box believes in you!",
  "Every jump counts!",
  "You're boxing above your weight!"
];

function getCommentary(): string {
  if (score.value < bestScore.value / 2) {
    return humorousComments[Math.floor(Math.random() * humorousComments.length)];
  } else {
    return encouragingComments[Math.floor(Math.random() * encouragingComments.length)];
  }
}

function loadBestScore(): void {
  const saved = localStorage.getItem('parkour-runner-best');
  if (saved) {
    bestScore.value = parseInt(saved);
  }
}

function saveBestScore(): void {
  localStorage.setItem('parkour-runner-best', bestScore.value.toString());
}

function generateObstacle(): Obstacle {
  const minHeight = 40;
  const maxHeight = 120;
  const height = minHeight + Math.random() * (maxHeight - minHeight);
  
  return {
    x: canvas.value!.width + 50,
    y: groundY.value - height,
    width: 20,
    height: height,
    passed: false
  };
}

function isValidObstacleSpacing(newObstacle: Obstacle): boolean {
  if (obstacles.length === 0) return true;
  
  const lastObstacle = obstacles[obstacles.length - 1];
  const distance = newObstacle.x - (lastObstacle.x + lastObstacle.width);
  
  const minDistance = 150;
  const maxJumpDistance = 200;
  
  const totalHeight = Math.max(lastObstacle.height, newObstacle.height);
  const requiredDistance = Math.max(minDistance, totalHeight * 1.5);
  
  return distance >= requiredDistance && distance <= maxJumpDistance;
}

function updateObstacles(): void {
  for (let i = obstacles.length - 1; i >= 0; i--) {
    obstacles[i].x -= gameSpeed;
    
    if (!obstacles[i].passed && obstacles[i].x + obstacles[i].width < player.x) {
      obstacles[i].passed = true;
      score.value += 1;
    }
    
    if (obstacles[i].x + obstacles[i].width < 0) {
      obstacles.splice(i, 1);
    }
  }
  
  if (obstacles.length === 0 || obstacles[obstacles.length - 1].x < canvas.value!.width - 200) {
    let attempts = 0;
    let newObstacle: Obstacle;
    
    do {
      newObstacle = generateObstacle();
      attempts++;
    } while (!isValidObstacleSpacing(newObstacle) && attempts < 10);
    
    if (attempts < 10) {
      obstacles.push(newObstacle);
    }
  }
}

function updatePlayer(): void {
  if (!player.isGrounded) {
    player.velocityY += gravity;
  }
  
  player.y += player.velocityY;
  
  if (player.y >= groundY.value - player.height) {
    player.y = groundY.value - player.height;
    player.velocityY = 0;
    player.isGrounded = true;
    player.isJumping = false;
  }
}

function checkCollisions(): boolean {
  for (const obstacle of obstacles) {
    if (player.x < obstacle.x + obstacle.width &&
        player.x + player.width > obstacle.x &&
        player.y < obstacle.y + obstacle.height &&
        player.y + player.height > obstacle.y) {
      return true;
    }
  }
  return false;
}

function jump(): void {
  if (gameState.value === 'playing' && player.isGrounded) {
    player.velocityY = jumpPower;
    player.isGrounded = false;
    player.isJumping = true;
  }
}

function draw(): void {
  if (!ctx || !canvas.value) return;
  
  ctx.fillStyle = '#87CEEB';
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);
  
  const gradient = ctx.createLinearGradient(0, 0, 0, canvas.value.height);
  gradient.addColorStop(0, '#87CEEB');
  gradient.addColorStop(1, '#90EE90');
  ctx.fillStyle = gradient;
  ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);
  
  ctx.fillStyle = '#228B22';
  ctx.fillRect(0, groundY.value, canvas.value.width, canvas.value.height - groundY.value);
  
  ctx.fillStyle = '#000000';
  ctx.fillRect(player.x, player.y, player.width, player.height);
  
  if (player.isJumping) {
    ctx.fillStyle = '#333333';
    ctx.fillRect(player.x + 5, player.y + 5, 20, 20);
  }
  
  ctx.fillStyle = '#8B4513';
  obstacles.forEach(obstacle => {
    ctx!.fillRect(obstacle.x, obstacle.y, obstacle.width, obstacle.height);
    
    ctx!.fillStyle = '#A0522D';
    ctx!.fillRect(obstacle.x + 2, obstacle.y + 2, obstacle.width - 4, obstacle.height - 4);
    ctx!.fillStyle = '#8B4513';
  });
}

function gameLoop(): void {
  if (gameState.value !== 'playing') return;
  
  updatePlayer();
  updateObstacles();
  
  if (checkCollisions()) {
    gameOver();
    return;
  }
  
  draw();
  animationId = requestAnimationFrame(gameLoop);
}

function startGame(): void {
  gameState.value = 'playing';
  score.value = 0;
  isNewRecord.value = false;
  
  player.x = 100;
  player.y = groundY.value - player.height;
  player.velocityY = 0;
  player.isGrounded = true;
  player.isJumping = false;
  
  obstacles.length = 0;
  
  gameLoop();
  
  if (canvas.value) {
    canvas.value.focus();
  }
}

function gameOver(): void {
  gameState.value = 'gameOver';
  
  if (score.value > bestScore.value) {
    bestScore.value = score.value;
    isNewRecord.value = true;
    saveBestScore();
  }
  
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
}

function goToMenu(): void {
  gameState.value = 'menu';
}

function resizeCanvas(): void {
  if (canvas.value) {
    canvas.value.width = window.innerWidth;
    canvas.value.height = window.innerHeight;
    groundY.value = canvas.value.height * 0.8;
  }
}

function handleKeydown(event: KeyboardEvent): void {
  if (event.code === 'Space') {
    event.preventDefault();
    jump();
  }
}

onMounted(() => {
  loadBestScore();
  
  if (canvas.value) {
    ctx = canvas.value.getContext('2d');
    resizeCanvas();
  }
  
  window.addEventListener('resize', resizeCanvas);
  window.addEventListener('keydown', handleKeydown);
});

onBeforeUnmount(() => {
  if (animationId) {
    cancelAnimationFrame(animationId);
  }
  window.removeEventListener('resize', resizeCanvas);
  window.removeEventListener('keydown', handleKeydown);
});
</script>

<style scoped>
.game-container {
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

canvas {
  display: block;
}

button {
  transition: all 0.2s ease;
}

button:hover {
  transform: translateY(-2px);
}

.animate-pulse {
  animation: pulse 1s ease-in-out infinite alternate;
}

@keyframes pulse {
  from {
    opacity: 1;
  }
  to {
    opacity: 0.5;
  }
}
</style>