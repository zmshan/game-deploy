<template>
  <div class="slot-machine">
    <div class="game-container">
      <div class="reels-container">
        <div v-for="(reel, index) in reels" :key="index" class="reel" :class="{ spinning: spinning[index] }">
          <div class="reel-items" :style="{ transform: `translateY(${reelPositions[index]}px)` }">
            <div v-for="(symbol, sIndex) in symbols" :key="sIndex" class="symbol">
              {{ symbol }}
            </div>
          </div>
        </div>
      </div>

      <div class="controls">
        <div class="balance">余额: {{ balance }} 币</div>
        <div class="bet-controls">
          <button @click="decreaseBet" :disabled="spinning.some(s => s)">-</button>
          <span>下注: {{ currentBet }} 币</span>
          <button @click="increaseBet" :disabled="spinning.some(s => s)">+</button>
        </div>
        <button class="spin-button" @click="spin" :disabled="spinning.some(s => s) || balance < currentBet">
          开始
        </button>
        <button class="auto-play" @click="toggleAutoPlay" :class="{ active: autoPlay }">
          {{ autoPlay ? '停止自动' : '自动游戏' }}
        </button>
      </div>

      <div class="win-message" v-if="winAmount > 0">
        恭喜获得 {{ winAmount }} 币!
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 游戏配置
const symbols = ['🍎', '🍋', '🍇', '🍒', '💎', '7️⃣'];
const REEL_COUNT = 3;
const SYMBOL_HEIGHT = 60;
const SPIN_DURATION = 2000;

// 游戏状态
const balance = ref(1000);
const currentBet = ref(10);
const reels = ref(Array(REEL_COUNT).fill([]));
const reelPositions = ref(Array(REEL_COUNT).fill(0));
const spinning = ref(Array(REEL_COUNT).fill(false));
const winAmount = ref(0);
const autoPlay = ref(false);
let autoPlayInterval = null;

// 投注控制
const increaseBet = () => {
  if (currentBet.value < 100) {
    currentBet.value += 10;
  }
};

const decreaseBet = () => {
  if (currentBet.value > 10) {
    currentBet.value -= 10;
  }
};

// 自动游戏控制
const toggleAutoPlay = () => {
  autoPlay.value = !autoPlay.value;
  if (autoPlay.value) {
    autoPlayInterval = setInterval(() => {
      if (balance.value >= currentBet.value && !spinning.value.some(s => s)) {
        spin();
      } else {
        toggleAutoPlay();
      }
    }, SPIN_DURATION + 1000);
  } else {
    clearInterval(autoPlayInterval);
  }
};

// 开始游戏
const spin = async () => {
  if (balance.value < currentBet.value) return;

  balance.value -= currentBet.value;
  winAmount.value = 0;

  // 设置每个转轮的旋转
  for (let i = 0; i < REEL_COUNT; i++) {
    spinning.value[i] = true;
    const finalPosition = -Math.floor(Math.random() * symbols.length) * SYMBOL_HEIGHT;

    // 添加延迟使转轮依次停止
    setTimeout(() => {
      reelPositions.value[i] = finalPosition;
      spinning.value[i] = false;

      // 最后一个转轮停止后检查结果
      if (i === REEL_COUNT - 1) {
        checkWin();
      }
    }, SPIN_DURATION + (i * 500));
  }
};

// 检查是否中奖
const checkWin = () => {
  const results = reelPositions.value.map(pos => {
    const index = Math.abs(Math.round(pos / SYMBOL_HEIGHT)) % symbols.length;
    return symbols[index];
  });

  // 计算奖励
  if (results.every(symbol => symbol === results[0])) {
    const multiplier = getMultiplier(results[0]);
    winAmount.value = currentBet.value * multiplier;
    balance.value += winAmount.value;
  }
};

// 获取不同符号的倍数
const getMultiplier = (symbol) => {
  const multipliers = {
    '💎': 10,
    '7️⃣': 7,
    '🍒': 5,
    '🍇': 4,
    '🍋': 3,
    '🍎': 2
  };
  return multipliers[symbol] || 1;
};

// 组件卸载时清理自动游戏定时器
onUnmounted(() => {
  if (autoPlayInterval) {
    clearInterval(autoPlayInterval);
  }
});
</script>

<style scoped>
.slot-machine {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 20px;
  background: #2c3e50;
  border-radius: 10px;
  color: white;
}

.game-container {
  background: #34495e;
  padding: 20px;
  border-radius: 8px;
  box-shadow: 0 0 20px rgba(0, 0, 0, 0.3);
}

.reels-container {
  display: flex;
  gap: 10px;
  background: #2c3e50;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
}

.reel {
  width: 80px;
  height: 180px;
  background: #1a1a1a;
  overflow: hidden;
  border-radius: 4px;
}

.reel-items {
  transition: transform 2s cubic-bezier(0.21, 0.53, 0.29, 0.99);
}

.spinning .reel-items {
  transition: none;
  transform: translateY(1000px);
}

.symbol {
  height: 60px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2em;
  background: #2c3e50;
  border: 1px solid #34495e;
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 15px;
  align-items: center;
}

.balance {
  font-size: 1.2em;
  font-weight: bold;
  color: #2ecc71;
}

.bet-controls {
  display: flex;
  align-items: center;
  gap: 10px;
}

button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  background: #3498db;
  color: white;
  cursor: pointer;
  font-size: 1em;
}

button:disabled {
  background: #95a5a6;
  cursor: not-allowed;
}

.spin-button {
  background: #e74c3c;
  padding: 12px 24px;
  font-size: 1.2em;
  font-weight: bold;
}

.auto-play {
  background: #f39c12;
}

.auto-play.active {
  background: #d35400;
}

.win-message {
  margin-top: 20px;
  font-size: 1.5em;
  color: #f1c40f;
  text-align: center;
  animation: pulse 1s infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
  }

  50% {
    transform: scale(1.1);
  }

  100% {
    transform: scale(1);
  }
}
</style>