<template>
  <div class="tetris-game">
    <div class="game-container">
      <div class="game-board">
        <div v-for="(row, y) in gameBoard" :key="y" class="row">
          <div v-for="(cell, x) in row" :key="x" class="cell"
            :class="{ filled: cell || (currentPiece && isCurrentPieceCellFilled(x, y)) }">
          </div>
        </div>
      </div>

      <div class="game-info">
        <div class="next-piece">
          <h3>下一个方块</h3>
          <div class="next-piece-board">
            <div v-for="(row, y) in nextPieceBoard" :key="y" class="row">
              <div v-for="(cell, x) in row" :key="x" class="cell" :class="{ filled: cell }">
              </div>
            </div>
          </div>
        </div>

        <div class="score-info">
          <div class="score">分数: {{ score }}</div>
          <div class="level">等级: {{ level }}</div>
          <div class="lines">消除行数: {{ lines }}</div>
        </div>

        <div class="controls">
          <button @click="startGame" :disabled="isPlaying">开始游戏</button>
          <button @click="pauseGame" :disabled="!isPlaying">{{ isPaused ? '继续' : '暂停' }}</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 游戏配置
const BOARD_WIDTH = 10;
const BOARD_HEIGHT = 20;
const INITIAL_SPEED = 1000;

// 俄罗斯方块形状定义
const TETROMINOES = {
  I: [
    [1, 1, 1, 1]
  ],
  O: [
    [1, 1],
    [1, 1]
  ],
  T: [
    [0, 1, 0],
    [1, 1, 1]
  ],
  S: [
    [0, 1, 1],
    [1, 1, 0]
  ],
  Z: [
    [1, 1, 0],
    [0, 1, 1]
  ],
  J: [
    [1, 0, 0],
    [1, 1, 1]
  ],
  L: [
    [0, 0, 1],
    [1, 1, 1]
  ]
};

// 游戏状态
const gameBoard = ref(Array(BOARD_HEIGHT).fill().map(() => Array(BOARD_WIDTH).fill(0)));
const nextPieceBoard = ref(Array(4).fill().map(() => Array(4).fill(0)));
const currentPiece = ref(null);
const nextPiece = ref(null);
const currentPosition = ref({ x: 0, y: 0 });
const score = ref(0);
const level = ref(1);
const lines = ref(0);
const isPlaying = ref(false);
const isPaused = ref(false);

let gameInterval = null;

// 游戏控制
const startGame = () => {
  resetGame();
  isPlaying.value = true;
  generateNewPiece();
  startGameLoop();
};

const pauseGame = () => {
  isPaused.value = !isPaused.value;
  if (isPaused.value) {
    clearInterval(gameInterval);
  } else {
    startGameLoop();
  }
};

const resetGame = () => {
  gameBoard.value = Array(BOARD_HEIGHT).fill().map(() => Array(BOARD_WIDTH).fill(0));
  score.value = 0;
  level.value = 1;
  lines.value = 0;
  currentPiece.value = null;
  nextPiece.value = null;
  isPaused.value = false;
};

// 游戏循环
const startGameLoop = () => {
  clearInterval(gameInterval);
  gameInterval = setInterval(() => {
    if (!isPaused.value) {
      moveDown();
    }
  }, INITIAL_SPEED / level.value);
};

// 方块操作
const generateNewPiece = () => {
  const pieces = Object.keys(TETROMINOES);
  if (!nextPiece.value) {
    nextPiece.value = {
      shape: TETROMINOES[pieces[Math.floor(Math.random() * pieces.length)]],
      type: pieces[Math.floor(Math.random() * pieces.length)]
    };
  }

  currentPiece.value = nextPiece.value;
  nextPiece.value = {
    shape: TETROMINOES[pieces[Math.floor(Math.random() * pieces.length)]],
    type: pieces[Math.floor(Math.random() * pieces.length)]
  };

  currentPosition.value = {
    x: Math.floor((BOARD_WIDTH - currentPiece.value.shape[0].length) / 2),
    y: 0
  };

  updateNextPieceBoard();

  if (!canMove(0, 0)) {
    gameOver();
  }
};

const updateNextPieceBoard = () => {
  nextPieceBoard.value = Array(4).fill().map(() => Array(4).fill(0));
  const shape = nextPiece.value.shape;
  const offsetY = Math.floor((4 - shape.length) / 2);
  const offsetX = Math.floor((4 - shape[0].length) / 2);

  shape.forEach((row, y) => {
    row.forEach((cell, x) => {
      if (cell) {
        nextPieceBoard.value[y + offsetY][x + offsetX] = cell;
      }
    });
  });
};

const canMove = (moveX, moveY) => {
  return currentPiece.value.shape.every((row, y) => {
    return row.every((cell, x) => {
      const newX = currentPosition.value.x + x + moveX;
      const newY = currentPosition.value.y + y + moveY;
      return (
        !cell ||
        (newX >= 0 &&
          newX < BOARD_WIDTH &&
          newY < BOARD_HEIGHT &&
          newY >= 0 &&
          !gameBoard.value[newY][newX])
      );
    });
  });
};

