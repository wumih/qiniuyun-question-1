<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
import OpenAI from 'openai'

// ===== 配置 =====
const apiKey = ref('')
const baseURL = ref('https://api.deepseek.com/v1') // DeepSeek 标准接口
const modelName = ref('deepseek-chat')             // 填入你的具体模型名称
const isConfigModalOpen = ref(true)
const errorMessage = ref('')

// ===== 硬件引用 =====
const videoRef = ref(null)
const stream = ref(null)
const hasPermission = ref(false)

// ===== 应用状态 =====
// IDLE | RECORDING | PROCESSING | SPEAKING
const appState = ref('IDLE')
const transcript = ref('')
const aiReply = ref('')
const volumeLevel = ref(0)
const systemFrozen = ref(false)
const voiceOutputEnabled = ref(true)
const visionEnabled = ref(true)
const chatHistory = ref([]) // 多轮对话历史记录 [{role, content, time}]

// ===== 内部变量 =====
let openaiClient = null
let recognition = null
let audioContext = null
let analyser = null
let microphone = null
let abortController = null
let vadRafId = null
let ttsQueueCount = 0 // 用于跟踪未读完的 TTS 队列数量

// ===== 初始化 =====
onMounted(() => {
  const storedKey = localStorage.getItem('VISUAL_ASSISTANT_KEY')
  const storedURL = localStorage.getItem('VISUAL_ASSISTANT_URL')
  const storedModel = localStorage.getItem('VISUAL_ASSISTANT_MODEL')
  if (storedURL) baseURL.value = storedURL
  if (storedModel) modelName.value = storedModel
  if (storedKey) {
    apiKey.value = storedKey
    isConfigModalOpen.value = false
    initSystem()
  }
})

onUnmounted(() => {
  if (vadRafId) cancelAnimationFrame(vadRafId)
})

const saveApiKey = () => {
  if (!apiKey.value.trim()) { errorMessage.value = 'API Key 不能为空'; return }
  localStorage.setItem('VISUAL_ASSISTANT_KEY', apiKey.value)
  localStorage.setItem('VISUAL_ASSISTANT_URL', baseURL.value)
  localStorage.setItem('VISUAL_ASSISTANT_MODEL', modelName.value)
  isConfigModalOpen.value = false
  errorMessage.value = ''
  initSystem()
}

// ===== 系统初始化 =====
const initSystem = async () => {
  try {
    openaiClient = new OpenAI({
      apiKey: apiKey.value,
      dangerouslyAllowBrowser: true,
      baseURL: baseURL.value
    })

    stream.value = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720 },
      audio: true
    })

    if (videoRef.value) {
      videoRef.value.srcObject = stream.value
      hasPermission.value = true
      initVADVisualizer(stream.value)
      initRecognition()
    }
  } catch (err) {
    errorMessage.value = '无法访问摄像头或麦克风，请授予权限后刷新页面。'
  }
}

// ===== VAD（仅用于波形可视化，不再驱动识别触发）=====
const initVADVisualizer = (mediaStream) => {
  audioContext = new (window.AudioContext || window.webkitAudioContext)()
  analyser = audioContext.createAnalyser()
  analyser.fftSize = 256
  microphone = audioContext.createMediaStreamSource(mediaStream)
  microphone.connect(analyser)

  const dataArray = new Uint8Array(analyser.frequencyBinCount)
  const tick = () => {
    if (systemFrozen.value) { volumeLevel.value = 0; vadRafId = requestAnimationFrame(tick); return }
    analyser.getByteFrequencyData(dataArray)
    let sum = 0
    for (let i = 0; i < dataArray.length; i++) sum += dataArray[i]
    volumeLevel.value = (sum / dataArray.length) * 2
    vadRafId = requestAnimationFrame(tick)
  }
  tick()
}

