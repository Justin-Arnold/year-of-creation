<script setup lang="ts">
import { onMounted, ref } from 'vue';

const canvas = ref<HTMLCanvasElement>();
const container = ref<HTMLDivElement>();

onMounted(() => {
    if (!canvas.value || !container.value) {
        alert('No canvas found!');
        return
    }

    const ctx = canvas.value.getContext('2d');
    if (!ctx) {
        alert('No context found!');
        return
    }

    ctx.canvas.height = container.value.clientHeight;
    ctx.canvas.width = ctx.canvas.height * 9 / 16;

    const PIPE_WIDTH = 100;
    const PIPE_GAP = 200;
    const PIPE_SPEED = 5;
    const PIPE_FREQUENCY = 1000;
    const timeSinceLastPipe = ref(0);
    const pipes = ref<Pipe[]>([]);
    const bird = new Bird(ctx.canvas.width / 2, ctx.canvas.height / 2, 20, 0, 0.5);
    const jumpFrameTimeout = 3;
    const jumpTimeout = ref(0)

    // Animation Loop
    function animationLoop() {
        if(!ctx) return;

        ctx.fillStyle = 'lightblue';
        ctx.fillRect(0, 0, ctx.canvas.width, ctx.canvas.height);

        // Drawing the Ground
        const GROUND_HEIGHT = 100;
        ctx.fillStyle = 'Green';
        ctx.fillRect(0, ctx.canvas.height - GROUND_HEIGHT, ctx.canvas.width, GROUND_HEIGHT);

        // Animate Pipes
        if (timeSinceLastPipe.value > PIPE_FREQUENCY) {
            const pipeHeight = Math.random() * (ctx.canvas.height - PIPE_GAP - GROUND_HEIGHT);
            pipes.value.push(new Pipe(ctx.canvas.width, 0, PIPE_WIDTH, pipeHeight, PIPE_SPEED));
            pipes.value.push(new Pipe(ctx.canvas.width, pipeHeight + PIPE_GAP, PIPE_WIDTH, ctx.canvas.height - pipeHeight - PIPE_GAP - GROUND_HEIGHT, PIPE_SPEED));
            timeSinceLastPipe.value = 0;
        } else {
            timeSinceLastPipe.value += 16;
        }
        pipes.value.forEach(pipe => {
            pipe.draw(ctx);
            pipe.update();
        });
        // Remove Pipes that are off screen
        pipes.value = pipes.value.filter(pipe => pipe.x + pipe.width > 0);

        //there should be no player input, calculate based on bird position and next pipe gap and jump if needed
        const nextPipe = pipes.value.find(pipe => pipe.x > ctx.canvas.width / 2 );
        if (nextPipe && nextPipe.x < ctx.canvas.width / 2 + PIPE_WIDTH / 2) {
            if (bird.y < nextPipe.y || bird.y > nextPipe.y + nextPipe.height) {

                jumpTimeout.value ++;
                if (jumpTimeout.value > jumpFrameTimeout) {
                    bird.jump();
                    jumpTimeout.value = 0;
                }
            }
        } else if (bird.y > ctx.canvas.height - GROUND_HEIGHT) {
            bird.jump();
        }
        if (!nextPipe) {
            bird.startIdle();
        } else {
            bird.stopIdle();
        }


        //Render Bird
        bird.draw(ctx);
        bird.update();

        requestAnimationFrame(animationLoop);
    }
    animationLoop()
})



class Pipe {
    x: number;
    y: number;
    width: number;
    height: number;
    speed: number;
    constructor(x: number, y: number, width: number, height: number, speed: number) {
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
        this.speed = speed;
    }
    draw(ctx: CanvasRenderingContext2D) {
        ctx.fillStyle = 'gray';
        ctx.fillRect(this.x, this.y, this.width, this.height);
    }
    update() {
        this.x -= this.speed;
    }
}

class Bird {
    x: number;
    y: number;
    radius: number;
    speed: number;
    gravity: number;
    constructor(x: number, y: number, radius: number, speed: number, gravity: number) {
        this.x = x;
        this.y = y;
        this.radius = radius;
        this.speed = speed;
        this.gravity = gravity;
    }
    draw(ctx: CanvasRenderingContext2D) {
        ctx.fillStyle = 'yellow';
        ctx.beginPath();
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fill();
    }
    update() {
        this.y += this.speed;
        this.speed += this.gravity;
    }
    jump() {
        console.log('jump');
        this.speed = -10;
    }
    startIdle() {
        this.gravity = 0;
    }
    stopIdle() {
        this.gravity = 0.5;
    }
}
</script>

<template>
    <div ref="container" class="h-full">
        <canvas ref="canvas" />
    </div>
</template>