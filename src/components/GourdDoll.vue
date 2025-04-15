<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const gameState = ref('ready') // ready, playing, over
const score = ref(0)
const level = ref(1)
const playerPosition = ref({ x: 0, y: 0 })
const enemies = ref([])
const skills = ref([
  { name: '火球', cooldown: 0, maxCooldown: 30 },
  { name: '冰冻', cooldown: 0, maxCooldown: 60 },
  { name: '治疗', cooldown: 0, maxCooldown: 45 }
])
const playerHealth = ref(100)

const moveKeys = {
  ArrowUp: { x: 0, y: -1 },
  ArrowDown: { x: 0, y: 1 },
  ArrowLeft: { x: -1, y: 0 },
  ArrowRight: { x: 1, y: 0 }
}

const startGame = () => {
  gameState.value = 'playing'
  score.value = 0
  level.value = 1
  playerPosition.value = { x: 0, y: 0 }
  playerHealth.value = 100
  enemies.value = generateEnemies()
  skills.value.forEach(skill => skill.cooldown = 0)
}

const generateEnemies = () => {
  const count = Math.min(3 + level.value, 8)
  return Array.from({ length: count }, () => ({
    x: Math.floor(Math.random() * 10),
    y: Math.floor(Math.random() * 10),
    health: 50
  }))
}

const movePlayer = (direction) => {
  if (gameState.value !== 'playing') return

  const newX = playerPosition.value.x + direction.x
  const newY = playerPosition.value.y + direction.y

  if (newX >= 0 && newX < 10 && newY >= 0 && newY < 10) {
    playerPosition.value = { x: newX, y: newY }
  }
}

const useSkill = (index) => {
  if (gameState.value !== 'playing') return
  if (skills.value[index].cooldown > 0) return

  const skill = skills.value[index]
  skill.cooldown = skill.maxCooldown

  switch (index) {
    case 0: // 火球
      damageNearbyEnemies()
      break
    case 1: // 冰冻
      freezeEnemies()
      break
    case 2: // 治疗
      healPlayer()
      break
  }
}

const damageNearbyEnemies = () => {
  enemies.value.forEach(enemy => {
    const distance = Math.abs(enemy.x - playerPosition.value.x) +
      Math.abs(enemy.y - playerPosition.value.y)
    if (distance <= 2) {
      enemy.health -= 30
    }
  })
  checkEnemies()
}

const freezeEnemies = () => {
  // 实现冰冻效果
  score.value += 5
}

const healPlayer = () => {
  playerHealth.value = Math.min(100, playerHealth.value + 30)
}

const checkEnemies = () => {
  enemies.value = enemies.value.filter(enemy => enemy.health > 0)
  if (enemies.value.length === 0) {
    level.value++
    enemies.value = generateEnemies()
    score.value += 10 * level.value
  }
}

const gameLoop = () => {
  if (gameState.value !== 'playing') return

  // 更新技能冷却
  skills.value.forEach(skill => {
    if (skill.cooldown > 0) skill.cooldown--
  })

  // 检查游戏结束条件
  if (playerHealth.value <= 0) {
    gameState.value = 'over'
  }
}

let gameInterval
onMounted(() => {
  gameInterval = setInterval(gameLoop, 1000)
  window.addEventListener('keydown', handleKeyPress)
})

onUnmounted(() => {
  clearInterval(gameInterval)
  window.removeEventListener('keydown', handleKeyPress)
})

const handleKeyPress = (event) => {
  if (moveKeys[event.key]) {
    movePlayer(moveKeys[event.key])
  } else if (event.key >= '1' && event.key <= '3') {
    useSkill(parseInt(event.key) - 1)
  }
}
</script>

<template>
  <div class="game-container">
    <h1>葫芦娃大战</h1>

    <div v-if="gameState === 'ready'" class="start-screen">
      <button @click="startGame">开始游戏</button>
    </div>

    <div v-else-if="gameState === 'playing'" class="game-board">
      <div class="status-bar">
        <div>等级: {{ level }}</div>
        <div>分数: {{ score }}</div>
        <div>生命值: {{ playerHealth }}</div>
      </div>

      <div class="skills">
        <button v-for="(skill, index) in skills" :key="index" @click="useSkill(index)" :disabled="skill.cooldown > 0">
          {{ skill.name }}
          <div v-if="skill.cooldown > 0" class="cooldown">
            {{ skill.cooldown }}
          </div>
        </button>
      </div>

      <div class="grid">
        <div v-for="y in 10" :key="y - 1" class="row">
          <div v-for="x in 10" :key="x - 1" class="cell" :class="{
            'player': playerPosition.x === x - 1 && playerPosition.y === y - 1,
            'enemy': enemies.some(e => e.x === x - 1 && e.y === y - 1)
          }"></div>
        </div>
      </div>
    </div>

    <div v-else class="game-over">
      <h2>游戏结束</h2>
      <p>最终得分: {{ score }}</p>
      <button @click="startGame">重新开始</button>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
}

.status-bar {
  display: flex;
  justify-content: space-between;
  margin-bottom: 20px;
  font-size: 18px;
}

.skills {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.skills button {
  padding: 10px 20px;
  position: relative;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.skills button:disabled {
  background: #ccc;
}

.cooldown {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 2px 6px;
  border-radius: 4px;
}

.grid {
  display: grid;
  grid-template-rows: repeat(10, 1fr);
  gap: 2px;
  background: #eee;
  padding: 2px;
  border-radius: 4px;
}

.row {
  display: grid;
  grid-template-columns: repeat(10, 1fr);
  gap: 2px;
}

.cell {
  aspect-ratio: 1;
  background: white;
  border-radius: 2px;
}

.cell.player {
  background: #4CAF50;
}

.cell.enemy {
  background: #f44336;
}

.start-screen,
.game-over {
  text-align: center;
  margin-top: 40px;
}

button {
  font-size: 18px;
  padding: 10px 20px;
  cursor: pointer;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  transition: background 0.3s;
}

button:hover:not(:disabled) {
  background: #45a049;
}
</style>