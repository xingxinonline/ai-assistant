# AGENTS.md - AI Assistant 语音聊天项目

## 项目概述
这是一个端到端的语音聊天助手/陪伴机器人项目，支持语音输入、智能对话和语音输出。

基于 **小智 AI** 开源生态，本项目包含五个核心组件：
- 硬件端 (ESP32): `SDK/xiaozhi-esp32`
- 云端服务 (Python): `SDK/xiaozhi-esp32-server`
- MQTT 网关 (Node.js): `SDK/xiaozhi-mqtt-gateway`
- 声纹识别 (Python): `SDK/voiceprint-api`
- MCP 接入点 (Python): `SDK/mcp-endpoint-server`

### 系统架构

**直连模式 (WebSocket):**
```
┌───────────────────┐    WebSocket + Opus     ┌─────────────────────────────┐
│   ESP32 硬件端     │ ◄─────────────────────► │      Python 云端服务          │
│  xiaozhi-esp32    │     双向音频+控制         │   xiaozhi-esp32-server       │
│                   │                         │                             │
│  • 音频采集/播放    │                         │  • ASR: FunASR/火山/讯飞     │
│  • 唤醒词检测      │                         │  • LLM: 智谱/电豆/DeepSeek   │
│  • 显示屏/LED      │                         │  • TTS: Edge/火山/阿里云     │
│  • MCP 设备控制    │                         │  • VAD: SileroVAD          │
└───────────────────┘                         │  • 意图识别/记忆/插件       │
                                              └──────────────┬──────────────┘
                                                             │
                                                             ▼
                                              ┌─────────────────────────────┐
                                              │         AI 服务              │
                                              │  • 智谱 GLM  • 电豆 Doubao   │
                                              │  • Gitee AI  • 火山引擎      │
                                              └─────────────────────────────┘
```

**全模块部署架构:**
```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                              全模块部署架构                                     │
├───────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌───────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│  │ ESP32 设备  │     │  MQTT 网关    │     │ Python Server │     │   智控台      │      │
│  │           │ ◄─► │ :1883/:8007  │ ◄─► │    :8000     │ ◄─► │   :8002     │      │
│  │ • 音频采集   │     │ • 协议转换    │     │ • ASR/LLM/TTS │     │ • 用户管理   │      │
│  │ • MCP 工具  │     │ • 音频加解密  │     │ • VAD/Intent  │     │ • 模型配置   │      │
│  └───────────┘     │ • 设备认证    │     │ • Memory      │     │ • 参数配置   │      │
│        │           └─────────────┘     │ • Plugins     │     │ • OTA 升级   │      │
│        │                               └──────┬──────┘     └─────────────┘      │
│        │                                      │                                  │
│        ▼                                      ▼                                  │
│  ┌───────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│  │ MCP 接入点  │     │  声纹识别     │     │   MySQL      │     │    Redis     │      │
│  │   :8004    │     │   :8005      │     │   :3306      │     │    :6379     │      │
│  │           │     │              │     │              │     │              │      │
│  │ • 工具注册   │     │ • 3D-Speaker │     │ • 用户数据    │     │ • 会话缓存   │      │
│  │ • 消息转发   │     │ • 说话人识别  │     │ • 设备绑定    │     │ • Token缓存  │      │
│  └───────────┘     └─────────────┘     └─────────────┘     └─────────────┘      │
│                                                                                 │
└───────────────────────────────────────────────────────────────────────────────────┘
```

**网关模式 (MQTT+UDP → WebSocket):**
```
┌───────────────────┐                     ┌───────────────────┐                     ┌─────────────────────┐
│   ESP32 硬件端     │   MQTT (JSON控制)   │    MQTT 网关       │   WebSocket (JSON)  │   Python 云端服务    │
│  xiaozhi-esp32    │ ◄────────────────► │ xiaozhi-mqtt-gw   │ ◄────────────────► │ xiaozhi-esp32-server│
│                   │   UDP (加密Opus)    │                   │   WebSocket (Binary)│                     │
│  • 音频采集/播放    │ ◄────────────────► │  • 协议转换        │ ◄────────────────► │  • ASR/LLM/TTS      │
│  • 唤醒词检测      │                     │  • 音频加解密      │                     │  • MCP 调度         │
│  • MCP 设备控制    │                     │  • 设备认证        │                     │  • 意图/记忆        │
└───────────────────┘                     │  • REST API       │                     └─────────────────────┘
                                          └───────────────────┘
```

