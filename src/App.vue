<script setup>
import SnakeGame from './components/SnakeGame.vue'
import SlotMachine from './components/SlotMachine.vue'
import Tetris from './components/Tetris.vue'
import FlappyBird from './components/FlappyBird.vue'
import BoneChickenTiger from './components/BoneChickenTiger.vue'
import GourdDoll from './components/GourdDoll.vue'
import Gobang from './components/Gobang.vue'
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'
import 'element-plus/dist/index.css'

const currentGame = ref('snake')
const menuState = ref('expanded') // 'expanded', 'collapsed', 'hidden'
const menuOpacity = ref(1)
const allElementsVisible = ref(true) // 控制所有界面元素的显示状态
const isFullscreen = ref(false) // 控制全屏模式
let inactivityTimer

// 菜单拖拽相关状态
const menuPosition = reactive({
  x: 0,
  y: 0,
  isDragging: false,
  startX: 0,
  startY: 0,
  edge: 'left' // 'left', 'right', 'top', 'bottom'
})

// 游戏面板拖拽相关状态
const gamePanel = reactive({
  x: 0,
  y: 0,
  width: 800,
  height: 600,
  isDragging: false,
  isResizing: false,
  startX: 0,
  startY: 0,
  longPressTimer: null,
  isLocked: false,
  resizeHandle: '' // 'nw', 'ne', 'sw', 'se', 'n', 'e', 's', 'w'
})

// 计算菜单样式
const menuStyle = computed(() => {
  const style = {
    transform: `translate(${menuPosition.x}px, ${menuPosition.y}px)`,
    opacity: menuOpacity.value,
    transition: menuPosition.isDragging ? 'none' : 'all 0.3s ease'
  }

  // 根据吸附边缘调整样式
  if (menuPosition.edge === 'right') {
    style.flexDirection = 'column'
    style.right = '0'
    style.left = 'auto'
  } else if (menuPosition.edge === 'top') {
    style.flexDirection = 'row'
    style.top = '0'
    style.bottom = 'auto'
  } else if (menuPosition.edge === 'bottom') {
    style.flexDirection = 'row'
    style.bottom = '0'
    style.top = 'auto'
  } else { // 默认左侧
    style.flexDirection = 'column'
    style.left = '0'
    style.right = 'auto'
  }

  return style
})

// 计算游戏面板样式
const gamePanelStyle = computed(() => {
  return {
    transform: `translate(${gamePanel.x}px, ${gamePanel.y}px)`,
    width: `${gamePanel.width}px`,
    height: `${gamePanel.height}px`,
    transition: gamePanel.isDragging || gamePanel.isResizing ? 'none' : 'all 0.3s ease',
    position: 'relative',
    cursor: gamePanel.isLocked ? 'default' : 'move'
  }
})

const toggleMenu = () => {
  if (menuState.value === 'expanded') {
    menuState.value = 'collapsed'
  } else if (menuState.value === 'collapsed') {
    menuState.value = 'hidden'
  } else {
    menuState.value = 'expanded'
  }
}

const resetInactivityTimer = () => {
  menuOpacity.value = 1
  if (inactivityTimer) clearTimeout(inactivityTimer)
  inactivityTimer = setTimeout(() => {
    menuOpacity.value = 0.5
  }, 3000)
}

// 菜单拖拽开始
const startMenuDrag = (event) => {
  menuPosition.isDragging = true
  menuPosition.startX = event.clientX - menuPosition.x
  menuPosition.startY = event.clientY - menuPosition.y
}

// 菜单拖拽中
const onMenuDrag = (event) => {
  if (!menuPosition.isDragging) return

  menuPosition.x = event.clientX - menuPosition.startX
  menuPosition.y = event.clientY - menuPosition.startY
}

