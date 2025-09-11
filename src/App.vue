<script setup lang="ts">
import { ref } from 'vue'
import { PROMPT_LIST } from './constant/index'

// 提示词接口定义
interface Prompt {
  id: number
  name: string
  prompt: string
  attention?: string
  imgUrl?: string
}

// Toast 消息系统
interface ToastMessage {
  id: number
  message: string
  type: 'success' | 'error' | 'info'
  duration?: number
}

const toasts = ref<ToastMessage[]>([])
let toastIdCounter = 0

// 显示 toast 消息
const showToast = (message: string, type: 'success' | 'error' | 'info' = 'info', duration = 3000) => {
  const id = ++toastIdCounter
  const toast: ToastMessage = { id, message, type, duration }
  toasts.value.push(toast)

  // 自动移除 toast
  setTimeout(() => {
    removeToast(id)
  }, duration)
}

// 移除 toast
const removeToast = (id: number) => {
  const index = toasts.value.findIndex(toast => toast.id === id)
  if (index > -1) {
    toasts.value.splice(index, 1)
  }
}

// 弹窗控制
const showModal = ref(false)
const selectedCard = ref<(Prompt & { displayIndex: number }) | null>(null)
const isFlipping = ref(false)

// 图片预览弹窗控制
const showImagePreview = ref(false)
const previewImage = ref('')
const imageLoading = ref(false)
const imageError = ref(false)

// 已打开的卡片状态管理
const openedCards = ref(new Set<number>())
// 正在翻转到正面的卡片
const flippingToFront = ref(new Set<number>())

// 打开卡片详情
const openCard = (prompt: Prompt, index: number) => {
  // 立即设置选中的卡片和旋转状态，并添加显示索引
  selectedCard.value = { ...prompt, displayIndex: index }
  isFlipping.value = true

  // 短暂延迟后显示弹窗，让旋转动画先完成
  setTimeout(() => {
    showModal.value = true
    isFlipping.value = false
    // 禁止页面滚动
    document.body.style.overflow = 'hidden'
  }, 600) // 旋转动画持续时间
}

