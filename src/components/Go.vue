<script setup>
import { ref, onMounted, computed } from 'vue'

// 棋盘大小（标准围棋为19x19）
const BOARD_SIZE = 19

// 初始化棋盘数据
const board = ref(Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(0)))
const currentPlayer = ref(1) // 1代表黑棋，2代表白棋
const gameOver = ref(false)
const winner = ref(0)
const lastMove = ref(null)
const capturedStones = ref({ 1: 0, 2: 0 }) // 黑方和白方各自提子数量
const koPoint = ref(null) // 打劫禁入点
const pass = ref(false) // 是否有玩家选择跳过
const consecutivePasses = ref(0) // 连续跳过次数
const territory = ref({ 1: 0, 2: 0 }) // 黑方和白方的领地
const gamePhase = ref('playing') // playing, scoring, finished

// 计算当前玩家显示文本
const currentPlayerText = computed(() => {
  return currentPlayer.value === 1 ? '黑方' : '白方'
})

// 落子函数
function placePiece (row, col) {
  // 如果游戏已结束或者正在计算得分，不允许落子
  if (gameOver.value || gamePhase.value === 'scoring') return

  // 如果位置已有棋子，不允许落子
  if (board.value[row][col] !== 0) return

  // 检查是否是打劫禁入点
  if (koPoint.value && koPoint.value.row === row && koPoint.value.col === col) {
    alert('此处为打劫禁入点，不能落子')
    return
  }

  // 临时落子以检查是否合法
  board.value[row][col] = currentPlayer.value

  // 检查落子后的气
  const group = findGroup(row, col)
  const liberties = countLiberties(group)

  // 提取对方的棋子
  const capturedGroups = []
  const opponent = currentPlayer.value === 1 ? 2 : 1

  // 检查四周的对方棋子是否被提走
  const directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]
  let capturedCount = 0
  let potentialKoPoint = null

  for (const [dx, dy] of directions) {
    const newRow = row + dx
    const newCol = col + dy

    if (newRow >= 0 && newRow < BOARD_SIZE && newCol >= 0 && newCol < BOARD_SIZE &&
      board.value[newRow][newCol] === opponent) {
      const adjacentGroup = findGroup(newRow, newCol)
      const adjacentLiberties = countLiberties(adjacentGroup)

      if (adjacentLiberties === 0) {
        capturedGroups.push(adjacentGroup)
        capturedCount += adjacentGroup.length

        // 如果只提了一个子，记录可能的打劫点
        if (adjacentGroup.length === 1) {
          potentialKoPoint = { row: newRow, col: newCol }
        }
      }
    }
  }

  // 提子
  for (const group of capturedGroups) {
    for (const { row: r, col: c } of group) {
      board.value[r][c] = 0
    }
  }

  // 如果自己没气了，并且没有提子，则为自杀，不允许落子
  if (liberties === 0 && capturedCount === 0) {
    board.value[row][col] = 0 // 恢复棋盘
    alert('此处落子为自杀，不允许落子')
    return
  }

  // 更新打劫点
  koPoint.value = capturedCount === 1 && potentialKoPoint && liberties > 0 ? potentialKoPoint : null

  // 更新提子数量
  capturedStones.value[currentPlayer.value] += capturedCount

  // 记录最后一步
  lastMove.value = { row, col }

  // 重置连续跳过次数
  consecutivePasses.value = 0
  pass.value = false

  // 切换玩家
  currentPlayer.value = opponent
}

// 查找一个棋子所在的整个连通区域（棋串）
function findGroup (row, col) {
  const color = board.value[row][col]
  if (color === 0) return []

  const visited = Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(false))
  const group = []
  const queue = [{ row, col }]
  visited[row][col] = true

  while (queue.length > 0) {
    const { row: r, col: c } = queue.shift()
    group.push({ row: r, col: c })

    const directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]
    for (const [dx, dy] of directions) {
      const newRow = r + dx
      const newCol = c + dy

      if (newRow >= 0 && newRow < BOARD_SIZE && newCol >= 0 && newCol < BOARD_SIZE &&
        !visited[newRow][newCol] && board.value[newRow][newCol] === color) {
        queue.push({ row: newRow, col: newCol })
        visited[newRow][newCol] = true
      }
    }
  }

  return group
}

// 计算一个棋串的气数
function countLiberties (group) {
  if (group.length === 0) return 0

  const libertySet = new Set()
  const directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]

  for (const { row, col } of group) {
    for (const [dx, dy] of directions) {
      const newRow = row + dx
      const newCol = col + dy

      if (newRow >= 0 && newRow < BOARD_SIZE && newCol >= 0 && newCol < BOARD_SIZE &&
        board.value[newRow][newCol] === 0) {
        libertySet.add(`${newRow},${newCol}`)
      }
    }
  }

  return libertySet.size
}