// ===== 语音识别初始化（按需启动，不自动常驻）=====
const initRecognition = () => {
  const SR = window.SpeechRecognition || window.webkitSpeechRecognition
  if (!SR) { errorMessage.value = '当前浏览器不支持语音识别，请使用 Chrome / Edge'; return }

  recognition = new SR()
  recognition.continuous = true
  recognition.interimResults = true
  recognition.lang = 'zh-CN'

  recognition.onresult = (event) => {
    let text = ''
    for (let i = event.resultIndex; i < event.results.length; ++i) {
      text += event.results[i][0].transcript
    }
    transcript.value = text
  }

  recognition.onerror = (event) => {
    if (event.error === 'aborted' || event.error === 'no-speech') return
    console.warn('识别错误:', event.error)
  }

  recognition.onend = () => {
    // 若仍在录音状态（用户未手动停止），自动重启防止引擎超时停止
    if (appState.value === 'RECORDING') {
      try { recognition.start() } catch (e) {}
    }
  }
}

// ===== 核心交互：录音按钮 =====
const startRecording = () => {
  if (!hasPermission.value || systemFrozen.value) return
  if (appState.value !== 'IDLE') return

  transcript.value = ''
  aiReply.value = ''
  appState.value = 'RECORDING'

  try {
    recognition.start()
  } catch (e) {
    // 识别引擎可能已在运行，忽略
  }
}

const stopRecording = () => {
  if (appState.value !== 'RECORDING') return
  try { recognition.stop() } catch (e) {}
  // 不立即清空 transcript，保留内容供用户确认后发送
  appState.value = 'IDLE'
}

// ===== 视觉帧捕获：从 video 元素截取当前画面帧 =====
const captureFrame = () => {
  if (!videoRef.value || !videoRef.value.videoWidth) return null
  const canvas = document.createElement('canvas')
  canvas.width = 512   // 降采样以控制 Token 消耗
  canvas.height = Math.round(512 * videoRef.value.videoHeight / videoRef.value.videoWidth)
  const ctx = canvas.getContext('2d')
  ctx.drawImage(videoRef.value, 0, 0, canvas.width, canvas.height)
  return canvas.toDataURL('image/jpeg', 0.7) // base64 JPEG
}

// ===== 发送指令给 AI =====
const sendCommand = async () => {
  const text = transcript.value.trim()
  if (!text || appState.value === 'PROCESSING' || appState.value === 'SPEAKING') return

  // 若仍在录音，先停止
  if (appState.value === 'RECORDING') {
    try { recognition.stop() } catch (e) {}
  }

  appState.value = 'PROCESSING'
  aiReply.value = '神经元推理中...'
  abortController = new AbortController()

  try {
    // 构建消息内容：视觉模式下包含图像，纯文字模式下只发文本
    let messageContent
    if (visionEnabled.value) {
      const frameBase64 = captureFrame()
      if (frameBase64) {
        messageContent = [
          { type: 'text', text: text || '请描述你在这张图中看到了什么，用中文回答，不超过3句。' },
          { type: 'image_url', image_url: { url: frameBase64 } }
        ]
      } else {
        messageContent = text // 摄像头未就绪时降级为纯文字
      }
    } else {
      messageContent = text
    }

    // 构建完整 messages 数组：系统提示词 + 历史对话 + 当前轮
    const historyMessages = chatHistory.value.map(turn => ({
      role: turn.role,
      content: turn.content  // 历史记录只发文字，不重发图片（节省 Token）
    }))

    const stream = await openaiClient.chat.completions.create({
      model: modelName.value,
      messages: [
        { role: 'system', content: '你是一个智能语音视觉助手。请用中文回答，简洁准确。' },
        ...historyMessages,
        { role: 'user', content: messageContent }
      ],
      max_tokens: 300,
      stream: true // 开启流式输出
    }, { signal: abortController.signal })

    aiReply.value = ''
    let ttsBuffer = ''
    window.speechSynthesis.cancel()
    ttsQueueCount = 0

    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta?.content || ''
      if (delta) {
        if (aiReply.value === '神经元推理中...') aiReply.value = '' // 清除占位符
        aiReply.value += delta
        ttsBuffer += delta

        if (voiceOutputEnabled.value) {
          // 句子切片：遇到标点符号就读
          const sentenceEndRegex = /([。！？\n.!?]+)/
          const match = ttsBuffer.match(sentenceEndRegex)
          
          if (match) {
            const splitIndex = match.index + match[0].length
            const sentenceToSpeak = ttsBuffer.substring(0, splitIndex).trim()
            ttsBuffer = ttsBuffer.substring(splitIndex)
            
            if (sentenceToSpeak) {
              playTTSChunk(sentenceToSpeak)
            }
          }
        }
      }
    }

    // 流结束后，把剩余的缓冲读完
    if (voiceOutputEnabled.value && ttsBuffer.trim()) {
      playTTSChunk(ttsBuffer.trim())
    }

    // 流完整结束后，将这一轮记录进历史记录
    const now = new Date().toLocaleTimeString('zh-CN', { hour: '2-digit', minute: '2-digit' })
    chatHistory.value.push({ role: 'user',      content: text,  time: now })
    chatHistory.value.push({ role: 'assistant', content: aiReply.value, time: now })
    transcript.value = '' // 发送成功后清空输入框

    // 纯文字模式下，流结束后直接复位
    if (!voiceOutputEnabled.value) {
      appState.value = 'IDLE'
    }

  } catch (error) {
    if (error.name === 'AbortError') return
    errorMessage.value = '请求失败：' + error.message
    setTimeout(() => { appState.value = 'IDLE'; errorMessage.value = '' }, 4000)
  }
}

