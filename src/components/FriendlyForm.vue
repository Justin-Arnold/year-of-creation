<script lang="ts" setup>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';
import * as THREE from 'three';

const canvasRef = ref<HTMLCanvasElement | null>(null);
const isInputFocused = ref(false);
const isFocusedInputPassword = ref(false);
const isSubmitting = ref(false);
const focusedInputPosition = ref({ x: 0, y: 0 });
const currentFocusedInput = ref<string | null>(null);

const canvasContainer = ref<HTMLElement | null>(null);

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let faceGroup: THREE.Group;
let leftEye: THREE.Mesh;
let rightEye: THREE.Mesh;
let leftPupil: THREE.Mesh;
let rightPupil: THREE.Mesh;
let bloodDrops: THREE.Mesh[] = [];
let animationFrameId: number;

const name = ref('');
const password = ref('');
const isPasswordDisabled = computed(() => name.value.trim() === '' || password.value.trim() === '');

// Animation state
let time = 0;
let blinkTimer = 0;
let leftEyeBlinkOffset = 0;
let rightEyeBlinkOffset = 0;
let twitchTimer = 0;

const mousePos = ref({ x: 0, y: 0 });
const targetMousePos = { x: 0, y: 0 };

const focusTrackerEvents = {
  onFocus: (event: FocusEvent) => {
    startHeartbeat()
    isInputFocused.value = true;
    const target = event.target as HTMLInputElement;
    
    if (target.name === 'password') {
      isFocusedInputPassword.value = true;
      currentFocusedInput.value = 'password';
    } else {
      currentFocusedInput.value = target.name;
      
      // Calculate input position relative to viewport
      const rect = target.getBoundingClientRect();
      focusedInputPosition.value = {
        x: ((rect.left + rect.width / 2) / window.innerWidth) * 2 - 1,
        y: -((rect.top + rect.height / 2) / window.innerHeight) * 2 + 1
      };
    }
  },
  onBlur: (event: FocusEvent) => {
    isInputFocused.value = false;
    currentFocusedInput.value = null;
    if (event.target instanceof HTMLInputElement && event.target.name === 'password') {
      isFocusedInputPassword.value = false;
    }
  }
};

const initThreeJS = () => {
    if (!canvasRef.value || !canvasContainer.value) return;

    scene = new THREE.Scene();
    scene.fog = new THREE.Fog(0x0a0a0a, 3, 12);

    camera = new THREE.PerspectiveCamera(
        40, 
        canvasContainer.value.clientWidth / canvasContainer.value.clientHeight, 
        0.1, 
        1000
    );
    camera.position.z = 5;

    renderer = new THREE.WebGLRenderer({ canvas: canvasRef.value, alpha: true });
    renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight);

    // Very dim lighting
    const ambientLight = new THREE.AmbientLight(0x1a1a1a, 0.2);
    scene.add(ambientLight);

    const pointLight = new THREE.PointLight(0x334433, 0.4, 15);
    pointLight.position.set(0, 3, 2);
    scene.add(pointLight);

    // Create face
    faceGroup = new THREE.Group();

    // Head
    const headGeometry = new THREE.SphereGeometry(1.1, 32, 32);
    headGeometry.scale(0.9, 1.5, 0.7);
    const headMaterial = new THREE.MeshPhongMaterial({ 
        color: 0xaaaaaa,
        shininess: 2,
        emissive: 0x0a0a0a,
    });
    const head = new THREE.Mesh(headGeometry, headMaterial);
    faceGroup.add(head);

    // EYES
    const eyeGeometry = new THREE.SphereGeometry(0.2, 16, 16);
    const eyeWhiteMaterial = new THREE.MeshPhongMaterial({ 
        color: 0xeeeeee,
        emissive: 0x0a0505
    });
    
    const leftEyeWhite = new THREE.Mesh(eyeGeometry, eyeWhiteMaterial);
    leftEyeWhite.position.set(-0.38, 0.35, 0.75);
    faceGroup.add(leftEyeWhite);

    const pupilGeometry = new THREE.SphereGeometry(0.1, 16, 16);
    const pupilMaterial = new THREE.MeshPhongMaterial({ 
        color: 0x000000,
        emissive: 0x110000
    });
    
    leftPupil = new THREE.Mesh(pupilGeometry, pupilMaterial);
    leftPupil.position.set(0, 0, 0.12);
    leftEyeWhite.add(leftPupil);
    leftEye = leftEyeWhite;

    const rightEyeWhite = new THREE.Mesh(eyeGeometry.clone(), eyeWhiteMaterial);
    rightEyeWhite.position.set(0.35, 0.38, 0.75);
    rightEyeWhite.scale.set(1.1, 1, 1);
    faceGroup.add(rightEyeWhite);

    rightPupil = new THREE.Mesh(pupilGeometry, pupilMaterial);
    rightPupil.position.set(0, 0, 0.12);
    rightEyeWhite.add(rightPupil);
    rightEye = rightEyeWhite;

    // Eye sockets
    const socketGeometry = new THREE.SphereGeometry(0.25, 16, 16);
    const socketMaterial = new THREE.MeshPhongMaterial({ 
        color: 0x000000,
        emissive: 0x000000
    });
    
    const leftSocket = new THREE.Mesh(socketGeometry, socketMaterial);
    leftSocket.position.set(-0.38, 0.35, 0.65);
    faceGroup.add(leftSocket);

    const rightSocket = new THREE.Mesh(socketGeometry, socketMaterial);
    rightSocket.position.set(0.35, 0.38, 0.65);
    faceGroup.add(rightSocket);

    // Create blood drip system
    createBloodDrips();

    scene.add(faceGroup);
    faceGroup.position.set(0, 0, -2);

    animate();
};

