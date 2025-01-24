<script setup lang="ts">
import { ref, onUnmounted } from 'vue'

// 音效资源
const chickenSound = new Audio('/src/assets/sounds/鸡叫.mp3')
const chickSound = new Audio('/src/assets/sounds/小鸡叫.mp3')
const cowSound = new Audio('/src/assets/sounds/牛叫.mp3')
const horseSound = new Audio('/src/assets/sounds/马叫声.mp3')
const brownGoatSound = new Audio('/src/assets/sounds/棕色的羊叫.mp3')
const donkeySound = new Audio('/src/assets/sounds/驴叫.mp3')
const sheepSound = new Audio('/src/assets/sounds/绵羊叫.mp3')
const gooseSound = new Audio('/src/assets/sounds/鹅叫.mp3')
const goatSound = new Audio('/src/assets/sounds/山羊叫.mp3')
const failSound = new Audio('/src/assets/sounds/失败.mp3')

// 按钮音效
const buttonClickSound = new Audio('/src/assets/sounds/按钮点击.mp3')
buttonClickSound.volume = 0.5

// 播放按钮音效的函数
const playButtonSound = () => {
  buttonClickSound.currentTime = 0
  buttonClickSound.play()
}

// 图片样式配置

// 游戏状态
const gameStarted = ref(false)
const gameOver = ref(false)
const gameResult = ref<'timeout' | 'full' | 'win' | null>(null)
const timeLeft = ref(600) // 10分钟 = 600秒
const timer = ref<number | null>(null)
const showConfirmDialog = ref(false)
const isPaused = ref(false) // 游戏暂停状态
const currentLevel = ref(1) // 当前关卡
const showLevelTransition = ref(false) // 关卡过渡动画

// 待消除区域（最多6个格子）
const pendingArea = ref<Array<{ id: number; type: string }>>([]);

// 动物类型和图片路径
const animalTypes = ['sheep', 'goose', 'chicken', 'chick', 'cow', 'goat', 'brown_goat', 'donkey', 'horse']
const animalImages = {
  goose: "https://img.z4a.net/images/2025/01/21/ccd1b713c84ed4a0226a9896c11312df.png",
  sheep: "https://img.z4a.net/images/2025/01/21/7914ffb601280ee2b153987a2ec61254.png",
  chicken: "https://img.z4a.net/images/2025/01/21/907056c68627d80ba00ffb8121b613a2.png",
  chick: "https://img.z4a.net/images/2025/01/21/01640dec8c93ce588ef149431f0b6c21.png",
  cow: "https://img.z4a.net/images/2025/01/21/be69d38164e6be70b0475948951cbe05.png",
  goat: "https://img.z4a.net/images/2025/01/21/aa065361fe486a1c7b9679849c5bb00b.png",
  brown_goat: "https://img.z4a.net/images/2025/01/21/c7d7d252eb9c21a4ce900129a07e24bb.png",
  donkey: "https://img.z4a.net/images/2025/01/21/df14fb83b47fd261886b4116c2115c0f.png",
  horse: "https://img.z4a.net/images/2025/01/21/3049d4665f79df527382466721717fa7.png"
}

// 游戏区域的动物
const animals = ref<Array<{ id: number; type: string; x: number; y: number; visible: boolean; zIndex: number; offsetX?: number; offsetY?: number }>>([]);

// 检查动物是否被遮挡
const isAnimalCovered = (targetAnimal: { x: number; y: number; zIndex: number }) => {
  const VISIBLE_THRESHOLD = 0.1; // 当动物被遮挡超过90%面积时判定为不可选中
  let totalOverlapArea = 0;
  const animalSize = 80;
  const animalArea = animalSize * animalSize;

  // 计算目标动物的边界框
  const targetLeft = (targetAnimal.x / 100) * window.innerWidth - animalSize / 2;
  const targetRight = targetLeft + animalSize;
  const targetTop = (targetAnimal.y / 100) * window.innerHeight - animalSize / 2;
  const targetBottom = targetTop + animalSize;

  // 检查所有可能遮挡的动物
  for (const animal of animals.value) {
    if (!animal.visible || animal.zIndex <= targetAnimal.zIndex) continue;

    // 计算遮挡动物的边界框
    const otherLeft = (animal.x / 100) * window.innerWidth - animalSize / 2;
    const otherRight = otherLeft + animalSize;
    const otherTop = (animal.y / 100) * window.innerHeight - animalSize / 2;
    const otherBottom = otherTop + animalSize;

    // 计算重叠区域
    const overlapLeft = Math.max(targetLeft, otherLeft);
    const overlapRight = Math.min(targetRight, otherRight);
    const overlapTop = Math.max(targetTop, otherTop);
    const overlapBottom = Math.min(targetBottom, otherBottom);

    if (overlapLeft < overlapRight && overlapTop < overlapBottom) {
      const overlapArea = (overlapRight - overlapLeft) * (overlapBottom - overlapTop);
      totalOverlapArea += overlapArea;

      // 如果被遮挡面积超过阈值，则认为动物不可选中
      if (totalOverlapArea >= animalArea * (1 - VISIBLE_THRESHOLD)) {
        return true;
      }
    }
  }

  return false; // 如果总遮挡面积小于阈值，则动物可选中
}