// ===== TTS (流式切片队列) =====
const playTTSChunk = (text) => {
  const utterance = new SpeechSynthesisUtterance(text)
  utterance.lang = 'zh-CN'
  utterance.rate = 1.1
  
  ttsQueueCount++
  
  utterance.onstart = () => { appState.value = 'SPEAKING' }
  utterance.onend = () => {
    ttsQueueCount--
    // 队列清空且请求已完成时（非 PROCESSING），回到待机
    if (ttsQueueCount <= 0) {
      ttsQueueCount = 0
      if (appState.value !== 'PROCESSING') {
        appState.value = 'IDLE'
      }
    }
  }
  utterance.onerror = () => {
    ttsQueueCount--
    if (ttsQueueCount <= 0) {
      ttsQueueCount = 0
      appState.value = 'IDLE'
    }
  }
  window.speechSynthesis.speak(utterance)
}

// ===== 打断 AI 播报 =====
const interruptAI = () => {
  window.speechSynthesis.cancel()
  ttsQueueCount = 0
  if (abortController) { abortController.abort(); abortController = null }
  appState.value = 'IDLE'
}

// ===== 清空重置 =====
const clearAll = () => {
  interruptAI()
  transcript.value = ''
  aiReply.value = ''
  chatHistory.value = [] // 同时清空历史
}

// ===== 全局冻结 =====
const toggleFreeze = () => {
  systemFrozen.value = !systemFrozen.value
  if (systemFrozen.value) {
    if (abortController) { abortController.abort(); abortController = null }
    window.speechSynthesis.cancel()
    ttsQueueCount = 0
    try { recognition.stop() } catch (e) {}
    if (videoRef.value) videoRef.value.pause()
    appState.value = 'IDLE'
  } else {
    if (videoRef.value) videoRef.value.play()
  }
}

// ===== 状态标签 =====
const stateLabel = {
  IDLE: '待机',
  RECORDING: '录音中',
  PROCESSING: 'AI 推理中',
  SPEAKING: 'AI 播报中'
}
</script>