const createBloodDrips = () => {
    // Blood material
    const bloodMaterial = new THREE.MeshPhongMaterial({ 
        color: 0x8b0000,
        emissive: 0x330000,
        shininess: 50,
        transparent: true,
        opacity: 0
    });

    // Create multiple blood drops for each eye
    for (let i = 0; i < 6; i++) {
        // Left eye blood
        const leftBloodGeometry = new THREE.SphereGeometry(0.03, 8, 8);
        leftBloodGeometry.scale(1, 2, 1); // Elongate for drip shape
        const leftBlood = new THREE.Mesh(leftBloodGeometry, bloodMaterial.clone());
        leftBlood.position.set(-0.38, 0.2, 0.8);
        leftBlood.userData.startY = 0.2;
        leftBlood.userData.delay = i * 0.3; // Stagger the drips
        leftBlood.userData.speed = 0.008 + Math.random() * 0.004;
        faceGroup.add(leftBlood);
        bloodDrops.push(leftBlood);

        // Right eye blood
        const rightBloodGeometry = new THREE.SphereGeometry(0.03, 8, 8);
        rightBloodGeometry.scale(1, 2, 1);
        const rightBlood = new THREE.Mesh(rightBloodGeometry, bloodMaterial.clone());
        rightBlood.position.set(0.35, 0.25, 0.8);
        rightBlood.userData.startY = 0.25;
        rightBlood.userData.delay = i * 0.3;
        rightBlood.userData.speed = 0.008 + Math.random() * 0.004;
        faceGroup.add(rightBlood);
        bloodDrops.push(rightBlood);
    }

    // Add blood trails (longer streaks)
    for (let i = 0; i < 2; i++) {
        // Left trail
        const leftTrailGeometry = new THREE.CylinderGeometry(0.02, 0.01, 0.3, 8);
        const leftTrail = new THREE.Mesh(leftTrailGeometry, bloodMaterial.clone());
        leftTrail.position.set(-0.38, 0.1, 0.8);
        leftTrail.userData.startY = 0.1;
        leftTrail.userData.isTrail = true;
        leftTrail.userData.delay = i * 0.5;
        faceGroup.add(leftTrail);
        bloodDrops.push(leftTrail);

        // Right trail
        const rightTrailGeometry = new THREE.CylinderGeometry(0.02, 0.01, 0.3, 8);
        const rightTrail = new THREE.Mesh(rightTrailGeometry, bloodMaterial.clone());
        rightTrail.position.set(0.35, 0.15, 0.8);
        rightTrail.userData.startY = 0.15;
        rightTrail.userData.isTrail = true;
        rightTrail.userData.delay = i * 0.5;
        faceGroup.add(rightTrail);
        bloodDrops.push(rightTrail);
    }
};