### 核心组件
- **服务端 (Python)**: 本项目，处理 ASR → LLM → TTS 流程
- **硬件端 (ESP32)**: `SDK/xiaozhi-esp32`，处理音频采集和播放
- **通信协议**: WebSocket + Opus 音频编解码 + MCP 控制协议

## 开发环境

### Python 服务端 (UV 管理)
- 使用 **UV** 管理 Python 虚拟环境和依赖
- Python 版本: 3.11+
- 激活环境: `uv venv && .venv\Scripts\activate` (Windows)
- 安装依赖: `uv pip install -r requirements.txt` 或 `uv sync`
- 运行脚本: `uv run python <script.py>`

### ESP32 硬件端 (ESP-IDF)
- SDK 路径: `SDK/xiaozhi-esp32`
- 开发环境: ESP-IDF 5.4+
- 支持芯片: ESP32-C3, ESP32-S3, ESP32-P4
- 参考文档: `SDK/xiaozhi-esp32/README_zh.md`

### 常用命令
- 创建虚拟环境: `uv venv`
- 安装依赖: `uv pip install <package>`
- 运行测试: `uv run pytest`
- 代码格式化: `uv run ruff format .`
- 代码检查: `uv run ruff check .`

## API 配置

### LLM 服务
- 一般对话使用 `glm-4.5-flash` 模型
- 摘要/总结使用 `glm-4-flash-250414` 模型
- API Host: `https://open.bigmodel.cn/api/paas/v4`

### Embedding 服务
- 模型: `bge-m3`
- 维度: 1024
- API Host: `https://ai.gitee.com/v1`

### Rerank 服务
- 模型: `bge-reranker-v2-m3`
- API Host: `https://ai.gitee.com/v1/rerank`

## 项目结构
```
ai-assistant/
├── AGENTS.md               # AI Agent 指导文件
├── CLAUDE.md               # Claude 专用配置
├── README.md               # 项目说明文档
├── .env.example            # 环境变量模板
├── .env                    # 实际环境变量 (不提交到 Git)
├── .gitignore              # Git 忽略配置
├── pyproject.toml          # 项目配置和依赖
├── src/                    # Python 服务端源代码 (本项目开发)
│   ├── __init__.py
│   ├── main.py             # 服务端入口
│   ├── llm/                # LLM 对话模块
│   ├── asr/                # 语音识别模块
│   ├── tts/                # 语音合成模块
│   ├── protocol/           # WebSocket/MCP 协议
│   └── utils/              # 工具函数
├── SDK/                    # 参考实现 (第三方开源项目)
│   ├── xiaozhi-esp32/      # ESP32 硬件端固件 (C++)
│   ├── xiaozhi-esp32-server/ # Python 云端服务参考实现
│   ├── xiaozhi-mqtt-gateway/ # MQTT 网关 (Node.js)
│   ├── voiceprint-api/     # 声纹识别服务 (Python)
│   └── mcp-endpoint-server/ # MCP 接入点服务 (Python)
├── tests/                  # 测试目录
└── docs/                   # 文档目录
```

## SDK 参考实现

### xiaozhi-esp32 (硬件端)
- 路径: `SDK/xiaozhi-esp32`
- 语言: C++ (ESP-IDF 5.4+)
- 功能: 音频采集/播放、唤醒词检测、MCP 设备控制
- 协议文档: `SDK/xiaozhi-esp32/docs/websocket.md`