// 关闭弹窗
const closeModal = () => {
  // 恢复页面滚动
  document.body.style.overflow = ''

  // 为弹窗添加关闭动画
  const modalOverlay = document.querySelector('.modal-overlay')
  if (modalOverlay) {
    (modalOverlay as any).style.animation = 'fadeOut 0.3s ease'
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
            // 保存到缓存
            saveToCache()
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
          // 保存到缓存
          saveToCache()
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
const usePrompt = (prompt: Prompt) => {
  navigator.clipboard.writeText(prompt.prompt).then(() => {
    showToast(`提示词已复制到剪贴板！\n\n${prompt.prompt.substring(0, 100)}...`)
  }).catch(() => {
    // 如果剪贴板 API 不可用，显示提示词内容
    showToast(`提示词：\n\n${prompt.prompt}`)
  })
}

// 缓存键名
const CACHE_KEY = 'JQ_RANDOM_PROMPT_CARD_DATA'

// 生成100张卡片数据
const generatePromptList = (): Prompt[] => {
  // 生成100张卡片，重复使用基础模板
  const fullList: Prompt[] = []
  for (let i = 0; i < PROMPT_LIST.length; i++) {
    const basePrompt = PROMPT_LIST[i]
    fullList.push({
      ...basePrompt,
      id: i + 1
    })
  }
  
  // 使用 Fisher-Yates 洗牌算法随机打乱数组
  for (let i = fullList.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [fullList[i], fullList[j]] = [fullList[j], fullList[i]]
  }
  
  return fullList
}

// 从缓存加载数据
const loadFromCache = (): { promptList: Prompt[], openedCards: number[] } | null => {
  try {
    const cached = localStorage.getItem(CACHE_KEY)
    if (cached) {
      const data = JSON.parse(cached)
      if (data.promptList && data.openedCards) {
        return data
      }
    }
  } catch (error) {
    console.error('加载缓存失败:', error)
  }
  return null
}

// 保存数据到缓存
const saveToCache = () => {
  try {
    const data = {
      promptList: promptList.value,
      openedCards: Array.from(openedCards.value)
    }
    localStorage.setItem(CACHE_KEY, JSON.stringify(data))
  } catch (error) {
    console.error('保存缓存失败:', error)
  }
}

// 初始化数据：优先使用缓存，否则生成新数据
const initializeData = (): { promptList: Prompt[], openedCards: Set<number> } => {
  const cached = loadFromCache()
  if (cached) {
    // 使用缓存数据
    return {
      promptList: cached.promptList,
      openedCards: new Set(cached.openedCards)
    }
  } else {
    // 生成新数据
    return {
      promptList: generatePromptList(),
      openedCards: new Set<number>()
    }
  }
}

const { promptList: initialPromptList, openedCards: initialOpenedCards } = initializeData()
const promptList = ref<Prompt[]>(initialPromptList)

// 使用初始化的已打开卡片状态
openedCards.value = initialOpenedCards

// 重置功能
const resetData = () => {
  // 清除缓存
  localStorage.removeItem(CACHE_KEY)
  
  // 重新生成随机数据
  promptList.value = generatePromptList()
  openedCards.value = new Set()
  flippingToFront.value = new Set()
  
  // 关闭弹窗和图片预览
  showModal.value = false
  showImagePreview.value = false
  selectedCard.value = null
  previewImage.value = ''
  
  // 恢复页面滚动
  document.body.style.overflow = ''
  
  // 平滑滚动到页面顶部
  window.scrollTo({
    top: 0,
    behavior: 'smooth'
  })
  
  showToast('数据已重置，生成新的随机卡片顺序！', 'success')
}

// 使用 Canvas 处理图片跨域的函数
const canvasLoadImage = (imageUrl: string): Promise<string> => {
  return new Promise((resolve, reject) => {
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')
    const img = new Image()

    img.crossOrigin = 'anonymous'

    img.onload = function () {
      canvas.width = img.width
      canvas.height = img.height

      ctx?.drawImage(img, 0, 0)

      const dataURL = canvas.toDataURL('image/png', 0.9)
      resolve(dataURL)
    }

    img.onerror = () => reject(new Error('图片加载失败'))
    img.src = imageUrl
  })
}

// 动态导入图片函数
const getImageUrl = (imgName: string): string => {
  try {
    // 使用 Vite 的动态导入功能
    return new URL(`./assets/${imgName}`, import.meta.url).href
  } catch (error) {
    console.error('图片路径解析失败:', error)
    return ''
  }
}

// 查看图片方法
const viewImage = async (prompt: Prompt) => {
  if (!prompt.imgUrl) {
    showToast('该卡片暂无效果图', 'info')
    return
  }

  imageLoading.value = true
  imageError.value = false
  showImagePreview.value = true
  // 禁止页面滚动
  document.body.style.overflow = 'hidden'

  try {
    const imageUrl = getImageUrl(prompt.imgUrl)

    // 尝试使用 Canvas 处理图片
    try {
      const processedImage = await canvasLoadImage(imageUrl)
      previewImage.value = processedImage
    } catch (canvasError) {
      // 如果 Canvas 处理失败，直接使用原图片 URL
      console.warn('Canvas 处理失败，使用原图片:', canvasError)
      previewImage.value = imageUrl
    }
  } catch (error) {
    console.error('图片加载失败:', error)
    imageError.value = true
  } finally {
    imageLoading.value = false
  }
}

// 关闭图片预览
const closeImagePreview = () => {
  showImagePreview.value = false
  previewImage.value = ''
  imageError.value = false
  // 恢复页面滚动
  document.body.style.overflow = ''
}
</script>

<template>
  <div class="app">
    <header class="header">
      <div class="header-content">
        <h1 class="app-title">AI 提示词卡片抽签</h1>
        <p class="app-subtitle">made by jiaqi shen</p>
      </div>
    </header>
    <main class="main">
      <div class="card-grid">
        <!-- 卡片显示：未打开显示背面，已打开显示正面 -->
        <div v-for="(prompt, index) in promptList" :key="prompt.id" :class="{
          'card-back': !openedCards.has(prompt.id) && !flippingToFront.has(prompt.id),
          'card-front-grid': openedCards.has(prompt.id),
          'flipping': isFlipping && selectedCard && selectedCard.id === prompt.id,
          'flipping-to-front': flippingToFront.has(prompt.id),
          'opened': openedCards.has(prompt.id)
        }" @click="!openedCards.has(prompt.id) ? openCard(prompt, index) : null">

          <!-- 卡片背面内容（未打开时显示） -->
          <template v-if="!openedCards.has(prompt.id)">
            <div class="card-back-pattern"></div>
            <div class="card-number-large">{{ (index + 1).toString().padStart(2, '0') }}</div>
            <div class="card-back-title">AI 提示词</div>
          </template>

          <!-- 卡片正面内容（已打开时显示） -->
          <template v-else>
            <div class="card-header">
              <div class="card-number">#{{ (index + 1).toString().padStart(3, '0') }}</div>
            </div>
            <div class="card-content-grid">
              <h3 class="card-title-grid">{{ prompt.name || '提示词标题' }}</h3>
              <div class="card-description-grid">{{ prompt.prompt }}</div>
            </div>
            <div class="opened-indicator" @click.stop="openCard(prompt, index)">查看提示词</div>
          </template>
        </div>
      </div>
    </main>

    <!-- 重置按钮 -->
    <footer class="footer">
      <button class="reset-button" @click="resetData">
        重置卡片
      </button>
    </footer>

    <!-- 弹窗模态框 -->
    <div v-if="showModal" class="modal-overlay" @click="closeModal">
      <div class="modal-container" @click.stop v-if="selectedCard">
        <button class="close-button" @click="closeModal">×</button>
        <div class="modal-content">
          <!-- 卡片正面详情 -->
          <div class="card-front" :class="`card-type-${(selectedCard.displayIndex % 5) + 1}`">
            <div class="card-header">
              <div class="card-number">#{{ (selectedCard.displayIndex + 1).toString().padStart(3, '0') }}</div>
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
                复制提示词
              </button>
              <button class="use-button" @click="viewImage(selectedCard)">
                查看效果图
              </button>
            </div>

            <div class="holographic-effect"></div>
          </div>
        </div>
      </div>
    </div>

    <!-- 图片预览弹窗 -->
    <div v-if="showImagePreview" class="image-preview-overlay" @click="closeImagePreview">
      <div class="image-preview-content" @click.stop>
        <button class="image-close-button" @click="closeImagePreview">×</button>

        <!-- 加载状态 -->
        <div v-if="imageLoading" class="image-loading">
          <div class="loading-spinner"></div>
          <p>正在加载图片...</p>
        </div>

        <!-- 错误状态 -->
        <div v-else-if="imageError" class="image-error">
          <div class="error-icon">⚠️</div>
          <p>图片加载失败</p>
          <button class="retry-button" @click="selectedCard && viewImage(selectedCard)">重试</button>
        </div>

        <!-- 图片显示 -->
        <div v-else class="image-container">
          <img :src="previewImage" :alt="selectedCard?.name || '效果图'" class="preview-image"
            @error="imageError = true" />
        </div>
      </div>
    </div>

    <!-- Toast 消息容器 -->
    <div class="toast-container">
      <transition-group name="toast" tag="div">
        <div v-for="toast in toasts" :key="toast.id" :class="[
          'toast',
          `toast-${toast.type}`
        ]" @click="removeToast(toast.id)">
          <div class="toast-message">{{ toast.message }}</div>
        </div>
      </transition-group>
    </div>
  </div>
