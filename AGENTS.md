# AGENTS.md - AI Assistant 语音聊天项目

> 本文档为 AI Agent 提供项目核心上下文。详细参考文档位于 `docs/agent_context/`。

## 项目概述

端到端语音聊天助手项目，基于 **小智 AI** 开源生态。

**核心流程**: 用户语音 → ASR识别 → LLM对话 → TTS合成 → 语音播放

## 项目结构

```
ai-assistant/
├── src/                    # 🔧 本项目 Python 服务端源代码 (主要开发区)
├── tests/                  # 测试目录
├── docs/                   # 文档目录
│   └── agent_context/      # AI Agent 详细参考文档
│       ├── sdk_reference.md    # SDK 组件详解
│       ├── protocols.md        # 通信协议详解
│       └── architecture.md     # 系统架构说明
├── SDK/                    # 参考实现 (第三方开源)
│   ├── xiaozhi-esp32/          # ESP32 硬件端 (C++)
│   ├── xiaozhi-esp32-server/   # Python 云端参考实现
│   ├── xiaozhi-mqtt-gateway/   # MQTT 网关 (Node.js)
│   ├── voiceprint-api/         # 声纹识别 (Python)
│   └── mcp-endpoint-server/    # MCP 接入点 (Python)
├── AGENTS.md               # 本文件
├── CLAUDE.md               # Claude 专用配置
└── README.md               # 项目说明
```

## 开发环境

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

- `SDK/xiaozhi-esp32/` - ESP32 C++ 代码

### ❌ 禁止修改

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

遵循 **Conventional Commits**，格式：`<emoji> <type>(<scope>): <subject>`

| Type | Emoji | 说明 |
|------|-------|------|
| `feat` | ✨ | 新功能 |
| `fix` | 🐛 | 修复 |
| `docs` | 📝 | 文档 |
| `refactor` | ♻️ | 重构 |
| `test` | ✅ | 测试 |
| `chore` | 🧹 | 杂务 |

**正文结构**: WHAT → WHY → HOW

```
✨ feat(asr): 支持火山引擎语音识别

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

| 需要了解                      | 阅读文档                                        |
| ----------------------------- | ----------------------------------------------- |
| SDK 各组件详细说明            | `docs/agent_context/sdk_reference.md`           |
| 通信协议 (WebSocket/MCP/MQTT) | `docs/agent_context/protocols.md`               |
| 系统架构和部署                | `docs/agent_context/architecture.md`            |
| ESP32 协议原始文档            | `SDK/xiaozhi-esp32/docs/`                       |
| 云端服务参考实现              | `SDK/xiaozhi-esp32-server/main/xiaozhi-server/` |
