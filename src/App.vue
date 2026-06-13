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
const allowInterruption = ref(false) // 新增打断选项状态

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
          aiReply.value = ''
          if (recognition) {
            try { recognition.start() } catch (e) {}
          }
        }
        clearTimeout(silenceTimer)
        silenceTimer = null
      } else {
        if (appState.value === 'LISTENING' && !silenceTimer) {
          silenceTimer = setTimeout(() => {
            triggerInference()
          }, SILENCE_THRESHOLD)
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
      
      let currentTranscript = ''
      for (let i = event.resultIndex; i < event.results.length; ++i) {
        currentTranscript += event.results[i][0].transcript
      }
      transcript.value = currentTranscript
    }
    
    recognition.onend = () => {
      if (appState.value === 'LISTENING' || allowInterruption.value) {
        try { recognition.start() } catch (e) {}
      }
    }
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

const triggerInference = async () => {
  enterOutputLock()
  aiReply.value = '网络神经元推理中...'
  abortController = new AbortController() // 每次网络请求签发全新的中止控制器
  
  try {
    const promptText = transcript.value || "你好"
    
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
        
        <div class="transcript-box" v-if="(transcript || aiReply) && appState !== 'IDLE'">
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
        <div class="audio-waves" :class="{ 'locked-waves': !allowInterruption && (appState === 'PROCESSING' || appState === 'SPEAKING') }">
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel * 1.5)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel * 2)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel * 1.5)) + '%' }"></div>
          <div class="wave" :style="{ height: Math.min(100, Math.max(20, volumeLevel)) + '%' }"></div>
        </div>
        <p>
          {{ appState === 'LISTENING' ? '正在接收语音指令...' : 
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
  gap: 30px;
}
.toggle-switch {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 1rem;
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