// 跳过当前回合
function passTurn () {
  pass.value = true
  consecutivePasses.value++

  // 如果连续两次跳过，进入计分阶段
  if (consecutivePasses.value >= 2) {
    gamePhase.value = 'scoring'
    calculateTerritory()
  } else {
    // 切换玩家
    currentPlayer.value = currentPlayer.value === 1 ? 2 : 1
  }
}

// 计算领地
function calculateTerritory () {
  // 创建一个临时棋盘来标记领地
  const territoryBoard = Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(0))
  const visited = Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(false))

  // 重置领地计数
  territory.value = { 1: 0, 2: 0 }

  // 遍历棋盘上的每个空点
  for (let row = 0; row < BOARD_SIZE; row++) {
    for (let col = 0; col < BOARD_SIZE; col++) {
      if (board.value[row][col] === 0 && !visited[row][col]) {
        // 找到一个空区域
        const emptyRegion = []
        const queue = [{ row, col }]
        visited[row][col] = true
        let surroundingBlack = 0
        let surroundingWhite = 0
        let isSurrounded = true

        // 广度优先搜索找到整个空区域
        while (queue.length > 0) {
          const { row: r, col: c } = queue.shift()
          emptyRegion.push({ row: r, col: c })

          // 检查四周
          const directions = [[0, 1], [1, 0], [0, -1], [-1, 0]]
          for (const [dx, dy] of directions) {
            const newRow = r + dx
            const newCol = c + dy

            if (newRow >= 0 && newRow < BOARD_SIZE && newCol >= 0 && newCol < BOARD_SIZE) {
              if (board.value[newRow][newCol] === 0 && !visited[newRow][newCol]) {
                // 继续搜索空区域
                queue.push({ row: newRow, col: newCol })
                visited[newRow][newCol] = true
              } else if (board.value[newRow][newCol] === 1) {
                // 黑棋
                surroundingBlack++
              } else if (board.value[newRow][newCol] === 2) {
                // 白棋
                surroundingWhite++
              }
            } else {
              // 如果到达棋盘边缘，则不是完全被围住的
              isSurrounded = false
            }
          }
        }

        // 判断这个空区域属于谁
        if (isSurrounded) {
          let owner = 0
          if (surroundingBlack > 0 && surroundingWhite === 0) {
            owner = 1 // 黑方领地
            territory.value[1] += emptyRegion.length
          } else if (surroundingWhite > 0 && surroundingBlack === 0) {
            owner = 2 // 白方领地
            territory.value[2] += emptyRegion.length
          }

          // 标记领地
          for (const { row: r, col: c } of emptyRegion) {
            territoryBoard[r][c] = owner
          }
        }
      }
    }
  }

  // 计算最终得分（领地 + 提子数）
  const blackScore = territory.value[1] + capturedStones.value[1]
  const whiteScore = territory.value[2] + capturedStones.value[2] + 6.5 // 白棋贴目6.5

  // 判断胜负
  if (blackScore > whiteScore) {
    winner.value = 1
  } else {
    winner.value = 2
  }

  gameOver.value = true
  gamePhase.value = 'finished'
}

// 重新开始游戏
function restartGame () {
  board.value = Array(BOARD_SIZE).fill().map(() => Array(BOARD_SIZE).fill(0))
  currentPlayer.value = 1
  gameOver.value = false
  winner.value = 0
  lastMove.value = null
  capturedStones.value = { 1: 0, 2: 0 }
  koPoint.value = null
  pass.value = false
  consecutivePasses.value = 0
  territory.value = { 1: 0, 2: 0 }
  gamePhase.value = 'playing'
}

// 悔棋功能（简化版，实际围棋中悔棋需要更复杂的处理）
function undoMove () {
  alert('围棋中通常不支持悔棋功能')
}

// 显示棋盘坐标
function getCoordinateLabel (index) {
  // 围棋坐标通常跳过字母I
  const letters = 'ABCDEFGHJKLMNOPQRST'
  return letters[index]
}
</script>

