# CLAUDE.md - AI Assistant 项目速查

> Claude 专用配置文件。完整指引见 `AGENTS.md`。

## CRITICAL: 默认架构

**网关模式 (MQTT+UDP → WebSocket)** 是默认通信架构。ESP32 是 IoT 设备。

```
ESP32 ──(MQTT控制 + UDP音频)──► 网关 ──(WebSocket)──► Python云端
```

## CRITICAL: 开发环境

**必须使用 UV** 管理 Python：

```bash
uv venv              # 创建环境
uv sync              # 安装依赖
uv run python <file> # 运行代码
uv run pytest        # 运行测试
```

## 代码质量

```bash
uv run ruff format . # 格式化
uv run ruff check .  # 检查
uv run pytest        # 测试
```

## 文件边界

| 状态 | 路径 | 说明 |
|------|------|------|
| ✅ 可编辑 | `src/`, `tests/`, `docs/`, `*.md` | 主要开发区 |
| ⚠️ 谨慎 | `SDK/xiaozhi-esp32/` | ESP32 C++ 代码 |
| ❌ 禁止 | `.venv/`, `__pycache__/`, `.env`, `uv.lock` | 系统/敏感文件 |

## API 速查

| 用途      | 模型                 |
| --------- | -------------------- |
| 对话      | `glm-4.5-flash`      |
| 摘要      | `glm-4-flash-250414` |
| Embedding | `bge-m3`             |
| Rerank    | `bge-reranker-v2-m3` |

## IMPORTANT: 代码规范

- **类型注解**: 所有函数必须有
- **文档字符串**: 中文编写
- **异步**: 使用 `async/await`
- **风格**: PEP 8

## Git 提交

格式：`<emoji> <type>(<scope>): <subject>`

| Type | Emoji | 说明 |
|------|-------|------|
| `feat` | ✨ | 新功能 |
| `fix` | 🐛 | 修复 |
| `docs` | 📝 | 文档 |
| `refactor` | ♻️ | 重构 |
| `test` | ✅ | 测试 |
| `chore` | 🧹 | 杂务 |

**正文**: WHAT → WHY → HOW

## 深入了解

| 主题     | 文档                                  |
| -------- | ------------------------------------- |
| SDK 详解 | `docs/agent_context/sdk_reference.md` |
| 通信协议 | `docs/agent_context/protocols.md`     |
| 系统架构 | `docs/agent_context/architecture.md`  |
