# CLAUDE.md - AI Assistant 语音聊天项目

## 项目简介
端到端语音聊天助手/陪伴机器人项目，提供语音输入 → 智能对话 → 语音输出的完整交互体验。

基于 **小智 AI** 开源生态，本项目实现 Python 服务端，配合 ESP32 硬件端 (`SDK/xiaozhi-esp32`) 使用。

## 开发环境

### Python 服务端 (重要: 使用 UV)
**必须使用 UV 管理虚拟环境和运行 Python 代码**
- 创建环境: `uv venv`
- 安装依赖: `uv pip install <package>` 或 `uv sync`
- 运行代码: `uv run python <script.py>`
- 运行测试: `uv run pytest`

### ESP32 硬件端
- SDK 位置: `SDK/xiaozhi-esp32/`
- 开发工具: VSCode + ESP-IDF 插件 (v5.4+)
- 协议文档: `SDK/xiaozhi-esp32/docs/`

### 快速命令
```bash
# 初始化项目
uv venv && uv sync

# 运行服务端
uv run python src/main.py

# 代码质量
uv run ruff format .
uv run ruff check .
uv run pytest
```

## API 配置说明

### LLM 模型选择
| 场景 | 模型 | 说明 |
|------|------|------|
| 一般对话 | glm-4.5-flash | 快速响应，适合实时对话 |
| 摘要/总结 | glm-4-flash-250414 | 适合文本处理任务 |

### 服务端点
- LLM: `https://open.bigmodel.cn/api/paas/v4`
- Embedding: `https://ai.gitee.com/v1`
- Rerank: `https://ai.gitee.com/v1/rerank`

## 文件边界

### 可安全编辑
- `src/` - Python 服务端源代码
- `tests/` - 测试代码
- `docs/` - 文档
- `*.md` - Markdown 文档

### 谨慎修改
- `SDK/xiaozhi-esp32/` - ESP32 硬件端源码 (C++ 代码)

### 禁止修改
- `.venv/` - 虚拟环境
- `__pycache__/` - Python 缓存
- `.env` - 包含敏感的 API 密钥
- `uv.lock` - 依赖锁定文件 (除非明确需要更新)

## 关键协议文档

需要实现服务端时，参考以下文档:
- `SDK/xiaozhi-esp32/docs/websocket.md` - WebSocket 通信协议
- `SDK/xiaozhi-esp32/docs/mcp-protocol.md` - MCP 设备控制协议
- `SDK/xiaozhi-esp32/docs/mcp-usage.md` - MCP 使用示例

云端服务参考实现:
- `SDK/xiaozhi-esp32-server/main/xiaozhi-server/config.yaml` - 完整配置参考
- `SDK/xiaozhi-esp32-server/main/xiaozhi-server/core/` - 核心实现代码
- `SDK/xiaozhi-esp32-server/main/xiaozhi-server/core/providers/` - ASR/LLM/TTS 提供者

## 代码规范

### 风格指南
- 使用类型注解
- 遵循 PEP 8
- 中文注释和文档字符串
- 异步函数使用 `async/await`

### 示例代码
```python
# 正确: 完整的类型注解和文档
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

# 错误: 缺少类型注解和文档
async def chat(msg, model="glm-4.5-flash"):
    ...
```

## 工作流程

### 添加新功能
1. 在 `src/` 下创建相应模块
2. 编写单元测试到 `tests/`
3. 运行 `uv run ruff check .` 检查代码
4. 运行 `uv run pytest` 确保测试通过

### 安装新依赖
```bash
uv pip install <package>
# 或添加到 pyproject.toml 后运行
uv sync
```

## 调试提示
- 使用 `uv run python -m debugpy ...` 进行调试
- 日志级别通过环境变量 `LOG_LEVEL` 控制
- API 调用失败时检查 `.env` 中的密钥配置