// 点击动物
// 计算两个点之间的距离
const calculateDistance = (x1: number, y1: number, x2: number, y2: number) => {
  return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
};

// 处理动物的鼠标悬停效果
const handleAnimalHover = (targetAnimal: { x: number; y: number; id: number }) => {
  const INFLUENCE_RADIUS = 15; // 影响半径（百分比）
  const MAX_OFFSET = 3; // 最大位移（百分比）

  animals.value.forEach(animal => {
    if (animal.id === targetAnimal.id || !animal.visible) return;

    const distance = calculateDistance(targetAnimal.x, targetAnimal.y, animal.x, animal.y);
    if (distance <= INFLUENCE_RADIUS) {
      const angle = Math.atan2(animal.y - targetAnimal.y, animal.x - targetAnimal.x);
      const offset = (1 - distance / INFLUENCE_RADIUS) * MAX_OFFSET;
      
      animal.offsetX = Math.cos(angle) * offset;
      animal.offsetY = Math.sin(angle) * offset;
    } else {
      animal.offsetX = 0;
      animal.offsetY = 0;
    }
  });
};

// 重置所有动物的位移
const resetAnimalOffsets = () => {
  animals.value.forEach(animal => {
    animal.offsetX = 0;
    animal.offsetY = 0;
  });
};

const handleAnimalClick = (animal: {
visible: any; id: number; type: string 
}) => {
  if (!animal.visible || pendingArea.value.length >= 6 || isPaused.value) return;
  
  // 获取目标动物
  const targetAnimal = animals.value.find(a => a.id === animal.id);
  if (!targetAnimal) return;
  
  // 检查是否被遮挡
  if (isAnimalCovered(targetAnimal)) return;
  
  // 获取目标位置（待消除区域的空位）
  const emptySlotIndex = pendingArea.value.length;
  const targetSlot = document.querySelector(`.pending-slot:nth-child(${emptySlotIndex + 1})`);
  const targetRect = targetSlot?.getBoundingClientRect();
  const animalElement = document.querySelector(`[data-animal-id="${animal.id}"]`);
  
  if (targetRect && animalElement) {
    const startRect = animalElement.getBoundingClientRect();
    const startX = startRect.left;
    const startY = startRect.top;
    const endX = targetRect.left;
    const endY = targetRect.top;

    // 设置动画样式
// 类型断言告诉 TypeScript 这是一个 HTMLElement
(animalElement as HTMLElement).style.position = 'fixed';
    (animalElement as HTMLElement).style.left = `${startX}px`;
    (animalElement as HTMLElement).style.top = `${startY}px`;
    (animalElement as HTMLElement).style.zIndex = '9999';
    
    // 使用线性动画
    const startTime = performance.now();
    const duration = 300; // 缩短动画时长到300ms以提升流畅度

    function animate(currentTime: number) {
      const elapsed = currentTime - startTime;
      const progress = Math.min(elapsed / duration, 1);

      // 使用线性动画
      const currentX = startX + (endX - startX) * progress;
      const currentY = startY + (endY - startY) * progress;

      // 添加缩放和旋转效果
      const scale = 1 + 0.2 * (1 - progress); // 开始时较大，逐渐恢复正常大小
      const rotation = progress * 360; // 完整旋转一圈

      const transform = `translate(${currentX - startX}px, ${currentY - startY}px) translate(-50%, -50%) scale(${scale}) rotate(${rotation}deg)`;
      // 确保 animalElement 不为空
      if (animalElement) {
// 使用类型断言确保 TypeScript 知道这是一个 HTMLElement
(animalElement as HTMLElement).style.transform = transform;
      }

      if (progress < 1) {
        requestAnimationFrame(animate);
      } else {
        // 动画结束，重置变换
// 确保 animalElement 不为空再设置样式
if (animalElement) {
  (animalElement as HTMLElement).style.transform = 'translate(-50%, -50%)';
}
      }
    }

    requestAnimationFrame(animate);
    
    // 动画结束后处理
    setTimeout(() => {
      // 恢复原始样式
// 使用类型断言确保 TypeScript 知道这是一个 HTMLElement
(animalElement as HTMLElement).style.position = '';
(animalElement as HTMLElement).style.left = '';
(animalElement as HTMLElement).style.top = '';
// 类型断言告诉 TypeScript 这是一个 HTMLElement
(animalElement as HTMLElement).style.zIndex = '';
// 类型断言告诉 TypeScript 这是一个 HTMLElement
(animalElement as HTMLElement).style.transition = '';
      
      // 添加到待消除区域
      pendingArea.value.push({ id: animal.id, type: animal.type });
      // 隐藏原位置的动物
      targetAnimal.visible = false;
    }, 300);
  } else {
    // 如果获取位置失败，直接添加到待消除区域
    pendingArea.value.push({ id: animal.id, type: animal.type });
    targetAnimal.visible = false;
  }
  
  // 在动画完成后检查是否可以消除和游戏是否结束
  requestAnimationFrame(() => {
    setTimeout(() => {
      // 检查是否可以消除
      checkMatch();
      
      // 检查游戏是否结束
      checkGameOver();
    }, 300); // 与动画时间同步
  });
}

