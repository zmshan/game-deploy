<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import { ElMessage } from 'element-plus'

// 游戏配置
const GRID_SIZE = 4
const PIECES_PER_TYPE = 2
const TOTAL_PIECES = PIECES_PER_TYPE * 4
const gameMode = ref('pvp') // pvp, pve

// 棋子类型
const PIECE_TYPES = {
  BONE: '棒子',
  TIGER: '老虎',
  CHICKEN: '鸡',
  BUG: '虫子'
}

// 相克关系
const COUNTER_RELATIONS = {
  [PIECE_TYPES.TIGER]: PIECE_TYPES.CHICKEN,
  [PIECE_TYPES.CHICKEN]: PIECE_TYPES.BUG,
  [PIECE_TYPES.BUG]: PIECE_TYPES.BONE,
  [PIECE_TYPES.BONE]: PIECE_TYPES.TIGER
}

// 游戏状态
const gameState = ref('ready') // ready, playing, over
const currentPlayer = ref('red') // red, blue
const winner = ref(null)
const isAIThinking = ref(false)

// 棋盘状态
const board = ref([])
const selectedPiece = ref(null)

// 初始化棋盘
const initBoard = () => {
  const pieces = []
  // 为每种类型创建红蓝双方的棋子
  Object.values(PIECE_TYPES).forEach(type => {
    for (let i = 0; i < PIECES_PER_TYPE; i++) {
      pieces.push({ type, team: 'red', revealed: false })
      pieces.push({ type, team: 'blue', revealed: false })
    }
  })

  // 随机打乱棋子
  for (let i = pieces.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
      ;[pieces[i], pieces[j]] = [pieces[j], pieces[i]]
  }

  // 创建4x4棋盘
  board.value = Array.from({ length: GRID_SIZE }, (_, row) =>
    Array.from({ length: GRID_SIZE }, (_, col) => ({
      piece: pieces[row * GRID_SIZE + col],
      position: { row, col }
    }))
  )
}

// 开始新游戏
const startGame = (mode = 'pvp') => {
  gameMode.value = mode
  gameState.value = 'playing'
  currentPlayer.value = 'red'
  winner.value = null
  initBoard()
}

// AI移动逻辑
const makeAIMove = async () => {
  if (currentPlayer.value !== 'blue' || gameMode.value !== 'pve') return

  isAIThinking.value = true
  await new Promise(resolve => setTimeout(resolve, 1000))

  // 获取已知的棋子信息
  const revealedPieces = board.value.flat().filter(cell => cell.piece && cell.piece.revealed)
  const unrevealedCells = board.value.flat().filter(cell => cell.piece && !cell.piece.revealed)
  const bluePieces = revealedPieces.filter(cell => cell.piece.team === 'blue')
  const redPieces = revealedPieces.filter(cell => cell.piece.team === 'red')

  // 策略1：如果场上有可以吃的棋子，优先吃子
  for (const bluePiece of bluePieces) {
    const possibleCaptures = board.value.flat().filter(targetCell => {
      if (!canMoveTo(bluePiece.position, targetCell.position)) return false
      if (!targetCell.piece || !targetCell.piece.revealed) return false
      if (targetCell.piece.team === 'red') {
        return canCapture(bluePiece.piece, targetCell.piece)
      }
      return false
    })

    if (possibleCaptures.length > 0) {
      selectedPiece.value = bluePiece
      handlePieceClick(possibleCaptures[0])
      isAIThinking.value = false
      return
    }
  }

  // 策略2：如果己方棋子可能被吃，尝试移动到安全位置
  for (const bluePiece of bluePieces) {
    const isInDanger = redPieces.some(redPiece =>
      canMoveTo(redPiece.position, bluePiece.position) &&
      canCapture(redPiece.piece, bluePiece.piece)
    )

    if (isInDanger) {
      const safeSpots = board.value.flat().filter(targetCell => {
        if (!canMoveTo(bluePiece.position, targetCell.position)) return false
        if (targetCell.piece) return false
        // 检查移动后是否安全
        return !redPieces.some(redPiece =>
          canMoveTo(redPiece.position, targetCell.position) &&
          canCapture(redPiece.piece, bluePiece.piece)
        )
      })

      if (safeSpots.length > 0) {
        selectedPiece.value = bluePiece
        handlePieceClick(safeSpots[0])
        isAIThinking.value = false
        return
      }
    }
  }

  // 策略3：翻开未知棋子
  if (unrevealedCells.length > 0) {
    // 优先翻开靠近对方棋子的未知棋子
    const bestUnrevealedCell = unrevealedCells.reduce((best, cell) => {
      const distanceToRedPieces = redPieces.reduce((minDist, redPiece) => {
        const dist = Math.abs(redPiece.position.row - cell.position.row) +
          Math.abs(redPiece.position.col - cell.position.col)
        return Math.min(minDist, dist)
      }, Infinity)
      if (!best || distanceToRedPieces < best.distance) {
        return { cell, distance: distanceToRedPieces }
      }
      return best
    }, null)

    if (bestUnrevealedCell) {
      handlePieceClick(bestUnrevealedCell.cell)
      isAIThinking.value = false
      return
    }
  }

  // 策略4：移动到有利位置
  if (bluePieces.length > 0) {
    const randomPiece = bluePieces[Math.floor(Math.random() * bluePieces.length)]
    const possibleMoves = board.value.flat().filter(targetCell => {
      if (!canMoveTo(randomPiece.position, targetCell.position)) return false
      return !targetCell.piece
    })

    if (possibleMoves.length > 0) {
      selectedPiece.value = randomPiece
      handlePieceClick(possibleMoves[0])
    }
  }

  isAIThinking.value = false
}