</template>

<style scoped>
.app {
  min-height: 100vh;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
}

/* 头部样式 */
.header {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  padding: 2rem 0;
  text-align: center;
}

.header-content {
  max-width: 1400px;
  margin: 0 auto;
  padding: 0 1rem;
}

.app-title {
  font-family: 'Inter', sans-serif;
  font-size: 2.5rem;
  font-weight: 700;
  color: white;
  margin: 0 0 0.5rem 0;
  text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.3);
  letter-spacing: -0.02em;
}

.app-subtitle {
  font-family: 'Inter', sans-serif;
  font-size: 1rem;
  font-weight: 400;
  color: rgba(255, 255, 255, 0.8);
  margin: 0;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.2);
  letter-spacing: 0.02em;
}

.main {
  max-width: 1400px;
  margin: 0 auto;
}

/* 底部样式 */
.footer {
  max-width: 1400px;
  margin: 0 auto;
  padding: 2rem 1rem;
  text-align: center;
}

.reset-button {
  font-family: 'Inter', sans-serif;
  background: linear-gradient(135deg, #ff6b6b, #ee5a6f);
  color: white;
  border: none;
  padding: 1rem 2rem;
  border-radius: 30px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 1rem;
  letter-spacing: 0.02em;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  box-shadow: 0 8px 25px rgba(255, 107, 107, 0.3);
  text-transform: none;
}

.reset-button:hover {
  background: linear-gradient(135deg, #ee5a6f, #e74c3c);
  transform: translateY(-2px) scale(1.05);
  box-shadow: 0 12px 30px rgba(255, 107, 107, 0.4);
}

.reset-button:active {
  transform: translateY(0) scale(1);
  box-shadow: 0 6px 20px rgba(255, 107, 107, 0.3);
}

.reset-icon {
  font-size: 1.2rem;
  display: inline-block;
  transition: transform 0.3s ease;
}

.reset-button:hover .reset-icon {
  transform: rotate(180deg);
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

.modal-container {
  position: relative;
  max-width: 90vw;
  max-height: 90vh;
  animation: scaleIn 0.5s ease;
}

.modal-content {
  max-height: 90vh;
  overflow-y: auto;
  overflow-x: hidden;
}

.close-button {
  position: absolute;
  top: -16px;
  right: -16px;
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
  width: 400px;
  max-width: 90vw;
}

/* 美化滚动条 */
.modal-content::-webkit-scrollbar {
  width: 6px;
}

.modal-content::-webkit-scrollbar-track {
  background: rgba(0, 0, 0, 0.1);
  border-radius: 3px;
}

.modal-content::-webkit-scrollbar-thumb {
  background: linear-gradient(45deg, #667eea, #764ba2);
  border-radius: 3px;
}

.modal-content::-webkit-scrollbar-thumb:hover {
  background: linear-gradient(45deg, #5a6fd8, #6a419a);
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
  line-clamp: 3;
  -webkit-line-clamp: 3;
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

/* 图片预览弹窗样式 */
.image-preview-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.95);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  backdrop-filter: blur(8px);
}

.image-preview-content {
  position: relative;
  max-width: 95vw;
  max-height: 95vh;
  background: white;
  border-radius: 20px;
  /* overflow: hidden; */
  box-shadow: 0 25px 80px rgba(0, 0, 0, 0.6);
  display: flex;
  flex-direction: column;
}

.image-close-button {
  position: absolute;
  top: -15px;
  right: -15px;
  width: 45px;
  height: 45px;
  border-radius: 50%;
  background: linear-gradient(135deg, #ff4757, #ff3742);
  color: white;
  border: 3px solid white;
  font-size: 1.8rem;
  cursor: pointer;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.3);
  transition: all 0.3s ease;
  font-weight: bold;
}

.image-close-button:hover {
  background: linear-gradient(135deg, #ff3742, #e63946);
  transform: scale(1.1) rotate(90deg);
  box-shadow: 0 8px 25px rgba(0, 0, 0, 0.4);
}

/* 加载状态样式 */
.image-loading {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 4rem 2rem;
  min-height: 400px;
  color: #323232;
  width: 300px;
}

.loading-spinner {
  width: 50px;
  height: 50px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #667eea;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 1.5rem;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}

.image-loading p {
  font-family: 'Inter', sans-serif;
  font-size: 1.1rem;
  margin: 0;
  color: #555;
}

/* 错误状态样式 */
.image-error {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 4rem 2rem;
  min-height: 400px;
  color: #666;
}

.error-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.image-error p {
  font-family: 'Inter', sans-serif;
  font-size: 1.1rem;
  margin: 0 0 1.5rem 0;
  color: #555;
}

.retry-button {
  background: linear-gradient(45deg, #667eea, #764ba2);
  color: white;
  border: none;
  padding: 0.8rem 1.5rem;
  border-radius: 25px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s ease;
  font-size: 0.95rem;
}

.retry-button:hover {
  background: linear-gradient(45deg, #5a6fd8, #6a419a);
  transform: scale(1.05);
}

/* 图片容器样式 */
.image-container {
  display: flex;
  flex-direction: column;
  max-height: 95vh;
  overflow: hidden;
}

.preview-image {
  max-width: 100%;
  max-height: 70vh;
  object-fit: contain;
  background: #f8f9fa;
  display: block;
  margin: 0 auto;
}

.image-info {
  padding: 1.5rem 2rem;
  background: white;
  border-top: 1px solid #eee;
  flex-shrink: 0;
}

.image-title {
  font-family: 'Inter', sans-serif;
  font-size: 1.4rem;
  font-weight: 600;
  color: #323232;
  margin: 0 0 1rem 0;
  text-align: center;
  letter-spacing: -0.02em;
}

.image-description {
  color: #666;
  line-height: 1.6;
  font-size: 1rem;
  margin: 0;
  text-align: left;
  max-height: 150px;
  overflow-y: auto;
}


/* 响应式设计 */
@media (max-width: 768px) {
  .app-title {
    font-size: 2rem;
  }

  .app-subtitle {
    font-size: 0.9rem;
  }

  .header {
    padding: 1.5rem 0;
  }

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
  .app-title {
    font-size: 1.5rem;
  }

  .app-subtitle {
    font-size: 0.8rem;
  }

  .header {
    padding: 1rem 0;
  }

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

  .footer {
    padding: 1.5rem 0.5rem;
  }

  .reset-button {
    padding: 0.8rem 3.5rem;
    font-size: 0.9rem;
  }

  .card-title-grid {
    font-size: 0.7rem;
  }

  .card-description-grid {
    font-size: 0.6rem;
    line-clamp: 4;
    -webkit-line-clamp: 4;
  }

  .opened-indicator {
    font-size: 0.55rem;
    padding: 0.15rem 0.3rem;
  }

  /* 移动端图片预览样式 */
  .image-preview-content {
    max-width: 95vw;
    max-height: 90vh;
    margin: 1rem;
  }

  .image-close-button {
    width: 40px;
    height: 40px;
    font-size: 1.5rem;
    top: -10px;
    right: -10px;
  }

  .image-info {
    padding: 1rem;
  }

  .image-title {
    font-size: 1.2rem;
  }

  .image-description {
    font-size: 0.9rem;
    max-height: 120px;
  }
}

/* Toast 样式 */
.toast-container {
  position: fixed;
  top: 20px;
  right: 20px;
  z-index: 3000;
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-width: 400px;
  pointer-events: none;
}

.toast {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  background: #009b67;
  border-radius: 12px;
  padding: 16px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
  border-left: 4px solid;
  pointer-events: auto;
  cursor: pointer;
  transition: all 0.3s ease;
  animation: slideIn 0.3s ease;
  max-width: 100%;
  word-wrap: break-word;
}
.toast-icon {
  font-size: 18px;
  flex-shrink: 0;
  margin-top: 2px;
}

.toast-message {
  flex: 1;
  font-size: 16px;
  line-height: 1.4;
  color: #f5f5f5;
  white-space: pre-line;
}


/* Toast 动画 */
.toast-enter-active,
.toast-leave-active {
  transition: all 0.3s ease;
}

.toast-enter-from {
  opacity: 0;
  transform: translateX(100%);
}

.toast-leave-to {
  opacity: 0;
  transform: translateX(100%);
}

.toast-move {
  transition: transform 0.3s ease;
}

@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateX(100%);
  }

  to {
    opacity: 1;
    transform: translateX(0);
  }
}

/* 移动端 Toast 样式 */
@media (max-width: 768px) {
  .toast-container {
    top: 10px;
    right: 10px;
    left: 10px;
    max-width: none;
  }

  .toast {
    padding: 12px;
  }

  .toast-message {
    font-size: 14px;
  }

  /* 移动端动画：从顶部滑入 */
  .toast-enter-from {
    opacity: 0;
    transform: translateY(-100%);
  }

  .toast-leave-to {
    opacity: 0;
    transform: translateY(-100%);
  }

  @keyframes slideIn {
    from {
      opacity: 0;
      transform: translateY(-100%);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }
}
</style>
