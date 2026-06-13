<script setup>
import { ref, onMounted } from 'vue'
import OpenAI from 'openai'

const videoRef = ref(null)
const stream = ref(null)
const hasPermission = ref(false)
const errorMessage = ref('')
const isConfigModalOpen = ref(true)
const apiKey = ref('')

const appState = ref('IDLE') 
const transcript = ref('')
const latestFrame = ref('')
const volumeLevel = ref(0)
const aiReply = ref('')
const allowInterruption = ref(false) // 打断选项状态
const manualTriggerMode = ref(false)  // 手动触发模式：开启后不自动推理，等待点击按钮
const pendingTranscript = ref('')     // 手动模式下缓存的待发送语音文本
const systemFrozen = ref(false)       // 全局冻结状态

let audioContext = null
let analyser = null
let microphone = null
let recognition = null
let silenceTimer = null
let currentUtterance = null
let abortController = null // 网络请求中断控制器
const SILENCE_THRESHOLD = 800 
const VOLUME_THRESHOLD = 15 
let openaiClient = null 

onMounted(() => {
  const storedKey = localStorage.getItem('VISUAL_ASSISTANT_KEY')
  const envKey = import.meta.env.VITE_OPENAI_API_KEY
  if (envKey) {
    apiKey.value = envKey
    isConfigModalOpen.value = false
    initCamera()
  } else if (storedKey) {
    apiKey.value = storedKey
    isConfigModalOpen.value = false
    initCamera()
  }
})

const saveApiKey = () => {
  if (!apiKey.value.trim()) {
    errorMessage.value = 'API Key 不能为空'
    return
  }
  localStorage.setItem('VISUAL_ASSISTANT_KEY', apiKey.value)
  isConfigModalOpen.value = false
  errorMessage.value = ''
  initCamera()
}

const initCamera = async () => {
  try {
    openaiClient = new OpenAI({ 
      apiKey: apiKey.value, 
      dangerouslyAllowBrowser: true,
      baseURL: 'https://api.deepseek.com/v1'
    })
    
    stream.value = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720 },
      audio: true
    })
    if (videoRef.value) {
      videoRef.value.srcObject = stream.value
      hasPermission.value = true
      
      initAudioAnalysis(stream.value)
      initSpeechRecognition()
    }
  } catch (err) {
    console.error('获取媒体设备失败:', err)
    errorMessage.value = '无法访问摄像头或麦克风，请在浏览器设置中授予权限并刷新页面。'
  }
}

const initAudioAnalysis = (mediaStream) => {
  audioContext = new (window.AudioContext || window.webkitAudioContext)()
  analyser = audioContext.createAnalyser()
  analyser.fftSize = 256
  microphone = audioContext.createMediaStreamSource(mediaStream)
  microphone.connect(analyser)

  const dataArray = new Uint8Array(analyser.frequencyBinCount)

  const checkVolume = () => {
    // 全局冻结守卫：冻结状态下跳过所有 VAD 逻辑，但保持 rAF 循环以便解冻后立即恢复
    if (systemFrozen.value) {
      volumeLevel.value = 0
      requestAnimationFrame(checkVolume)
      return
    }
    analyser.getByteFrequencyData(dataArray)
    let sum = 0
    for (let i = 0; i < dataArray.length; i++) {
      sum += dataArray[i]
    }
    volumeLevel.value = (sum / dataArray.length) * 2

    // 状态隔离判断：如果不允许打断，且处于输出状态，则直接冻结不处理音频流
    const isLocked = !allowInterruption.value && (appState.value === 'PROCESSING' || appState.value === 'SPEAKING')

    if (!isLocked) {
      if (volumeLevel.value > VOLUME_THRESHOLD) {
        
        // 全双工模式下的拦截阀门：若允许打断且正处于输出，瞬间触发强行打断
        if (allowInterruption.value && (appState.value === 'SPEAKING' || appState.value === 'PROCESSING')) {
          interruptPlayback()
        }

        if (appState.value === 'IDLE') {
          appState.value = 'LISTENING'
          transcript.value = ''
          if (!manualTriggerMode.value) aiReply.value = ''
          // 不再在此处启动 recognition，它已经在后台持续运行
        }
        clearTimeout(silenceTimer)
        silenceTimer = null
      } else {
        if (appState.value === 'LISTENING' && !silenceTimer) {
          if (manualTriggerMode.value) {
            // 手动模式：静默后缓存文本并回到 IDLE 等待用户点击按钮
            silenceTimer = setTimeout(() => {
              pendingTranscript.value = transcript.value
              appState.value = 'IDLE'
              silenceTimer = null
            }, SILENCE_THRESHOLD)
          } else {
            // 自动模式：静默后直接触发推理
            silenceTimer = setTimeout(() => {
              triggerInference()
            }, SILENCE_THRESHOLD)
          }
        }
      }
    }
    requestAnimationFrame(checkVolume)
  }
  checkVolume()
}

