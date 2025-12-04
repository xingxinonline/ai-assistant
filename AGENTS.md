# AGENTS.md - AI Assistant 语音聊天项目

## 项目概述
这是一个端到端的语音聊天助手/陪伴机器人项目，支持语音输入、智能对话和语音输出。

### 系统架构
```
┌─────────────────┐     WebSocket      ┌─────────────────┐
│  ESP32 硬件端    │ ◄──────────────► │   Python 服务端   │
│  (xiaozhi-esp32) │     Opus Audio    │   (本项目)       │
└─────────────────┘                    └─────────────────┘
        │                                      │
        │ 语音输入/输出                          │ LLM/ASR/TTS
        ▼                                      ▼
    用户交互                              智谱/Gitee AI
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
├── AGENTS.md           # AI Agent 指导文件
├── CLAUDE.md           # Claude 专用配置
├── README.md           # 项目说明文档
├── .env.example        # 环境变量模板
├── .env                # 实际环境变量 (不提交到 Git)
├── .gitignore          # Git 忽略配置
├── pyproject.toml      # 项目配置和依赖
├── src/                # Python 服务端源代码
│   ├── __init__.py
│   ├── main.py         # 服务端入口
│   ├── llm/            # LLM 对话模块
│   ├── asr/            # 语音识别模块
│   ├── tts/            # 语音合成模块
│   ├── protocol/       # WebSocket/MCP 协议
│   └── utils/          # 工具函数
├── SDK/                # 硬件端 SDK
│   └── xiaozhi-esp32/  # 小智 ESP32 固件源码
│       ├── main/       # 主程序代码
│       ├── docs/       # 协议文档
│       └── ...         # ESP-IDF 项目结构
├── tests/              # 测试目录
└── docs/               # 文档目录
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