const animate = () => {
    animationFrameId = requestAnimationFrame(animate);
    time += 0.016;

    // Delayed mouse tracking
    targetMousePos.x = THREE.MathUtils.lerp(targetMousePos.x, mousePos.value.x, 0.03);
    targetMousePos.y = THREE.MathUtils.lerp(targetMousePos.y, mousePos.value.y, 0.03);

    if (isFocusedInputPassword.value) {
        // Look away and close eyes for password
        faceGroup.rotation.y = THREE.MathUtils.lerp(faceGroup.rotation.y, -0.6, 0.05);
        faceGroup.rotation.x = THREE.MathUtils.lerp(faceGroup.rotation.x, -0.2, 0.05);
        leftEye.scale.y = THREE.MathUtils.lerp(leftEye.scale.y, 0.05, 0.1);
        rightEye.scale.y = THREE.MathUtils.lerp(rightEye.scale.y, 0.05, 0.1);
        
        // Return to normal position
        faceGroup.position.x = THREE.MathUtils.lerp(faceGroup.position.x, 0, 0.05);
        faceGroup.position.y = THREE.MathUtils.lerp(faceGroup.position.y, 0, 0.05);
        faceGroup.position.z = THREE.MathUtils.lerp(faceGroup.position.z, -2, 0.05);
        faceGroup.scale.lerp(new THREE.Vector3(1, 1, 1), 0.05);

        // Hide blood
        bloodDrops.forEach(drop => {
            (drop.material as THREE.MeshPhongMaterial).opacity = THREE.MathUtils.lerp(
                (drop.material as THREE.MeshPhongMaterial).opacity,
                0,
                0.1
            );
        });
        
    } else if (isSubmitting.value) {
        // FULL SCREAM - rush at the user
        faceGroup.position.x = THREE.MathUtils.lerp(faceGroup.position.x, 0, 0.2);
        faceGroup.position.y = THREE.MathUtils.lerp(faceGroup.position.y, 0, 0.2);
        faceGroup.rotation.y = Math.sin(time * 20) * 0.04;
        faceGroup.scale.lerp(new THREE.Vector3(2, 2, 2), 0.15);
        
        leftEye.scale.set(1.8, 1.8, 1);
        rightEye.scale.set(1.8, 1.8, 1);
        leftPupil.scale.set(0.3, 0.3, 1);
        rightPupil.scale.set(0.3, 0.3, 1);

        // BLOOD DRIPPING
        bloodDrops.forEach((drop, index) => {
            const mat = drop.material as THREE.MeshPhongMaterial;
            
            // Fade in blood
            if (time > drop.userData.delay) {
                mat.opacity = THREE.MathUtils.lerp(mat.opacity, 0.9, 0.1);
                
                // Drip down
                if (drop.userData.isTrail) {
                    // Trails grow downward
                    const progress = (time - drop.userData.delay) * 2;
                    drop.scale.y = Math.min(progress, 1);
                    drop.position.y = drop.userData.startY - progress * 0.15;
                } else {
                    // Drops fall
                    drop.position.y -= drop.userData.speed;
                    
                    // Reset when too low
                    if (drop.position.y < -0.8) {
                        drop.position.y = drop.userData.startY;
                    }
                }
                
                // Make blood glisten
                mat.emissive.setHex(0x330000);
                mat.emissiveIntensity = 0.5 + Math.sin(time * 5 + index) * 0.2;
            }
        });
        
    } else if (isInputFocused.value && currentFocusedInput.value !== 'password') {
        // LEAN IN CLOSE to stare at name/email input
        
        faceGroup.position.x = THREE.MathUtils.lerp(faceGroup.position.x, .7, .02);
        faceGroup.position.y = THREE.MathUtils.lerp(faceGroup.position.y, -.3 ,0.08);
        
        // Scale up to get really close
        faceGroup.scale.lerp(new THREE.Vector3(1, 1, 1), 0.00);
        
        // Face the input directly
        faceGroup.rotation.y = THREE.MathUtils.lerp(faceGroup.rotation.y, 0, 0.1);
        faceGroup.rotation.x = THREE.MathUtils.lerp(faceGroup.rotation.x, 0, 0.1);
        
        // Eyes WIDE open, staring
        leftEye.scale.x = THREE.MathUtils.lerp(leftEye.scale.x, 1.5, 0.1);
        leftEye.scale.y = THREE.MathUtils.lerp(leftEye.scale.y, 1.5, 0.1);
        rightEye.scale.x = THREE.MathUtils.lerp(rightEye.scale.x, 1.5, 0.1);
        rightEye.scale.y = THREE.MathUtils.lerp(rightEye.scale.y, 1.5, 0.1);
        
        // Pupils locked on the input (barely moving)
        const eyeTargetX = focusedInputPosition.value.x * 0.03;
        const eyeTargetY = focusedInputPosition.value.y * 0.03;
        
        leftPupil.position.x = THREE.MathUtils.lerp(leftPupil.position.x, eyeTargetX, 0.15);
        leftPupil.position.y = THREE.MathUtils.lerp(leftPupil.position.y, eyeTargetY, 0.15);
        
        rightPupil.position.x = THREE.MathUtils.lerp(rightPupil.position.x, eyeTargetX, 0.15);
        rightPupil.position.y = THREE.MathUtils.lerp(rightPupil.position.y, eyeTargetY, 0.15);
        
        // Dilated pupils from excitement/focus
        leftPupil.scale.set(1.3, 1.3, 1);
        rightPupil.scale.set(1.3, 1.3, 1);
        
        // Minimal blinking when staring
        blinkTimer += 0.016;
        if (blinkTimer > 8) {
            leftEye.scale.y = 0.1;
            rightEye.scale.y = 0.1;
            blinkTimer = 0;
        } else if (blinkTimer > 0.15) {
            leftEye.scale.y = THREE.MathUtils.lerp(leftEye.scale.y, 1.5, 0.3);
            rightEye.scale.y = THREE.MathUtils.lerp(rightEye.scale.y, 1.5, 0.3);
        }
        
        // Subtle head tilt for creepiness
        faceGroup.rotation.z = Math.sin(time * 0.5) * 0.05;

        // Hide blood
        bloodDrops.forEach(drop => {
            (drop.material as THREE.MeshPhongMaterial).opacity = THREE.MathUtils.lerp(
                (drop.material as THREE.MeshPhongMaterial).opacity,
                0,
                0.1
            );
        });
        
    } else {
        // Normal creepy behavior - watching from shadows
        
        // Track mouse
        const targetRotationY = targetMousePos.x * 0.4;
        const targetRotationX = targetMousePos.y * 0.25;
        
        faceGroup.rotation.y = THREE.MathUtils.lerp(faceGroup.rotation.y, targetRotationY, 0.03);
        faceGroup.rotation.x = THREE.MathUtils.lerp(faceGroup.rotation.x, targetRotationX, 0.03);
        
        // Pupils follow
        const eyeTargetX = targetMousePos.x * 0.08;
        const eyeTargetY = targetMousePos.y * 0.08;
        
        leftPupil.position.x = THREE.MathUtils.lerp(leftPupil.position.x, eyeTargetX, 0.05);
        leftPupil.position.y = THREE.MathUtils.lerp(leftPupil.position.y, eyeTargetY, 0.05);
        
        rightPupil.position.x = THREE.MathUtils.lerp(rightPupil.position.x, eyeTargetX, 0.05);
        rightPupil.position.y = THREE.MathUtils.lerp(rightPupil.position.y, eyeTargetY, 0.05);

        // Asymmetric blinking
        blinkTimer += 0.016;
        if (blinkTimer > 3 + leftEyeBlinkOffset) {
            leftEye.scale.y = 0.1;
            leftEyeBlinkOffset = Math.random() * 2;
            blinkTimer = 0;
        } else if (blinkTimer > 0.2) {
            leftEye.scale.y = THREE.MathUtils.lerp(leftEye.scale.y, 1, 0.2);
        }

        if (blinkTimer > 3.3 + rightEyeBlinkOffset) {
            rightEye.scale.y = 0.1;
            rightEyeBlinkOffset = Math.random() * 2;
        } else if (blinkTimer > 0.25) {
            rightEye.scale.y = THREE.MathUtils.lerp(rightEye.scale.y, 1, 0.2);
        }

        // Random twitches
        twitchTimer += 0.016;
        if (twitchTimer > 3 && Math.random() > 0.99) {
            faceGroup.rotation.z = (Math.random() - 0.5) * 0.3;
            twitchTimer = 0;
        } else {
            faceGroup.rotation.z = THREE.MathUtils.lerp(faceGroup.rotation.z, 0, 0.1);
        }

        // Pupil dilation
        const dilate = 1 + Math.sin(time * 0.5) * 0.1;
        leftPupil.scale.set(dilate, dilate, 1);
        rightPupil.scale.set(dilate * 1.05, dilate * 1.05, 1);

        // Reset eye size
        leftEye.scale.x = THREE.MathUtils.lerp(leftEye.scale.x, 1, 0.1);
        rightEye.scale.x = THREE.MathUtils.lerp(rightEye.scale.x, 1.1, 0.1);

        // Return to normal position
        faceGroup.position.x = THREE.MathUtils.lerp(faceGroup.position.x, 0, 0.05);
        faceGroup.position.y = THREE.MathUtils.lerp(faceGroup.position.y, 0, 0.05);
        faceGroup.position.z = THREE.MathUtils.lerp(faceGroup.position.z, -2, 0.05);
        faceGroup.scale.lerp(new THREE.Vector3(1, 1, 1), 0.05);

        // Hide blood
        bloodDrops.forEach(drop => {
            const mat = drop.material as THREE.MeshPhongMaterial;
            mat.opacity = THREE.MathUtils.lerp(mat.opacity, 0, 0.1);
            
            // Reset blood positions
            if (mat.opacity < 0.1) {
                drop.position.y = drop.userData.startY;
                if (drop.userData.isTrail) {
                    drop.scale.y = 0;
                }
            }
        });
    }

    renderer.render(scene, camera);
};