const initSpeechRecognition = () => {
  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
  if (SpeechRecognition) {
    recognition = new SpeechRecognition()
    recognition.continuous = true
    recognition.interimResults = true
    recognition.lang = 'zh-CN'

    recognition.onresult = (event) => {
      // 若处于物理锁定且不允许打断，则拒绝消费任何转译结果
      if (!allowInterruption.value && (appState.value === 'PROCESSING' || appState.value === 'SPEAKING')) return
      // 冻结状态下拒绝消费
      if (systemFrozen.value) return
      
      let currentTranscript = ''
      for (let i = event.resultIndex; i < event.results.length; ++i) {
        currentTranscript += event.results[i][0].transcript
      }
      transcript.value = currentTranscript
      // 手动模式下同步更新待发送缓存
      if (manualTriggerMode.value) {
        pendingTranscript.value = currentTranscript
      }
    }
    
    recognition.onend = () => {
      // 核心修复：只要系统未被冻结，且未处于非打断的输出锁定，就立即重启
      const isHardLocked = !allowInterruption.value && (appState.value === 'PROCESSING' || appState.value === 'SPEAKING')
      if (!systemFrozen.value && !isHardLocked) {
        try { recognition.start() } catch (e) {}
      }
    }

    recognition.onerror = (event) => {
      // aborted 是主动调用 stop() 触发的，不是真实错误，忽略
      if (event.error === 'aborted') return
      console.warn('语音识别错误:', event.error)
      // 其他错误（如 network、no-speech）延迟后重启
      setTimeout(() => {
        const isHardLocked = !allowInterruption.value && (appState.value === 'PROCESSING' || appState.value === 'SPEAKING')
        if (!systemFrozen.value && !isHardLocked) {
          try { recognition.start() } catch (e) {}
        }
      }, 300)
    }

    // 立即启动，保持永久后台监听
    try { recognition.start() } catch (e) {}
  } else {
    console.warn('当前浏览器不支持原生语音识别。')
  }
}

// 全双工核心：强行中断输出并复位
const interruptPlayback = () => {
  if (appState.value === 'SPEAKING' || appState.value === 'PROCESSING') {
    window.speechSynthesis.cancel() // 清空音频播报缓冲队列
    if (abortController) {
      abortController.abort() // 在物理底层切断悬而未决的 API HTTP 请求
      abortController = null
    }
    releaseOutputLock() // 瞬间解除状态机锁定
  }
}

const enterOutputLock = () => {
  appState.value = 'PROCESSING'
  if (!allowInterruption.value) {
    if (recognition) recognition.stop() 
    if (videoRef.value) videoRef.value.pause() 
  }
}

const releaseOutputLock = () => {
  appState.value = 'IDLE'
  transcript.value = ''
  aiReply.value = ''
  if (videoRef.value) videoRef.value.play() 
}

// 手动模式专用：点击按钮后触发
const submitManualCommand = () => {
  const text = pendingTranscript.value || transcript.value
  if (!text.trim()) return
  pendingTranscript.value = ''
  triggerInference(text)
}

const triggerInference = async (overrideText = null) => {
  enterOutputLock()
  aiReply.value = '网络神经元推理中...'
  abortController = new AbortController() // 每次网络请求签发全新的中止控制器
  
  try {
    const promptText = overrideText || transcript.value || "你好"
    
    const response = await openaiClient.chat.completions.create({
      model: "deepseek-chat",
      messages: [
        {
          role: "user",
          content: promptText
        }
      ],
      max_tokens: 300
    }, {
      signal: abortController.signal // 绑定熔断控制器信号
    })
    
    const responseText = response.choices[0].message.content
    aiReply.value = responseText
    
    playTTS(responseText)
  } catch (error) {
    if (error.name === 'AbortError') {
      console.log('网络请求已被用户发声强制打断')
      return // 被用户打断引发的报错静默处理，不再对外抛出
    }
    console.error('API Error:', error)
    errorMessage.value = '模型网关请求失败：' + error.message
    setTimeout(() => {
      releaseOutputLock()
      errorMessage.value = ''
    }, 4000)
  }
}

const playTTS = (text) => {
  window.speechSynthesis.cancel() 
  currentUtterance = new SpeechSynthesisUtterance(text)
  currentUtterance.lang = 'zh-CN'
  currentUtterance.rate = 1.1
  
  currentUtterance.onstart = () => {
    appState.value = 'SPEAKING'
  }
  
  currentUtterance.onend = () => {
    releaseOutputLock()
  }

  currentUtterance.onerror = () => {
    if (appState.value === 'SPEAKING') {
      releaseOutputLock()
    }
  }
  
  window.speechSynthesis.speak(currentUtterance)
}