<template>
  <div class="app-container">
    <!-- 初始化弹窗 -->
    <div v-if="isConfigModalOpen" class="modal-overlay">
      <div class="modal-content">
        <h2>系统初始化</h2>
        <div class="config-field">
          <label>API Key</label>
          <input v-model="apiKey" type="password" placeholder="sk-..." @keyup.enter="saveApiKey" />
        </div>
        <div class="config-field">
          <label>API 地址（Base URL）</label>
          <input v-model="baseURL" type="text" placeholder="https://dashscope.aliyuncs.com/compatible-mode/v1" />
          <div class="config-hint">
            🟢 DeepSeek 标准地址: <code>https://api.deepseek.com/v1</code><br>
            🟡 如需切换其他厂商: 填入对应 Base URL 即可
          </div>
        </div>
        <div class="config-field">
          <label>模型名称</label>
          <input v-model="modelName" type="text" placeholder="qwen-vl-plus" />
          <div class="config-hint">
            如果你的 DeepSeek 模型支持视觉，就填入其正式名称<br>
            示例: <code>deepseek-chat</code> / <code>deepseek-vl2</code> / <code>gpt-4o</code>
          </div>
        </div>
        <button @click="saveApiKey">连接系统</button>
        <p v-if="errorMessage" class="error-text">{{ errorMessage }}</p>
      </div>
    </div>

    <main class="main-content" :class="{ blur: isConfigModalOpen }">
      <!-- 顶栏 -->
      <header class="cyber-header">
        <h1>AI VOICE ASSISTANT</h1>
        <div class="header-controls">
          <span class="state-badge" :class="appState.toLowerCase()">
            {{ stateLabel[appState] }}
          </span>
          <!-- 语音 / 文字输出切换 -->
          <label class="toggle-switch" :class="{ 'voice-on': voiceOutputEnabled }">
            <input type="checkbox" v-model="voiceOutputEnabled">
            {{ voiceOutputEnabled ? '🔊 语音回答' : '💬 文字回答' }}
          </label>
          <!-- 视觉模式切换 -->
          <label class="toggle-switch" :class="{ 'voice-on': visionEnabled }">
            <input type="checkbox" v-model="visionEnabled">
            {{ visionEnabled ? '👁 视觉开启' : '👁 视觉关闭' }}
          </label>
          <button class="freeze-btn" :class="{ frozen: systemFrozen }" @click="toggleFreeze">
            {{ systemFrozen ? '❄️ 已冻结 · 点击恢复' : '⏸ 冻结系统' }}
          </button>
          <button class="ctrl-btn danger" v-if="appState === 'SPEAKING' || appState === 'PROCESSING'" @click="interruptAI">
            ✕ 打断
          </button>
        </div>
      </header>

      <!-- 摄像头 -->
      <section class="video-section">
        <div class="video-container">
          <video ref="videoRef" autoplay playsinline muted class="camera-feed"
            :class="{ 'paused-feed': systemFrozen }"></video>
          <div class="scanner-line" v-if="appState === 'PROCESSING'"></div>
          <!-- 录音时的发光边框 -->
          <div class="recording-ring" v-if="appState === 'RECORDING'"></div>
        </div>
      </section>

      <!-- 交互面板 -->
      <section class="interaction-panel" v-if="hasPermission">



        <!-- 对话历史气泡流 -->
        <div class="chat-history" v-if="chatHistory.length > 0 || appState === 'PROCESSING' || (appState === 'SPEAKING' && aiReply)">
          <div class="chat-history-header">
            <span>💬 对话历史（{{ Math.floor(chatHistory.length / 2) }} 轮）</span>
            <button class="clear-history-btn" @click="clearAll">清空全部</button>
          </div>
          <div class="chat-bubbles">
            <!-- 已完成的历史记录 -->
            <div
              v-for="(turn, idx) in chatHistory"
              :key="idx"
              class="bubble"
              :class="turn.role"
            >
              <div class="bubble-meta">{{ turn.role === 'user' ? '🧑 你' : '🤖 AI' }} <span>{{ turn.time }}</span></div>
              <div class="bubble-text">{{ turn.content }}</div>
            </div>
            
            <!-- 当前正在生成的回答（流式渲染区域） -->
            <div v-if="appState === 'PROCESSING' || (appState === 'SPEAKING' && aiReply)" class="bubble assistant live-reply">
              <div class="bubble-meta">🤖 AI (生成中...)</div>
              <div class="bubble-text">
                {{ aiReply || '⏳ 神经元推理中...' }}
              </div>
            </div>
          </div>
        </div>

        <!-- 实时转译区（可编辑：语音识别内容可手动修改） -->
        <div class="transcript-area">
          <div class="transcript-label">
            🗣️ 语音内容
            <span class="label-hint">{{ appState === 'RECORDING' ? '· 识别中，可直接修改文字' : '· 可手动输入或修改' }}</span>
          </div>
          <textarea
            class="transcript-textarea"
            :class="{ active: appState === 'RECORDING' }"
            v-model="transcript"
            :placeholder="appState === 'RECORDING' ? '正在聆听，请说话...' : '点击录音按钮开始，或直接在此输入文字'"
            :readonly="appState === 'PROCESSING' || appState === 'SPEAKING'"
            rows="3"
          ></textarea>
        </div>

        <!-- 操作按钮组 -->
        <div class="action-buttons">
          <!-- 录音控制按钮 -->
          <button
            class="record-btn"
            :class="{
              'recording': appState === 'RECORDING',
              'disabled': systemFrozen || appState === 'PROCESSING' || appState === 'SPEAKING'
            }"
            @mousedown="startRecording"
            @click="appState === 'RECORDING' ? stopRecording() : startRecording()"
            :disabled="systemFrozen || appState === 'PROCESSING' || appState === 'SPEAKING'"
          >
            <span class="record-icon">{{ appState === 'RECORDING' ? '⏹' : '🎙' }}</span>
            {{ appState === 'RECORDING' ? '停止录音' : '开始录音' }}
          </button>

          <!-- 发送按钮 -->
          <button
            class="send-btn"
            :disabled="!transcript.trim() || appState === 'PROCESSING' || appState === 'SPEAKING' || systemFrozen"
            @click="sendCommand"
          >
            <span>▶ 发送指令</span>
          </button>

          <!-- 清空按钮 -->
          <button class="clear-btn" @click="clearAll" :disabled="!transcript && !aiReply">
            ✕ 清空
          </button>
        </div>

        <!-- 波形频谱 -->
        <div class="audio-waves" :class="{ 'recording-waves': appState === 'RECORDING', 'frozen-waves': systemFrozen }">
          <div class="wave" v-for="m in [1, 1.5, 2, 1.5, 1]" :key="m"
            :style="{ height: (appState === 'RECORDING' ? Math.min(100, Math.max(10, volumeLevel * m)) : 10) + '%' }">
          </div>
        </div>

        <p v-if="errorMessage" class="error-text floating-error">{{ errorMessage }}</p>
      </section>
    </main>
  </div>
