<script setup lang="ts">
import { ref } from 'vue'
import { PROMPT_LIST } from './constant/index'

// 弹窗控制
const showModal = ref(false)
const selectedCard = ref<any>(null)
const isFlipping = ref(false)

// 已打开的卡片状态管理
const openedCards = ref(new Set<number>())
// 正在翻转到正面的卡片
const flippingToFront = ref(new Set<number>())

// 打开卡片详情
const openCard = (prompt: any) => {
  // 立即设置选中的卡片和旋转状态
  selectedCard.value = prompt
  isFlipping.value = true

  // 短暂延迟后显示弹窗，让旋转动画先完成
  setTimeout(() => {
    showModal.value = true
    isFlipping.value = false
  }, 600) // 旋转动画持续时间
}

// 关闭弹窗
const closeModal = () => {
  // 为弹窗添加关闭动画
  const modalOverlay = document.querySelector('.modal-overlay')
  if (modalOverlay) {
    modalOverlay.style.animation = 'fadeOut 0.3s ease'
    setTimeout(() => {
      showModal.value = false
      // 弹窗消失后，只有当卡片还未打开时才触发翻转动画
      if (selectedCard.value) {
        const cardId = selectedCard.value.id
        // 检查卡片是否已经是正面状态
        if (!openedCards.value.has(cardId)) {
          flippingToFront.value.add(cardId)
          
          // 动画中点时标记为已打开（切换内容）
          setTimeout(() => {
            openedCards.value.add(cardId)
          }, 300) // 动画中点
          
          // 翻转动画完成后，移除翻转状态
          setTimeout(() => {
            flippingToFront.value.delete(cardId)
          }, 600) // 翻转动画持续时间
        }
      }
      selectedCard.value = null
    }, 300)
  } else {
    showModal.value = false
    // 如果没有动画，也只有当卡片还未打开时才触发翻转动画
    if (selectedCard.value) {
      const cardId = selectedCard.value.id
      // 检查卡片是否已经是正面状态
      if (!openedCards.value.has(cardId)) {
        flippingToFront.value.add(cardId)
        
        // 动画中点时标记为已打开
        setTimeout(() => {
          openedCards.value.add(cardId)
        }, 300)
        
        // 动画完成后移除翻转状态
        setTimeout(() => {
          flippingToFront.value.delete(cardId)
        }, 600)
      }
    }
    selectedCard.value = null
  }
}

// 使用提示词的方法
const usePrompt = (prompt: any) => {
  navigator.clipboard.writeText(prompt.prompt).then(() => {
    alert(`提示词已复制到剪贴板！\n\n${prompt.prompt.substring(0, 100)}...`)
  }).catch(() => {
    // 如果剪贴板 API 不可用，显示提示词内容
    alert(`提示词：\n\n${prompt.prompt}`)
  })
}

// 生成100张卡片数据
const generatePromptList = () => {
  // 生成100张卡片，重复使用基础模板
  const fullList = []
  for (let i = 1; i <= 100; i++) {
    const baseIndex = (i - 1) % PROMPT_LIST.length
    const basePrompt = PROMPT_LIST[baseIndex]
    fullList.push({
      ...basePrompt,
      id: i
    })
  }
  return fullList
}

const promptList = generatePromptList()
</script>