// ====================== 全局冻结控制 ======================
const toggleFreeze = () => {
  systemFrozen.value = !systemFrozen.value

  if (systemFrozen.value) {
    // === 进入冻结 ===
    // 1. 中止任何正在进行的 API 请求
    if (abortController) {
      abortController.abort()
      abortController = null
    }
    // 2. 清空 TTS 音频队列
    window.speechSynthesis.cancel()
    // 3. 停止语音识别
    if (recognition) recognition.stop()
    // 4. 定格摄像头画面
    if (videoRef.value) videoRef.value.pause()
    // 5. 清除待触发的静默计时器
    clearTimeout(silenceTimer)
    silenceTimer = null
    // 6. 状态机复位
    appState.value = 'IDLE'
  } else {
    // === 解除冻结 ===
    // 1. 恢复摄像头画面流
    if (videoRef.value) videoRef.value.play()
    // 2. 重启语音识别监听
    if (recognition) {
      try { recognition.start() } catch (e) {}
    }
    // VAD checkVolume 循环已在后台保持运行，无需重启
  }
}
</script>

<template>
  <div class="app-container">
    <div v-if="isConfigModalOpen" class="modal-overlay">
      <div class="modal-content">
        <h2>系统初始化</h2>
        <p>请输入您的 DeepSeek API Key 以激活智能对话终端：</p>
        <input 
          v-model="apiKey" 
          type="password" 
          placeholder="sk-..." 
          @keyup.enter="saveApiKey"
        />
        <button @click="saveApiKey">连接系统</button>
        <p v-if="errorMessage" class="error-text">{{ errorMessage }}</p>
      </div>
    </div>

    <main class="main-content" :class="{ 'blur': isConfigModalOpen }">
      <header class="cyber-header">
        <h1>AI VOICE ASSISTANT</h1>
        <div class="header-controls">
          <label class="toggle-switch">
            <input type="checkbox" v-model="allowInterruption">
            开启智能打断
          </label>
          <label class="toggle-switch manual-mode-toggle" :class="{ 'active-mode': manualTriggerMode }">
            <input type="checkbox" v-model="manualTriggerMode">
            🎙️ 手动发送模式
          </label>
          <button class="freeze-btn" :class="{ 'frozen': systemFrozen }" @click="toggleFreeze">
            {{ systemFrozen ? '❄️ 已冻结 · 点击恢复' : '⏸ 冻结系统' }}
          </button>
          <div class="status-indicator">
            <span class="dot" :class="{ 'active': hasPermission }"></span>
            {{ hasPermission ? '系统在线' : '等待连接' }}
            <span v-if="hasPermission" class="state-label">[{{ appState }}]</span>
          </div>
        </div>
      </header>

      <section class="video-section">
        <div class="video-container">
          <video 
            ref="videoRef" 
            autoplay 
            playsinline 
            muted
            class="camera-feed"
            :class="{ 'paused-feed': !allowInterruption && (appState === 'PROCESSING' || appState === 'SPEAKING') }"
          ></video>
          <div class="scanner-line" v-if="appState === 'PROCESSING'"></div>
        </div>
        
        <!-- 手动模式：始终显示实时转译与发送按钮 -->
        <div class="transcript-box manual-box" v-if="manualTriggerMode && hasPermission">
          <p class="live-transcript">🎙️ 实时采集中：<span>{{ transcript || pendingTranscript || '等待您说话...' }}</span></p>
          <button 
            class="submit-btn" 
            :disabled="(!transcript && !pendingTranscript) || appState === 'PROCESSING' || appState === 'SPEAKING'"
            @click="submitManualCommand"
          >
            {{ appState === 'PROCESSING' || appState === 'SPEAKING' ? '⏳ AI 处理中...' : '▶ 发送指令' }}
          </button>
          <p v-if="aiReply" class="ai-reply">🤖 AI: {{ aiReply }}</p>
        </div>

        <!-- 自动模式：原有的状态提示框 -->
        <div class="transcript-box" v-else-if="(transcript || aiReply) && appState !== 'IDLE'">
          <p v-if="appState === 'LISTENING'">🗣️ 正在倾听: {{ transcript }}</p>
          <p v-if="appState === 'PROCESSING'">⏳ 已捕获指令: {{ transcript }}<br/><span class="ai-reply">{{ aiReply }}</span></p>
          <p v-if="appState === 'SPEAKING' || (aiReply && appState !== 'PROCESSING')" class="ai-reply">
            🤖 AI: {{ aiReply }}<br/>
            <span v-if="!allowInterruption" class="interrupt-hint">🔒 半双工模式：输出流独占，输入通道已物理冻结</span>
            <span v-else class="interrupt-hint">⚡ 全双工模式：可随时对着麦克风发声强行打断 AI 播报</span>
          </p>
        </div>

        <p v-if="errorMessage && !isConfigModalOpen" class="error-text floating-error">
          {{ errorMessage }}
        </p>
      </section>
      
      <footer class="cyber-footer" v-if="hasPermission">
        <div class="audio-waves" :class="{ 
          'locked-waves': !allowInterruption && (appState === 'PROCESSING' || appState === 'SPEAKING'),
          'frozen-waves': systemFrozen
        }">
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel * 1.5)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel * 2)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel * 1.5)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel)) + '%' }"></div>
        </div>
        <p>
          {{ systemFrozen ? '❄️ 系统冻结中 · 所有输入输出已全部暂停' :
             appState === 'LISTENING' ? '正在接收语音指令...' : 
             appState === 'PROCESSING' ? (allowInterruption ? '神经元推理中，您可以随时强行打断...' : '神经元推理中，输入设备已锁定') : 
             appState === 'SPEAKING' ? (allowInterruption ? 'AI 播报中，您可以直接说话打断...' : 'AI 播报中，输入设备已锁定') : '环境静默，系统待机' }}
        </p>
      </footer>
    </main>
  </div>
