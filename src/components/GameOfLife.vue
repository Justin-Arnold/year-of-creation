<script setup lang="ts">
import { onMounted, ref } from 'vue';

const canvas = ref<HTMLCanvasElement | null>(null);

onMounted(() => {
    if (canvas.value === null) throw new Error('Canvas is null');

    canvas.value.width = window.innerWidth;
    canvas.value.height = window.innerHeight;
    const ctx = canvas.value.getContext('2d');
    const gridResolution = 100;
    const cellSize = canvas.value.width / gridResolution;
    let grid = createGrid(gridResolution);

    function createGrid(size: number) {
        return new Array(size).fill(null).map(() => new Array(size).fill(false));
    }

    function drawGrid() {
        if (canvas.value === null) throw new Error('Canvas is null');
        ctx!.clearRect(0, 0, canvas.value.width, canvas.value.height);
        for (let x = 0; x < gridResolution; x++) {
            for (let y = 0; y < gridResolution; y++) {
                ctx!.beginPath();
                ctx!.rect(x * cellSize, y * cellSize, cellSize, cellSize);
                ctx!.fillStyle = grid[x]![y] ? 'black' : 'white';
                ctx!.fill();
                ctx!.stroke();
            }
        }
    }

    function updateGrid() {
        const nextGrid = createGrid(gridResolution);
        for (let x = 0; x < gridResolution; x++) {
            for (let y = 0; y < gridResolution; y++) {
                const alive = countAliveNeighbours(x, y);
                if (grid[x]![y]) {
                    nextGrid[x]![y] = alive === 2 || alive === 3;
                } else {
                    nextGrid[x]![y] = alive === 3;
                }
            }
        }
        grid = nextGrid;
    }

    function countAliveNeighbours(x: number, y: number) {
        let count = 0;
        for (let dx = -1; dx <= 1; dx++) {
            for (let dy = -1; dy <= 1; dy++) {
                if (dx === 0 && dy === 0) continue;
                const nx = x + dx, ny = y + dy;
                if (nx >= 0 && nx < gridResolution && ny >= 0 && ny < gridResolution) {
                    count += grid[nx]![ny] ? 1 : 0;
                }
            }
        }
        return count;
    }

    function gameLoop() {
        updateGrid();
        drawGrid();
        requestAnimationFrame(gameLoop);
    }

    // Randomly populate the grid for initial setup
    for (let x = 0; x < gridResolution; x++) {
        for (let y = 0; y < gridResolution; y++) {
            grid[x]![y] = Math.random() > 0.8;
        }
    }

    gameLoop();
})
</script>

<template>
    <canvas ref="canvas"></canvas>
</template>