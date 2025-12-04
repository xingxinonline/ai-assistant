# AGENTS.md - AI Assistant 语音聊天项目

> 本文档为 AI Agent 提供项目核心上下文。详细参考文档位于 `docs/agent_context/`。

## 项目概述

**全栈嵌入式 AI 语音助手项目**，基于 **小智 AI** 开源生态。

**核心流程**: 用户语音 → ASR识别 → LLM对话 → TTS合成 → 语音播放

> **CRITICAL: 默认架构模式**
> 
> 本项目采用 **网关模式 (MQTT+UDP → WebSocket)** 作为默认通信架构。
> ESP32 本质上是 IoT 设备，网关模式更符合 IoT 设备的通信范式。

### 全栈分层架构 (网关模式)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         全栈架构 (默认: 网关模式)                              │
├──────────────────────────────────────────────────────────────────────────────┤
│  硬件层        │  网关层        │  云端层        │  应用层                    │
│  Hardware     │  Gateway      │  Cloud        │  Apps                      │
│               │               │               │                            │
│  ESP32-S3     │  MQTT网关      │  Python服务    │  Web/App/小程序            │
│  传感器/音频   │  (Node.js)    │  ASR/LLM/TTS  │  智控台                    │
│               │               │               │                            │
│  ┌─────────┐  │  ┌─────────┐  │  ┌─────────┐  │  ┌─────────┐               │
│  │ESP32设备 │──┼─►│MQTT+UDP │──┼─►│WebSocket│──┼─►│用户界面  │               │
│  └─────────┘  │  └─────────┘  │  └─────────┘  │  └─────────┘               │
└──────────────────────────────────────────────────────────────────────────────┘

IoT 设备通信链路:
  ESP32 ──(MQTT控制 + UDP音频)──► 网关 ──(WebSocket)──► Python云端
```

### 硬件平台

**立创·实战派 ESP32-S3** ([详细规格](docs/agent_context/hardware.md))

| 核心 | 规格 |
|------|------|
| 芯片 | ESP32-S3 双核 240MHz, 16MB Flash, 8MB PSRAM |
| 显示 | 2寸 IPS 触摸屏 (320×240) |
| 音频 | 双麦克风 + 喇叭 (ES8311/ES7210/NS4150B) |
| 传感器 | 六轴姿态 (QMI8658) |
| 通信 | Wi-Fi + BLE 5.0 |

## 项目结构

```
ai-assistant/
├── src/                    # 🔧 本项目 Python 服务端源代码 (主要开发区)
├── tests/                  # 测试目录
├── docs/                   # 文档目录
│   └── agent_context/      # AI Agent 详细参考文档
│       ├── hardware.md              # 硬件平台说明
│       ├── third_party_reference.md # 第三方组件详解
│       ├── protocols.md             # 通信协议详解
│       └── architecture.md          # 系统架构说明
├── third_party/            # 第三方开源组件 (Git 子模块)
│   ├── xiaozhi-esp32/          # ESP32 硬件端 (C++)
│   ├── xiaozhi-esp32-server/   # Python 云端参考实现
│   ├── xiaozhi-mqtt-gateway/   # MQTT 网关 (Node.js)
│   ├── voiceprint-api/         # 声纹识别 (Python)
│   ├── mcp-endpoint-server/    # MCP 接入点 (Python)
│   ├── mem0/                   # AI 记忆管理库 (Python)
│   ├── LightRAG/               # 知识图谱 RAG (Python)
│   └── bullmq/                 # Redis 任务队列 (Python/TS)
├── AGENTS.md               # 本文件
└── README.md               # 项目说明
```

## 开发环境

### CRITICAL: 克隆项目 (含子模块)

```bash
# 首次克隆 (必须加 --recursive)
git clone --recursive <repo-url>

# 已克隆后初始化子模块
git submodule update --init --recursive

# 更新子模块到最新
git submodule update --remote
```

### CRITICAL: 使用 UV 管理 Python

```bash
# 创建环境
uv venv

# 安装依赖
uv sync
# 或
uv pip install <package>