const moveDown = () => {
  if (canMove(0, 1)) {
    currentPosition.value.y++;
  } else {
    placePiece();
    clearLines();
    generateNewPiece();
  }
};

const moveLeft = () => {
  if (canMove(-1, 0)) {
    currentPosition.value.x--;
  }
};

const moveRight = () => {
  if (canMove(1, 0)) {
    currentPosition.value.x++;
  }
};

const rotatePiece = () => {
  const rotated = currentPiece.value.shape[0].map((_, i) =>
    currentPiece.value.shape.map(row => row[i]).reverse()
  );

  const originalShape = currentPiece.value.shape;
  currentPiece.value.shape = rotated;

  if (!canMove(0, 0)) {
    currentPiece.value.shape = originalShape;
  }
};

const placePiece = () => {
  currentPiece.value.shape.forEach((row, y) => {
    row.forEach((cell, x) => {
      if (cell) {
        gameBoard.value[currentPosition.value.y + y][currentPosition.value.x + x] = 1;
      }
    });
  });
};

const clearLines = () => {
  let linesCleared = 0;

  for (let y = BOARD_HEIGHT - 1; y >= 0; y--) {
    if (gameBoard.value[y].every(cell => cell)) {
      gameBoard.value.splice(y, 1);
      gameBoard.value.unshift(Array(BOARD_WIDTH).fill(0));
      linesCleared++;
      y++;
    }
  }

  if (linesCleared > 0) {
    lines.value += linesCleared;
    score.value += calculateScore(linesCleared);
    level.value = Math.floor(lines.value / 10) + 1;
    startGameLoop(); // 更新游戏速度
  }
};

const calculateScore = (linesCleared) => {
  const basePoints = [40, 100, 300, 1200]; // 1、2、3、4行的基础分数
  return basePoints[linesCleared - 1] * level.value;
};

const gameOver = () => {
  isPlaying.value = false;
  clearInterval(gameInterval);
  alert(`游戏结束！\n最终分数: ${score.value}\n等级: ${level.value}`);
};

// 键盘控制
const handleKeydown = (event) => {
  if (!isPlaying.value || isPaused.value) return;

  switch (event.key) {
    case 'ArrowLeft':
      moveLeft();
      break;
    case 'ArrowRight':
      moveRight();
      break;
    case 'ArrowDown':
      moveDown();
      break;
    case 'ArrowUp':
      rotatePiece();
      break;
    case ' ':
      while (canMove(0, 1)) {
        moveDown();
      }
      break;
  }
};

// 生命周期钩子
onMounted(() => {
  window.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
  clearInterval(gameInterval);
});

// 在script setup中添加
const isCurrentPieceCellFilled = (x, y) => {
  if (!currentPiece.value) return false;
  const pieceX = x - currentPosition.value.x;
  const pieceY = y - currentPosition.value.y;
  return pieceX >= 0 && pieceX < currentPiece.value.shape[0].length &&
    pieceY >= 0 && pieceY < currentPiece.value.shape.length &&
    currentPiece.value.shape[pieceY][pieceX] === 1;
};
</script>

<style scoped>
.tetris-game {
  display: flex;
  justify-content: center;
  padding: 20px;
  background: #2c3e50;
  min-height: 100vh;
}

.game-container {
  display: flex;
  gap: 20px;
  background: #34495e;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.3);
}

.game-board {
  display: grid;
  grid-template-rows: repeat(20, 30px);
  grid-template-columns: repeat(10, 30px);
  gap: 1px;
  background: #2c3e50;
  padding: 10px;
  border-radius: 4px;
}

.game-info {
  display: flex;
  flex-direction: column;
  gap: 20px;
  color: white;
}

.next-piece {
  background: #2c3e50;
  padding: 15px;
  border-radius: 4px;
}

.next-piece h3 {
  margin: 0 0 10px 0;
  text-align: center;
}

.next-piece-board {
  display: grid;
  grid-template-rows: repeat(4, 30px);
  grid-template-columns: repeat(4, 30px);
  gap: 1px;
  background: #34495e;
  padding: 5px;
}

.row {
  display: contents;
}

.cell {
  background: #34495e;
  border: 1px solid #2c3e50;
}

.cell.filled {
  background: #e74c3c;
  box-shadow: inset 0 0 10px rgba(0, 0, 0, 0.3);
}

.score-info {
  background: #2c3e50;
  padding: 15px;
  border-radius: 4px;
  font-size: 1.2em;
}

.score-info>div {
  margin: 10px 0;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background: #3498db;
  color: white;
  cursor: pointer;
  font-size: 1.1em;
  transition: background-color 0.3s;
}

button:hover:not(:disabled) {
  background: #2980b9;
}

button:disabled {
  background: #95a5a6;
  cursor: not-allowed;
}
</style>