// 检查匹配
const checkMatch = () => {
  // 统计每种动物的数量
  const counts = new Map<string, number>();
  pendingArea.value.forEach(animal => {
    counts.set(animal.type, (counts.get(animal.type) || 0) + 1);
  });

  // 检查每种动物是否达到3个
  for (const [type, count] of counts.entries()) {
    if (count >= 3) {
      // 找出所有匹配的动物
      const matchedAnimals = pendingArea.value.filter(animal => animal.type === type);
      
      // 获取要移除的动物ID
      const animalIdsToRemove = matchedAnimals.slice(0, 3).map(animal => animal.id);
      
      // 创建动画效果
      const animationPromises = animalIdsToRemove.map(id => {
        return new Promise<void>(resolve => {
          const elements = document.querySelectorAll(`img[data-animal-id="${id}"]`);
          elements.forEach(element => {
            if (element instanceof HTMLElement) {
              element.style.transition = 'all 0.3s ease-out';
              element.style.transform = 'scale(1.5) rotate(360deg)';
              element.style.opacity = '0';
            }
          });
          setTimeout(resolve, 500);
        });
      });

      // 等待所有动画完成后再更新状态
      Promise.all(animationPromises).then(() => {
        // 从待消除区域移除已消除的动物
        pendingArea.value = pendingArea.value.filter(animal => !animalIdsToRemove.includes(animal.id));
        
        // 根据动物类型播放对应音效
        if (type === 'chicken') {
          chickenSound.currentTime = 0; // 重置音频播放位置
          chickenSound.playbackRate = 1.5; // 设置播放速度为1.5倍
          chickenSound.play();
        } else if (type === 'cow') {
          cowSound.currentTime = 0; // 重置音频播放位置
          cowSound.playbackRate = 2.0; // 设置播放速度为2.0倍
          cowSound.play();
        } else if (type === 'horse') {
          horseSound.currentTime = 0; // 重置音频播放位置
          horseSound.playbackRate = 2.0; // 设置播放速度为2.0倍
          horseSound.play();
        } else if (type === 'brown_goat') {
          brownGoatSound.currentTime = 0; // 重置音频播放位置
          brownGoatSound.play();
        } else if (type === 'donkey') {
          donkeySound.currentTime = 0; // 重置音频播放位置
          donkeySound.playbackRate = 2.0; // 设置播放速度为2.0倍
          donkeySound.play();
        } else if (type === 'goose') {
          gooseSound.currentTime = 0; // 重置音频播放位置
          gooseSound.playbackRate = 1.5; // 设置播放速度为1.5倍
          gooseSound.play();
        } else if (type === 'goat') {
          goatSound.currentTime = 0; // 重置音频播放位置
          goatSound.playbackRate = 1.5; // 设置播放速度为1.5倍
          goatSound.play();
        } else if (type === 'goat') {
          goatSound.currentTime = 0; // 重置音频播放位置
          goatSound.playbackRate = 1.5; // 设置播放速度为1.5倍
          goatSound.play();
        } else if (type === 'sheep') {
          sheepSound.currentTime = 0; // 重置音频播放位置
          sheepSound.playbackRate = 1.5; // 设置播放速度为1.5倍
          sheepSound.play();
        } else if (type === 'chick') {
          chickSound.currentTime = 0; // 重置音频播放位置
          chickSound.playbackRate = 2.0; // 设置播放速度为2.0倍
          chickSound.play();
        }
        
        // 检查层级可见性
// 移除了对不存在函数的调用，因为这个函数在当前代码中并未定义
// 动物消除后的层级可见性由 zIndex 属性自动处理
        
        // 检查游戏状态
        const visibleAnimals = animals.value.filter(animal => animal.visible);
        if (visibleAnimals.length === 0 && pendingArea.value.length === 0) {
          endGame('win');
        } else {
          // 检查是否还有可以消除的组合
          setTimeout(() => {
            checkGameOver();
            checkMatch();
          }, 100);
        }
      });

      return; // 找到一组匹配后就退出当前检查
    }
  }
}

