# 🤖 AI Voice Assistant

> 专为七牛云比赛设计的**纯前端无服务器**智能语音交互终端，基于 Vue 3 + Vite 构建，零后端依赖，支持流式输出与多模态接入。

---

## 🎬 项目演示 (Demo)

由于 GitHub 原生环境对大体积视频源文件的渲染限制，我们建议您点击下方链接，直接观看高清演示（带声音）：

👉 **[🎥 点击此处在线观看项目演示视频 (Watch Demo Video)](https://github.com/wumih/qiniuyun-question-1/blob/main/assets/demo.mp4)**

---
## ⚠️ 核心注意事项（关于视觉多模态能力）

**请注意：默认接入的文本大模型（如 DeepSeek `deepseek-chat` 等）在物理层面仅支持纯文本输入，无法“看到”摄像头的画面！**

虽然本项目已经内置了调用摄像头、截取帧画面并转为 Base64 图片格式发送的完整代码流（且界面提供了「👁 视觉开启」切换开关），但**如果您的 API 模型本身不支持视觉（Vision），强制发送带有图片数据的请求会导致 API 返回报错 (400 Bad Request)。**

*若要在 Hackathon 路演中展示真实的视觉对话能力，请在初次打开页面弹出的配置窗口中，将 API Base URL 和模型名称替换为真实支持多模态的厂商接口（如阿里云的 `qwen-vl-plus`，或 OpenAI 的 `gpt-4o`）。*

---

## ✨ 核心功能升级 (最新版本)

### 🚀 流式极速响应 (Streaming Response)
- 启用了 Server-Sent Events (SSE) 流式传输。
- 界面如同打字机般实时逐字渲染 AI 推理结果，将首字延迟 (TTFB) 降至极低，拒绝演示时的漫长等待焦虑。
- **动态切片播报 (Chunked TTS)**：无需等待长篇大论生成完毕，AI 一旦生成断句的标点符号（逗号、句号等），系统便会立刻触发原生语音朗读，边写边说。

### 🔄 多轮上下文记忆 (Multi-turn Memory)
- 突破单轮对话限制，系统后台默默维护对话队列。
- 支持基于上下文的连贯提问，生成的回答以专属发光气泡渲染，并伴随自动滚动体验。

### 🕹️ 手动精准控制流 (Manual Control Flow)
- 彻底抛弃了在嘈杂路演现场极易误触发的自动触发机制，改为最稳妥的**全手动按钮驱动**工作流（开始录音 -> 停止录音 -> 校对文本 -> 确认发送）。
- **可二次编辑的输入区**：语音识别转译出的文字支持在发送前，直接通过键盘进行二次删改与补全，大幅弥补语音识别不准确带来的尴尬。

### 🧩 现代企业级代码解耦 (Component Architecture)
- 原本混合了上千行逻辑的 `App.vue` 巨无霸文件已被完全解构。
- 逻辑被剥离为高复用性的独立 Hook (`useLLM`, `useCamera`, `useWebSpeech`)，UI 层拆分为独立的积木组件，即便比赛现场临时要求加功能也绝不会牵一发而动全身。

### ❄️ 全局紧急冻结 (System Freeze)
- 演示现场的终极“一键刹车”：一键瞬间截断进行中的 API 网络请求、切断 TTS 音频播放流、强制暂停摄像头画面并中断拾音。再次点击立刻复原。

---

## 🛠️ 极速启动

**环境要求**：Node.js ≥ 18、Chrome / Edge 浏览器（必须，Firefox 不支持原生 Web Speech API）。

```bash
# 安装依赖
npm install

# 启动开发服务器
npm run dev
```

首次访问 `http://localhost:5173/` 时，在初始化界面中填入对应供应商的 API Key 即可启动。

---

## 🏗️ 架构拓扑 (现存状态)

```text
src/
├── store/
│   └── appState.js           # 跨组件共享的全局响应式状态中心
├── composables/
│   ├── useCamera.js          # 摄像头唤醒、切片截帧、定格控制
│   ├── useLLM.js             # 封装 OpenAI 标准协议发包与流式解析机制
│   └── useWebSpeech.js       # 封装 STT(录音转译) 与 TTS(排队式朗读) API
├── components/
│   ├── SettingsModal.vue     # 首屏授权及配置初始化模块
│   ├── CameraFeed.vue        # 摄像头独立渲染器与 VAD 环形动画
│   └── InteractionPanel.vue  # 包含聊天气泡流、输入修改区及底部控制面板
└── App.vue                   # 根路由、子组件统筹及悬浮头部控制栏
```

---

## 🛡️ 演示安全性与隐私

- **零硬编码**：不随 Git 泄露任何秘钥，仅存于当前浏览器的 `localStorage` 中。
- **纯前端直连**：没有任何中间商后端或代理服务器（如果直连官方端口的话），避免数据中转带来的隐私争议。

---

## 📄 License

[MIT](LICENSE)