// 菜单拖拽结束，自动吸附到最近的边缘并确保在可视区域内
const endMenuDrag = () => {
  if (!menuPosition.isDragging) return

  menuPosition.isDragging = false

  // 获取视口尺寸
  const viewportWidth = window.innerWidth
  const viewportHeight = window.innerHeight

  // 获取菜单元素
  const menuElement = document.querySelector('.game-selector')
  if (!menuElement) return

  const menuRect = menuElement.getBoundingClientRect()

  // 确保菜单不会超出视口边界
  menuPosition.x = Math.max(0, Math.min(viewportWidth - menuRect.width, menuPosition.x))
  menuPosition.y = Math.max(0, Math.min(viewportHeight - menuRect.height, menuPosition.y))

  // 计算到各边的距离
  const distToLeft = menuPosition.x
  const distToRight = viewportWidth - (menuPosition.x + menuRect.width)
  const distToTop = menuPosition.y
  const distToBottom = viewportHeight - (menuPosition.y + menuRect.height)

  // 找出最近的边
  const minDist = Math.min(distToLeft, distToRight, distToTop, distToBottom)

  // 吸附到最近的边
  if (minDist === distToLeft) {
    menuPosition.x = 0
    menuPosition.edge = 'left'
  } else if (minDist === distToRight) {
    menuPosition.x = viewportWidth - menuRect.width
    menuPosition.edge = 'right'
  } else if (minDist === distToTop) {
    menuPosition.y = 0
    menuPosition.edge = 'top'
  } else {
    menuPosition.y = viewportHeight - menuRect.height
    menuPosition.edge = 'bottom'
  }
}

// 游戏面板长按开始
const startGamePanelLongPress = (event) => {
  if (gamePanel.isLocked) return

  // 清除之前的长按计时器
  if (gamePanel.longPressTimer) {
    clearTimeout(gamePanel.longPressTimer)
  }

  // 设置长按计时器，减少长按时间以提高响应速度
  gamePanel.longPressTimer = setTimeout(() => {
    gamePanel.isDragging = true
    gamePanel.startX = event.clientX - gamePanel.x
    gamePanel.startY = event.clientY - gamePanel.y
  }, 300) // 300ms长按触发，提高响应速度
}

// 游戏面板拖拽中
const onGamePanelDrag = (event) => {
  if (!gamePanel.isDragging && !gamePanel.isResizing) return

  if (gamePanel.isDragging) {
    // 计算新位置
    const newX = event.clientX - gamePanel.startX
    const newY = event.clientY - gamePanel.startY

    // 确保游戏面板不会完全移出视口
    const viewportWidth = window.innerWidth
    const viewportHeight = window.innerHeight

    gamePanel.x = Math.max(-gamePanel.width + 100, Math.min(viewportWidth - 100, newX))
    gamePanel.y = Math.max(-gamePanel.height + 100, Math.min(viewportHeight - 100, newY))
  } else if (gamePanel.isResizing) {
    const dx = event.clientX - gamePanel.startX
    const dy = event.clientY - gamePanel.startY

    // 根据不同的调整柄调整大小
    switch (gamePanel.resizeHandle) {
      case 'se': // 右下角
        gamePanel.width += dx
        gamePanel.height += dy
        break
      case 'sw': // 左下角
        gamePanel.x += dx
        gamePanel.width -= dx
        gamePanel.height += dy
        break
      case 'ne': // 右上角
        gamePanel.y += dy
        gamePanel.width += dx
        gamePanel.height -= dy
        break
      case 'nw': // 左上角
        gamePanel.x += dx
        gamePanel.y += dy
        gamePanel.width -= dx
        gamePanel.height -= dy
        break
      case 'n': // 上边
        gamePanel.y += dy
        gamePanel.height -= dy
        break
      case 's': // 下边
        gamePanel.height += dy
        break
      case 'e': // 右边
        gamePanel.width += dx
        break
      case 'w': // 左边
        gamePanel.x += dx
        gamePanel.width -= dx
        break
    }

    // 限制最小尺寸
    gamePanel.width = Math.max(300, gamePanel.width)
    gamePanel.height = Math.max(300, gamePanel.height)

    gamePanel.startX = event.clientX
    gamePanel.startY = event.clientY
  }
}

// 游戏面板拖拽结束
const endGamePanelDrag = () => {
  if (gamePanel.longPressTimer) {
    clearTimeout(gamePanel.longPressTimer)
    gamePanel.longPressTimer = null
  }

  if (gamePanel.isDragging) {
    gamePanel.isDragging = false
    gamePanel.isLocked = true // 放下后锁定
  }

  if (gamePanel.isResizing) {
    gamePanel.isResizing = false
  }
}