// 检查游戏是否结束
const checkGameOver = () => {
  // 如果待消除区域满了，且没有可以消除的组合，游戏结束
  if (pendingArea.value.length >= 6) {
    const counts = new Map<string, number>();
    pendingArea.value.forEach(animal => {
      counts.set(animal.type, (counts.get(animal.type) || 0) + 1);
    });
    
    let hasMatch = false;
    for (const count of counts.values()) {
      if (count >= 3) {
        hasMatch = true;
        break;
      }
    }
    
    if (!hasMatch) {
      endGame('full');
    }
  }
  
  // 检查是否获胜（所有动物都被消除）
  const visibleAnimals = animals.value.filter(animal => animal.visible);
  if (visibleAnimals.length === 0 && pendingArea.value.length === 0) {
    endGame('win');
  }
}

// 开始游戏
const startGame = () => {
  playButtonSound()
  gameStarted.value = true
  gameOver.value = false
  timeLeft.value = 600
  currentLevel.value = 1  // 确保每次重新开始都从第一关开始
  // 初始化游戏区域
  initializeGame()
  // 启动计时器
  timer.value = setInterval(() => {
    if (!isPaused.value && timeLeft.value > 0) {
      timeLeft.value--
    } else if (timeLeft.value === 0) {
      endGame('timeout')
    }
  }, 1000)
}