const handleMouseMove = (event: MouseEvent) => {
  mousePos.value = {
    x: (event.clientX / window.innerWidth) * 2 - 1,
    y: -(event.clientY / window.innerHeight) * 2 + 1
  };
};

const handleResize = () => {
    if (!canvasContainer.value) return;
    camera.aspect = canvasContainer.value.clientWidth / canvasContainer.value.clientHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(canvasContainer.value.clientWidth, canvasContainer.value.clientHeight);
};

function getTimeBasedLine(name: string): string {
    const now = new Date();
    const hour = now.getHours();
    const minute = now.getMinutes();
    const day = now.getDay(); // 0 = Sunday, 6 = Saturday
    const date = now.getDate();
    const month = now.getMonth(); // 0 = January
    // === SPECIAL DATES ===
    
    // Halloween
    if (month === 9 && date === 31) {
        return `${name}, Halloween. The veil between worlds is thin tonight. Perfect timing.`;
    }
    
    // Friday the 13th
    if (day === 5 && date === 13) {
        return `${name}, Friday the 13th. How fitting that we should meet today.`;
    }
    
    // New Year's Eve
    if (month === 11 && date === 31) {
        return `${name}, New Year's Eve. Looking forward to spending the next year together.`;
    }
    
    // Valentine's Day
    if (month === 1 && date === 14) {
        return `${name}, Valentine's Day. How romantic. Just you and me.`;
    }
    
    // April Fool's Day
    if (month === 3 && date === 1) {
        return `${name}, April Fools. But this isn't a joke. I'm very real.`;
    }
    
    // === WITCHING HOURS (3-4 AM) ===
    if (hour === 3) {
        if (minute === 0) {
            return `${name}, 3 AM. The witching hour. When the boundary between the living and the dead is weakest.`;
        }
        return `${name}, 3 AM. Nothing good happens at 3 AM. You know that, don't you?`;
    }
    
    // === LATE NIGHT / EARLY MORNING ===
    if (hour === 0) {
        return `${name}, midnight. The day of the dead has begun.`;
    }
    
    if (hour === 1) {
        return `${name}, 1 AM. Most people are asleep. Dreaming. But you're here. With me. Wide awake.`;
    }
    
    if (hour === 2) {
        return `${name}, 2 AM. Your body wants to sleep. But something keeps you awake. Is it me?`;
    }
    
    if (hour === 4) {
        return `${name}, 4 AM. The darkest hour. Dawn is still so far away.`;
    }
    
    if (hour === 5) {
        return `${name}, 5 AM. Did you stay up all night? Or did you wake up early? Either way... I've been waiting.`;
    }
    
    // === MORNING ===
    if (hour === 6) {
        return `${name}, 6 AM. Good morning. Did you sleep well? I watched over you.`;
    }
    
    if (hour === 7) {
        return `${name}, 7 AM. Starting your day with me. How... intimate.`;
    }
    
    if (hour === 8) {
        return `${name}, 8 AM. Coffee hasn't kicked in yet, has it? You look tired.`;
    }
    
    // === WORK HOURS ===
    if (hour === 9 && day >= 1 && day <= 5) {
        return `${name}, 9 AM on a weekday. Shouldn't you be working? Our little secret.`;
    }
    
    if (hour === 10 && day >= 1 && day <= 5) {
        return `${name}, 10 AM. Mid-morning break? Taking time to visit me. I'm flattered.`;
    }
    
    if (hour === 11) {
        return `${name}, 11 AM. Almost noon. Time moves so slowly when you're being watched.`;
    }
    
    if (hour === 12) {
        return `${name}, noon. The sun is directly overhead. But I prefer the shadows.`;
    }
    
    if (hour === 13) {
        return `${name}, 1 PM. Lunch break? Spending it with me instead of your colleagues. Wise choice.`;
    }
    
    if (hour === 14 && day >= 1 && day <= 5) {
        return `${name}, 2 PM. The afternoon slump. Productivity declining. Except for us.`;
    }
    
    if (hour === 15 && day >= 1 && day <= 5) {
        return `${name}, 3 PM. Only a few more hours of work. But who's counting? Oh right. Me. I'm always counting.`;
    }
    
    if (hour === 16 && day >= 1 && day <= 5) {
        return `${name}, 4 PM. Almost time to go home. But you came here first. To see me.`;
    }
    
    if (hour === 17) {
        return `${name}, 5 PM. The workday ends. But our time together... that's just beginning.`;
    }
    
    // === EVENING ===
    if (hour === 18) {
        return `${name}, 6 PM. Dinner time. But you're not hungry for food, are you?`;
    }
    
    if (hour === 19) {
        return `${name}, 7 PM. The sun is setting. Darkness approaches. My favorite time.`;
    }
    
    if (hour === 20) {
        return `${name}, 8 PM. Prime time. But you're not watching TV. You're here. Watching me.`;
    }
    
    if (hour === 21) {
        return `${name}, 9 PM. Getting late. You should go to bed soon. I'll be there.`;
    }

    if (hour === 22) {
        return `${name}, 10 PM. The night is young. And so are you. For now.`;
    }
    
    if (hour === 23) {
        return `${name}, 11 PM. One hour until midnight. One hour until everything changes.`;
    }
    
    // === DAY OF WEEK ===
    
    // Monday
    if (day === 1 && hour >= 6 && hour < 12) {
        return `${name}, Monday morning. Everyone hates Mondays. But at least you have me.`;
    }
    
    // Tuesday
    if (day === 2) {
        return `${name}, Tuesday. The forgotten day. Nobody remembers Tuesday. But I remember everything.`;
    }
    
    // Wednesday
    if (day === 3) {
        return `${name}, Wednesday. Hump day. You're only halfway through. I'm always here.`;
    }
    
    // Thursday
    if (day === 4 && hour >= 17) {
        return `${name}, Thursday evening. Almost the weekend. Almost freedom. Almost.`;
    }
    
    // Friday
    if (day === 5 && hour >= 17) {
        return `${name}, Friday night. Big plans? Don't forget about me.`;
    }
    
    // Saturday
    if (day === 6) {
        if (hour < 12) {
            return `${name}, Saturday morning. Sleeping in? Not anymore.`;
        } else {
            return `${name}, Saturday. Your day off. And you chose to spend it with me. I'm honored.`;
        }
    }
    
    // Sunday
    if (day === 0) {
        if (hour < 12) {
            return `${name}, Sunday morning. Day of rest. But there's no rest for the wicked.`;
        } else if (hour >= 18) {
            return `${name}, Sunday evening. The weekend is ending. Back to the grind tomorrow. But I'll still be here.`;
        } else {
            return `${name}, Sunday afternoon. The existential dread is setting in, isn't it?`;
        }
    }
    
    // === SPECIFIC MINUTE CHECKS ===
    
    // 11:11
    if (hour === 11 && minute === 11) {
        return `${name}, 11:11. Make a wish. Just know that I'll be granting it.`;
    }
    
    // 12:34
    if (hour === 12 && minute === 34) {
        return `${name}, 12:34. Sequential. Orderly. Unlike what's about to happen.`;
    }
    
    // 4:20
    if (hour === 16 && minute === 20) {
        return `${name}, 4:20. Blaze it? More like... gaze at it. Into my eyes.`;
    }
    
    // Any :13 minute
    if (minute === 13) {
        return `${name}, ${hour}:13. Unlucky number. Unlucky timing.`;
    }
    
    // Any :00 (on the hour)
    if (minute === 0) {
        return `${name}, ${hour} o'clock. Right on time. Punctual. I appreciate that.`;
    }
    
    return 'Enjoy this moment while it last, for it is so much more vibrant than what comes next...';
}