### xiaozhi-esp32-server (云端服务)
- 路径: `SDK/xiaozhi-esp32-server/main/xiaozhi-server`
- 语言: Python 3.10
- 功能: 完整的 ASR/LLM/TTS 流程实现
- 配置: `config.yaml` - 模块配置参考
- 核心目录:
  - `core/providers/` - ASR/LLM/TTS/VAD/VLLM 提供者实现
  - `core/handle/` - 连接处理和消息分发
  - `plugins_func/` - 插件函数 (天气/新闻/音乐等)
- 支持的模块:
  | 模块 | 支持平台 |
  |------|----------|
  | **ASR** | FunASR(本地), 火山引擎, 讯飞, 阿里云, 腾讯, 百度, OpenAI |
  | **LLM** | 智谱GLM, 阿里百炼, 电豆Doubao, DeepSeek, Gemini, Ollama, Dify, Coze |
  | **TTS** | EdgeTTS(免费), 火山引擎, 阿里云, 腾讯, 讯飞, FishSpeech, GPT-SoVITS |
  | **VAD** | SileroVAD(本地) |
  | **VLLM** | 智谱glm-4v-flash, 千问qwen2.5-vl |
  | **Memory** | mem0ai, 本地短期记忆 |
  | **Intent** | function_call, intent_llm |
- 全模块部署端口:
  | 服务 | 端口 | 说明 |
  |------|------|------|
  | xiaozhi-server | 8000 | WebSocket 服务 |
  | manager-api | 8002 | 智控台后端 (Java) |
  | manager-web | 8001 | 智控台前端 (Vue) |
  | MySQL | 3306 | 数据库 |
  | Redis | 6379 | 缓存 |

### manager-api (智控台后端)
- 路径: `SDK/xiaozhi-esp32-server/main/manager-api`
- 语言: Java (Spring Boot)
- 功能: 用户管理、设备绑定、模型配置、OTA 升级
- 核心模块:
  - `modules/agent/` - 智能体/角色管理
  - `modules/device/` - 设备管理
  - `modules/model/` - 模型配置 (ASR/LLM/TTS)
  - `modules/knowledge/` - 知识库管理
  - `modules/timbre/` - 音色管理
  - `modules/voiceclone/` - 语音克隆
  - `modules/security/` - 安全认证
  - `modules/sys/` - 系统配置
- 国际化: 支持中/英/德/越南语

### manager-web (智控台前端)
- 路径: `SDK/xiaozhi-esp32-server/main/manager-web`
- 语言: Vue.js
- 功能: Web 管理界面
- 核心页面:
  - `DeviceManagement.vue` - 设备管理
  - `ModelConfig.vue` - 模型配置
  - `AgentTemplateManagement.vue` - 智能体模板
  - `KnowledgeBaseManagement.vue` - 知识库管理
  - `VoiceCloneManagement.vue` - 语音克隆
  - `VoicePrint.vue` - 声纹管理
  - `OtaManagement.vue` - OTA 升级
  - `UserManagement.vue` - 用户管理

### manager-mobile (智控台移动版)
- 路径: `SDK/xiaozhi-esp32-server/main/manager-mobile`
- 语言: Vue 3 + TypeScript (uni-app)
- 功能: 跨端移动管理端 (App/H5/小程序)
- 平台兼容性:
  | H5 | iOS | Android | 微信小程序 |
  |----|-----|---------|-----------|
  | ✓  | ✓   | ✓       | ✓         |
- 核心功能:
  - `pages/agent/` - 智能体管理
  - `pages/device/` - 设备管理
  - `pages/device-config/` - 设备配网 (WiFi/超声波)
  - `pages/voiceprint/` - 声纹管理
  - `pages/chat-history/` - 聊天记录
- 技术栈: alova (请求库) + pinia (状态) + wot-design-uni (UI)
- 开发命令:
  ```bash
  pnpm dev:h5          # H5 开发
  pnpm dev:mp          # 微信小程序
  pnpm build:app       # App 构建
  ```