// 开始调整游戏面板大小
const startGamePanelResize = (event, handle) => {
  if (!gamePanel.isLocked) return

  event.stopPropagation()
  gamePanel.isResizing = true
  gamePanel.resizeHandle = handle
  gamePanel.startX = event.clientX
  gamePanel.startY = event.clientY
}

// 解锁游戏面板
const unlockGamePanel = () => {
  gamePanel.isLocked = false
}

// 注册全局事件
onMounted(() => {
  resetInactivityTimer()

  window.addEventListener('mousemove', (event) => {
    onMenuDrag(event)
    onGamePanelDrag(event)
  })

  window.addEventListener('mouseup', () => {
    endMenuDrag()
    endGamePanelDrag()
  })

  // 添加ESC键监听
  window.addEventListener('keydown', handleKeyDown)
})

// 切换全屏模式
const toggleFullscreen = () => {
  isFullscreen.value = !isFullscreen.value

  if (isFullscreen.value) {
    // 进入全屏模式时，可以尝试调用浏览器的全屏API
    const docElm = document.documentElement
    if (docElm.requestFullscreen) {
      docElm.requestFullscreen()
    } else if (docElm.mozRequestFullScreen) { // Firefox
      docElm.mozRequestFullScreen()
    } else if (docElm.webkitRequestFullscreen) { // Chrome, Safari
      docElm.webkitRequestFullscreen()
    } else if (docElm.msRequestFullscreen) { // IE/Edge
      docElm.msRequestFullscreen()
    }
  } else {
    // 退出全屏模式
    if (document.exitFullscreen) {
      document.exitFullscreen()
    } else if (document.mozCancelFullScreen) { // Firefox
      document.mozCancelFullScreen()
    } else if (document.webkitExitFullscreen) { // Chrome, Safari
      document.webkitExitFullscreen()
    } else if (document.msExitFullscreen) { // IE/Edge
      document.msExitFullscreen()
    }
  }
}

// 监听ESC键退出全屏
const handleKeyDown = (event) => {
  if (event.key === 'Escape' && isFullscreen.value) {
    isFullscreen.value = false
  }
}

onUnmounted(() => {
  if (inactivityTimer) clearTimeout(inactivityTimer)

  window.removeEventListener('mousemove', onMenuDrag)
  window.removeEventListener('mousemove', onGamePanelDrag)
  window.removeEventListener('mouseup', endMenuDrag)
  window.removeEventListener('mouseup', endGamePanelDrag)
  window.removeEventListener('keydown', handleKeyDown)
})
</script>

