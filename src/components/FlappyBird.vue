<template>
  <div class="game-container">
    <canvas ref="gameCanvas" :width="canvasWidth" :height="canvasHeight"></canvas>
    <div class="game-overlay" v-if="!gameStarted || gameOver || isPreparing">
      <h1>Flappy Bird</h1>
      <p v-if="gameOver">游戏结束! 得分: {{ score }}</p>
      <p v-if="isPreparing" class="countdown">{{ countdown }}</p>
      <button @click="startGame" v-if="!isPreparing">{{ gameOver ? '重新开始' : '开始游戏' }}</button>
    </div>
    <div class="score" v-if="gameStarted && !gameOver">{{ score }}</div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';

// 游戏配置
const canvasWidth = 288;
const canvasHeight = 512;
const gravity = 0.25;
const jumpForce = -4.6;
const pipeGap = 85;
const pipeWidth = 52;
const birdSize = 24;
const baseSpeed = 1.5; // 基础移动速度
const speedIncrement = 0.1; // 每得10分增加的速度
const countdownTime = 3; // 准备时间（秒）

// 游戏状态
const gameCanvas = ref(null);
const ctx = ref(null);
const gameStarted = ref(false);
const gameOver = ref(false);
const score = ref(0);
const isPreparing = ref(false);
const countdown = ref(countdownTime);
const currentSpeed = ref(baseSpeed);

// 游戏对象
const bird = ref({
  x: canvasWidth / 3,
  y: canvasHeight / 2,
  velocity: 0
});

const pipes = ref([]);
let animationFrame = null;

// 加载图片资源
const sprites = {
  background: new Image(),
  bird: new Image(),
  pipe: new Image()
};

sprites.background.src = '/sprites/background.png';
sprites.bird.src = '/sprites/bird.png';
sprites.pipe.src = '/sprites/pipe.png';

// 初始化游戏
const initGame = () => {
  bird.value = {
    x: canvasWidth / 3,
    y: canvasHeight / 2,
    velocity: 0
  };
  pipes.value = [];
  score.value = 0;
  gameOver.value = false;
  currentSpeed.value = baseSpeed;
  startPreparePhase();
};

// 准备阶段
const startPreparePhase = () => {
  isPreparing.value = true;
  countdown.value = countdownTime;
  const countdownInterval = setInterval(() => {
    countdown.value--;
    if (countdown.value <= 0) {
      clearInterval(countdownInterval);
      isPreparing.value = false;
      gameStarted.value = true;
      gameLoop();
    }
  }, 1000);
};

// 开始游戏
const startGame = () => {
  initGame();
};

// 游戏主循环
const gameLoop = () => {
  if (!gameStarted.value || gameOver.value || isPreparing.value) return;

  // 更新小鸟位置
  bird.value.velocity += gravity;
  bird.value.y += bird.value.velocity;

  // 生成管道
  if (pipes.value.length === 0 || pipes.value[pipes.value.length - 1].x < canvasWidth - 150) {
    const pipeY = Math.random() * (canvasHeight - pipeGap - 100) + 50;
    pipes.value.push({
      x: canvasWidth,
      y: pipeY
    });
  }

  // 更新管道位置和速度
  pipes.value.forEach(pipe => {
    pipe.x -= currentSpeed.value;
  });

  // 移除超出屏幕的管道
  pipes.value = pipes.value.filter(pipe => pipe.x > -pipeWidth);

  // 碰撞检测
  if (checkCollision()) {
    gameOver.value = true;
    return;
  }

  // 计算得分和更新速度
  pipes.value.forEach(pipe => {
    if (pipe.x + pipeWidth < bird.value.x && !pipe.passed) {
      pipe.passed = true;
      score.value++;
      // 每得10分增加速度
      if (score.value % 10 === 0) {
        currentSpeed.value += speedIncrement;
      }
    }
  });

  // 绘制游戏画面
  draw();

  animationFrame = requestAnimationFrame(gameLoop);
};

// 碰撞检测
const checkCollision = () => {
  // 检查是否撞到地面或天花板
  if (bird.value.y < 0 || bird.value.y + birdSize > canvasHeight) {
    return true;
  }

  // 检查是否撞到管道
  return pipes.value.some(pipe => {
    return (
      bird.value.x + birdSize > pipe.x &&
      bird.value.x < pipe.x + pipeWidth &&
      (bird.value.y < pipe.y || bird.value.y + birdSize > pipe.y + pipeGap)
    );
  });
};

// 绘制游戏画面
const draw = () => {
  if (!ctx.value) return;

  // 绘制背景
  ctx.value.drawImage(sprites.background, 0, 0, canvasWidth, canvasHeight);

  // 绘制管道
  pipes.value.forEach(pipe => {
    // 绘制上管道
    ctx.value.drawImage(
      sprites.pipe,
      pipe.x,
      pipe.y - 320,
      pipeWidth,
      320
    );

    // 绘制下管道
    ctx.value.drawImage(
      sprites.pipe,
      pipe.x,
      pipe.y + pipeGap,
      pipeWidth,
      320
    );
  });

  // 绘制小鸟
  ctx.value.drawImage(
    sprites.bird,
    bird.value.x,
    bird.value.y,
    birdSize,
    birdSize
  );
};

// 处理点击/按键事件
const handleJump = () => {
  if (gameStarted.value && !gameOver.value) {
    bird.value.velocity = jumpForce;
  }
};

// 生命周期钩子
onMounted(() => {
  ctx.value = gameCanvas.value.getContext('2d');
  window.addEventListener('click', handleJump);
  window.addEventListener('keydown', (e) => {
    if (e.code === 'Space') handleJump();
  });
});

onUnmounted(() => {
  window.removeEventListener('click', handleJump);
  window.removeEventListener('keydown', handleJump);
  if (animationFrame) {
    cancelAnimationFrame(animationFrame);
  }
});
</script>

<style scoped>
.game-container {
  position: relative;
  width: 288px;
  height: 512px;
  margin: 0 auto;
}

canvas {
  border: 2px solid #000;
}

.game-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background: rgba(0, 0, 0, 0.5);
  color: white;
}

.game-overlay h1 {
  font-size: 32px;
  margin-bottom: 20px;
}

.game-overlay button {
  padding: 10px 20px;
  font-size: 18px;
  background: #4CAF50;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.game-overlay button:hover {
  background: #45a049;
}

.score {
  position: absolute;
  top: 20px;
  right: 20px;
  font-size: 24px;
  font-weight: bold;
  color: white;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
}
.countdown {
  font-size: 48px;
  font-weight: bold;
  margin: 20px 0;
}
</style>