### xiaozhi-mqtt-gateway (MQTT 网关)
- 路径: `SDK/xiaozhi-mqtt-gateway`
- 语言: Node.js
- 功能: MQTT+UDP 到 WebSocket 桥接，设备指令下发
- 核心文件:
  - `app.js` - 主入口，MQTT/UDP 服务器，WebSocket 桥接
  - `mqtt-protocol.js` - MQTT 3.1.1 协议解析与封装
  - `utils/mqtt_config_v2.js` - 设备认证签名生成与验证

### voiceprint-api (声纹识别服务)
- 路径: `SDK/voiceprint-api`
- 语言: Python 3.10 + FastAPI
- 功能: 基于 3D-Speaker 模型的声纹识别，用于说话人识别
- 核心文件:
  - `app/services/voiceprint_service.py` - 声纹提取/注册/识别服务
  - `app/api/v1/voiceprint.py` - REST API 接口
  - `app/database/voiceprint_db.py` - MySQL 声纹存储
- API 接口:
  - `POST /voiceprint/register` - 注册声纹 (speaker_id + WAV音频)
  - `POST /voiceprint/identify` - 识别声纹 (返回 speaker_id + 相似度)
  - `DELETE /voiceprint/{speaker_id}` - 删除声纹
- 技术特点:
  - 模型: `iic/speech_campplus_sv_zh-cn_3dspeaker_16k` (ModelScope)
  - 特征计算: 余弦相似度
  - 音频处理: 自动重采样到 16kHz

### mcp-endpoint-server (MCP 接入点服务)
- 路径: `SDK/mcp-endpoint-server`
- 语言: Python 3.10 + FastAPI
- 功能: MCP 工具注册中心，转发小智端和工具端消息
- 核心文件:
  - `src/server.py` - FastAPI 主服务，WebSocket 端点
  - `src/core/connection_manager.py` - 连接管理和消息转发
  - `src/handlers/websocket_handler.py` - 工具端/小智端消息处理
  - `src/utils/aes_utils.py` - Token 加解密
- WebSocket 端点:
  - `WS /mcp_endpoint/mcp/` - 工具端连接 (注册 MCP 工具)
  - `WS /mcp_endpoint/call/` - 小智端连接 (调用 MCP 工具)
  - `GET /mcp_endpoint/health` - 健康检查和连接统计
- 架构设计:
  ```
  工具端 (MCP Server) ──WS──► MCP Endpoint ◄──WS── 小智端 (ESP32/Cloud)
       │                        │                      │
       │  注册工具列表            │  转发 JSON-RPC        │  调用工具
       └──────────────────────►◄──────────────────────┘
  ```

## 通信协议

### WebSocket 协议
详见 `SDK/xiaozhi-esp32/docs/websocket.md`

**握手流程:**
1. 设备连接 → 发送 `hello` 消息 (含 audio_params)
2. 服务端回复 `hello` 消息 (含 session_id)
3. 双向音频流 (Opus 编码) + JSON 控制消息

**消息类型:**
- `hello`: 握手消息
- `listen`: 开始/停止监听
- `stt`: 语音识别结果
- `tts`: 语音合成控制
- `llm`: 表情/情绪控制
- `mcp`: 设备控制协议

### MCP 协议 (设备控制)
详见 `SDK/xiaozhi-esp32/docs/mcp-protocol.md`

基于 JSON-RPC 2.0，支持:
- `tools/list`: 获取设备能力列表
- `tools/call`: 调用设备功能 (音量、灯光、GPIO 等)

### MQTT 网关协议 (IoT 设备接入)

网关将 IoT 设备常用的 MQTT+UDP 协议转换为 WebSocket 协议。