// 初始化游戏
const initializeGame = () => {
  // 清空现有动物
  animals.value = []
  pendingArea.value = []
  
  // 计算布局参数
  const areaSize = {
    width: 90,    // 增加宽度到90%
    height: 35    // 保持高度不变
  }
  const margin = {
    left: 5,      // 调整左边距到5%
    top: 28,      // 将上边距从25%调整到28%
    right: 95,    // 调整右边界到95%
    bottom: 65    // 保持下边界不变
  }
  const layers = currentLevel.value === 1 ? 2 : 6 // 第一关2层，第二关6层
  
  // 生成初始动物布局
  const numAnimalsPerLayer = currentLevel.value === 1 ? 6 : 18  // 第一关每层6个动物，第二关每层18个动物
  const positions = []

  // 为每层生成随机位置
  for (let layer = 0; layer < layers; layer++) {
    // 确保第一关的动物类型分布合理
    const availableTypes = currentLevel.value === 1 
      ? [...animalTypes].sort(() => Math.random() - 0.5).slice(0, 2) // 第一关只使用2种动物类型
      : animalTypes
  
    for (let i = 0; i < numAnimalsPerLayer; i++) {
      // 在调整后的区域内生成随机位置
      const x = margin.left + Math.random() * areaSize.width
      const y = margin.top + Math.random() * areaSize.height
      
      positions.push({
        x,
        y,
        layer,
        type: availableTypes[i % availableTypes.length] // 确保动物类型均匀分布
      })
    }
  }
  
  // 随机打乱每层的位置
  for (let layer = 0; layer < layers; layer++) {
    const layerPositions = positions.filter(pos => pos.layer === layer)
    for (let i = layerPositions.length - 1; i > 0; i--) {
      const j = Math.floor(Math.random() * (i + 1))
      const temp = layerPositions[i]
      layerPositions[i] = layerPositions[j]
      layerPositions[j] = temp
    }
  }
  
  // 生成动物
  let id = 0
  for (let layer = 0; layer < layers; layer++) {
    const layerPositions = positions.filter(pos => pos.layer === layer)
    
    // 第一关特殊处理：确保每层都有两种动物，且每种动物的数量相等
    if (currentLevel.value === 1) {
      const selectedTypes = [...animalTypes].sort(() => Math.random() - 0.5).slice(0, 2)
      
      // 为每层分配动物，确保每种动物在每层都有3个
      for (let i = 0; i < Math.min(numAnimalsPerLayer, layerPositions.length); i++) {
        const pos = layerPositions[i]
        const animal = {
          id: id++,
          type: selectedTypes[Math.floor(i / 3) % 2], // 每3个动物使用同一种类型
          x: pos.x,
          y: pos.y,
          visible: true,
          zIndex: layer * 100 + i
        }
        animals.value.push(animal)
      }
    } else {
      // 第二关保持原有逻辑
      for (let i = 0; i < Math.min(numAnimalsPerLayer, layerPositions.length); i++) {
        const pos = layerPositions[i]
        const animal = {
          id: id++,
          type: animalTypes[Math.floor(Math.random() * animalTypes.length)],
          x: pos.x,
          y: pos.y,
          visible: true,
          zIndex: layer * 100 + i
        }
        animals.value.push(animal)
      }
    }
  }
}

// 返回主页
const handleReturnClick = () => {
  playButtonSound()
  showConfirmDialog.value = true
}

// 重置游戏状态
const resetGameState = () => {
  currentLevel.value = 1
  showLevelTransition.value = false
  gameStarted.value = false
  gameOver.value = false
  if (timer.value) {
    clearInterval(timer.value)
    timer.value = null
  }
  animals.value = []
  pendingArea.value = []
}

// 确认返回主页
const returnToHome = () => {
  playButtonSound()
  showConfirmDialog.value = false
  resetGameState()
}

// 取消返回
const cancelReturn = () => {
  playButtonSound()
  showConfirmDialog.value = false
}

// 结束游戏
const endGame = (reason: 'timeout' | 'full' | 'win') => {
  if (reason === 'timeout' || reason === 'full') {
    failSound.currentTime = 0
    failSound.play()
  }
  
  if (reason === 'win' && currentLevel.value === 1) {
    // 第一关胜利，显示过渡动画
    showLevelTransition.value = true
    setTimeout(() => {
      showLevelTransition.value = false
      currentLevel.value = 2
      initializeGame()
    }, 2000)
  } else {
    gameOver.value = true
    gameResult.value = reason
    if (timer.value) {
      clearInterval(timer.value)
      timer.value = null
    }
  }
}

// 组件卸载时清理计时器
onUnmounted(() => {
  if (timer.value) {
    clearInterval(timer.value)
  }
})
</script>

