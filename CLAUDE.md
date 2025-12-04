# CLAUDE.md - AI Assistant 语音聊天项目

## 项目简介
端到端语音聊天助手/陪伴机器人项目，提供语音输入 → 智能对话 → 语音输出的完整交互体验。

基于 **小智 AI** 开源生态，本项目包含五个核心组件：
- **硬件端** (ESP32): `SDK/xiaozhi-esp32` - 音频采集/播放、唤醒词检测
- **云端服务** (Python): `SDK/xiaozhi-esp32-server` - ASR/LLM/TTS 处理
- **MQTT 网关** (Node.js): `SDK/xiaozhi-mqtt-gateway` - 协议桥接
- **声纹识别** (Python): `SDK/voiceprint-api` - 说话人识别
- **MCP 接入点** (Python): `SDK/mcp-endpoint-server` - 工具注册中心

### 系统架构
```
直连模式:  ESP32 ←──WebSocket+Opus──→ Python Cloud ←→ AI服务
网关模式:  ESP32 ←──MQTT+UDP──→ Gateway ←──WebSocket──→ Python Cloud
```

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

MQTT 网关核心文件:
- `SDK/xiaozhi-mqtt-gateway/app.js` - 主入口，MQTT/UDP 服务器，WebSocket 桥接
- `SDK/xiaozhi-mqtt-gateway/mqtt-protocol.js` - MQTT 3.1.1 协议解析与封装
- `SDK/xiaozhi-mqtt-gateway/utils/mqtt_config_v2.js` - 设备认证签名 (HMAC-SHA256)

声纹识别核心文件:
- `SDK/voiceprint-api/app/services/voiceprint_service.py` - 声纹提取/识别服务
- `SDK/voiceprint-api/app/api/v1/voiceprint.py` - REST API 接口
- `SDK/voiceprint-api/voiceprint.yaml` - 配置模板

MCP 接入点核心文件:
- `SDK/mcp-endpoint-server/src/server.py` - FastAPI 主服务
- `SDK/mcp-endpoint-server/src/core/connection_manager.py` - 连接管理和消息转发
- `SDK/mcp-endpoint-server/mcp-endpoint-server.cfg` - 配置文件

## 通信协议速查

### 直连模式 (WebSocket)
- 握手: `hello` 消息交换 (含 audio_params, session_id)
- 音频: Opus 编码，Binary 帧传输
- 控制: JSON 消息 (listen, stt, tts, llm, mcp, goodbye)

### 网关模式 (MQTT+UDP)
- MQTT 认证: `clientId=GID@@@MAC@@@UUID`, `password=HMAC-SHA256`
- UDP 加密: AES-128-CTR, 每会话随机密钥
- UDP Header: 16字节 (type, payloadLen, connId, timestamp, sequence)

### MCP 协议 (设备控制)
- 基于 JSON-RPC 2.0
- `tools/list`: 获取设备能力
- `tools/call`: 调用设备功能 (音量、灯光、GPIO)

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