// 检查是否可以移动到目标位置
const canMoveTo = (fromPos, toPos, toCell) => {
  const rowDiff = Math.abs(fromPos.row - toPos.row)
  const colDiff = Math.abs(fromPos.col - toPos.col)

  // 中间四格允许斜向移动
  const isCenterCell = (pos) => {
    return pos.row >= 1 && pos.row <= 2 && pos.col >= 1 && pos.col <= 2
  }

  // 如果是中间四格，允许斜向移动
  if (isCenterCell(fromPos) && isCenterCell(toPos)) {
    return rowDiff <= 1 && colDiff <= 1
  }

  // 其他位置只能横向或纵向移动
  return (rowDiff === 1 && colDiff === 0) || (rowDiff === 0 && colDiff === 1)
}

// 检查是否可以吃掉对方棋子
const canCapture = (attacker, defender) => {
  // 相同棋子可以相互抵消
  if (attacker.type === defender.type) return true
  // 相克关系判断
  return COUNTER_RELATIONS[attacker.type] === defender.type
}

// 处理棋子点击
const handlePieceClick = (cell) => {
  if (gameMode.value === 'pve' && currentPlayer.value === 'blue') return
  if (gameState.value !== 'playing') return

  // 如果点击的是未翻开的棋子
  if (cell.piece && !cell.piece.revealed) {
    cell.piece.revealed = true
    currentPlayer.value = currentPlayer.value === 'red' ? 'blue' : 'red'
    checkGameOver()
    if (gameMode.value === 'pve' && currentPlayer.value === 'blue' && gameState.value === 'playing') {
      setTimeout(() => makeAIMove(), 500)
    }
    return
  }

  // 如果是当前玩家的棋子
  if (cell.piece && cell.piece.team === currentPlayer.value) {
    selectedPiece.value = cell
    return
  }

  // 如果已选择棋子，尝试移动或吃子
  if (selectedPiece.value) {
    const fromCell = selectedPiece.value
    const fromPiece = fromCell.piece
    const toPiece = cell.piece

    if (canMoveTo(fromCell.position, cell.position)) {
      // 移动到空格
      if (!toPiece) {
        cell.piece = fromPiece
        fromCell.piece = null
        currentPlayer.value = currentPlayer.value === 'red' ? 'blue' : 'red'
        checkGameOver()
        if (gameMode.value === 'pve' && currentPlayer.value === 'blue' && gameState.value === 'playing') {
          setTimeout(() => makeAIMove(), 500)
        }
      }
      // 吃子或相消
      else if (toPiece.team !== fromPiece.team && canCapture(fromPiece, toPiece)) {
        // 如果是相同棋子，两个都消失
        if (fromPiece.type === toPiece.type) {
          cell.piece = null
          fromCell.piece = null
        } else {
          // 吃子
          cell.piece = fromPiece
          fromCell.piece = null
        }
        currentPlayer.value = currentPlayer.value === 'red' ? 'blue' : 'red'
        checkGameOver()
        if (gameMode.value === 'pve' && currentPlayer.value === 'blue' && gameState.value === 'playing') {
          setTimeout(() => makeAIMove(), 500)
        }
      } else {
        // 错误操作提示
        ElMessage.error('无效的移动！')
        const cellElement = document.querySelector(`.cell[data-row="${cell.position.row}"][data-col="${cell.position.col}"]`)
        if (cellElement) {
          cellElement.classList.add('shake')
          setTimeout(() => cellElement.classList.remove('shake'), 500)
        }
      }
    } else {
      // 错误移动提示
      ElMessage.error('不能移动到该位置！')
      const cellElement = document.querySelector(`.cell[data-row="${cell.position.row}"][data-col="${cell.position.col}"]`)
      if (cellElement) {
        cellElement.classList.add('shake')
        setTimeout(() => cellElement.classList.remove('shake'), 500)
      }
    }
    selectedPiece.value = null
  }
}