<template>
  <div class="game-container">
    <!-- 首页 -->
    <div v-if="!gameStarted" class="start-screen">
      <div class="start-content">
        <img src="https://img.z4a.net/images/2025/01/22/b1dd4ee8071e03de4d52889e6ec05932.png" alt="擒兽记" class="game-title" />
        <p class="game-desc">收集相同的动物，三个一组可以消除。看看你能坚持多久！</p>
        <button @click="startGame" class="start-btn">开始游戏</button>
      </div>
    </div>

    <!-- 游戏中页面 -->
    <div v-else class="game-area">
      <!-- 关卡过渡动画 -->
      <div v-if="showLevelTransition" class="level-transition">
        <div class="level-transition-content">
          <h2>恭喜通过！</h2>
          <p>进入最后一关</p>
        </div>
      </div>
      <div class="game-header">
        <div class="timer">{{ Math.floor(timeLeft / 60) }}:{{ (timeLeft % 60).toString().padStart(2, '0') }}</div>
        <button class="pause-btn" @click="isPaused = !isPaused">{{ isPaused ? '继续' : '暂停' }}</button>
        <button class="return-btn" @click="handleReturnClick">返回主页</button>
      </div>

      <div class="main-area">
        <div
          v-for="animal in animals"
          :key="animal.id"
          class="animal"
          :data-animal-id="animal.id"
          :style="{
            left: `${animal.x + (animal.offsetX || 0)}%`,
            top: `${animal.y + (animal.offsetY || 0)}%`,
            zIndex: animal.zIndex,
            display: animal.visible ? 'flex' : 'none'
          }"
          @mouseenter="handleAnimalHover(animal)"
          @mouseleave="resetAnimalOffsets"
          @click="handleAnimalClick(animal)"
        >
          <img :src="animalImages[animal.type as keyof typeof animalImages]" :alt="animal.type" :style="{
            width: '100%',
            height: '100%',
            objectFit: 'contain',
            display: 'block'
          }" />
        </div>
      </div>

      <div class="pending-area">
        <div
          v-for="(_, index) in Array(6).fill(null)"
          :key="`pending-slot-${index}`"
          class="pending-slot"
        >
          <template v-if="pendingArea[index]">
            <img :src="animalImages[pendingArea[index].type as keyof typeof animalImages]" :alt="pendingArea[index].type" :style="{
              width: '100%',
              height: '100%',
              objectFit: 'contain' as const,
              display: 'block' as const
            }" />
          </template>
        </div>
      </div>
      <div class="button-area">
        <button class="game-btn">按钮1</button>
        <button class="game-btn">按钮2</button>
        <button class="game-btn">按钮3</button>
      </div>
    </div>

    <!-- 游戏结束页面 -->
    <div v-if="gameOver" class="game-over-overlay">
      <div class="game-over">
      <h2 :class="['game-over-title', gameResult]">
        {{ gameResult === 'win' ? '恭喜过关！' :
           gameResult === 'timeout' ? '时间到！' :
           gameResult === 'full' ? '格子已满！' : '游戏结束' }}
      </h2>
      <p class="game-result" v-if="gameResult === 'win'">
        恭喜你成功通关！你用时 {{ 600 - timeLeft }} 秒完成了游戏。
      </p>
      <p class="game-result" v-else-if="gameResult === 'timeout'">
        时间已用完，再接再厉！
      </p>
      <p class="game-result" v-else-if="gameResult === 'full'">
        格子已满，游戏结束！
      </p>
      <div class="game-over-buttons">
        <button @click="startGame" class="restart-btn">重新开始</button>
        <button @click="returnToHome" class="home-btn">返回主页</button>
      </div>
    </div>
    </div>

    <!-- 确认弹窗 -->
    <div v-if="showConfirmDialog" class="confirm-dialog">
      <div class="confirm-dialog-content">
        <p>返回主页？游戏将会终止</p>
        <div class="confirm-dialog-buttons">
          <button @click="returnToHome" class="confirm-btn">确认</button>
          <button @click="cancelReturn" class="cancel-btn">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.game-container {
  width: 100%;
  height: 100vh;
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  background-color: #f5f5f5;
  box-sizing: border-box;
  overflow: hidden;
  position: fixed;
  top: 0;
  left: 0;
}

.game-area {
  position: relative;
  display: flex;
  flex-direction: column;
  gap: 10px;
  width: 100%;
  max-width: 414px;
  height: 100%;
  max-height: 896px;
  aspect-ratio: 414/896;
  background-image: url('https://img.z4a.net/images/2025/01/23/-2356215feae935ad3.png');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  padding: 10px;
}

.main-area {
  flex: 1;
  position: relative;
  height: calc(100% - 120px);
  margin-top: 40px;
}

.animal {
  position: absolute;
  width: 80px;
  height: 80px;
  background-color: transparent;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transform: translate(-50%, -50%);
  transition: transform 0.2s, left 0.3s ease-out, top 0.3s ease-out;
  overflow: hidden;
  touch-action: manipulation;
}

.pending-area {
  width: 100%;
  display: grid;
  grid-template-columns: repeat(6, 1fr);
  gap: 4px;
  padding: 4px;
  height: 60px;
  position: relative;
  transform: translateY(-140px);
}

.pending-slot {
  aspect-ratio: 1;
  border: 5px solid #453D38;
  border-radius: 20px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 2px;
  width: 45px;
  height: 45px;
  background: #FFF;
  box-shadow: 0px 6px 0px 0px rgba(0, 0, 0, 0.13), 0px -6px 0px 0px #A4B3C7 inset;
}