<template>
  <div class="game-container" :class="{ 'all-hidden': !allElementsVisible, 'fullscreen-mode': isFullscreen }">
    <!-- 一键隐藏/显示按钮 -->
    <div class="global-toggle" @click="allElementsVisible = !allElementsVisible">
      {{ allElementsVisible ? '隐藏' : '显示' }}
    </div>

    <!-- 沉浸模式按钮 -->
    <div class="immersive-toggle" @click="toggleFullscreen">
      {{ isFullscreen ? '退出全屏' : '沉浸模式' }}
    </div>
    <!-- 菜单拖拽区域 -->
    <div class="menu-toggle" @click="toggleMenu" @mouseover="resetInactivityTimer">
      {{ menuState === 'expanded' ? '◀' : (menuState === 'collapsed' ? '◀' : '▶') }}
    </div>
    <div class="game-selector" :class="{
      'collapsed': menuState === 'collapsed',
      'hidden': menuState === 'hidden',
      'edge-left': menuPosition.edge === 'left',
      'edge-right': menuPosition.edge === 'right',
      'edge-top': menuPosition.edge === 'top',
      'edge-bottom': menuPosition.edge === 'bottom'
    }" :style="menuStyle" @mouseover="resetInactivityTimer" @mousedown.prevent="startMenuDrag">
      <div class="menu-drag-handle">拖动</div>
      <button :class="{ active: currentGame === 'snake' }" @click="currentGame = 'snake'">
        🐍 <span class="game-text">贪吃蛇</span>
      </button>
      <button :class="{ active: currentGame === 'slot' }" @click="currentGame = 'slot'">
        🎰 <span class="game-text">老虎机</span>
      </button>
      <button :class="{ active: currentGame === 'tetris' }" @click="currentGame = 'tetris'">
        🟦 <span class="game-text">俄罗斯方块</span>
      </button>
      <button :class="{ active: currentGame === 'flappy' }" @click="currentGame = 'flappy'">
        🐦 <span class="game-text">Flappy Bird</span>
      </button>
      <button :class="{ active: currentGame === 'bct' }" @click="currentGame = 'bct'">
        🎮 <span class="game-text">棒子鸡子老虎</span>
      </button>
      <button :class="{ active: currentGame === 'gourd' }" @click="currentGame = 'gourd'">
        🎯 <span class="game-text">葫芦娃</span>
      </button>
      <button :class="{ active: currentGame === 'gobang' }" @click="currentGame = 'gobang'">
        ⭕ <span class="game-text">五子棋</span>
      </button>
    </div>

    <!-- 游戏面板区域 -->
    <div class="game-panel" :style="gamePanelStyle" @mousedown="startGamePanelLongPress" @dblclick="unlockGamePanel">
      <component :is="currentGame === 'snake' ? SnakeGame :
        (currentGame === 'slot' ? SlotMachine :
          (currentGame === 'tetris' ? Tetris :
            (currentGame === 'flappy' ? FlappyBird :
              (currentGame === 'bct' ? BoneChickenTiger :
                (currentGame === 'gourd' ? GourdDoll : Gobang)))))" />

      <!-- 调整大小的手柄，仅在锁定状态显示 -->
      <template v-if="gamePanel.isLocked">
        <!-- 四个角的调整手柄 -->
        <div class="resize-handle nw" @mousedown.stop="startGamePanelResize($event, 'nw')"></div>
        <div class="resize-handle ne" @mousedown.stop="startGamePanelResize($event, 'ne')"></div>
        <div class="resize-handle sw" @mousedown.stop="startGamePanelResize($event, 'sw')"></div>
        <div class="resize-handle se" @mousedown.stop="startGamePanelResize($event, 'se')"></div>

        <!-- 四条边的调整手柄 -->
        <div class="resize-handle n" @mousedown.stop="startGamePanelResize($event, 'n')"></div>
        <div class="resize-handle e" @mousedown.stop="startGamePanelResize($event, 'e')"></div>
        <div class="resize-handle s" @mousedown.stop="startGamePanelResize($event, 's')"></div>
        <div class="resize-handle w" @mousedown.stop="startGamePanelResize($event, 'w')"></div>
      </template>
    </div>
  </div>
</template>

<style>
/* 全局隐藏样式 */
.all-hidden .game-selector,
.all-hidden .menu-toggle,
.all-hidden .resize-handle,
.all-hidden .immersive-toggle,
.all-hidden .game-panel {
  display: none !important;
}

/* 全屏模式样式 */
.fullscreen-mode .game-panel {
  position: fixed !important;
  top: 0 !important;
  left: 0 !important;
  width: 100vw !important;
  height: 100vh !important;
  z-index: 1000 !important;
  transform: none !important;
  border-radius: 0 !important;
}

/* 一键隐藏/显示按钮样式 */
.global-toggle {
  position: fixed;
  top: 10px;
  right: 10px;
  background: rgba(76, 175, 80, 0.8);
  color: white;
  border: none;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  z-index: 100;
  font-size: 12px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
}

.global-toggle:hover {
  background: rgba(76, 175, 80, 1);
  transform: scale(1.1);
}

/* 沉浸模式按钮样式 */
.immersive-toggle {
  position: fixed;
  top: 10px;
  right: 60px;
  background: rgba(33, 150, 243, 0.8);
  color: white;
  border: none;
  border-radius: 4px;
  padding: 8px 12px;
  display: flex;
  justify-content: center;
  align-items: center;
  cursor: pointer;
  z-index: 100;
  font-size: 12px;
  box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
}

.immersive-toggle:hover {
  background: rgba(33, 150, 243, 1);
  transform: scale(1.05);
}