</template>

<style scoped>
.header-controls {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}
.toggle-switch {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.9rem;
  color: #fff;
  font-weight: bold;
  cursor: pointer;
  background: rgba(0, 255, 204, 0.1);
  padding: 5px 15px;
  border-radius: 20px;
  border: 1px solid var(--primary-color);
  transition: all 0.3s;
}
.toggle-switch:hover {
  background: rgba(0, 255, 204, 0.2);
}
.manual-mode-toggle {
  border-color: #f5a623;
  background: rgba(245, 166, 35, 0.1);
}
.manual-mode-toggle.active-mode {
  background: rgba(245, 166, 35, 0.25);
  box-shadow: 0 0 10px rgba(245, 166, 35, 0.4);
}
.freeze-btn {
  padding: 6px 18px;
  font-size: 0.9rem;
  font-weight: bold;
  border: 1px solid #4fc3f7;
  border-radius: 20px;
  background: rgba(79, 195, 247, 0.1);
  color: #4fc3f7;
  cursor: pointer;
  transition: all 0.3s;
  white-space: nowrap;
}
.freeze-btn:hover {
  background: rgba(79, 195, 247, 0.25);
  box-shadow: 0 0 12px rgba(79, 195, 247, 0.5);
}
.freeze-btn.frozen {
  border-color: #00cfff;
  background: rgba(0, 207, 255, 0.2);
  color: #00cfff;
  box-shadow: 0 0 18px rgba(0, 207, 255, 0.6);
  animation: freeze-pulse 1.5s ease-in-out infinite;
}
@keyframes freeze-pulse {
  0%, 100% { box-shadow: 0 0 12px rgba(0, 207, 255, 0.4); }
  50%       { box-shadow: 0 0 28px rgba(0, 207, 255, 0.9); }
}
.frozen-waves .wave {
  background: #4fc3f7 !important;
  height: 20% !important;
  opacity: 0.3;
  transition: all 0.4s;
}
.manual-box {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.live-transcript {
  color: #ccc;
  font-size: 0.95rem;
}
.live-transcript span {
  color: #fff;
  font-weight: bold;
}
.submit-btn {
  align-self: flex-start;
  padding: 10px 28px;
  background: linear-gradient(135deg, #f5a623, #e8850a);
  color: #000;
  font-weight: bold;
  font-size: 1rem;
  border: none;
  border-radius: 25px;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 0 15px rgba(245, 166, 35, 0.5);
}
.submit-btn:hover:not(:disabled) {
  transform: scale(1.05);
  box-shadow: 0 0 25px rgba(245, 166, 35, 0.8);
}
.submit-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  transform: none;
}
.ai-reply {
  color: var(--primary-color);
  font-weight: bold;
  margin-top: 5px;
  display: block;
}
.interrupt-hint {
  font-size: 0.8rem;
  color: #ff3366;
  font-weight: normal;
  margin-top: 8px;
  display: inline-block;
}
.paused-feed {
  filter: grayscale(100%) brightness(50%);
  transition: filter 0.3s ease;
}
.locked-waves .wave {
  background: #ff3366 !important;
  opacity: 0.5;
}
</style>