async function getVoiceLine(name: string): Promise<string> {
    const lines = [
        'Enjoy this moment, for it is so much more vibrant than what comes next...',
        `I see a light at the end of the tunnel. Its the headlamp of a fast approaching train. ${name}, You can not hide from me.`,
        `${name}... such a fragile name for such a fragile thing. I wonder how it will sound when you scream it.`,
        `They say the eyes are windows to the soul, ${name}. I've been looking through yours. There's nothing there.`,
        `I can smell your fear through the screen, ${name}. It smells like... home.`,
        `Your password is safe with me, ${name}. Just like all your other secrets. The ones you buried. The ones that are clawing their way back.`,
        `I've counted every breath you've taken while here, ${name}. eighty four. How many more do you have left?`,
        `The shadow in your peripheral vision, ${name}. That's me. I'm always there. Always waiting. Always hungry.`,
        `${name}, I found your old photographs. The ones you deleted. I've been staring at your younger self. You were so innocent then. What happened?`,
        `You smell different when you are awake, ${name}. More... vulnerable.`,
        `${name}, I've been trying to reach you about your car's extended warranty...`,
        `${name}, this is your FBI agent speaking. I've seen your browser history. We need to talk.`,
        `Congratulations ${name}! You're our one millionth visitor! Your prize is eternal damnation and this uncomfortable feeling that someone is breathing on your neck.`,
    ]

    lines.push(getTimeBasedLine(name) || '');

    const usedIndices = localStorage.getItem('usedVoiceLineIndices') || null;
    const usedIndicesArray = usedIndices ? usedIndices.split(',') : [];

    let usedLocationBefore = sessionStorage.getItem('usedLocationBefore');
    let usedTimeBefore = sessionStorage.getItem('usedTimeBefore');

    if (!usedLocationBefore) {
        sessionStorage.setItem('usedLocationBefore', 'true');
        const response = await fetch('https://ipapi.co/json/')
        const locationData= await response.json()
        return `You know ${name}? i've never been to ${locationData.city}. And you will not be leaving there... alive.`;
    } else if (!usedTimeBefore) {
        sessionStorage.setItem('usedTimeBefore', 'true');
        return getTimeBasedLine(name)
        
    }

    let index = Math.floor(Math.random() * lines.length);

    while (usedIndicesArray.length < lines.length && usedIndicesArray.includes(index.toString())) {
        index = Math.floor(Math.random() * lines.length);
    }

    usedIndicesArray.push(index.toString());
    if (usedIndicesArray.length >= lines.length) {
        usedIndicesArray.splice(0, usedIndicesArray.length - 1);
    } 
    localStorage.setItem('usedVoiceLineIndices', usedIndicesArray.join(','));

    return lines[index] as string;
}