// 检查游戏是否结束
const checkGameOver = () => {
  const redPieces = countTeamPieces('red')
  const bluePieces = countTeamPieces('blue')

  if (redPieces === 0) {
    winner.value = 'blue'
    gameState.value = 'over'
  } else if (bluePieces === 0) {
    winner.value = 'red'
    gameState.value = 'over'
  }
}

// 计算某一方剩余棋子数量
const countTeamPieces = (team) => {
  return board.value.flat().reduce((count, cell) => {
    return count + (cell.piece && cell.piece.team === team ? 1 : 0)
  }, 0)
}
</script>

<template>
  <div class="game-container">
    <h1>棒子鸡子老虎虫子</h1>

    <div v-if="gameState === 'ready'" class="start-screen">
      <button @click="startGame('pvp')">双人对战</button>
      <button @click="startGame('pve')">电脑对战</button>
    </div>

    <div v-else class="game-board">
      <div class="status-bar">
        <div>当前回合: {{ currentPlayer === 'red' ? '红方' : (gameMode === 'pve' ? 'AI' : '蓝方') }}</div>
        <div v-if="isAIThinking" class="ai-thinking">AI思考中...</div>
      </div>

      <div class="grid">
        <div v-for="row in board" :key="row[0].position.row" class="row">
          <div v-for="cell in row" :key="`${cell.position.row}-${cell.position.col}`" class="cell"
            :data-row="cell.position.row" :data-col="cell.position.col" :class="{
              'selected': selectedPiece && selectedPiece.position.row === cell.position.row && selectedPiece.position.col === cell.position.col,
              'red': cell.piece && cell.piece.team === 'red' && cell.piece.revealed,
              'blue': cell.piece && cell.piece.team === 'blue' && cell.piece.revealed,
              'unrevealed': cell.piece && !cell.piece.revealed
            }" @click="handlePieceClick(cell)">
            <span v-if="cell.piece && cell.piece.revealed">
              {{ cell.piece.type }}
            </span>
            <span v-else-if="cell.piece">?</span>
          </div>
        </div>
      </div>

      <div v-if="gameState === 'over'" class="game-over">
        <h2>游戏结束</h2>
        <p>{{ winner === 'red' ? '红方' : '蓝方' }}获胜！</p>
        <button @click="startGame">重新开始</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
@keyframes shake {

  0%,
  100% {
    transform: translateX(0);
  }

  25% {
    transform: translateX(-5px);
  }

  75% {
    transform: translateX(5px);
  }
}

.shake {
  animation: shake 0.5s ease-in-out;
}

.game-container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  text-align: center;
}

.status-bar {
  margin-bottom: 20px;
  font-size: 18px;
}

.grid {
  display: inline-grid;
  gap: 4px;
  padding: 10px;
  background: #eee;
  border-radius: 8px;
}

.row {
  display: flex;
  gap: 4px;
}

.cell {
  width: 80px;
  height: 80px;
  background: white;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  cursor: pointer;
  transition: all 0.3s ease;
  position: relative;
}

.cell[data-row="1"][data-col="1"],
.cell[data-row="1"][data-col="2"],
.cell[data-row="2"][data-col="1"],
.cell[data-row="2"][data-col="2"] {
  background: #e8f5e9;
}

.cell.unrevealed {
  background: #ddd;
}

.cell.red {
  background: #ffebee;
  color: #d32f2f;
}

.cell.blue {
  background: #e3f2fd;
  color: #1976d2;
}

.cell.selected {
  border: 2px solid #ffd700;
}

.game-over {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(255, 255, 255, 0.9);
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.start-screen button {
  margin: 0 10px;
}

.ai-thinking {
  color: #1976d2;
  margin-top: 10px;
  font-style: italic;
}

button {
  padding: 10px 20px;
  font-size: 16px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  transition: background 0.3s;
}

button:hover {
  background: #45a049;
}
</style>