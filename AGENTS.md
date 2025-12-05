# AGENTS.md - AI Assistant 语音聊天项目

> 本文档为 AI Agent 提供项目核心上下文。详细参考文档位于 `docs/agent_context/`。

## 项目概述

**全栈嵌入式 AI 语音助手项目**，基于 **小智 AI** 开源生态。

**核心流程**: 用户语音 → ASR → LLM → TTS → 语音播放

> **CRITICAL: 默认架构模式**
> 
> 本项目采用 **网关模式 (MQTT+UDP → WebSocket)** 作为默认通信架构。
> ESP32 本质上是 IoT 设备，网关模式更符合 IoT 设备的通信范式。

**硬件平台**: 立创·实战派 ESP32-S3 (详见 `docs/agent_context/hardware.md`)

## 项目结构

```
ai-assistant/
├── src/                    # Python 服务端源代码 (主要开发区)
├── tests/                  # 测试目录
├── docs/agent_context/     # AI Agent 详细参考文档
└── third_party/            # 第三方开源组件 (Git 子模块, 只读)
```

## 开发命令

```bash
# 克隆 (必须含子模块)
git clone --recursive <repo-url>
git submodule update --init --recursive

# Python 环境 (必须使用 uv)
uv venv && uv sync
uv run python <script.py>
uv run pytest

# 代码质量
uv run ruff format . && uv run ruff check .
uv run pytest --cov=src
```

## 文件边界

| 状态 | 路径 | 说明 |
|------|------|------|
| ✅ 可编辑 | `src/`, `tests/`, `docs/`, `*.md` | 主要开发区 |
| ⚠️ 谨慎 | `third_party/xiaozhi-esp32/` | ESP32 C++ 代码 |
| ❌ 禁止 | `third_party/*/`, `.venv/`, `.env`, `uv.lock` | 子模块/敏感文件 |

## 模型配置

### LLM

| 用途 | 模型 | API Host |
|------|------|----------|
| 对话 | `glm-4.5-flash` | `https://open.bigmodel.cn/api/paas/v4` |

### 向量化与重排序 (推荐: 混合部署)

| 组件      | 模型                   | 部署     | 延迟  |
| --------- | ---------------------- | -------- | ----- |
| Embedding | `bge-m3` (1024维)      | 远程 API | ~80ms |
| Reranker  | `bce-reranker-base_v1` | 本地 CPU | ~30ms |

> 详细选型说明见 `docs/agent_context/architecture.md` → 模型选型

## 代码规范

### CRITICAL: 语言偏好

> **中文为主，英文为辅**

- **中文**: 文档字符串、注释、提交消息
- **英文**: 变量名、函数名、类名、常量

### IMPORTANT: 必须遵守

- 所有函数必须有**类型注解**
- 文档字符串使用**中文**编写
- 遵循 **PEP 8** 风格指南
- 使用 **async/await** 异步模式

> 代码风格参考: `src/` 目录下现有代码

## Git 提交规范

格式: `<type>(<scope>): <subject>` (中文描述)

| Type | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 |
| `docs` | 文档 |
| `refactor` | 重构 |
| `test` | 测试 |
| `chore` | 杂务 |

### CRITICAL: 拆分提交

> **每个提交只做一件事**

必须拆分的场景:
- 不同模块的独立修改 → 分别提交
- 新功能 + Bug 修复 → `feat` + `fix`
- 代码 + 文档 → `feat/fix` + `docs`
- 重构 + 新功能 → `refactor` + `feat`

提交正文结构: **WHAT → WHY → HOW**

### PR 规范

- 标题格式: `[模块名] 简要描述`
- 提交前: `uv run ruff check . && uv run pytest`

---

## 渐进式披露: 深入了解

当需要更详细信息时，阅读以下文档:

### 系统文档

| 需要了解 | 阅读文档 |
|----------|----------|
| 硬件平台 (ESP32-S3 规格/引脚) | `docs/agent_context/hardware.md` |
| 系统架构和部署 | `docs/agent_context/architecture.md` |
| 通信协议 (WebSocket/MCP/MQTT) | `docs/agent_context/protocols.md` |
| 第三方组件详解 | `docs/agent_context/third_party_reference.md` |

### 模块设计文档

| 模块 | 阅读文档 |
|------|----------|
| Mem0 记忆服务 (遗忘曲线/压缩/检索) | `docs/agent_context/design_memory.md` |
| LightRAG 知识库 (图谱/混合检索) | `docs/agent_context/design_knowledge.md` |
| Scheduler 定时提醒 (确认重试/话术) | `docs/agent_context/design_scheduler.md` |

### 参考实现

| 需要了解 | 阅读文档 |
|----------|----------|
| ESP32 协议原始文档 | `third_party/xiaozhi-esp32/docs/` |
| 云端服务参考实现 | `third_party/xiaozhi-esp32-server/main/xiaozhi-server/` |
