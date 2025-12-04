# 🎙️ AI Assistant - 语音聊天助手

一个端到端的语音聊天助手/陪伴机器人项目，提供自然的语音交互体验。

基于 [小智 AI](https://github.com/78/xiaozhi-esp32) 开源生态，实现 Python 服务端，配合 ESP32 硬件端使用。

## ✨ 功能特性

- 🎤 **语音输入** - 实时语音识别，支持中文对话
- 🤖 **智能对话** - 基于大语言模型的智能回复
- 🔊 **语音输出** - 自然语音合成，流畅的对话体验
- 📝 **文本处理** - 支持摘要、总结等文本任务
- 🔌 **硬件支持** - 兼容 70+ 种 ESP32 开发板
- 🌐 **MCP 协议** - 支持设备控制和智能家居联动

## 🏗️ 系统架构

本项目支持两种连接模式：

### 直连模式 (推荐)
ESP32 通过 WebSocket 直接连接云端服务：
```
┌─────────────────────┐   WebSocket + Opus   ┌─────────────────────┐
│    ESP32 硬件端      │ ◄──────────────────► │    Python 服务端     │
│  (xiaozhi-esp32)    │    双向音频+控制       │     (本项目)         │
│                     │                      │                     │
│  • 语音采集 (Mic)    │                      │  • ASR 语音识别      │
│  • 语音播放 (Speaker)│                      │  • LLM 对话生成      │
│  • 显示屏/LED        │                      │  • TTS 语音合成      │
│  • 唤醒词检测        │                      │  • MCP 设备控制      │
└─────────────────────┘                      └──────────┬──────────┘
                                                        │
                                                        ▼
                                             ┌─────────────────────┐
                                             │      AI 服务        │
                                             │  • 智谱 GLM         │
                                             │  • Gitee AI         │
                                             └─────────────────────┘
```

### 网关模式 (IoT 设备)
低功耗设备通过 MQTT 网关接入：
```
┌──────────────┐  MQTT+UDP  ┌──────────────┐  WebSocket  ┌──────────────┐
│  ESP32 设备   │ ◄────────► │  MQTT 网关    │ ◄─────────► │  Python 云端  │
│              │  (加密)     │              │             │              │
│ • 音频采集    │            │ • 协议转换    │             │ • ASR/LLM/TTS│
│ • MCP 工具   │            │ • 音频加解密  │             │ • MCP 调度   │
└──────────────┘            │ • 设备认证    │             └──────────────┘
                            └──────────────┘
```

## 🚀 快速开始

### 环境要求

- Python 3.11+
- [UV](https://github.com/astral-sh/uv) - 现代化的 Python 包管理器

### 安装 UV

```bash
# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### 项目设置

```bash
# 克隆项目
git clone <repository-url>
cd ai-assistant

# 创建虚拟环境并安装依赖
uv venv
uv sync

# 配置环境变量
cp .env.example .env
# 编辑 .env 文件，填入你的 API 密钥
```

### 运行项目

```bash
# 激活虚拟环境 (可选，uv run 会自动使用)
# Windows
.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate

# 运行主程序
uv run python src/main.py
```

## 📁 项目结构

```
ai-assistant/
├── AGENTS.md               # AI Agent 指导文件
├── CLAUDE.md               # Claude 专用配置
├── README.md               # 项目说明文档 (本文件)
├── .env.example            # 环境变量模板
├── .env                    # 实际环境变量 (需自行创建)
├── pyproject.toml          # 项目配置和依赖
├── src/                    # 本项目 Python 服务端源代码
├── SDK/                    # 参考实现 (Git Submodule)
│   ├── xiaozhi-esp32/      # ESP32 硬件端固件 (C++)
│   ├── xiaozhi-esp32-server/ # Python 云端参考实现
│   └── xiaozhi-mqtt-gateway/ # MQTT 网关 (Node.js)
├── tests/                  # 测试目录
└── docs/                   # 文档目录
```

### SDK 组件说明

| 组件 | 路径 | 说明 |
|------|------|------|
| **硬件端** | `SDK/xiaozhi-esp32` | ESP32 固件，支持 70+ 开发板 |
| **云端服务** | `SDK/xiaozhi-esp32-server` | 完整 Python 服务端参考实现 |
| **MQTT 网关** | `SDK/xiaozhi-mqtt-gateway` | MQTT+UDP → WebSocket 协议桥接 |

## ⚙️ 配置说明

### 环境变量

复制 `.env.example` 为 `.env` 并配置以下变量：

| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `LLM_API_KEY` | 智谱 AI API 密钥 | `your-api-key` |
| `EMBEDDING_API_KEY` | Gitee AI Embedding 密钥 | `your-api-key` |
| `RERANK_API_KEY` | Gitee AI Rerank 密钥 | `your-api-key` |

### 模型配置

| 用途 | 模型 | 说明 |
|------|------|------|
| 一般对话 | `glm-4.5-flash` | 快速响应，适合实时对话 |
| 摘要总结 | `glm-4-flash-250414` | 适合文本处理任务 |
| 文本嵌入 | `bge-m3` | 1024 维向量 |
| 重排序 | `bge-reranker-v2-m3` | 语义相关性排序 |

### 云端服务支持的模块 (参考 SDK)

| 模块 | 支持的平台 |
|------|---------------|
| **ASR** | FunASR(本地)、火山引擎、讯飞、阿里云、腾讯、百度 |
| **LLM** | 智谱 GLM、阿里百炼、电豆 Doubao、DeepSeek、Gemini、Ollama |
| **TTS** | EdgeTTS(免费)、火山引擎、阿里云、腾讯、讯飞、FishSpeech |
| **VAD** | SileroVAD(本地) |
| **Memory** | mem0ai、本地短期记忆 |
| **Intent** | function_call、intent_llm |

### MQTT 网关数据流

MQTT 网关将低功耗 IoT 设备的 MQTT+UDP 协议转换为 WebSocket，实现与云端服务的双向通信：

```
ESP32 ──MQTT(JSON)──► Gateway ──WebSocket──► Python Cloud
      ──UDP(Opus)───►         ──Binary────►
      ◄──MQTT(JSON)──         ◄──WebSocket──
      ◄──UDP(Opus)───         ◄──Binary────
```

**特点:**
- 🔐 **HMAC-SHA256 设备认证** - 防止未授权设备接入
- 🔒 **AES-128-CTR 音频加密** - 每会话随机密钥
- 📡 **UDP 低延迟传输** - 60ms 帧，支持序列号纠错
- 🛠️ **REST API 设备管理** - 远程下发指令、查询状态

## 🧪 开发指南

### 代码质量

```bash
# 格式化代码
uv run ruff format .

# 检查代码
uv run ruff check .

# 运行测试
uv run pytest

# 查看测试覆盖率
uv run pytest --cov=src
```

### 添加依赖

```bash
# 安装单个包
uv pip install <package-name>

# 同步所有依赖
uv sync
```

## 🔗 相关资源

### 小智 AI 生态
- [xiaozhi-esp32](https://github.com/78/xiaozhi-esp32) - ESP32 硬件端固件
- [xiaozhi.me](https://xiaozhi.me) - 官方云服务
- [小智百科全书](https://ccnphfhqs21z.feishu.cn/wiki/F5krwD16viZoF0kKkvDcrZNYnhb) - 详细教程

### 第三方服务端实现
- [xinnan-tech/xiaozhi-esp32-server](https://github.com/xinnan-tech/xiaozhi-esp32-server) - Python
- [joey-zhou/xiaozhi-esp32-server-java](https://github.com/joey-zhou/xiaozhi-esp32-server-java) - Java
- [AnimeAIChat/xiaozhi-server-go](https://github.com/AnimeAIChat/xiaozhi-server-go) - Golang

## 📄 许可证

MIT License

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m '✨ feat: 添加某功能'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 创建 Pull Request
