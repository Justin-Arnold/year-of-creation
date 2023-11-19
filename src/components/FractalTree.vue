<script setup lang="ts">
import { onMounted, ref } from 'vue';

const canvas = ref<HTMLCanvasElement>()

onMounted(() => {
    if (!canvas.value) return
    const ctx = canvas.value.getContext('2d');
    canvas.value.width = window.innerWidth;
    canvas.value.height = window.innerHeight;

    const branchWidth = 1;
    const maxDepth = 10;
    const branchLength = 100;
    const branches = <any>[];

    class Branch {

        startX: number;
        startY: number;
        length: number;
        angle: number;
        depth: number;
        color: string;
        currentLength: number;
        growing: boolean;

        constructor(startX: number, startY: number, length: number, angle: number, depth: number, color: string) {
            this.startX = startX;
            this.startY = startY;
            this.length = length;
            this.angle = angle;
            this.depth = depth;
            this.color = color;
            this.currentLength = 0;
            this.growing = true;
        }

        grow() {
            if (this.growing) {
                this.currentLength += 1; // Speed of growth
                if (this.currentLength >= this.length) {
                    this.currentLength = this.length;
                    this.growing = false;
                    if (this.depth > 0) {
                        branches.push(new Branch(this.endX(), this.endY(), this.length * 0.9, this.angle - 20, this.depth - 1, this.color));
                        branches.push(new Branch(this.endX(), this.endY(), this.length * 0.9, this.angle + 20, this.depth - 1, this.color));
                    }
                }
            }
        }

        draw() {
            if (!ctx) return
            ctx.strokeStyle = this.color;
            ctx.lineWidth = branchWidth;
            ctx.beginPath();
            ctx.moveTo(this.startX, this.startY);
            ctx.lineTo(this.startX + this.currentLength * Math.cos(this.angle * Math.PI / 180),
                    this.startY + this.currentLength * Math.sin(this.angle * Math.PI / 180));
            ctx.stroke();
        }

        endX() {
            return this.startX + this.currentLength * Math.cos(this.angle * Math.PI / 180);
        }

        endY() {
            return this.startY + this.currentLength * Math.sin(this.angle * Math.PI / 180);
        }
    }

    // Create the initial branch
    branches.push(new Branch(canvas.value.width / 2, canvas.value.height, branchLength, -90, maxDepth, 'brown'));

    function animate() {
      requestAnimationFrame(animate);
      ctx.clearRect(0, 0, canvas.value.width, canvas.value.height);

      branches.forEach(branch => {
        branch.grow();
        branch.draw();
      });
    }

    animate();

    // Resize canvas on window resize
    window.addEventListener('resize', () => {
      canvas.width = window.innerWidth;
      canvas.height = window.innerHeight;
    });
});
</script>

<template>
    <canvas ref="canvas"></canvas>
</template>


