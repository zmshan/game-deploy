<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { ElMessage } from 'element-plus'

// 游戏配置
const CELL_SIZE = 20
const GRID_SIZE = 20
const INITIAL_SNAKE_LENGTH = 3

// 游戏状态
const gameStarted = ref(false)
const gameOver = ref(false)
const score = ref(0)
const level = ref(1)
const speed = ref(150)

// 蛇和食物的状态
const snake = ref([])
const food = ref({ x: 0, y: 0 })
const direction = ref('right')
let gameLoop = null

// 初始化游戏
const initGame = () => {
  // 初始化蛇的位置
  snake.value = Array.from({ length: INITIAL_SNAKE_LENGTH }, (_, i) => ({
    x: Math.floor(GRID_SIZE / 2) - i,
    y: Math.floor(GRID_SIZE / 2)
  }))
  generateFood()
  score.value = 0
  gameOver.value = false
  direction.value = 'right'
}

// 生成食物
const generateFood = () => {
  let newFood
  do {
    newFood = {
      x: Math.floor(Math.random() * GRID_SIZE),
      y: Math.floor(Math.random() * GRID_SIZE)
    }
  } while (snake.value.some(segment => segment.x === newFood.x && segment.y === newFood.y))
  food.value = newFood
}

// 移动蛇
const moveSnake = () => {
  if (gameOver.value || !gameStarted.value) return

  const head = { ...snake.value[0] }

  switch (direction.value) {
    case 'up': head.y--; break
    case 'down': head.y++; break
    case 'left': head.x--; break
    case 'right': head.x++; break
  }

  // 检查碰撞
  if (checkCollision(head)) {
    handleGameOver()
    return
  }

  snake.value.unshift(head)

  // 检查是否吃到食物
  if (head.x === food.value.x && head.y === food.value.y) {
    score.value += 10
    generateFood()
    checkLevelUp()
  } else {
    snake.value.pop()
  }
}

// 检查碰撞
const checkCollision = (head) => {
  // 检查是否撞墙
  if (head.x < 0 || head.x >= GRID_SIZE || head.y < 0 || head.y >= GRID_SIZE) {
    return true
  }

  // 检查是否撞到自己
  return snake.value.some(segment => segment.x === head.x && segment.y === head.y)
}

// 处理游戏结束
const handleGameOver = () => {
  gameOver.value = true
  gameStarted.value = false
  clearInterval(gameLoop)
  ElMessage.error('游戏结束！')
}

// 开始游戏
const startGame = () => {
  if (gameLoop) clearInterval(gameLoop)
  initGame()
  gameStarted.value = true
  gameLoop = setInterval(moveSnake, speed.value)
}

// 检查升级
const checkLevelUp = () => {
  if (score.value > 0 && score.value % 50 === 0) {
    level.value++
    speed.value = Math.max(50, 150 - (level.value - 1) * 20)
    clearInterval(gameLoop)
    gameLoop = setInterval(moveSnake, speed.value)
    ElMessage.success(`升级到第 ${level.value} 关！`)
  }
}

// 键盘控制
const handleKeydown = (e) => {
  if (!gameStarted.value) return

  const key = e.key.toLowerCase()
  const newDirection = {
    'arrowup': 'up',
    'arrowdown': 'down',
    'arrowleft': 'left',
    'arrowright': 'right',
    'w': 'up',
    's': 'down',
    'a': 'left',
    'd': 'right'
  }[key]

  if (newDirection) {
    const oppositeDirections = {
      up: 'down',
      down: 'up',
      left: 'right',
      right: 'left'
    }

    if (oppositeDirections[newDirection] !== direction.value) {
      direction.value = newDirection
    }
  }
}

// 生命周期钩子
onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  if (gameLoop) clearInterval(gameLoop)
})
</script>

<template>
  <div class="snake-game">
    <div class="game-header">
      <h2>贪吃蛇游戏</h2>
      <div class="game-info">
        <span>得分: {{ score }}</span>
        <span>等级: {{ level }}</span>
      </div>
    </div>

    <div class="game-container" :style="{ width: GRID_SIZE * CELL_SIZE + 'px', height: GRID_SIZE * CELL_SIZE + 'px' }">
      <!-- 蛇身 -->
      <div v-for="(segment, index) in snake" :key="index" class="snake-segment"
        :style="{ left: segment.x * CELL_SIZE + 'px', top: segment.y * CELL_SIZE + 'px' }">
      </div>

      <!-- 食物 -->
      <div class="food" :style="{ left: food.x * CELL_SIZE + 'px', top: food.y * CELL_SIZE + 'px' }">
      </div>
    </div>

    <div class="game-controls">
      <el-button type="primary" @click="startGame" :disabled="gameStarted">
        {{ gameOver ? '重新开始' : '开始游戏' }}
      </el-button>
    </div>

    <div class="game-instructions">
      <h3>游戏说明：</h3>
      <p>使用方向键或 WASD 控制蛇的移动</p>
      <p>每吃到一个食物得10分</p>
      <p>每得50分升一级，移动速度会加快</p>
      <p>撞到墙壁或自己的身体游戏结束</p>
    </div>
  </div>
</template>

<style scoped>
.snake-game {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
}

.game-header {
  text-align: center;
  margin-bottom: 20px;
}

.game-info {
  display: flex;
  gap: 20px;
  margin-top: 10px;
  font-size: 18px;
}

.game-container {
  position: relative;
  background-color: #f0f0f0;
  border: 2px solid #666;
  margin: 20px 0;
}

.snake-segment {
  position: absolute;
  width: 20px;
  height: 20px;
  background-color: #42b983;
  border: 1px solid #2c3e50;
  box-sizing: border-box;
}

.food {
  position: absolute;
  width: 20px;
  height: 20px;
  background-color: #ff6b6b;
  border-radius: 50%;
  box-sizing: border-box;
}

.game-controls {
  margin: 20px 0;
}

.game-instructions {
  max-width: 400px;
  margin-top: 20px;
  padding: 15px;
  background-color: #f8f9fa;
  border-radius: 8px;
}

.game-instructions h3 {
  color: #2c3e50;
  margin-bottom: 10px;
}

.game-instructions p {
  margin: 5px 0;
  color: #666;
}
</style>