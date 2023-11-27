<script setup lang="ts">
import { ref } from 'vue';

type MineSweeperCell = {
    isMine: boolean;
    isFlagged: boolean;
    isRevealed: boolean;
    adjacentMines: number;
}

const grid = ref<MineSweeperCell[]>([]);

function generateBoard() {
    const boardSize = 10;
    const numberOfMines = 10;

    const board = new Array(boardSize * boardSize).fill(0).map(() => ({
        isMine: false,
        isFlagged: false,
        isRevealed: false,
        adjacentMines: 0,
    }));

    for (let i = 0; i < numberOfMines; i++) {
        const randomIndex = Math.floor(Math.random() * board.length);
        board[randomIndex].isMine = true;
    }

    grid.value = board;

    grid.value.forEach((cell, index) => {
        cell.adjacentMines = getAdjacentMinesByIndex(index);
    });
}

generateBoard();

function checkCell(index: number) {
    const cell = grid.value[index];
    if (cell.isMine) {
        alert('Game over!');
        generateBoard();
    } else {
        cell.isRevealed = true;
        if (cell.adjacentMines === 0) {
            revealConnectedCells(index);
        }
    }
}

function getAdjacentMinesByIndex(index: number) {
    const boardSize = 10;
    const board = grid.value;
    const adjacentCells = [
        board[index - boardSize - 1],
        board[index - boardSize],
        board[index - boardSize + 1],
        board[index - 1],
        board[index + 1],
        board[index + boardSize - 1],
        board[index + boardSize],
        board[index + boardSize + 1],
    ];

    return adjacentCells.filter(cell => cell?.isMine).length;
}

function revealConnectedCells(index: number) {
    const boardSize = 10;
    const board = grid.value;
    const adjacentCells = [
        board[index - boardSize - 1],
        board[index - boardSize],
        board[index - boardSize + 1],
        board[index - 1],
        board[index + 1],
        board[index + boardSize - 1],
        board[index + boardSize],
        board[index + boardSize + 1],
    ];

    adjacentCells.forEach(cell => {
        if (cell?.adjacentMines === 0 && !cell.isRevealed) {
            cell.isRevealed = true;
            revealConnectedCells(board.indexOf(cell));
        } else if (cell?.adjacentMines > 0) {
            cell.isRevealed = true;
        }
    });

}

function flagCell(index: number) {
    const cell = grid.value[index];
    cell.isFlagged = !cell.isFlagged;
}

</script>

<template>
   <div class="w-[400px] h-[400px] rounded-sm border border-black grid grid-cols-10 grid-rows-10">
        <span v-for="cell, index in grid" class="border border-black bg-white aspect-square grid place-items-center text-lg font-bold" :key="index" @click="checkCell(index)" @click.right.prevent="flagCell(index)">
            {{ cell.isFlagged ? '🇫🇷' :  cell.isRevealed ? cell.isMine ? '💣' : cell.adjacentMines : ''}}
        </span>
   </div>
</template>