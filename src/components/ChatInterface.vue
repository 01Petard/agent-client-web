<template>
  <div class="chat-interface">
    <!-- 聊天消息区域 -->
    <div ref="chatHistoryRef" class="chat-history">
      <div class="messages-container">
        <!-- 欢迎消息 -->
        <div v-if="messages.length === 0" class="welcome-message">
          <div class="welcome-icon">👋</div>
          <h3>欢迎使用智能咖啡Agent</h3>
          <p>我是您的智能助手，有什么可以帮助您的吗？</p>
        </div>

        <!-- 消息列表 -->
        <div v-for="message in messages" :key="message.id"
             :class="['message', message.type]">
          <div class="message-content">
            <!-- 使用 v-html 渲染富文本 -->
            <div class="message-text" v-html="renderMarkdown(message.text)"></div>
            <div class="message-time">{{ formatTime(message.timestamp) }}</div>
          </div>
        </div>

        <!-- 加载状态 -->
        <div v-if="isLoading" class="message assistant">
          <div class="message-content">
            <div class="typing-indicator">
              <span></span>
              <span></span>
              <span></span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 输入区域 -->
    <div class="chat-input-area">
      <div class="input-container">
        <textarea ref="textareaRef"
                  v-model="inputMessage"
                  :disabled="isLoading"
                  class="message-input"
                  placeholder="输入您的问题... (Alt+Enter 换行)"
                  rows="1"
                  @input="adjustTextareaHeight"
                  @keydown="handleKeyDown"></textarea>
        <button :disabled="!inputMessage.trim() || isLoading"
                class="send-button"
                @click="sendMessage">
          <svg fill="none" height="20" stroke="currentColor" stroke-width="2" viewBox="0 0 24 24" width="20">
            <line x1="22" x2="11" y1="2" y2="13"></line>
            <polygon points="22,2 15,22 11,13 2,9"></polygon>
          </svg>
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import {inject, nextTick, onMounted, ref} from 'vue'
import {marked} from 'marked'
import DOMPurify from 'dompurify'
import {sendWorkflowRequestStream, getUserHistory} from '../services/api.js'

const props = defineProps({currentUser: Object})
const currentUser = inject('currentUser')
const emit = defineEmits(['refreshMessages'])

const messages = ref([])
const inputMessage = ref('')
const isLoading = ref(false)
const chatHistoryRef = ref(null)
const textareaRef = ref(null)

/**
 * 发送消息（流式）
 */
const sendMessage = async () => {
  const message = inputMessage.value.trim()
  if (!message || isLoading.value) return

  // 添加用户消息
  const userMessage = {
    id: Date.now(),
    type: 'user',
    text: message,
    timestamp: new Date()
  }
  messages.value.push(userMessage)
  inputMessage.value = ''
  resetTextareaHeight()
  await nextTick()
  scrollToBottom()

  // 加载状态 + 空助手消息
  isLoading.value = true
  const assistantMsg = {
    id: Date.now() + 1,
    type: 'assistant',
    text: '',
    timestamp: new Date()
  }
  messages.value.push(assistantMsg)

  let fullText = ''
  const uid = currentUser?.value?.id?.toString() || '0'
  sendWorkflowRequestStream({
    userId: uid,
    message: message,
    onChunk: (chunk) => {
      fullText += chunk
      assistantMsg.text = fullText
      // 触发响应式更新
      messages.value = [...messages.value]
      nextTick().then(scrollToBottom)
    },
    onDone: () => {
      if (!fullText.trim()) {
        assistantMsg.text = '抱歉，我暂时无法回答您的问题。'
      }
      isLoading.value = false
      nextTick().then(scrollToBottom)
    },
    onError: (err) => {
      console.error('流式请求失败:', err)
      if (!fullText.trim()) {
        assistantMsg.text = '抱歉，服务暂时不可用，请稍后再试。'
      }
      isLoading.value = false
      nextTick().then(scrollToBottom)
    }
  })
}

/**
 * 键盘事件：Enter 发送，Alt+Enter 换行
 */
const handleKeyDown = (event) => {
  if (event.key === 'Enter' && !event.altKey && !event.shiftKey) {
    event.preventDefault()
    sendMessage()
  }
}

const adjustTextareaHeight = () => {
  const ta = textareaRef.value
  if (ta) {
    ta.style.height = 'auto'
    ta.style.height = Math.min(ta.scrollHeight, 120) + 'px'
  }
}

const resetTextareaHeight = () => {
  const ta = textareaRef.value
  if (ta) ta.style.height = 'auto'
}

const scrollToBottom = () => {
  const el = chatHistoryRef.value
  if (el) el.scrollTop = el.scrollHeight
}

const formatTime = (ts) => {
  return new Date(ts).toLocaleTimeString('zh-CN', {
    hour: '2-digit',
    minute: '2-digit'
  })
}

const renderMarkdown = (text) => {
  if (!text) return ''
  const html = marked.parse(text, {async: false})
  return DOMPurify.sanitize(html)
}

// 加载历史消息
const loadHistory = async () => {
  const uid = currentUser?.value?.id?.toString()
  if (!uid) return
  try {
    const history = await getUserHistory(uid)
    if (history && history.length > 0) {
      messages.value = history.map((msg, i) => ({
        id: msg.id || Date.now() + i,
        type: msg.role === 'user' ? 'user' : 'assistant',
        text: msg.content,
        timestamp: msg.createTime ? new Date(msg.createTime) : new Date()
      }))
    }
  } catch (e) {
    console.error('加载历史失败:', e)
  }
}

