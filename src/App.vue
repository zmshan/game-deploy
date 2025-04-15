<script setup>
import SnakeGame from './components/SnakeGame.vue'
import SlotMachine from './components/SlotMachine.vue'
import Tetris from './components/Tetris.vue'
import FlappyBird from './components/FlappyBird.vue'
import BoneChickenTiger from './components/BoneChickenTiger.vue'
import GourdDoll from './components/GourdDoll.vue'
import { ref } from 'vue'
import 'element-plus/dist/index.css'

const currentGame = ref('snake')
</script>

<template>
  <div class="game-container">
    <div class="game-selector">
      <button :class="{ active: currentGame === 'snake' }" @click="currentGame = 'snake'">贪吃蛇</button>
      <button :class="{ active: currentGame === 'slot' }" @click="currentGame = 'slot'">老虎机</button>
      <button :class="{ active: currentGame === 'tetris' }" @click="currentGame = 'tetris'">俄罗斯方块</button>
      <button :class="{ active: currentGame === 'flappy' }" @click="currentGame = 'flappy'">Flappy Bird</button>
      <button :class="{ active: currentGame === 'bct' }" @click="currentGame = 'bct'">棒子鸡子老虎</button>
      <button :class="{ active: currentGame === 'gourd' }" @click="currentGame = 'gourd'">葫芦娃</button>
    </div>

    <component :is="currentGame === 'snake' ? SnakeGame :
      (currentGame === 'slot' ? SlotMachine :
        (currentGame === 'tetris' ? Tetris :
          (currentGame === 'flappy' ? FlappyBird :
            (currentGame === 'bct' ? BoneChickenTiger : GourdDoll))))" />
  </div>
</template>

<style>
#app {
  font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  margin: 0;
  padding: 20px;
  box-sizing: border-box;
}

.game-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.game-selector {
  display: flex;
  justify-content: center;
  gap: 20px;
  margin-bottom: 30px;
}

.game-selector button {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background: #4CAF50;
  color: white;
  cursor: pointer;
  font-size: 1.1em;
  transition: all 0.3s ease;
}

.game-selector button:hover {
  background: #45a049;
}

.game-selector button.active {
  background: #2c3e50;
  transform: scale(1.05);
}
</style>