<template>
  <div class="app">
    <main class="main">
      <div class="card-grid">
        <!-- 卡片显示：未打开显示背面，已打开显示正面 -->
        <div v-for="prompt in promptList" :key="prompt.id" 
             :class="{ 
               'card-back': !openedCards.has(prompt.id) && !flippingToFront.has(prompt.id),
               'card-front-grid': openedCards.has(prompt.id),
               'flipping': isFlipping && selectedCard && selectedCard.id === prompt.id,
               'flipping-to-front': flippingToFront.has(prompt.id),
               'opened': openedCards.has(prompt.id)
             }"
             @click="!openedCards.has(prompt.id) ? openCard(prompt) : null">
          
          <!-- 卡片背面内容（未打开时显示） -->
          <template v-if="!openedCards.has(prompt.id)">
            <div class="card-back-pattern"></div>
            <div class="card-number-large">{{ prompt.id.toString().padStart(2, '0') }}</div>
            <div class="card-back-title">AI 提示词</div>
          </template>
          
          <!-- 卡片正面内容（已打开时显示） -->
          <template v-else>
            <div class="card-header">
              <div class="card-number">#{{ prompt.id.toString().padStart(3, '0') }}</div>
            </div>
            <div class="card-content-grid">
              <h3 class="card-title-grid">{{ prompt.name || '提示词标题' }}</h3>
              <div class="card-description-grid">{{ prompt.prompt.substring(0, 80) }}...</div>
              <div v-if="prompt.attention" class="card-attention-grid">
                <strong>Tips：</strong>{{ prompt.attention.substring(0, 50) }}...
              </div>
            </div>
            <div class="opened-indicator" @click.stop="openCard(prompt)">重新查看</div>
          </template>
        </div>
      </div>
    </main>

    <!-- 弹窗模态框 -->
    <div v-if="showModal" class="modal-overlay" @click="closeModal">
      <div class="modal-content" @click.stop v-if="selectedCard">
        <button class="close-button" @click="closeModal">×</button>

        <!-- 卡片正面详情 -->
        <div class="card-front" :class="`card-type-${(selectedCard.id % 5) + 1}`">
          <div class="card-header">
            <div class="card-number">#{{ selectedCard.id.toString().padStart(3, '0') }}</div>
          </div>

          <div class="card-content">
            <h3 class="card-title">{{ selectedCard.name || '提示词标题' }}</h3>
            <div class="card-description" :class="{ 'has-attention': selectedCard.attention }">
              {{ selectedCard.prompt }}
            </div>
            <div v-if="selectedCard.attention" class="card-attention">
              <strong>Tips：</strong>{{ selectedCard.attention }}
            </div>
          </div>

          <div class="card-footer">
            <button class="use-button" @click="usePrompt(selectedCard)">
              使用提示词
            </button>
          </div>

          <div class="holographic-effect"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.main {
  max-width: 1400px;
  margin: 0 auto;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
  gap: 1.5rem;
  padding: 1rem;
  width: 100%;
  box-sizing: border-box;
}

/* 卡片背面样式 */
.card-back {
  aspect-ratio: 3/4;
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  border-radius: 15px;
  padding: 1rem;
  box-shadow:
    0 8px 32px rgba(0, 0, 0, 0.2),
    0 2px 8px rgba(0, 0, 0, 0.1);
  border: 2px solid #4a6fa5;
  position: relative;
  transition: all 0.3s ease;
  cursor: pointer;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
  box-sizing: border-box;
}

.card-back:hover {
  transform: translateY(-5px) scale(1.05);
  box-shadow:
    0 15px 30px rgba(0, 0, 0, 0.3),
    0 5px 15px rgba(0, 0, 0, 0.2);
  border-color: #6b8aca;
}

/* 旋转动画效果 */
.card-back.flipping {
  animation: cardFlip 0.6s ease-in-out;
  transform-style: preserve-3d;
}

/* 翻转到正面动画 */
.flipping-to-front {
  animation: flipToFront 0.6s ease-in-out;
  transform-style: preserve-3d;
}

/* 翻转时背面样式保持到动画中点 */
.flipping-to-front:not(.opened) {
  background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
  border: 2px solid #4a6fa5;
  border-radius: 15px;
  color: white;
  /* 保持背面的布局样式 */
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}

/* 翻转时背面内容样式 */
.flipping-to-front:not(.opened) .card-number-large {
  font-family: 'JetBrains Mono', 'Consolas', 'Monaco', monospace;
  font-size: 2.5rem;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  margin-bottom: 0.5rem;
  position: relative;
  z-index: 1;
}

.flipping-to-front:not(.opened) .card-back-title {
  font-size: 0.9rem;
  font-weight: 500;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
  position: relative;
  z-index: 1;
  text-align: center;
}

.flipping-to-front:not(.opened) .card-back-pattern {
  border-radius: 15px;
}

/* 翻转时正面样式在动画中点后显示 */
.flipping-to-front.opened {
  background: linear-gradient(145deg, #ffffff, #f0f0f0);
  border: 2px solid #4a6fa5;
  border-radius: 15px;
  color: #333;
  /* 正面布局样式 */
  display: flex;
  flex-direction: column;
  padding: 0.6rem;
}

.card-back-pattern {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: repeating-linear-gradient(45deg,
      rgba(255, 255, 255, 0.1) 0px,
      rgba(255, 255, 255, 0.1) 10px,
      transparent 10px,
      transparent 20px);
  border-radius: 15px;
}

.card-number-large {
  font-family: 'JetBrains Mono', 'Consolas', 'Monaco', monospace;
  font-size: 2.5rem;
  font-weight: bold;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5);
  margin-bottom: 0.5rem;
  position: relative;
  z-index: 1;
}

.card-back-title {
  font-size: 0.9rem;
  font-weight: 500;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.5);
  position: relative;
  z-index: 1;
  text-align: center;
}

