<script setup lang="ts">
import { ref, onMounted } from 'vue'
import { Icon } from '@iconify/vue';

const MAX_PARTICLES = 250;
const FORCE_SPACING = 20

class Vector2D {
    x: number;
    y: number;
    constructor(x?: number, y?: number) {
        this.x = x || 0;
        this.y = y || 0;
    }
    /**
     *  @param vector
     */
    add(vector: {x: number, y: number}) {
        this.x += vector.x;
        this.y += vector.y;
    }
    /**
     *  Resets the vector to a new position
     *  @param x
     *  @param y
     */
    reset(x:number, y:number) {
        this.x = x;
        this.y = y;
    }
    /**
     *  uses linear interpolation to move the vector towards another vector
     *  @param vector - target vector
     *  @param n - interpolation factor, between 0 and 1. A lower value will cause a smaller movement in the vector.
     */
    interpolate(vector: {x: number, y: number}, n:number) {
        this.x += (vector.x - this.x)*n;
        this.y += (vector.y - this.y)*n;
    }
}

class Particle {
    position: Vector2D;
    velocity: Vector2D;
    acceleration: Vector2D;
    alpha: number;
    color: string;
    points: Vector2D[];

    constructor() {
        this.position = new Vector2D(-100,-100);
        this.velocity = new Vector2D();
        this.acceleration = new Vector2D();
        this.alpha = 0;
        this.color = '#000000';
        this.points = [new Vector2D(-10 + Math.random()*20, -10 + Math.random()*20),
                    new Vector2D(-10 + Math.random()*20, -10 + Math.random()*20),
                    new Vector2D(-10 + Math.random()*20, -10 + Math.random()*20)];
    }

    update() {
        this.velocity.add(this.acceleration);
        this.position.add(this.velocity);
        this.acceleration.reset(0,0);
        this.alpha -= 0.008;
        if (this.alpha < 0) this.alpha = 0;
    }

    follow() {
        if (!canvas.value) {return;}
        var x = Math.floor(this.position.x / 20);
        var y = Math.floor(this.position.y / 20);
        var index = x * Math.floor(canvas.value.height/20) + y;
        var force = forces.value[index];
        if (force) this.applyForce(force);
    }

    applyForce(force: {x: number, y: number}) {
        this.acceleration.add(force);
    }

    draw() {
        const ctx = canvas.value?.getContext('2d')
        if (!ctx) return;
        ctx.globalAlpha = this.alpha;
        ctx.beginPath();
        ctx.moveTo(this.position.x+this.points[0]!.x, this.position.y+this.points[0]!.y);
        ctx.lineTo(this.position.x+this.points[1]!.x, this.position.y+this.points[1]!.y);
        ctx.lineTo(this.position.x+this.points[2]!.x, this.position.y+this.points[2]!.y);
        ctx.closePath();
        ctx.fillStyle = this.color;
        ctx.fill();
    }
}

const canvas = ref<HTMLCanvasElement>()

const fan = ref<HTMLElement>()
const forces = ref<Vector2D[]>([])
const particles = ref<Particle[]>([]);
const pointerPosition = ref<Vector2D>(new Vector2D());
let currentParticle = 0

onMounted(() => {
    if (!canvas.value) {
        console.error('Canvas not found');
        return;
    }
    if (!fan.value) {
        console.error('Fan not found');
        return;
    }

    console.log(fan)

    const ctx = canvas.value.getContext('2d');
    const fanDimensions = fan.value.getBoundingClientRect();

    const centerOfFan = new Vector2D(
        fanDimensions.x + window.scrollX + fanDimensions.width / 2,
        fanDimensions.y + window.scrollY + fanDimensions.height / 2
    );

    let emitter = new Vector2D(window.innerWidth/2, window.innerHeight/2);
    let mouse = new Vector2D(window.innerWidth/2, window.innerHeight/2);

    window.addEventListener('mousemove', onPointerMove);
    window.addEventListener('touchmove', onPointerMove);
    window.addEventListener('resize', () =>{
        resize();
        updateFanCenter();
    });

    function animate() {
        if (!canvas.value) {return;}

        updateFanCenter();
        ctx!.clearRect(0, 0, canvas.value.width, canvas.value.height);
        updateEmitter(emitter);
        launchParticle(emitter);
        // launchParticle(emitter);
        updateForces(centerOfFan);
        drawParticles();
        requestAnimationFrame(animate);
    }

    resize();
    initParticles();
    requestAnimationFrame(animate);
})