onMounted(() => {
  loadHistory()
  textareaRef.value?.focus()
})
</script>

<style scoped>
.chat-interface {
  height: 100%;
  display: flex;
  flex-direction: column;
  background: white;
}

/* ===== 消息历史 ===== */
.chat-history {
  flex: 1;
  overflow-y: auto;
  padding: 1rem 1.5rem;
  scroll-behavior: smooth;
  /* 隐藏滚动条但保持可滚动 */
  scrollbar-width: thin;
  scrollbar-color: transparent transparent;
}

.chat-history:hover {
  scrollbar-color: #d0d0d0 transparent;
}

.chat-history::-webkit-scrollbar {
  width: 4px;
}

.chat-history::-webkit-scrollbar-track {
  background: transparent;
}

.chat-history::-webkit-scrollbar-thumb {
  background: transparent;
  border-radius: 2px;
}

.chat-history:hover::-webkit-scrollbar-thumb {
  background: #d0d0d0;
}

.messages-container {
  display: flex;
  flex-direction: column;
  gap: 0.8rem;
  min-height: 100%;
}

/* ===== 欢迎消息 ===== */
.welcome-message {
  text-align: center;
  padding: 3rem 2rem;
  color: #888;
  margin-top: auto;
  margin-bottom: auto;
}

.welcome-icon {
  font-size: 3rem;
  margin-bottom: 1rem;
}

.welcome-message h3 {
  margin-bottom: 0.5rem;
  color: #444;
}

/* ===== 单条消息 ===== */
.message {
  display: flex;
}

.message.user {
  justify-content: flex-end;
}

.message.assistant {
  justify-content: flex-start;
}

.message-content {
  max-width: 75%;
  padding: 0.75rem 1rem;
  border-radius: 14px;
  position: relative;
}

.message.user .message-content {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border-bottom-right-radius: 4px;
}

.message.assistant .message-content {
  background: #f1f3f5;
  color: #333;
  border-bottom-left-radius: 4px;
}

/* 富文本消息样式 */
.message-text {
  line-height: 1.6;
  word-wrap: break-word;
  white-space: pre-wrap;
}

.message-text :deep(p) {
  margin: 0.4em 0;
}

.message-text :deep(p:first-child) {
  margin-top: 0;
}

.message-text :deep(p:last-child) {
  margin-bottom: 0;
}

.message-text :deep(ul), .message-text :deep(ol) {
  padding-left: 1.2em;
  margin: 0.3em 0;
}

.message-text :deep(li) {
  margin: 0.15em 0;
}

.message-text :deep(strong) {
  font-weight: 600;
}

.message-text :deep(code) {
  font-family: ui-monospace, monospace;
  background: rgba(0,0,0,0.06);
  padding: 0.15em 0.4em;
  border-radius: 3px;
  font-size: 0.9em;
}

.message.user .message-text :deep(code) {
  background: rgba(255,255,255,0.15);
}

.message-text :deep(pre) {
  background: #282c34;
  color: #abb2bf;
  padding: 0.8em;
  border-radius: 6px;
  overflow-x: auto;
  margin: 0.4em 0;
  font-size: 0.9em;
}

.message-time {
  font-size: 0.7rem;
  opacity: 0.6;
  margin-top: 0.3rem;
  text-align: right;
}

/* ===== 打字指示器 ===== */
.typing-indicator {
  display: flex;
  gap: 4px;
  align-items: center;
  padding: 0.3rem 0;
}

.typing-indicator span {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #999;
  animation: typing 1.4s infinite ease-in-out;
}

.typing-indicator span:nth-child(1) { animation-delay: -0.32s; }
.typing-indicator span:nth-child(2) { animation-delay: -0.16s; }

@keyframes typing {
  0%, 80%, 100% { transform: scale(0.8); opacity: 0.5; }
  40% { transform: scale(1); opacity: 1; }
}

/* ===== 输入区域 ===== */
.chat-input-area {
  padding: 0.8rem 1.5rem 1rem;
  background: white;
  border-top: 1px solid #e9ecef;
  flex-shrink: 0;
}

.input-container {
  display: flex;
  gap: 0.75rem;
  align-items: flex-end;
}

.message-input {
  flex: 1;
  padding: 0.7rem 1rem;
  border: 2px solid #e9ecef;
  border-radius: 18px;
  font-size: 0.95rem;
  font-family: inherit;
  resize: none;
  outline: none;
  transition: border-color 0.2s ease;
  min-height: 42px;
  max-height: 120px;
  line-height: 1.5;
}

.message-input:focus {
  border-color: #667eea;
}

.message-input:disabled {
  background: #f8f9fa;
  cursor: not-allowed;
}

.send-button {
  width: 42px;
  height: 42px;
  min-width: 42px;
  border: none;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}

.send-button:hover:not(:disabled) {
  transform: scale(1.05);
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.send-button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

/* ===== 响应式 ===== */
@media (max-width: 768px) {
  .chat-history {
    padding: 0.75rem 1rem;
  }
  .message-content {
    max-width: 85%;
  }
  .chat-input-area {
    padding: 0.6rem 1rem 0.8rem;
  }
}
</style>
