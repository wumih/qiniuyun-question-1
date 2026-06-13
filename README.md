# 🤖 AI Voice Assistant

> 专为黑客马拉松设计的**纯前端无服务器**智能语音对话终端，基于 Vue 3 + Vite 构建，零后端依赖，直连 DeepSeek 大模型。

---

## ✨ 核心功能

### 🎙️ 智能语音交互
- **永久后台监听**：语音识别引擎在授权后立即启动，始终保持运行，不会丢失说话开头内容
- **VAD 自动断句**：基于 Web Audio API 的 `AnalyserNode` 实时分析麦克风音量，静默 800ms 后自动触发推理，无需唤醒词
- **实时转译显示**：说话过程中文字实时滚屏更新

### 🧠 三种交互控制模式

| 模式 | 触发方式 | 适用场景 |
|------|----------|----------|
| **自动模式** | 静默后自动发送 | 日常对话 |
| **🎙️ 手动发送模式** | 点击「▶ 发送指令」按钮才执行 | 需要精确控制发送时机 |
| **⚡ 智能打断模式** | 说话可随时打断 AI 播报 | 全双工自然对话 |

### ❄️ 全局冻结功能
点击「⏸ 冻结系统」后，系统执行全量暂停：
- 摄像头画面定格
- 语音识别停止监听
- 正在进行的 API 请求被物理截断
- TTS 音频队列清空
- 再次点击立即恢复所有子系统

### 🔒 输入/输出互斥机制（半双工保护）
- AI 播报期间可选择锁定所有输入通道，彻底防止环境噪音引发的回音死循环
- 锁定时摄像头画面变为黑白定格，波形频谱变为警告红色

---

## 🛠️ 极速启动

**环境要求**：Node.js ≥ 18、Chrome / Edge 浏览器（必须，Firefox 不支持原生 SpeechRecognition）

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev
```

访问 `http://localhost:5173/`，在初始化弹窗中输入 **DeepSeek API Key**（以 `sk-` 开头）即可开始使用。

---

## 🏗️ 技术架构

```
浏览器硬件层
├── navigator.mediaDevices.getUserMedia  →  摄像头 + 麦克风
├── Web Audio API (AnalyserNode)         →  VAD 音量检测
└── SpeechRecognition API                →  实时语音转文字

状态机层 (FSM)
├── IDLE       →  空闲待机
├── LISTENING  →  正在采集语音
├── PROCESSING →  等待 AI 响应
└── SPEAKING   →  TTS 播报中

AI 调用层
├── DeepSeek Chat API (OpenAI 兼容协议)
├── AbortController                      →  网络层请求熔断
└── SpeechSynthesis API                  →  系统原生 TTS
```

---

## 🛡️ 安全设计

- **零硬编码密钥**：API Key 仅通过 UI 输入，存储于浏览器 `localStorage` 同源沙盒，不随源码分发
- **无后端依赖**：无服务器路由，无数据中转，用户数据不经过任何第三方服务器

---

## 📄 License

[MIT](LICENSE)
