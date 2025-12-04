# AGENTS.md - AI Assistant 语音聊天项目

## 项目概述
这是一个端到端的语音聊天助手/陪伴机器人项目，支持语音输入、智能对话和语音输出。

基于 **小智 AI** 开源生态，本项目包含三个核心组件：
- 硬件端 (ESP32): `SDK/xiaozhi-esp32`
- 云端服务 (Python): `SDK/xiaozhi-esp32-server`
- MQTT 网关 (Node.js): `SDK/xiaozhi-mqtt-gateway`

### 系统架构
```
┌───────────────────┐              ┌─────────────────────────────┐
│   ESP32 硬件端     │  WebSocket   │      Python 云端服务          │
│  xiaozhi-esp32    │ ◄──────────► │   xiaozhi-esp32-server       │
│                   │  Opus Audio  │                             │
│  • 音频采集/播放    │              │  • ASR: FunASR/火山/讯飞     │
│  • 唤醒词检测      │              │  • LLM: 智谱/电豆/DeepSeek   │
│  • 显示屏/LED      │              │  • TTS: Edge/火山/阿里云    │
│  • MCP 设备控制    │              │  • VAD: SileroVAD          │
└───────────────────┘              │  • 意图识别/记忆/插件      │
        │                          └─────────────────────────────┘
        │ MQTT+UDP (IoT控制)                    │
        ▼                                      ▼
┌───────────────────┐              ┌─────────────────────┐
│    MQTT 网关        │              │      AI 服务           │
│ xiaozhi-mqtt-gw   │              │  • 智谱 GLM            │
│                   │              │  • 电豆 Doubao         │
│  • 设备指令下发     │              │  • Gitee AI           │
│  • WebSocket桥接   │              │  • 火山引擎            │
└───────────────────┘              └─────────────────────┘
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
│   └── xiaozhi-mqtt-gateway/ # MQTT 网关 (Node.js)
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
- 核心代码: `core/` 目录

### xiaozhi-mqtt-gateway (MQTT 网关)
- 路径: `SDK/xiaozhi-mqtt-gateway`
- 语言: Node.js
- 功能: MQTT+UDP 到 WebSocket 桥接，设备指令下发
- 核心文件:
  - `app.js` - 主入口，MQTT/UDP 服务器，WebSocket 桥接
  - `mqtt-protocol.js` - MQTT 3.1.1 协议解析与封装
  - `utils/mqtt_config_v2.js` - 设备认证签名生成与验证

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

## PR 指令
- 标题格式: `[模块名] 简要描述`
- 提交前运行: `uv run ruff check . && uv run pytest`
- 确保所有测试通过
