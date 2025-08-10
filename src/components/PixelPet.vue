<template>
  <div class="pixel-world" @click="onWorldClick" @mousemove="onMouseMove">
    <div class="background">
      <div class="clouds">
        <div class="cloud" :style="{ left: cloud1X + 'px' }"></div>
        <div class="cloud" :style="{ left: cloud2X + 'px' }"></div>
      </div>
      <div class="ground"></div>
    </div>
    
    <div 
      class="pixel-pet" 
      :class="[petState, direction, { jumping: isJumping, dead: petState === 'dead', celebrating: celebrationActive }]"
      :style="{ 
        left: petX + 'px', 
        bottom: petY + 'px',
        transform: `scale(${petScale})`
      }"
      @click.stop="onPetClick"
    >
      <div class="pet-sprite">
        <div class="pet-body"></div>
        <div class="pet-head">
          <div class="eyes">
            <div class="eye"></div>
            <div class="eye"></div>
          </div>
        </div>
        <div class="pet-legs">
          <div class="leg"></div>
          <div class="leg"></div>
        </div>
      </div>
    </div>

    <div v-if="showHeartEffect" class="heart-effect" :style="{ left: heartX + 'px', bottom: heartY + 'px' }">
      ❤️
    </div>


    <div v-if="targetX !== null && targetY !== null" class="target-marker" :style="{ left: targetX + 'px', bottom: targetY + 'px' }">
      ✨
    </div>

    <!-- Status Bar -->
    <div class="status-bar" :class="{ celebrating: celebrationActive }">
      <div class="status-section">
        <div class="status-label">Loyalty</div>
        <div class="loyalty-hearts" :class="{ 'max-loyalty': loyaltyLevel >= 5 && maxLoyaltyReached }">
          <span v-for="i in 5" :key="i" class="heart" :class="{ filled: i <= loyaltyLevel }">❤️</span>
        </div>
        <div v-if="loyaltyLevel >= 5 && maxLoyaltyReached" class="achievement-badge">
          👑 BEST FRIEND! 👑
        </div>
      </div>
      
      <div class="status-section">
        <div class="status-label">Patience</div>
        <div class="patience-bar">
          <div class="patience-fill" :style="{ width: `${Math.max(0, (maxCommands - commandCount) / maxCommands * 100)}%` }" :class="{ danger: commandCount >= maxCommands - 1 }"></div>
        </div>
        <div class="patience-text">{{ Math.max(0, maxCommands - commandCount) }}/{{ maxCommands }}</div>
      </div>

      <div class="status-section">
        <div class="status-label">State</div>
        <div class="pet-status" :class="petState">
          {{ petState === 'dead' ? '💀 Dead' : 
               petState === 'following' ? '🐕 Following' :
               petState === 'investigating' ? '🔍 Curious' :
               petState === 'playing' ? '🎾 Playing' :
               petState === 'walking' ? '🚶 Walking' : '😴 Idle' }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'
import confetti from 'canvas-confetti'

const petX = ref(200)
const petY = ref(150)
const petScale = ref(1)
const petState = ref<'idle' | 'walking' | 'playing' | 'jumping' | 'dead' | 'investigating' | 'following'>('idle')
const direction = ref<'left' | 'right'>('right')
const isJumping = ref(false)

const targetX = ref<number | null>(null)
const targetY = ref<number | null>(null)
const moveSpeed = 2

const groundHeight = 120
const toolbarHeight = 60

// Rebellion mechanics
const commandCount = ref(0)
const maxCommands = 3
const rebellionCooldown = ref(0)

// Curiosity mechanics  
const mouseX = ref(0)
const mouseY = ref(0)
const investigationCooldown = ref(0)

// Loyalty mechanics
const petClicks = ref(0)
const loyaltyLevel = ref(0)
const followingPlayer = ref(false)
const maxLoyaltyReached = ref(false)
const celebrationActive = ref(false)

const showHeartEffect = ref(false)
const heartX = ref(0)
const heartY = ref(0)

const showFeedback = ref(false)
const feedbackText = ref('')
const feedbackType = ref('')
const feedbackX = ref(0)
const feedbackY = ref(0)

const cloud1X = ref(50)
const cloud2X = ref(300)

let animationFrame: number | null = null
let idleTimer: number | null = null
let cloudTimer: number | null = null

function onWorldClick(event: MouseEvent) {
  if (petState.value === 'dead') return // Dead pets don't respond
  
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  const clickX = event.clientX - rect.left
  const clickY = event.clientY - rect.top
  
  // Calculate grass area boundaries - full grass field from 40% to ground
  const skyEndY = window.innerHeight * 0.4 // Sky ends at 40% down
  const grassStartY = skyEndY // Grass starts where sky ends
  const grassEndY = window.innerHeight - groundHeight - toolbarHeight // Grass ends above ground
  
  // Only respond to clicks in the grass area
  if (clickY < grassStartY || clickY > grassEndY) {
    return // Ignore clicks outside grass area
  }
  
  // Rebellion mechanic - check if pet is being commanded too much
  if (petState.value === 'walking' && targetX.value !== null) {
    commandCount.value++
    showFeedbackEffect('⚠️ Impatient!', 'warning')
    
    if (commandCount.value >= maxCommands && rebellionCooldown.value <= 0) {
      petState.value = 'dead'
      targetX.value = null
      targetY.value = null
      showFeedbackEffect('💀 Rebellion!', 'death')
      return
    }
  } else {
    // Reset - loyalty gain happens when pet actually reaches destination
    commandCount.value = 0
  }
  
  // Convert click coordinates to pet positioning (bottom-up coordinate system)
  const minY = groundHeight + toolbarHeight + 20 // Bottom boundary (above ground)
  const maxY = window.innerHeight - grassStartY - 20 // Top boundary (below sky)
  
  targetX.value = Math.max(32, Math.min(window.innerWidth - 64, clickX - 32))
  targetY.value = Math.max(minY, Math.min(maxY, window.innerHeight - clickY - 32))
  
  if (clickX > petX.value + 32) {
    direction.value = 'right'
  } else {
    direction.value = 'left'
  }
  
  petState.value = 'walking'
  followingPlayer.value = false // Stop following when given direct command
  resetIdleTimer()
}

function onMouseMove(event: MouseEvent) {
  const rect = (event.currentTarget as HTMLElement).getBoundingClientRect()
  mouseX.value = event.clientX - rect.left
  mouseY.value = event.clientY - rect.top
  const petCenterX = petX.value + 32
  
  const distance = Math.abs(mouseX.value - petCenterX)
  const maxDistance = 200
  
  if (distance < maxDistance) {
    petScale.value = 1 + (maxDistance - distance) / maxDistance * 0.3
  } else {
    petScale.value = 1
  }
  
  // Curiosity mechanic - pet investigates mouse if idle and close
  if (petState.value === 'idle' && distance < 150 && investigationCooldown.value <= 0) {
    investigateMouse()
  }
}

function onPetClick(event: MouseEvent) {
  if (petState.value === 'dead') {
    // Revive pet if clicked when dead
    petState.value = 'idle'
    commandCount.value = 0
    rebellionCooldown.value = 100 // Give pet some time before it can rebel again
    showReviveEffect()
    return
  }
  
  // Loyalty mechanic - build trust
  petClicks.value++
  const previousLoyalty = loyaltyLevel.value
  loyaltyLevel.value = Math.min(5, Math.floor(petClicks.value / 2)) // Faster loyalty gain from clicks
  
  // Check if we just reached max loyalty for the first time
  if (loyaltyLevel.value === 5 && !maxLoyaltyReached.value) {
    maxLoyaltyReached.value = true
    triggerGrandCelebration()
  }
  
  showHeartEffect.value = true
  heartX.value = petX.value + 20
  heartY.value = petY.value + 80
  
  isJumping.value = true
  
  setTimeout(() => {
    showHeartEffect.value = false
    isJumping.value = false
  }, 1000)
  
  const behaviors = ['spin', 'bounce', 'wiggle']
  const randomBehavior = behaviors[Math.floor(Math.random() * behaviors.length)]
  
  petState.value = 'playing'
  setTimeout(() => {
    if (targetX.value === null && petState.value !== 'following') {
      petState.value = 'idle'
    }
  }, 2000)
  
  // High loyalty makes pet follow player
  if (loyaltyLevel.value >= 5 && !followingPlayer.value) {
    followingPlayer.value = true
    petState.value = 'following'
  }
  
  resetIdleTimer()
}

function showDeathEffect() {
  showHeartEffect.value = true
  heartX.value = petX.value + 20
  heartY.value = petY.value + 80
  setTimeout(() => showHeartEffect.value = false, 2000)
}

function showReviveEffect() {
  showHeartEffect.value = true  
  heartX.value = petX.value + 20
  heartY.value = petY.value + 80
  setTimeout(() => showHeartEffect.value = false, 2000)
}

function showFeedbackEffect(text: string, type: string) {
  // Removed floating text - feedback now only via status bar
}

function investigateMouse() {
  if (petState.value !== 'idle') return
  
  const minY = groundHeight + toolbarHeight + 20
  const maxY = window.innerHeight - (window.innerHeight * 0.4) - 20
  
  // Only investigate if mouse is in grass area
  const mouseYConverted = window.innerHeight - mouseY.value - 32
  if (mouseYConverted < minY || mouseYConverted > maxY) return
  
  targetX.value = Math.max(32, Math.min(window.innerWidth - 64, mouseX.value - 32))
  targetY.value = Math.max(minY, Math.min(maxY, mouseYConverted))
  
  petState.value = 'investigating'
  investigationCooldown.value = 300 // 30 seconds cooldown
  
  if (targetX.value > petX.value) {
    direction.value = 'right'
  } else {
    direction.value = 'left'
  }
  
  resetIdleTimer()
}

function updateFollowing() {
  if (!followingPlayer.value || petState.value !== 'following') return
  
  const minY = groundHeight + toolbarHeight + 20
  const maxY = window.innerHeight - (window.innerHeight * 0.4) - 20
  const mouseYConverted = window.innerHeight - mouseY.value - 32
  
  // Only follow if mouse is in grass area
  if (mouseYConverted < minY || mouseYConverted > maxY) return
  
  const distanceToMouse = Math.sqrt(
    Math.pow(mouseX.value - petX.value - 32, 2) + 
    Math.pow(mouseYConverted - petY.value, 2)
  )
  
  // Follow if mouse is far enough away
  if (distanceToMouse > 80) {
    targetX.value = Math.max(32, Math.min(window.innerWidth - 64, mouseX.value - 32))
    targetY.value = Math.max(minY, Math.min(maxY, mouseYConverted))
    
    if (targetX.value > petX.value) {
      direction.value = 'right'
    } else {
      direction.value = 'left'
    }
  }
}

function updatePetMovement() {
  // Handle following behavior
  if (petState.value === 'following') {
    updateFollowing()
  }
  
  if (targetX.value !== null && targetY.value !== null && (petState.value === 'walking' || petState.value === 'investigating' || petState.value === 'following')) {
    const distanceX = Math.abs(targetX.value - petX.value)
    const distanceY = Math.abs(targetY.value - petY.value)
    const totalDistance = Math.sqrt(distanceX * distanceX + distanceY * distanceY)
    
    if (totalDistance > 5) {
      const ratio = moveSpeed / totalDistance
      const moveX = (targetX.value - petX.value) * ratio
      const moveY = (targetY.value - petY.value) * ratio
      
      petX.value += moveX
      petY.value += moveY
      
      if (moveX > 0.5) {
        direction.value = 'right'
      } else if (moveX < -0.5) {
        direction.value = 'left'
      }
    } else {
      targetX.value = null
      targetY.value = null
      if (petState.value === 'investigating') {
        petState.value = 'idle'
        startRandomWander()
      } else if (petState.value === 'following') {
        // Stay in following mode
      } else if (petState.value === 'walking') {
        // Only reward patience for user-commanded walks, not passive wandering
        if (commandCount.value > 0) {
          const previousLoyalty = loyaltyLevel.value
          loyaltyLevel.value = Math.min(5, loyaltyLevel.value + 1) // More direct loyalty gain
          
          // Check if we just reached max loyalty for the first time
          if (loyaltyLevel.value === 5 && !maxLoyaltyReached.value) {
            maxLoyaltyReached.value = true
            triggerGrandCelebration()
          }
          
          commandCount.value = 0
        }
        petState.value = 'idle'
        startRandomWander()
      } else {
        petState.value = 'idle'
        startRandomWander()
      }
    }
  }
  
  const minY = groundHeight + toolbarHeight + 20 // Bottom boundary (above ground)
  const maxY = window.innerHeight - (window.innerHeight * 0.4) - 20 // Top boundary (below sky)
  
  if (petX.value < 32) petX.value = 32
  if (petX.value > window.innerWidth - 64) petX.value = window.innerWidth - 64
  if (petY.value < minY) petY.value = minY
  if (petY.value > maxY) petY.value = maxY
}

function startRandomWander() {
  resetIdleTimer()
  idleTimer = setTimeout(() => {
    if (petState.value === 'idle' && !followingPlayer.value) {
      const minY = groundHeight + toolbarHeight + 20 // Bottom boundary (above ground)
      const maxY = window.innerHeight - (window.innerHeight * 0.4) - 20 // Top boundary (below sky)
      
      const randomX = Math.random() * (window.innerWidth - 96) + 32
      const randomY = Math.random() * (maxY - minY) + minY
      
      targetX.value = randomX
      targetY.value = randomY
      petState.value = 'walking'
      
      if (randomX > petX.value) {
        direction.value = 'right'
      } else {
        direction.value = 'left'
      }
    }
  }, 3000 + Math.random() * 2000)
}

function resetIdleTimer() {
  if (idleTimer) {
    clearTimeout(idleTimer)
    idleTimer = null
  }
}

function updateClouds() {
  cloud1X.value -= 0.5
  cloud2X.value -= 0.3
  
  if (cloud1X.value < -100) cloud1X.value = window.innerWidth + 50
  if (cloud2X.value < -100) cloud2X.value = window.innerWidth + 50
}

function gameLoop() {
  updatePetMovement()
  updateClouds()
  updateCooldowns()
  animationFrame = requestAnimationFrame(gameLoop)
}

function updateCooldowns() {
  if (rebellionCooldown.value > 0) rebellionCooldown.value--
  if (investigationCooldown.value > 0) investigationCooldown.value--
}

function triggerGrandCelebration() {
  celebrationActive.value = true
  
  // Multi-stage confetti celebration
  const duration = 5000
  const animationEnd = Date.now() + duration
  const defaults = { startVelocity: 30, spread: 360, ticks: 60, zIndex: 2000 }

  function randomInRange(min: number, max: number) {
    return Math.random() * (max - min) + min
  }

  // Stage 1: Massive burst from pet location
  const petScreenX = petX.value + 32
  const petScreenY = window.innerHeight - petY.value - 32
  
  confetti({
    ...defaults,
    particleCount: 100,
    origin: { 
      x: petScreenX / window.innerWidth, 
      y: petScreenY / window.innerHeight 
    },
    colors: ['#FFD700', '#FF6B9D', '#4CAF50', '#2196F3', '#FF9800']
  })

  // Stage 2: Rainbow bursts from corners
  setTimeout(() => {
    confetti({
      ...defaults,
      particleCount: 50,
      origin: { x: 0, y: 0.6 },
      colors: ['#FF1744', '#FF6D00', '#FFD600']
    })
    confetti({
      ...defaults,
      particleCount: 50,
      origin: { x: 1, y: 0.6 },
      colors: ['#00C853', '#00BCD4', '#3F51B5']
    })
  }, 500)

  // Stage 3: Continuous celebration
  const interval = setInterval(() => {
    const timeLeft = animationEnd - Date.now()

    if (timeLeft <= 0) {
      clearInterval(interval)
      celebrationActive.value = false
      return
    }

    const particleCount = 30 * (timeLeft / duration)
    
    // Random bursts across the screen
    confetti({
      ...defaults,
      particleCount,
      origin: { 
        x: randomInRange(0.1, 0.9), 
        y: randomInRange(0.2, 0.8) 
      },
      colors: ['#FFD700', '#FF69B4', '#00FF7F', '#1E90FF', '#FFA500', '#FF1493']
    })
  }, 250)

  // Stage 4: Grand finale
  setTimeout(() => {
    for (let i = 0; i < 3; i++) {
      setTimeout(() => {
        confetti({
          ...defaults,
          particleCount: 150,
          startVelocity: 45,
          origin: { x: 0.5, y: 0.8 },
          colors: ['#FFD700', '#FFA500', '#FF6B9D', '#4CAF50', '#2196F3', '#9C27B0']
        })
      }, i * 200)
    }
  }, 4000)
  
  // Pet celebration behaviors
  petScale.value = 1.5
  petState.value = 'playing'
  isJumping.value = true
  
  setTimeout(() => {
    petScale.value = 1
    isJumping.value = false
  }, 3000)
}

onMounted(() => {
  const minY = groundHeight + toolbarHeight + 20 // Bottom boundary (above ground)
  const maxY = window.innerHeight - (window.innerHeight * 0.4) - 20 // Top boundary (below sky)
  
  petX.value = window.innerWidth / 2 - 32
  petY.value = minY + (maxY - minY) / 2 // Center of grass area
  gameLoop()
  startRandomWander()
})

onUnmounted(() => {
  if (animationFrame) cancelAnimationFrame(animationFrame)
  if (idleTimer) clearTimeout(idleTimer)
  if (cloudTimer) clearInterval(cloudTimer)
})
</script>

<style scoped>
.pixel-world {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: linear-gradient(180deg, #87CEEB 0%, #98D8E8 40%, #32CD32 40%, #228B22 100%);
  cursor: crosshair;
  overflow: hidden;
  image-rendering: pixelated;
  image-rendering: -moz-crisp-edges;
  image-rendering: crisp-edges;
}

.background {
  position: absolute;
  width: 100%;
  height: 100%;
}

.clouds {
  position: absolute;
  top: 10%;
  width: 100%;
  height: 30%;
}

.cloud {
  position: absolute;
  width: 80px;
  height: 40px;
  background: white;
  border-radius: 40px;
  opacity: 0.8;
}

.cloud:before {
  content: '';
  position: absolute;
  top: -20px;
  left: 20px;
  width: 40px;
  height: 40px;
  background: white;
  border-radius: 50%;
}

.cloud:after {
  content: '';
  position: absolute;
  top: -10px;
  right: 15px;
  width: 25px;
  height: 25px;
  background: white;
  border-radius: 50%;
}

.ground {
  position: absolute;
  bottom: 0;
  width: 100%;
  height: 120px;
  background: linear-gradient(180deg, #32CD32 0%, #228B22 100%);
  border-top: 4px solid #2E8B57;
}

.pixel-pet {
  position: absolute;
  width: 64px;
  height: 64px;
  transition: transform 0.1s ease;
  cursor: pointer;
  z-index: 100;
}

.pixel-pet.jumping {
  animation: jump 1s ease-out;
}

.pixel-pet.walking .pet-legs {
  animation: walk 0.6s ease-in-out infinite;
}

.pixel-pet.walking.right .pet-sprite {
  transform: scaleX(1);
}

.pixel-pet.walking.left .pet-sprite {
  transform: scaleX(-1);
}

.pixel-pet.right .pet-sprite {
  transform: scaleX(1);
}

.pixel-pet.left .pet-sprite {
  transform: scaleX(-1);
}

.pixel-pet.playing {
  animation: spin 2s ease-in-out;
}

.pixel-pet.dead {
  transform: rotate(90deg) !important;
  opacity: 0.7;
  filter: grayscale(0.8);
}

.pixel-pet.investigating .pet-head {
  animation: curious 1s ease-in-out infinite;
}

.pixel-pet.following .pet-sprite {
  animation: excited 0.8s ease-in-out infinite;
}

.pixel-pet.celebrating {
  animation: grand-celebration 3s ease-in-out;
  filter: drop-shadow(0 0 20px #FFD700);
}

.pet-sprite {
  position: relative;
  width: 100%;
  height: 100%;
  transform-origin: center;
}

.pet-body {
  position: absolute;
  bottom: 8px;
  left: 50%;
  transform: translateX(-50%);
  width: 32px;
  height: 24px;
  background: #8B4513;
  border: 2px solid #654321;
  border-radius: 8px;
}

.pet-head {
  position: absolute;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  width: 28px;
  height: 28px;
  background: #DEB887;
  border: 2px solid #8B7355;
  border-radius: 14px;
}

.eyes {
  position: absolute;
  top: 8px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 6px;
}

.eye {
  width: 4px;
  height: 4px;
  background: #000;
  border-radius: 50%;
}

.pet-legs {
  position: absolute;
  bottom: 0;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 8px;
  width: 20px;
  justify-content: center;
}

.leg {
  width: 4px;
  height: 10px;
  background: #8B4513;
  border: 1px solid #654321;
  border-radius: 0 0 2px 2px;
}

.heart-effect {
  position: absolute;
  font-size: 24px;
  z-index: 200;
  animation: heart-float 1s ease-out forwards;
  pointer-events: none;
}

.target-marker {
  position: absolute;
  font-size: 16px;
  z-index: 50;
  animation: sparkle 1s ease-in-out infinite;
  pointer-events: none;
}

@keyframes jump {
  0% { transform: translateY(0) scale(1); }
  25% { transform: translateY(-40px) scale(1.1); }
  50% { transform: translateY(-60px) scale(1.2); }
  75% { transform: translateY(-40px) scale(1.1); }
  100% { transform: translateY(0) scale(1); }
}

@keyframes walk {
  0%, 100% { transform: translateY(0); }
  25% { transform: translateY(-2px); }
  50% { transform: translateY(0); }
  75% { transform: translateY(-2px); }
}

@keyframes spin {
  0% { transform: rotate(0deg) scale(1); }
  50% { transform: rotate(180deg) scale(1.2); }
  100% { transform: rotate(360deg) scale(1); }
}

@keyframes heart-float {
  0% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
  100% {
    opacity: 0;
    transform: translateY(-50px) scale(1.5);
  }
}

@keyframes sparkle {
  0%, 100% { 
    opacity: 0.5; 
    transform: scale(1); 
  }
  50% { 
    opacity: 1; 
    transform: scale(1.2); 
  }
}

@keyframes curious {
  0%, 100% { transform: rotate(-2deg); }
  50% { transform: rotate(2deg); }
}

@keyframes excited {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.05); }
}

@keyframes grand-celebration {
  0%, 100% { 
    transform: scale(1) rotate(0deg); 
    filter: drop-shadow(0 0 20px #FFD700);
  }
  25% { 
    transform: scale(1.3) rotate(-10deg); 
    filter: drop-shadow(0 0 30px #FF6B9D);
  }
  50% { 
    transform: scale(1.5) rotate(0deg); 
    filter: drop-shadow(0 0 40px #4CAF50);
  }
  75% { 
    transform: scale(1.3) rotate(10deg); 
    filter: drop-shadow(0 0 30px #2196F3);
  }
}

.pixel-pet:hover {
  filter: brightness(1.1);
}

.pixel-world:active {
  cursor: crosshair;
}

/* Status Bar */
.status-bar {
  position: fixed;
  top: 20px;
  left: 20px;
  background: rgba(0, 0, 0, 0.8);
  color: white;
  padding: 15px;
  border-radius: 10px;
  font-family: 'Courier New', monospace;
  font-size: 12px;
  display: flex;
  gap: 20px;
  z-index: 1000;
  backdrop-filter: blur(5px);
  transition: all 0.3s ease;
}

.status-bar.celebrating {
  background: linear-gradient(45deg, #FFD700, #FF6B9D, #4CAF50, #2196F3);
  background-size: 400% 400%;
  animation: rainbow-flow 2s ease-in-out infinite;
  transform: scale(1.05);
  box-shadow: 0 0 30px rgba(255, 215, 0, 0.6);
}

.status-section {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 5px;
}

.status-label {
  font-weight: bold;
  font-size: 10px;
  opacity: 0.8;
}

.loyalty-hearts {
  display: flex;
  gap: 2px;
}

.heart {
  font-size: 14px;
  filter: grayscale(1);
  opacity: 0.3;
  transition: all 0.3s ease;
}

.heart.filled {
  filter: grayscale(0);
  opacity: 1;
  animation: heartbeat 2s ease-in-out infinite;
}

.loyalty-hearts.max-loyalty .heart.filled {
  animation: mega-heartbeat 1s ease-in-out infinite;
  filter: drop-shadow(0 0 10px #FFD700);
}

.achievement-badge {
  background: linear-gradient(45deg, #FFD700, #FFA500);
  color: #333;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 8px;
  font-weight: bold;
  margin-top: 5px;
  text-align: center;
  animation: badge-glow 2s ease-in-out infinite;
  box-shadow: 0 0 15px rgba(255, 215, 0, 0.8);
}

.patience-bar {
  width: 60px;
  height: 8px;
  background: #444;
  border-radius: 4px;
  overflow: hidden;
}

.patience-fill {
  height: 100%;
  background: linear-gradient(90deg, #4CAF50, #8BC34A);
  transition: all 0.3s ease;
  border-radius: 4px;
}

.patience-fill.danger {
  background: linear-gradient(90deg, #FF4444, #FF6B6B);
  animation: pulse-danger 0.5s ease-in-out infinite;
}

.patience-text {
  font-size: 10px;
  text-align: center;
  margin-top: 2px;
}

.pet-status {
  background: #333;
  padding: 4px 8px;
  border-radius: 12px;
  font-size: 10px;
  transition: all 0.3s ease;
}

.pet-status.dead {
  background: #FF4444;
  animation: blink 1s ease-in-out infinite;
}

.pet-status.following {
  background: #4CAF50;
}

.pet-status.investigating {
  background: #2196F3;
}

.pet-status.playing {
  background: #FF9800;
}

/* Feedback Effects */
.feedback-effect {
  position: absolute;
  font-size: 14px;
  font-weight: bold;
  z-index: 200;
  animation: feedback-float 2s ease-out forwards;
  pointer-events: none;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.8);
}

.feedback-effect.warning {
  color: #FFA500;
}

.feedback-effect.reward {
  color: #4CAF50;
}

.feedback-effect.death {
  color: #FF4444;
  font-size: 18px;
}

@keyframes heartbeat {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.1); }
}

@keyframes pulse-danger {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.7; }
}

@keyframes blink {
  0%, 50%, 100% { opacity: 1; }
  25%, 75% { opacity: 0.5; }
}

@keyframes feedback-float {
  0% {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
  100% {
    opacity: 0;
    transform: translateY(-60px) scale(1.2);
  }
}

@keyframes rainbow-flow {
  0% { background-position: 0% 50%; }
  50% { background-position: 100% 50%; }
  100% { background-position: 0% 50%; }
}

@keyframes mega-heartbeat {
  0%, 100% { 
    transform: scale(1); 
    filter: drop-shadow(0 0 10px #FFD700);
  }
  50% { 
    transform: scale(1.3); 
    filter: drop-shadow(0 0 20px #FFD700) drop-shadow(0 0 30px #FF6B9D);
  }
}

@keyframes badge-glow {
  0%, 100% { 
    box-shadow: 0 0 15px rgba(255, 215, 0, 0.8);
    transform: scale(1);
  }
  50% { 
    box-shadow: 0 0 25px rgba(255, 215, 0, 1), 0 0 35px rgba(255, 107, 157, 0.6);
    transform: scale(1.05);
  }
}
</style>