#app {
  font-family: 'Helvetica Neue', Helvetica, 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  margin: 0;
  padding: 20px;
  box-sizing: border-box;
  width: 100%;
  max-width: 100%;
  height: 100vh;
  overflow: hidden;
}

.game-container {
  width: 100%;
  height: 100vh;
  padding: 0;
  margin: 0;
  display: flex;
  position: relative;
  overflow: hidden;
  /* background-color: aqua; */
  box-sizing: border-box;
}

.menu-toggle {
  position: absolute;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 0 4px 4px 0;
  padding: 10px;
  cursor: pointer;
  z-index: 10;
  transition: all 0.3s ease;
}

.game-selector {
  display: flex;
  flex-direction: column;
  gap: 10px;
  position: absolute;
  background: white;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.2);
  transition: all 0.3s ease;
  z-index: 5;
  width: 200px;
  user-select: none;
}

/* 菜单拖动手柄 */
.menu-drag-handle {
  background: #4CAF50;
  color: white;
  text-align: center;
  padding: 5px;
  margin-bottom: 10px;
  border-radius: 4px;
  cursor: move;
  font-size: 12px;
}

/* 边缘吸附样式 */
.game-selector.edge-left {
  border-radius: 0 8px 8px 0;
  left: 0;
}

.game-selector.edge-right {
  border-radius: 8px 0 0 8px;
  right: 0;
}

.game-selector.edge-top {
  border-radius: 0 0 8px 8px;
  top: 0;
  flex-direction: row;
  width: auto;
  height: auto;
}

.game-selector.edge-bottom {
  border-radius: 8px 8px 0 0;
  bottom: 0;
  flex-direction: row;
  width: auto;
  height: auto;
}

.game-selector.edge-top .menu-drag-handle,
.game-selector.edge-bottom .menu-drag-handle {
  writing-mode: vertical-rl;
  margin-bottom: 0;
  margin-right: 10px;
  height: auto;
}

.game-selector.collapsed {
  width: 60px;
}

.game-selector.hidden {
  transform: translate(-100%, 0);
}

.game-selector.edge-right.hidden {
  transform: translate(100%, 0);
}

.game-selector.edge-top.hidden {
  transform: translate(0, -100%);
}

.game-selector.edge-bottom.hidden {
  transform: translate(0, 100%);
}

.game-selector.collapsed .game-text {
  display: none;
}

.game-selector button {
  padding: 10px 15px;
  border: none;
  border-radius: 4px;
  background: #4CAF50;
  color: white;
  cursor: pointer;
  font-size: 1.1em;
  transition: all 0.3s ease;
  text-align: left;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.game-selector button:hover {
  background: #45a049;
}

.game-selector button.active {
  background: #2c3e50;
}

.game-text {
  margin-left: 8px;
}

/* 游戏面板样式 */
.game-panel {
  background: rgb(66, 63, 63);
  border-radius: 8px;
  box-shadow: 0 0 15px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  margin: 0 auto;
}

/* 调整大小的手柄 */
.resize-handle {
  position: absolute;
  width: 10px;
  height: 10px;
  background: #4CAF50;
  border: 2px solid white;
  border-radius: 50%;
  z-index: 6;
}

.resize-handle.nw {
  top: -5px;
  left: -5px;
  cursor: nw-resize;
}

.resize-handle.ne {
  top: -5px;
  right: -5px;
  cursor: ne-resize;
}

.resize-handle.sw {
  bottom: -5px;
  left: -5px;
  cursor: sw-resize;
}

.resize-handle.se {
  bottom: -5px;
  right: -5px;
  cursor: se-resize;
}

/* 四条边的调整手柄样式 */
.resize-handle.n {
  top: -5px;
  left: 50%;
  transform: translateX(-50%);
  width: 30px;
  cursor: n-resize;
}

.resize-handle.s {
  bottom: -5px;
  left: 50%;
  transform: translateX(-50%);
  width: 30px;
  cursor: s-resize;
}

.resize-handle.e {
  right: -5px;
  top: 50%;
  transform: translateY(-50%);
  height: 30px;
  cursor: e-resize;
}

.resize-handle.w {
  left: -5px;
  top: 50%;
  transform: translateY(-50%);
  height: 30px;
  cursor: w-resize;
}
</style>