**完整数据流:**
```
┌─────────────────── 上行数据流 (用户语音 → AI) ───────────────────┐
│                                                                  │
│  ESP32 ──MQTT──► Gateway ──WebSocket──► Cloud                   │
│         (JSON控制)        (JSON控制)                              │
│                                                                  │
│  ESP32 ──UDP───► Gateway ──WebSocket──► Cloud                   │
│      (加密Opus)     (解密)    (Binary)                           │
│                                                                  │
├─────────────────── 下行数据流 (AI回复 → 用户) ───────────────────┤
│                                                                  │
│  Cloud ──WebSocket──► Gateway ──MQTT──► ESP32                   │
│        (JSON控制)            (JSON控制)                          │
│                                                                  │
│  Cloud ──WebSocket──► Gateway ──UDP───► ESP32                   │
│        (Binary)        (加密)   (加密Opus)                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**MQTT 设备认证:**
- `clientId`: `{GroupID}@@@{MAC地址}@@@{UUID}`
- `username`: Base64 编码的 JSON 用户数据
- `password`: `HMAC-SHA256(clientId + "|" + username, SIGNATURE_KEY)`

**UDP 音频加密:**
- 算法: AES-128-CTR
- 密钥: 每次会话生成随机 16 字节
- 帧格式: 16字节Header + 加密Opus数据

**UDP Header 格式 (16字节):**
| Offset | Size | Field |
|--------|------|-------|
| 0 | 1 | type (1=音频) |
| 2 | 2 | payloadLength |
| 4 | 4 | connectionId |
| 8 | 4 | timestamp (60ms/帧) |
| 12 | 4 | sequence |

**网关管理 API:**
```bash
# 设备指令下发 (MCP)
POST /api/commands/:clientId
Body: {type: "mcp", payload: {method: "tools/call", params: {...}}}

# 批量设备状态查询
POST /api/devices/status
Body: {clientIds: ["GID@@@MAC@@@UUID", ...]}
```

## 测试指令
- 运行全部测试: `uv run pytest`
- 运行指定测试: `uv run pytest tests/test_<name>.py -v`
- 查看覆盖率: `uv run pytest --cov=src`

## 代码规范
- 使用 Ruff 进行代码格式化和检查
- 所有函数需要类型注解
- 遵循 PEP 8 风格指南
- 使用中文编写文档字符串

## Git 提交规范 (Conventional Commits)

本项目遵循 **Conventional Commits** 规范，采用 **Commit-As-Prompt** 哲学，确保提交信息既适合人类阅读，也适合 AI 助手理解。

### 提交格式

```
<emoji> <type>(<scope>): <subject>

<WHAT>
<WHY>
<HOW>
```

### Type 与 Emoji 映射

| Type | Emoji | 说明 |
|------|-------|------|
| `feat` | ✨ | 新功能 |
| `fix` | 🐛 | 修复 Bug |
| `docs` | 📝 | 文档变更 |
| `style` | 🎨 | 代码格式 (不影响功能) |
| `refactor` | ♻️ | 代码重构 |
| `perf` | ⚡️ | 性能优化 |
| `test` | ✅ | 测试相关 |
| `build` | 🏗️ | 构建系统 |
| `ci` | 🤖 | CI/CD 配置 |
| `chore` | 🧹 | 杂务 (依赖更新等) |
| `revert` | ⏪ | 回滚提交 |
| `prompt` | 🧠 | AI 上下文/Prompt 相关 |

### 正文结构 (WHAT/WHY/HOW)

提交正文必须遵循三段式结构：

- **WHAT**: 一句话描述动作与对象
- **WHY**: 阐述业务目标、用户需求或缺陷背景（关联 Issue）
- **HOW**: 概述实现策略、兼容性影响、验证方式

### 示例

```
✨ feat(asr): 支持火山引擎语音识别

WHAT: 新增 VolcEngine ASR Provider 实现

WHY: 为满足高并发场景下的语音识别需求，接入火山引擎 ASR 服务。
关联需求 #123

HOW: 
- 实现 VolcEngineASRProvider 类，继承 BaseASRProvider
- 支持流式音频输入和实时转写
- 已通过单元测试和集成测试验证
```

## PR 指令
- 标题格式: `[模块名] 简要描述`
- 提交前运行: `uv run ruff check . && uv run pytest`
- 确保所有测试通过
