<script setup>
import { ref, onMounted } from 'vue'

// 棋盘大小
const BOARD_SIZE = 15

// 初始化棋盘数据
const board = ref(Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(0)))
const currentPlayer = ref(1) // 1代表黑棋，2代表白棋
const gameOver = ref(false)
const winner = ref(0)
const lastMove = ref(null)

// 落子函数
function placePiece (row, col) {
  if (board.value[row][col] === 0 && !gameOver.value) {
    board.value[row][col] = currentPlayer.value
    lastMove.value = { row, col }

    if (checkWin(row, col)) {
      gameOver.value = true
      winner.value = currentPlayer.value
    } else {
      currentPlayer.value = currentPlayer.value === 1 ? 2 : 1
    }
  }
}

// 检查胜利条件
function checkWin (row, col) {
  const directions = [
    [1, 0], [0, 1], [1, 1], [1, -1] // 水平、垂直、对角线方向
  ]

  for (const [dx, dy] of directions) {
    let count = 1
    let r = row, c = col

    // 正向检查
    while (true) {
      r += dx
      c += dy
      if (r < 0 || r >= BOARD_SIZE || c < 0 || c >= BOARD_SIZE ||
        board.value[r][c] !== currentPlayer.value) break
      count++
    }

    // 反向检查
    r = row
    c = col
    while (true) {
      r -= dx
      c -= dy
      if (r < 0 || r >= BOARD_SIZE || c < 0 || c >= BOARD_SIZE ||
        board.value[r][c] !== currentPlayer.value) break
      count++
    }

    if (count >= 5) return true
  }
  return false
}

// 重新开始游戏
function restartGame () {
  board.value = Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(0))
  currentPlayer.value = 1
  gameOver.value = false
  winner.value = 0
  lastMove.value = null
}

// 悔棋功能
function undoMove () {
  if (lastMove.value && !gameOver.value) {
    const { row, col } = lastMove.value
    board.value[row][col] = 0
    currentPlayer.value = currentPlayer.value === 1 ? 2 : 1
    lastMove.value = null
  }
}
</script>

<template>
  <div class="gobang-container">
    <div class="game-info">
      <h2>五子棋</h2>
      <div class="player-info" :class="{ 'game-over': gameOver }">
        <template v-if="!gameOver">
          当前玩家: <span :class="currentPlayer === 1 ? 'black' : 'white'">{{ currentPlayer === 1 ? '黑棋' : '白棋' }}</span>
        </template>
        <template v-else>
          游戏结束！获胜者: <span :class="winner === 1 ? 'black' : 'white'">{{ winner === 1 ? '黑棋' : '白棋' }}</span>
        </template>
      </div>
      <div class="controls">
        <button @click="restartGame">重新开始</button>
        <button @click="undoMove" :disabled="!lastMove || gameOver">悔棋</button>
      </div>
    </div>

    <div class="board">
      <div v-for="(row, i) in board" :key="i" class="board-row">
        <div v-for="(cell, j) in row" :key="j" class="cell" @click="placePiece(i, j)">
          <div v-if="cell !== 0" class="piece" :class="{
            'black': cell === 1,
            'white': cell === 2,
            'last-move': lastMove && lastMove.row === i && lastMove.col === j
          }"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.gobang-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
}

.game-info {
  text-align: center;
  margin-bottom: 20px;
}

.player-info {
  font-size: 1.2em;
  margin: 10px 0;
}

.player-info.game-over {
  color: #e74c3c;
  font-weight: bold;
}

.controls {
  margin: 15px 0;
}

.controls button {
  margin: 0 10px;
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  background: #3498db;
  color: white;
  cursor: pointer;
  transition: background 0.3s;
}

.controls button:hover {
  background: #2980b9;
}

.controls button:disabled {
  background: #bdc3c7;
  cursor: not-allowed;
}

.board {
  display: inline-block;
  background: #d4b887;
  padding: 15px;
  border-radius: 5px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.board-row {
  display: flex;
}

.cell {
  width: 35px;
  height: 35px;
  border: 1px solid #8b4513;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  position: relative;
}

.cell::before,
.cell::after {
  content: '';
  position: absolute;
  background: #8b4513;
}

.cell::before {
  width: 100%;
  height: 1px;
  top: 50%;
  transform: translateY(-50%);
}

.cell::after {
  width: 1px;
  height: 100%;
  left: 50%;
  transform: translateX(-50%);
}

.piece {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  z-index: 1;
  transition: all 0.3s ease;
}

.piece.black {
  background: radial-gradient(circle at 35% 35%, #666, #000);
  box-shadow: 2px 2px 4px rgba(0, 0, 0, 0.4);
}

.piece.white {
  background: radial-gradient(circle at 35% 35%, #fff, #ddd);
  box-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
}

.piece.last-move {
  box-shadow: 0 0 8px 4px #f39c12;
}

.black {
  color: #000;
}

.white {
  color: #666;
}
</style>