function resize() {
    if (!canvas.value) {return;}
    canvas.value.width = window.innerWidth;
    canvas.value.height = window.innerHeight;
    initForces();
}

function initForces() {
    if (!canvas.value) {return;}
    let i = 0;
    for (let x = 0; x < canvas.value.width; x += FORCE_SPACING) {
        for (let y = 0; y < canvas.value.height; y += FORCE_SPACING) {
            if (!forces.value[i]) forces.value[i] = new Vector2D();
            i++;
        }
    }
    // Here, we make sure to remove force vectors that may be extra after a resize
    if (i < forces.value.length) {
        forces.value.splice(i+1);
    }
}

function updateFanCenter() {
    if (!fan.value) {return;}
    const fanDimensions = fan.value.getBoundingClientRect();
    fanDimensions.x = fanDimensions.x + window.scrollX + fanDimensions.width / 2;
    fanDimensions.y = fanDimensions.y + window.scrollY + fanDimensions.height / 2;
};

function updateForces(fanCenter: {x: number, y: number}) {
    if (!canvas.value) {return;}
    var i = 0;
    for (var x = 0; x < canvas.value.width; x += 20) {
        for (var y = 0; y < canvas.value.height; y += 20) {
            // Calculate vector pointing away from the fan
            let dx = x - fanCenter.x;
            let dy = y - fanCenter.y;
            let distance = Math.sqrt(dx * dx + dy * dy);

            // Calculate force magnitude inversely proportional to distance
            // Ensure that you do not divide by zero when the distance is very small
            let forceMagnitude = distance > 0 ? 1 / distance : 1;

            // The force should be stronger when closer to the fan
            // You can adjust the strength of the force by changing the coefficient (e.g., 50)
            forceMagnitude *= 50; // Adjust this value as needed

            // Normalize and scale the force
            let fx = (dx / distance) * forceMagnitude;
            let fy = (dy / distance) * forceMagnitude;

            // Set the force for the current grid point
            if (forces.value[i]) forces.value[i]?.reset(fx, fy);
            i++;
        }
    }
}

function initParticles() {
    for (var i = 0; i < MAX_PARTICLES; i++) {
        particles.value.push(new Particle());
        particles.value[i]!.velocity.y = 0.1;
    }
}

function launchParticle(emitter: Vector2D) {
    if(!canvas.value) {return;}

    particles.value[currentParticle]!.position.reset(emitter.x, emitter.y);
    particles.value[currentParticle]!.velocity.reset(-1+ Math.random()*2, -1+Math.random()*2);
    particles.value[currentParticle]!.color = `hsl(${Math.floor(emitter.x/canvas.value.width*256)},40%,${60+Math.random()*20}%)`;
    particles.value[currentParticle]!.alpha = 1;
    currentParticle++;
    if (currentParticle === MAX_PARTICLES) {
        currentParticle = 0;
    }
}

function drawParticles() {
    for (var i = 0; i < MAX_PARTICLES; i++) {
        particles.value[i]?.follow();
        particles.value[i]?.update();
        particles.value[i]?.draw();
    }
}

function updateEmitter(emitter: Vector2D) {
    emitter.interpolate(pointerPosition.value, 0.2);
}

function onPointerMove(e: MouseEvent | TouchEvent) {
    if (e instanceof MouseEvent) {
        pointerPosition.value.x = e.pageX;
        pointerPosition.value.y = e.pageY;
    } else if (e instanceof TouchEvent) {
        pointerPosition.value.x = e.touches[0]?.pageX || 0;
        pointerPosition.value.y = e.touches[0]?.pageY || 0;
    }
}
</script>



<template>
    <div class="grid place-items-center h-full w-full">
        <span ref="fan">
            <Icon icon="mdi:fan" class="spin text-5xl"></Icon>
        </span>
    </div>
    <canvas ref="canvas"></canvas>
</template>

<style scoped>
@keyframes spin {
    to {
        transform: rotate(360deg);
    }
}
.spin {
    animation: spin .5s linear infinite;
}

canvas {
    position: absolute;
    top: 0;
    left: 0;
}
</style>