</template>

<style scoped>
/* ===== 状态徽章 ===== */
.state-badge {
  padding: 4px 14px;
  border-radius: 20px;
  font-size: 0.85rem;
  font-weight: bold;
  border: 1px solid currentColor;
}
.state-badge.idle      { color: #888; border-color: #555; }
.state-badge.recording { color: #ff4444; border-color: #ff4444; animation: blink 1s ease-in-out infinite; }
.state-badge.processing { color: #f5a623; border-color: #f5a623; }
.state-badge.speaking  { color: var(--primary-color); border-color: var(--primary-color); }

@keyframes blink {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.4; }
}

/* ===== 视频区 ===== */
.recording-ring {
  position: absolute;
  inset: 0;
  border: 3px solid #ff4444;
  border-radius: 8px;
  animation: ring-pulse 1s ease-in-out infinite;
  pointer-events: none;
}
@keyframes ring-pulse {
  0%, 100% { box-shadow: 0 0 12px rgba(255,68,68,0.5); }
  50%       { box-shadow: 0 0 30px rgba(255,68,68,0.9); }
}
.paused-feed {
  filter: grayscale(100%) brightness(40%);
  transition: filter 0.3s;
}

/* ===== 交互面板 ===== */
.interaction-panel {
  padding: 20px 30px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

/* ===== 转译区 ===== */
.transcript-area, .reply-area {
  background: rgba(255,255,255,0.04);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 10px;
  padding: 14px 18px;
}
.transcript-label, .reply-label {
  font-size: 0.75rem;
  color: #666;
  margin-bottom: 6px;
  text-transform: uppercase;
  letter-spacing: 1px;
}
.transcript-content {
  font-size: 1.05rem;
  color: #ccc;
  min-height: 40px;
  line-height: 1.6;
  transition: color 0.3s;
}
.transcript-content.active {
  color: #fff;
  border-left: 3px solid #ff4444;
  padding-left: 10px;
}
.reply-content {
  font-size: 1.05rem;
  color: var(--primary-color);
  line-height: 1.7;
  font-weight: 500;
}

/* ===== 操作按钮组 ===== */
.action-buttons {
  display: flex;
  gap: 12px;
  align-items: center;
  flex-wrap: wrap;
}

.record-btn {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 12px 28px;
  font-size: 1rem;
  font-weight: bold;
  border-radius: 30px;
  border: 2px solid #888;
  background: rgba(255,255,255,0.05);
  color: #fff;
  cursor: pointer;
  transition: all 0.2s;
}
.record-btn:hover:not(:disabled) {
  border-color: #ff4444;
  background: rgba(255,68,68,0.1);
  box-shadow: 0 0 16px rgba(255,68,68,0.4);
}
.record-btn.recording {
  border-color: #ff4444;
  background: rgba(255,68,68,0.2);
  box-shadow: 0 0 20px rgba(255,68,68,0.6);
  animation: record-glow 1.2s ease-in-out infinite;
}
@keyframes record-glow {
  0%, 100% { box-shadow: 0 0 14px rgba(255,68,68,0.4); }
  50%       { box-shadow: 0 0 32px rgba(255,68,68,0.9); }
}
.record-btn:disabled { opacity: 0.35; cursor: not-allowed; }

.send-btn {
  padding: 12px 28px;
  font-size: 1rem;
  font-weight: bold;
  border-radius: 30px;
  border: none;
  background: linear-gradient(135deg, var(--primary-color), #00a8cc);
  color: #000;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 0 14px rgba(0,255,204,0.4);
}
.send-btn:hover:not(:disabled) {
  transform: scale(1.04);
  box-shadow: 0 0 28px rgba(0,255,204,0.7);
}
.send-btn:disabled { opacity: 0.35; cursor: not-allowed; transform: none; }

.clear-btn {
  padding: 10px 20px;
  font-size: 0.9rem;
  border-radius: 20px;
  border: 1px solid #444;
  background: transparent;
  color: #888;
  cursor: pointer;
  transition: all 0.2s;
}
.clear-btn:hover:not(:disabled) { border-color: #888; color: #ccc; }
.clear-btn:disabled { opacity: 0.3; cursor: not-allowed; }

.ctrl-btn.danger {
  padding: 6px 16px;
  border-radius: 16px;
  border: 1px solid #ff4444;
  background: rgba(255,68,68,0.1);
  color: #ff4444;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
}
.ctrl-btn.danger:hover { background: rgba(255,68,68,0.25); }

/* ===== 冻结按钮 ===== */
.freeze-btn {
  padding: 6px 18px;
  font-size: 0.85rem;
  font-weight: bold;
  border: 1px solid #4fc3f7;
  border-radius: 20px;
  background: rgba(79,195,247,0.1);
  color: #4fc3f7;
  cursor: pointer;
  transition: all 0.3s;
}
.freeze-btn.frozen {
  border-color: #00cfff;
  background: rgba(0,207,255,0.2);
  color: #00cfff;
  animation: freeze-pulse 1.5s ease-in-out infinite;
}
@keyframes freeze-pulse {
  0%, 100% { box-shadow: 0 0 10px rgba(0,207,255,0.3); }
  50%       { box-shadow: 0 0 24px rgba(0,207,255,0.8); }
}

/* ===== 波形 ===== */
.audio-waves {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 5px;
  height: 40px;
}
.wave {
  width: 4px;
  background: #333;
  border-radius: 2px;
  transition: height 0.08s ease;
}
.recording-waves .wave { background: #ff4444; }
.frozen-waves .wave { background: #4fc3f7; height: 10% !important; opacity: 0.3; }

/* ===== 语音/文字切换 Toggle ===== */
.toggle-switch {
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 5px 14px;
  border-radius: 20px;
  border: 1px solid #555;
  background: rgba(255,255,255,0.05);
  color: #888;
  font-size: 0.85rem;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.25s;
  user-select: none;
}
.toggle-switch input { display: none; }
.toggle-switch:hover {
  border-color: #888;
  color: #ccc;
}
.toggle-switch.voice-on {
  border-color: var(--primary-color);
  background: rgba(0, 255, 204, 0.08);
  color: var(--primary-color);
  box-shadow: 0 0 10px rgba(0,255,204,0.2);
}

/* ===== 可编辑转译框 ===== */
.label-hint {
  font-size: 0.72rem;
  color: #556677;
  font-weight: normal;
  margin-left: 6px;
  text-transform: none;
  letter-spacing: 0;
}
.transcript-textarea {
  width: 100%;
  min-height: 70px;
  background: rgba(0, 0, 0, 0.3);
  border: 1px solid #334;
  border-radius: 6px;
  color: #ccc;
  font-size: 1rem;
  font-family: inherit;
  line-height: 1.6;
  padding: 10px 14px;
  box-sizing: border-box;
  resize: vertical;
  outline: none;
  transition: border-color 0.2s, color 0.2s, box-shadow 0.2s;
}
.transcript-textarea::placeholder {
  color: #445566;
  font-style: italic;
}
.transcript-textarea:focus {
  border-color: #667788;
  box-shadow: 0 0 8px rgba(100,150,200,0.2);
}
.transcript-textarea.active {
  color: #fff;
  border-color: #ff4444;
  border-left: 3px solid #ff4444;
  box-shadow: 0 0 10px rgba(255,68,68,0.2);
}
.transcript-textarea[readonly] {
  opacity: 0.5;
  cursor: not-allowed;
}

/* ===== 对话历史气泡流 ===== */
.chat-history {
  border: 1px solid #223;
  border-radius: 10px;
  overflow: hidden;
}
.chat-history-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 14px;
  background: rgba(0,0,0,0.3);
  font-size: 0.8rem;
  color: #667788;
  border-bottom: 1px solid #223;
}
.clear-history-btn {
  background: transparent;
  border: 1px solid #445;
  color: #556677;
  border-radius: 10px;
  padding: 2px 10px;
  font-size: 0.75rem;
  cursor: pointer;
  transition: all 0.2s;
}
.clear-history-btn:hover { border-color: #ff4444; color: #ff4444; }
.chat-bubbles {
  max-height: 160px;
  overflow-y: auto;
  padding: 12px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  scroll-behavior: smooth;
}
.chat-bubbles::-webkit-scrollbar { width: 4px; }
.chat-bubbles::-webkit-scrollbar-track { background: transparent; }
.chat-bubbles::-webkit-scrollbar-thumb { background: #334; border-radius: 2px; }
.bubble {
  max-width: 85%;
  padding: 8px 12px;
  border-radius: 10px;
  font-size: 0.9rem;
  line-height: 1.5;
}
.bubble.user {
  align-self: flex-end;
  background: rgba(0, 100, 80, 0.3);
  border: 1px solid rgba(0,255,204,0.15);
  color: #cce;
}
.bubble.assistant {
  align-self: flex-start;
  background: rgba(20, 30, 50, 0.6);
  border: 1px solid #334;
  color: #ccd;
}
.bubble.live-reply {
  border-color: #00ffcc;
  box-shadow: 0 0 10px rgba(0,255,204,0.1);
}
.bubble-meta {
  font-size: 0.72rem;
  color: #556;
  margin-bottom: 4px;
}
.bubble-meta span { margin-left: 6px; }
.bubble-text { word-break: break-word; white-space: pre-wrap; }
</style>