/* 弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.4s ease;
  backdrop-filter: blur(5px);
}

.modal-content {
  position: relative;
  max-width: 90vw;
  max-height: 90vh;
  overflow: auto;
  animation: scaleIn 0.5s ease;
}

.close-button {
  position: absolute;
  top: -10px;
  right: -10px;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: #ff4757;
  color: white;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  transition: all 0.3s ease;
}

.close-button:hover {
  background: #ff3742;
  transform: scale(1.1);
}

/* 卡片正面样式（在弹窗中显示） */
.card-front {
  background: linear-gradient(145deg, #ffffff, #f0f0f0);
  border-radius: 20px;
  padding: 2rem;
  box-shadow:
    0 20px 60px rgba(0, 0, 0, 0.3),
    0 8px 24px rgba(0, 0, 0, 0.2);
  border: 3px solid #ddd;
  position: relative;
  overflow: hidden;
  width: 400px;
  max-width: 90vw;
}

.card-type-1 {
  border-color: #ff6b6b;
}

.card-type-2 {
  border-color: #4ecdc4;
}

.card-type-3 {
  border-color: #45b7d1;
}

.card-type-4 {
  border-color: #96ceb4;
}

.card-type-5 {
  border-color: #feca57;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.card-number {
  font-family: 'JetBrains Mono', 'Consolas', 'Monaco', monospace;
  font-weight: bold;
  color: #666;
  font-size: 0.9rem;
}

.card-type {
  background: linear-gradient(45deg, #ff6b6b, #ee5a6f);
  color: white;
  padding: 0.3rem 0.8rem;
  border-radius: 15px;
  font-size: 0.8rem;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.card-content {
  margin-bottom: 1.5rem;
}

.card-title {
  font-family: 'Inter', sans-serif;
  font-size: 1.4rem;
  font-weight: 600;
  color: #333;
  margin: 0 0 1rem 0;
  text-align: center;
  letter-spacing: -0.02em;
}

.card-description {
  color: #323232;
  line-height: 1.5;
  font-size: 1rem;
  margin-bottom: 1rem;
}

.card-description.has-attention {
  padding-bottom: 1rem;
  border-bottom: 2px dashed #ddd;
}

.card-attention {
  color: rgba(0, 0, 0, 0.6);
  font-size: 0.9rem;
  line-height: 1.4;
  border-radius: 8px;
  margin-top: 1rem;
}

.card-stats {
  margin-bottom: 1.5rem;
}

.stat {
  display: flex;
  align-items: center;
  margin-bottom: 0.5rem;
}

.stat-label {
  width: 60px;
  font-size: 0.8rem;
  color: #666;
  font-weight: bold;
}

.stat-bar {
  flex: 1;
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
  margin-left: 1rem;
}

.stat-fill {
  height: 100%;
  background: linear-gradient(90deg, #ff6b6b, #4ecdc4);
  transition: width 0.5s ease;
  border-radius: 4px;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.use-button {
  font-family: 'Inter', sans-serif;
  background: linear-gradient(45deg, #667eea, #764ba2);
  color: white;
  border: none;
  padding: 0.7rem 1.2rem;
  border-radius: 25px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 0.9rem;
  letter-spacing: 0.01em;
}

.use-button:hover {
  background: linear-gradient(45deg, #5a6fd8, #6a419a);
  transform: scale(1.05);
}

.card-rarity {
  padding: 0.3rem 0.8rem;
  border-radius: 12px;
  font-size: 0.7rem;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.rarity-common {
  background: #95a5a6;
  color: white;
}

.rarity-uncommon {
  background: #3498db;
  color: white;
}

.rarity-rare {
  background: #9b59b6;
  color: white;
}

.rarity-epic {
  background: #e67e22;
  color: white;
}

.rarity-legendary {
  background: linear-gradient(45deg, #f39c12, #e74c3c);
  color: white;
  animation: glow 2s ease-in-out infinite alternate;
}

@keyframes glow {
  from {
    box-shadow: 0 0 5px rgba(243, 156, 18, 0.5);
  }

  to {
    box-shadow: 0 0 15px rgba(243, 156, 18, 0.8);
  }
}

.holographic-effect {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: linear-gradient(45deg,
      transparent 30%,
      rgba(255, 255, 255, 0.1) 50%,
      transparent 70%);
  transform: translateX(-100%);
  transition: transform 0.6s ease;
  pointer-events: none;
  border-radius: 20px;
}

.pokemon-card:hover .holographic-effect {
  transform: translateX(100%);
}

/* 动画 */
@keyframes fadeIn {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}

@keyframes fadeOut {
  from {
    opacity: 1;
  }

  to {
    opacity: 0;
  }
}

@keyframes scaleIn {
  from {
    opacity: 0;
    transform: scale(0.8) rotateY(180deg);
  }

  to {
    opacity: 1;
    transform: scale(1) rotateY(0deg);
  }
}

@keyframes cardFlip {
  0% {
    transform: rotateY(0deg) scale(1);
  }

  25% {
    transform: rotateY(90deg) scale(1.1);
  }

  50% {
    transform: rotateY(180deg) scale(1.2);
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.4);
  }

  75% {
    transform: rotateY(270deg) scale(1.1);
  }

  100% {
    transform: rotateY(360deg) scale(1);
  }
}

@keyframes flipToFront {
  0% {
    transform: rotateY(0deg) scale(1);
  }

  50% {
    transform: rotateY(90deg) scale(1.1);
    box-shadow: 0 15px 30px rgba(0, 0, 0, 0.3);
  }

  100% {
    transform: rotateY(0deg) scale(1);
  }
}

/* 卡片正面网格样式（在网格中显示的简化版本） */
.card-front-grid {
  aspect-ratio: 3/4;
  background: linear-gradient(145deg, #e0e0e0, #d0d0d0);
  border-radius: 15px;
  padding: 0.6rem;
  box-shadow:
    0 4px 16px rgba(0, 0, 0, 0.1),
    0 1px 4px rgba(0, 0, 0, 0.05);
  border: 2px solid #b8b8b8;
  position: relative;
  transition: all 0.3s ease;
  cursor: default;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  box-sizing: border-box;
  opacity: 0.8;
}

.card-front-grid:hover {
  transform: translateY(-1px);
  box-shadow:
    0 6px 16px rgba(0, 0, 0, 0.12),
    0 2px 6px rgba(0, 0, 0, 0.08);
  opacity: 0.8;
}

.card-front-grid .card-header {
  margin-bottom: 0.5rem;
}

.card-front-grid .card-number {
  font-family: 'JetBrains Mono', 'Consolas', 'Monaco', monospace;
  font-weight: bold;
  color: #999;
  font-size: 0.7rem;
}

.card-content-grid {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
}

.card-title-grid {
  font-family: 'Inter', sans-serif;
  font-size: 0.9rem;
  font-weight: 600;
  color: #777;
  margin: 0 0 0.5rem 0;
  text-align: center;
  letter-spacing: -0.02em;
  line-height: 1.2;
}

.card-description-grid {
  color: #888;
  line-height: 1.3;
  font-size: 0.7rem;
  text-align: center;
  display: -webkit-box;
  -webkit-line-clamp: 3;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.card-attention-grid {
  color: rgba(0, 0, 0, 0.6);
  font-size: 0.6rem;
  line-height: 1.2;
  padding: 0.3rem;
  border-radius: 4px;
  margin-top: 0.3rem;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.opened-indicator {
  position: absolute;
  bottom: 8px;
  right: 8px;
  background: linear-gradient(45deg, #4CAF50, #45a049);
  color: white;
  padding: 0.2rem 0.4rem;
  border-radius: 10px;
  font-size: 0.6rem;
  font-weight: bold;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
}

.opened {
  opacity: 0.9;
}

.opened::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  border-radius: 15px;
  pointer-events: none;
}

/* 响应式设计 */
@media (max-width: 768px) {
  .card-grid {
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    padding: 1rem;
    gap: 1.5rem;
  }

  .title {
    font-size: 2rem;
  }

  .card-front {
    width: 350px;
    padding: 1.5rem;
  }

  .card-number-large {
    font-size: 2rem;
  }

  .card-title-grid {
    font-size: 0.8rem;
  }

  .card-description-grid {
    font-size: 0.65rem;
  }
}

@media (max-width: 480px) {
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
    gap: 0.6rem;
    padding: 0.5rem;
    width: 100%;
    box-sizing: border-box;
  }

  .card-front {
    width: 300px;
    padding: 1rem;
  }

  .card-number-large {
    font-size: 1.8rem;
  }

  .card-back-title {
    font-size: 0.8rem;
  }

  .card-footer {
    flex-direction: column;
    gap: 1rem;
  }

  .use-button {
    width: 100%;
  }

  .card-title-grid {
    font-size: 0.7rem;
  }

  .card-description-grid {
    font-size: 0.6rem;
    -webkit-line-clamp: 2;
  }

  .card-attention-grid {
    font-size: 0.55rem;
    -webkit-line-clamp: 1;
  }

  .opened-indicator {
    font-size: 0.55rem;
    padding: 0.15rem 0.3rem;
  }
}
</style>