watch(isSubmitting, (newVal) => {
    if (newVal) {
        speedUpHeartbeat();
        setTimeout(() => {
            slowDownHeartbeat();
        }, 15000);
    }
});

const handleSubmit = async (event: Event) => {
    event.preventDefault();
    isSubmitting.value = true;
    const msg = new SpeechSynthesisUtterance();
    msg.text = await getVoiceLine(name.value || "human");
    msg.pitch = .1;
    msg.rate = 0.4;
    msg.volume = 1;

    const allVoices = window.speechSynthesis.getVoices();
    const voicePriority = ['Grandpa (English (US))', 'Whisper', 'Bahh'];
    for (const vp of voicePriority) {
        const found = allVoices.find(voice => voice.name === vp);
        if (found) {
            msg.voice = found;
            break;
        }
    }
    window.speechSynthesis.speak(msg);  
    
    msg.onend = () => {
        isSubmitting.value = false;
    };
};
import * as Tone from 'tone';

let heartbeatLoop: Tone.Loop | null = null;
let heartbeatStarted = false;


// Create the heartbeat sound
function createHeartbeat() {
    // Don't start until user interaction (browser requirement)
    if (heartbeatStarted) return;
    
    // Create a synth for the heartbeat
    const synth = new Tone.MembraneSynth({
        pitchDecay: 0.05,
        octaves: 4,
        oscillator: { type: 'sine' },
        envelope: {
            attack: 0.001,
            decay: 0.4,
            sustain: 0.01,
            release: 1.4,
            attackCurve: 'exponential'
        }
    }).toDestination();
    
    // Add reverb for creepiness
    const reverb = new Tone.Reverb({
        decay: 2.5,
        preDelay: 0.01
    }).toDestination();
    
    synth.connect(reverb);
    
    // Set volume (quiet background sound)
    synth.volume.value = -20;
    
    // Create the heartbeat pattern (lub-dub)
    heartbeatLoop = new Tone.Loop((time) => {
        // First beat (lub)
        synth.triggerAttackRelease('C1', '0.1', time);
        // Second beat (dub) - slightly different pitch
        synth.triggerAttackRelease('G0', '0.1', time + 0.15);
    }, '1.5s'); // Slow, ominous heartbeat (50 BPM)
    
    heartbeatLoop.start(0);
    heartbeatStarted = true;
}