<template>
  <div class="go-container">
    <div class="game-info">
      <h2>围棋</h2>
      <div class="player-info" :class="{ 'game-over': gameOver }">
        <template v-if="gamePhase === 'playing'">
          当前玩家: <span :class="currentPlayer === 1 ? 'black' : 'white'">{{ currentPlayerText }}</span>
        </template>
        <template v-else-if="gamePhase === 'scoring'">
          计算得分中...
        </template>
        <template v-else>
          游戏结束！获胜者: <span :class="winner === 1 ? 'black' : 'white'">{{ winner === 1 ? '黑方' : '白方' }}</span>
          <div class="score-info">
            黑方: {{ territory[1] + capturedStones[1] }} 分 (领地: {{ territory[1] }}, 提子: {{ capturedStones[1] }})<br>
            白方: {{ territory[2] + capturedStones[2] + 6.5 }} 分 (领地: {{ territory[2] }}, 提子: {{ capturedStones[2] }}, 贴目:
            6.5)
          </div>
        </template>
      </div>

      <div class="captured-info">
        <div>黑方提子: {{ capturedStones[1] }}</div>
        <div>白方提子: {{ capturedStones[2] }}</div>
      </div>

      <div class="controls">
        <button @click="restartGame">重新开始</button>
        <button @click="passTurn" :disabled="gameOver || gamePhase !== 'playing'">跳过</button>
      </div>
    </div>

    <div class="board">
      <!-- 坐标标签 - 顶部 -->
      <div class="coordinate-row">
        <div class="coordinate-cell"></div>
        <div v-for="(_, j) in board[0]" :key="'top-' + j" class="coordinate-cell">
          {{ getCoordinateLabel(j) }}
        </div>
      </div>

      <div v-for="(row, i) in board" :key="i" class="board-row">
        <!-- 坐标标签 - 左侧 -->
        <div class="coordinate-cell">{{ BOARD_SIZE - i }}</div>

        <div v-for="(cell, j) in row" :key="j" class="cell" @click="placePiece(i, j)">
          <div v-if="cell !== 0" class="piece" :class="{
            'black': cell === 1,
            'white': cell === 2,
            'last-move': lastMove && lastMove.row === i && lastMove.col === j
          }"></div>

          <!-- 星位标记 -->
          <div v-if="cell === 0 && [
            [3, 3], [3, 9], [3, 15],
            [9, 3], [9, 9], [9, 15],
            [15, 3], [15, 9], [15, 15]
          ].some(([r, c]) => r === i && c === j)" class="star-point"></div>
        </div>

        <!-- 坐标标签 - 右侧 -->
        <div class="coordinate-cell">{{ BOARD_SIZE - i }}</div>
      </div>

      <!-- 坐标标签 - 底部 -->
      <div class="coordinate-row">
        <div class="coordinate-cell"></div>
        <div v-for="(_, j) in board[0]" :key="'bottom-' + j" class="coordinate-cell">
          {{ getCoordinateLabel(j) }}
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.go-container {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

.game-info {
  text-align: center;
  margin-bottom: 20px;
  width: 100%;
  max-width: 600px;
}

.player-info {
  font-size: 1.2em;
  margin: 10px 0;
}

.player-info.game-over {
  color: #e74c3c;
  font-weight: bold;
}

.score-info {
  margin-top: 10px;
  font-size: 0.9em;
}

.captured-info {
  display: flex;
  justify-content: space-around;
  margin: 10px 0;
  font-size: 1.1em;
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
  background: #e6bf83;
  padding: 10px;
  border-radius: 5px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.coordinate-row {
  display: flex;
  height: 20px;
  margin: 2px 0;
}

.coordinate-cell {
  width: 20px;
  height: 20px;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 12px;
  color: #333;
}

.board-row {
  display: flex;
}

.cell {
  width: 30px;
  height: 30px;
  position: relative;
  cursor: pointer;
}

/* 绘制棋盘线 */
.cell::before,
.cell::after {
  content: '';
  position: absolute;
  background: #000;
  z-index: 1;
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

/* 边缘格子只显示一半线条 */
.board-row:first-child .cell::before {
  top: 50%;
  height: 50%;
}

.board-row:last-child .cell::before {
  top: 0;
  height: 50%;
}

.board-row .cell:first-of-type::after {
  left: 50%;
  width: 50%;
}

.board-row .cell:last-of-type::after {
  left: 0;
  width: 50%;
}

.piece {
  width: 26px;
  height: 26px;
  border-radius: 50%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 2;
  transition: all 0.3s ease;
}

.piece.black {
  background: radial-gradient(circle at 35% 35%, #444, #000);
  box-shadow: 1px 1px 3px rgba(0, 0, 0, 0.5);
}

.piece.white {
  background: radial-gradient(circle at 35% 35%, #fff, #eee);
  box-shadow: 1px 1px 3px rgba(0, 0, 0, 0.3);
}

.piece.last-move {
  box-shadow: 0 0 8px 4px #f39c12;
}

.star-point {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #000;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 1;
}

.black {
  color: #000;
}

.white {
  color: #666;
}
</style>