.pending-slot img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

.button-area {
  display: flex;
  justify-content: space-between;
  width: 100%;
  padding: 0 4px;
  transform: translateY(-30px);
}

.game-btn {
  flex: 1;
  margin: 0 2px;
  padding: 10px 0;
  font-size: 16px;
  color: white;
  cursor: pointer;
  border-radius: 20px;
  border: 2px solid #453D38;
  background: #FFD440;
  box-shadow: 0px 4px 2.1px 0px #FFFF92 inset, 0px 6px 0px 0px rgba(0, 0, 0, 0.13), 0px -8px 0px 0px #E48B0D inset;
  transition: transform 0.2s;
}

.game-btn:hover {
  transform: translateY(2px);
  background: #FFC340;
  box-shadow: 0px 4px 2.1px 0px #FFFF92 inset, 0px 4px 0px 0px rgba(0, 0, 0, 0.13), 0px -8px 0px 0px #E48B0D inset;
}

.start-screen {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100%;
  max-width: 414px;
  height: 100%;
  max-height: 896px;
  aspect-ratio: 414/896;
  background-image: url('https://img.z4a.net/images/2025/01/23/-2.png');
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.start-content {
  position: relative;
  width: 100%;
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  transform: translateY(-55px);
}

.game-over-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.game-over {
  position: relative;
  transform: none;
  text-align: center;
  padding: 40px;
  background-color: rgba(255, 255, 255, 0.95);
  border-radius: 12px;
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.15);
  min-width: 300px;
}

.game-over-title {
  font-size: 32px;
  margin-bottom: 20px;
  font-weight: bold;
}

.game-over-title.win {
  color: #42b883;
}

.game-over-title.timeout,
.game-over-title.full {
  color: #e74c3c;
}

.game-over-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
  margin-top: 25px;
}

button {
  padding: 12px 24px;
  font-size: 18px;
  background-color: #42b883;
  color: white;
  border: none;
  border-radius: 50px;
  cursor: pointer;
  transition: background-color 0.3s;
}

button:hover {
  background-color: #3aa876;
}

.game-title {
  width: 270px;
  height: auto;
  position: relative;
  z-index: 1;
  margin: 0 auto 20px;
  display: block;
}

.game-desc {
  font-size: 18px;
  color: #666;
  margin-bottom: 30px;
  line-height: 1.5;
}

.start-btn {
  font-size: 24px;
  padding: 15px 40px;
}

.game-header {
  position: relative;
  width: 100%;
  padding: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: calc(14px + 1vmin);
  background-color: rgba(255, 255, 255, 0.8);
  border-radius: 8px;
  margin-bottom: 10px;
}

.pause-btn {
  padding: 8px 16px;
  font-size: 16px;
  background-color: #42b883;
  margin: 0 10px;
}

.game-result {
  font-size: 20px;
  color: #666;
  margin: 15px 0;
}

.restart-btn,
.home-btn {
  margin: 10px;
  min-width: 120px;
}

.home-btn {
  background-color: #666;
}

.home-btn:hover {
  background-color: #555;
}

.confirm-dialog {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.confirm-dialog-content {
  background-color: white;
  padding: 30px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}

.confirm-dialog-content p {
  margin-bottom: 20px;
  font-size: 18px;
  color: #333;
}

.confirm-dialog-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
}

.confirm-btn {
  background-color: #42b883;
}

.cancel-btn {
  background-color: #666;
}

.cancel-btn:hover {
  background-color: #555;
}

.level-transition {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 2000;
  width: auto;
  height: auto;
}

.level-transition-content {
  width: 80%;
  max-width: 320px;
  background-color: rgba(0, 0, 0, 0.85);
  border-radius: 16px;
  padding: 24px;
  text-align: center;
  color: #fff;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  animation: scaleIn 0.5s ease-in-out;
}

.level-transition-content h2 {
  font-size: 24px;
  margin-bottom: 16px;
  color: #4aeea0;
}

.level-transition-content p {
  font-size: 18px;
  color: #e0e0e0;
  margin: 0;
  line-height: 1.5;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

@keyframes scaleIn {
  from {
    transform: scale(0.8);
    opacity: 0;
  }
  to {
    transform: scale(1);
    opacity: 1;
  }
}
</style>