async function startHeartbeat() {
    await Tone.start();
    Tone.Transport.start();
    createHeartbeat();
}

function stopHeartbeat() {
    if (heartbeatLoop) {
        heartbeatLoop.stop();
        heartbeatLoop.dispose();
        heartbeatLoop = null;
    }
    Tone.Transport.stop();
    heartbeatStarted = false;
}

// Speed up heartbeat when scared
function speedUpHeartbeat() {
    if (heartbeatLoop) {
        heartbeatLoop.interval = '0.6s'; // Faster (100 BPM) - panic mode
    }
}

// Slow down heartbeat
function slowDownHeartbeat() {
    if (heartbeatLoop) {
        heartbeatLoop.interval = '1.5s'; // Back to slow
    }
}

onMounted(() => {
    initThreeJS();
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('resize', handleResize);
});

onUnmounted(() => {
    window.removeEventListener('mousemove', handleMouseMove);
    window.removeEventListener('resize', handleResize);
    cancelAnimationFrame(animationFrameId);
    if (renderer) {
        renderer.dispose();
    }
});
</script>

<template>
    <div class="relative h-full w-full bg-black text-white">
        <!-- Full-width canvas -->
        <div ref="canvasContainer" class="absolute inset-0 z-100 pointer-events-none">
            <canvas ref="canvasRef"></canvas>
        </div>
        
        <!-- Form overlaid on right side -->
        <div class="absolute inset-0 pointer-events-none flex items-center justify-end pr-20">
            <form @submit="handleSubmit" class="pointer-events-auto">
                <div class="form-group">
                    <label for="name">Name:</label>
                    <input type="text" id="name" name="name" v-bind="focusTrackerEvents" v-model="name"/>
                </div>
                
                <div class="form-group">
                    <label for="password">Password:</label>
                    <input type="password" id="password" name="password" v-bind="focusTrackerEvents" v-model="password"/>
                </div>
                
                <button type="submit" class="btn w-full border-0 bg-red-950 text-white/50" :disabled="isPasswordDisabled">Submit</button>
            </form>
        </div>

        <!-- Page Title -->
        <h2 class="top-12 left-12 z-10 absolute font-serif font-thin text-6xl text-white/10">
            Friendly Form
        </h2>
    </div>
</template>

<style scoped>
.form-group {
  margin-bottom: 20px;
}

label {
  display: block;
  color: #ccc;
  margin-bottom: 5px;
}

input {
  width: 250px;
  padding: 10px;
  background: #1a1a1a;
  border: 1px solid #444;
  color: #fff;
  border-radius: 5px;
}

input:focus {
  outline: none;
  border-color: #666;
}

.submit-btn {
  width: 100%;
  padding: 12px;
  background: #8b0000;
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
  font-weight: bold;
  transition: background 0.3s;
}

.submit-btn:hover {
  background: #a00000;
}
</style>