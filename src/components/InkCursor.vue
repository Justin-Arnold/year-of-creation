<script setup lang="ts">
import { ref, onMounted } from 'vue';


const canvas = ref<HTMLCanvasElement>();
const mouseMoved = ref(false);

onMounted(() => {
    if (!canvas.value) return
    const ctx = canvas.value.getContext('2d');

    const pointer = {
        x: .5 * window.innerWidth,
        y: .5 * window.innerHeight,
    }

    const params = {
        pointsNumber: 40,
        widthFactor: .3,
        mouseThreshold: .6,
        spring: .4,
        friction: .5
    };

    const trail = new Array(params.pointsNumber);
    for (let i = 0; i < params.pointsNumber; i++) {
        trail[i] = {
            x: pointer.x,
            y: pointer.y,
            dx: 0,
            dy: 0,
        }
    }

    window.addEventListener("click", e => {
        updateMousePosition(e.pageX, e.pageY);
    });
    window.addEventListener("mousemove", e => {
        mouseMoved.value = true;
        updateMousePosition(e.pageX, e.pageY);
    });
    window.addEventListener("touchmove", e => {
        if (!e.targetTouches[0]) return;
        mouseMoved.value = true;
        updateMousePosition(e.targetTouches[0].pageX, e.targetTouches[0].pageY);
    });


    function updateMousePosition(eX:number, eY:number) {
        pointer.x = eX;
        pointer.y = eY;
    }

    function update(t: number) {
        if (ctx === null) return
        if (canvas.value === null || canvas.value === undefined) throw new Error('Canvas is null');

        // for intro motion
        if (!mouseMoved) {
            pointer.x = (.5 + .3 * Math.cos(.002 * t) * (Math.sin(.005 * t))) * window.innerWidth;
            pointer.y = (.5 + .2 * (Math.cos(.005 * t)) + .1 * Math.cos(.01 * t)) * window.innerHeight;
        }

        ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);
        trail.forEach((p, pIdx) => {
            const prev = pIdx === 0 ? pointer : trail[pIdx - 1];
            const spring = pIdx === 0 ? .4 * params.spring : params.spring;
            p.dx += (prev.x - p.x) * spring;
            p.dy += (prev.y - p.y) * spring;
            p.dx *= params.friction;
            p.dy *= params.friction;
            p.x += p.dx;
            p.y += p.dy;
        });

        ctx.beginPath();
        ctx.moveTo(trail[0].x, trail[0].y);

        for (let i = 1; i < trail.length - 1; i++) {
            const xc = .5 * (trail[i].x + trail[i + 1].x);
            const yc = .5 * (trail[i].y + trail[i + 1].y);
            ctx.quadraticCurveTo(trail[i].x, trail[i].y, xc, yc);
            ctx.lineWidth = params.widthFactor * (params.pointsNumber - i);
            ctx.stroke();
        }
        ctx.lineTo(trail[trail.length - 1].x, trail[trail.length - 1].y);
        ctx.stroke();

        window.requestAnimationFrame(update);
        }

        function setupCanvas() {
            if (!canvas.value) return
            canvas.value.width = window.innerWidth;
            canvas.value.height = window.innerHeight;
        }


        setupCanvas();
        update(0);
        window.addEventListener("resize", setupCanvas);
    })
    // for intro motion













</script>

<template>
    <canvas ref="canvas" />
</template>