# 运行代码 - 必须使用 uv run
uv run python <script.py>
uv run pytest
```

### 代码质量检查

```bash
uv run ruff format .    # 格式化
uv run ruff check .     # 检查
uv run pytest           # 测试
uv run pytest --cov=src # 覆盖率
```

## 文件编辑边界

### ✅ 可安全编辑

- `src/` - Python 服务端源代码
- `tests/` - 测试代码
- `docs/` - 文档
- 根目录 `*.md` 文件

### ⚠️ 谨慎修改

- `third_party/xiaozhi-esp32/` - ESP32 C++ 代码

### ❌ 禁止修改

- `third_party/` - 第三方子模块 (只读参考)

- `.venv/` - 虚拟环境
- `__pycache__/` - Python 缓存
- `.env` - 敏感 API 密钥
- `uv.lock` - 依赖锁定 (除非明确需要)

## API 配置

| 用途      | 模型                 | API Host                               |
| --------- | -------------------- | -------------------------------------- |
| 一般对话  | `glm-4.5-flash`      | `https://open.bigmodel.cn/api/paas/v4` |
| 摘要总结  | `glm-4-flash-250414` | 同上                                   |
| Embedding | `bge-m3` (1024维)    | `https://ai.gitee.com/v1`              |
| Rerank    | `bge-reranker-v2-m3` | `https://ai.gitee.com/v1/rerank`       |

## 代码规范

### IMPORTANT: 必须遵守

- **类型注解**: 所有函数必须有类型注解
- **文档字符串**: 使用中文编写
- **PEP 8**: 遵循 Python 风格指南
- **异步**: 使用 `async/await`

### 代码示例

```python
async def chat_with_llm(
    message: str,
    model: str = "glm-4.5-flash",
    temperature: float = 0.7
) -> str:
    """
    与 LLM 进行对话。
    
    Args:
        message: 用户输入的消息
        model: 使用的模型名称
        temperature: 采样温度
        
    Returns:
        LLM 的回复内容
    """
    ...
```

## Git 提交规范

遵循 **Conventional Commits**，格式：`<type>(<scope>): <subject>`

| Type | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 |
| `docs` | 文档 |
| `refactor` | 重构 |
| `test` | 测试 |
| `chore` | 杂务 |

### CRITICAL: 拆分提交

> **每个提交应只做一件事**。如果一次任务涉及多个独立的改动，必须拆分为多个提交。

**必须拆分的场景**:
- 不同模块/文件的独立修改 → 分别提交
- 新功能 + Bug 修复 → 分别提交 (`feat` + `fix`)
- 代码修改 + 文档更新 → 分别提交 (`feat/fix` + `docs`)
- 重构 + 新功能 → 分别提交 (`refactor` + `feat`)
- 多个不相关的 Bug 修复 → 分别提交

**示例** (一次任务产生 3 个提交):
```bash
# 提交 1: 重构
git commit -m "refactor(asr): 抽取公共音频处理逻辑"

# 提交 2: 新功能
git commit -m "feat(asr): 支持火山引擎语音识别"

# 提交 3: 文档
git commit -m "docs: 更新 ASR 提供者配置说明"
```

### 提交正文结构

**WHAT → WHY → HOW**

```
feat(asr): 支持火山引擎语音识别

WHAT: 新增 VolcEngine ASR Provider 实现

WHY: 满足高并发场景需求，关联 #123

HOW: 
- 实现 VolcEngineASRProvider 类
- 支持流式音频输入
- 已通过单元测试验证
```

## PR 规范

- **标题格式**: `[模块名] 简要描述`
- **提交前**: `uv run ruff check . && uv run pytest`
- **确保**: 所有测试通过

---

## 渐进式披露：深入了解

当需要更详细的信息时，请阅读以下文档：

### 系统文档

| 需要了解 | 阅读文档 |
|----------|----------|
| **硬件平台** (ESP32-S3 规格/引脚) | `docs/agent_context/hardware.md` |
| 系统架构和部署 | `docs/agent_context/architecture.md` |
| 通信协议 (WebSocket/MCP/MQTT) | `docs/agent_context/protocols.md` |
| 第三方组件详细说明 | `docs/agent_context/third_party_reference.md` |

### 模块设计文档

| 模块                                         | 阅读文档                                 |
| -------------------------------------------- | ---------------------------------------- |
| **Mem0 记忆服务** (遗忘曲线/压缩/检索)       | `docs/agent_context/design_memory.md`    |
| **LightRAG 知识库** (图谱/混合检索/增量更新) | `docs/agent_context/design_knowledge.md` |
| **Scheduler 定时提醒** (确认重试/话术生成)   | `docs/agent_context/design_scheduler.md` |

### 参考实现

| 需要了解 | 阅读文档 |
|----------|----------|
| ESP32 协议原始文档 | `third_party/xiaozhi-esp32/docs/` |
| 云端服务参考实现 | `third_party/xiaozhi-esp32-server/main/xiaozhi-server/` |
