# 第三方组件参考

> 本文档包含 AI Assistant 项目使用的第三方开源组件详细说明。

## 组件总览

| 组件 | 类型 | 用途 | 路径 |
|------|------|------|------|
| xiaozhi-esp32 | 硬件端 | ESP32 固件 | `third_party/xiaozhi-esp32/` |
| xiaozhi-esp32-server | 云端参考 | Python 后端参考实现 | `third_party/xiaozhi-esp32-server/` |
| xiaozhi-mqtt-gateway | 网关 | MQTT+UDP 网关 | `third_party/xiaozhi-mqtt-gateway/` |
| voiceprint-api | 声纹服务 | 说话人识别 | `third_party/voiceprint-api/` |
| mcp-endpoint-server | MCP 接入 | MCP 协议接入点 | `third_party/mcp-endpoint-server/` |
| mem0 | AI 记忆 | 对话记忆管理 | `third_party/mem0/` |
| LightRAG | 知识图谱 | RAG + 知识图谱 | `third_party/LightRAG/` |
| bullmq | 任务队列 | Redis 消息队列 | `third_party/bullmq/` |

---

## xiaozhi-esp32 (硬件端)

> 小智 AI 聊天机器人 ESP32 固件，支持多种开发板。

### 核心模块

| 模块 | 路径 | 功能 |
|------|------|------|
| Application | `main/application.cc` | 主应用状态机 |
| AudioCodec | `main/audio_codecs/` | 音频编解码 (Opus) |
| Board | `main/boards/` | 硬件抽象层 |
| Protocol | `main/protocols/` | 通信协议 (MQTT/WebSocket) |
| IoT | `main/iot/` | 智能设备控制 |

### 关键接口

```cpp
// 音频输入回调
void OnAudioInput(std::vector<int16_t>& data);

// 音频输出回调
void OnAudioOutput(std::vector<int16_t>& data);

// 聊天状态变化
void SetChatState(ChatState state);

// IoT 设备控制
void OnIoTCommand(const cJSON* command);
```

### 参考文档

- 协议文档: `third_party/xiaozhi-esp32/docs/websocket_protocol_*.md`
- 开发指南: `third_party/xiaozhi-esp32/docs/development-guide.md`

---

## xiaozhi-esp32-server (云端参考)

> Python 实现的云端服务，包含 ASR/LLM/TTS 完整链路。

### 核心组件

```
main/xiaozhi-server/
├── core/
│   ├── connection.py      # WebSocket 连接管理
│   ├── handler.py         # 消息处理器
│   └── providers/         # AI 服务提供者
│       ├── asr/           # 语音识别
│       ├── llm/           # 大语言模型
│       └── tts/           # 语音合成
└── config/
    └── config.yaml        # 配置文件
```

### Provider 接口模式

```python
# ASR Provider 接口
class ASRProviderBase:
    async def transcribe(self, audio: bytes) -> str: ...

# LLM Provider 接口
class LLMProviderBase:
    async def chat(self, messages: list) -> str: ...
    async def chat_stream(self, messages: list) -> AsyncIterator[str]: ...

# TTS Provider 接口
class TTSProviderBase:
    async def synthesize(self, text: str) -> bytes: ...
```

---

## xiaozhi-mqtt-gateway (MQTT 网关)

> Node.js 实现的 MQTT+UDP 网关，桥接 ESP32 设备与云端服务。

### 架构

```
ESP32 设备
    │
    ├── MQTT ──► 网关 ──► WebSocket ──► Python 云端
    │   (控制)
    └── UDP ───► 网关 ──► WebSocket ──► Python 云端
        (音频)
```

### 核心文件

- `app.js` - 网关主程序
- `mqtt-protocol.js` - MQTT 协议处理
- `config/` - 配置目录

### 配置示例

```javascript
// config/default.js
module.exports = {
  mqtt: {
    broker: 'mqtt://localhost:1883',
    clientId: 'gateway-' + process.pid
  },
  websocket: {
    serverUrl: 'ws://localhost:8080/ws'
  },
  udp: {
    port: 5000
  }
};
```

---

## voiceprint-api (声纹识别)

> 基于 SpeechBrain 的声纹识别服务。

### API 端点

```yaml
POST /register:
  description: 注册声纹
  body: { user_id, audio_file }

POST /verify:
  description: 验证声纹
  body: { user_id, audio_file }

POST /identify:
  description: 识别说话人
  body: { audio_file }
```

### 启动服务

```bash
cd third_party/voiceprint-api
pip install -r requirements.txt
python start_server.py
```

---

## mcp-endpoint-server (MCP 接入点)

> MCP (Model Context Protocol) 协议接入点服务器。

### 用途

- 为 LLM 提供外部工具调用能力
- 支持动态注册 MCP 工具
- 管理工具执行和结果返回

### 核心接口

```python
# 注册工具
async def register_tool(name: str, handler: Callable): ...

# 执行工具
async def execute_tool(name: str, params: dict) -> Any: ...
```

---

## mem0 (AI 记忆管理)

> 为 AI 应用提供智能记忆层。

### 核心概念

- **Memory**: 存储和检索对话记忆
- **User**: 用户身份关联
- **Metadata**: 记忆元数据

### 核心 API

```python
from mem0 import Memory

# 初始化
memory = Memory()

# 添加记忆
memory.add(
    messages="用户喜欢咖啡",
    user_id="user123",
    metadata={"category": "preference"}
)

# 搜索记忆
results = memory.search(
    query="用户的饮品偏好",
    user_id="user123"
)

# 获取所有记忆
all_memories = memory.get_all(user_id="user123")

# 更新记忆
memory.update(memory_id="xxx", data="用户喜欢拿铁咖啡")

# 删除记忆
memory.delete(memory_id="xxx")
memory.delete_all(user_id="user123")
```

### 配置选项

```python
config = {
    "llm": {
        "provider": "openai",
        "config": {"model": "gpt-4o-mini"}
    },
    "embedder": {
        "provider": "openai",
        "config": {"model": "text-embedding-3-small"}
    },
    "vector_store": {
        "provider": "qdrant",
        "config": {"host": "localhost", "port": 6333}
    }
}

memory = Memory.from_config(config)
```

### MCP 服务封装设计 (P0)

```python
# 目标: src/mcp/memory_service.py
class MemoryMCPService:
    """mem0 MCP 服务封装"""
    
    tools = [
        "memory_add",      # 添加记忆
        "memory_search",   # 搜索记忆
        "memory_get_all",  # 获取所有记忆
        "memory_update",   # 更新记忆
        "memory_delete",   # 删除记忆
    ]
```

---

## LightRAG (知识图谱 RAG)

> 结合知识图谱的 RAG 系统，提供更精准的知识检索。

### 核心特性

- **双模式检索**: 向量 + 图谱混合检索
- **知识图谱**: 自动构建实体关系图
- **增量更新**: 支持知识库增量更新

### 核心 API

```python
from lightrag import LightRAG, QueryParam

# 初始化
rag = LightRAG(
    working_dir="./rag_storage",
    llm_model_func=llm_model_func,
    embedding_func=embedding_func,
)

# 插入文档
await rag.ainsert(["文档内容1", "文档内容2"])

# 查询 - 支持多种模式
# naive: 仅向量检索
# local: 局部图谱检索
# global: 全局图谱检索
# hybrid: 混合检索 (推荐)
result = await rag.aquery(
    "查询问题",
    param=QueryParam(mode="hybrid")
)

# 删除实体
await rag.adelete_by_entity("实体名称")
```

### 检索模式详解

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| naive | 纯向量检索 | 简单语义匹配 |
| local | 局部子图检索 | 实体关系查询 |
| global | 全局图谱检索 | 跨文档推理 |
| hybrid | 混合检索 | 通用场景 (推荐) |

### MCP 服务封装设计 (P2)

```python
# 目标: src/mcp/knowledge_service.py
class KnowledgeMCPService:
    """LightRAG MCP 服务封装"""
    
    tools = [
        "knowledge_insert",  # 插入知识
        "knowledge_query",   # 查询知识
        "knowledge_delete",  # 删除知识
        "knowledge_stats",   # 知识库统计
    ]
```

---

## bullmq (任务队列)

> 基于 Redis 的高性能消息队列，支持 TypeScript 和 Python。

### 核心概念

- **Queue**: 任务队列
- **Worker**: 任务处理器
- **Job**: 单个任务
- **Flow**: 任务流 (父子任务)

### Python API

```python
from bullmq import Queue, Worker

# 创建队列
queue = Queue("my-queue", {"connection": redis_connection})

# 添加任务
await queue.add("job-name", {"data": "value"})

# 添加延迟任务
await queue.add("delayed-job", {"data": "value"}, {"delay": 5000})

# 添加定时任务 (Cron)
await queue.upsertJobScheduler(
    "scheduler-id",
    {"pattern": "0 9 * * *"},  # 每天 9 点
    {"name": "daily-task", "data": {"type": "daily"}}
)

# 创建 Worker 处理任务
async def process_job(job, token):
    print(f"Processing {job.name}: {job.data}")
    return {"result": "done"}

worker = Worker("my-queue", process_job, {"connection": redis_connection})
```

### 任务调度选项

```python
# 延迟执行
{"delay": 5000}  # 5 秒后执行

# 重试策略
{
    "attempts": 3,
    "backoff": {"type": "exponential", "delay": 1000}
}

# 优先级
{"priority": 1}  # 数字越小优先级越高

# Cron 表达式
{"pattern": "*/5 * * * *"}  # 每 5 分钟
```

### MCP 服务封装设计 (P1)

```python
# 目标: src/mcp/scheduler_service.py
class SchedulerMCPService:
    """BullMQ MCP 服务封装 - 定时任务"""
    
    tools = [
        "schedule_create",    # 创建定时任务
        "schedule_list",      # 列出定时任务
        "schedule_delete",    # 删除定时任务
        "schedule_pause",     # 暂停定时任务
        "schedule_resume",    # 恢复定时任务
        "job_add",            # 添加一次性任务
        "job_status",         # 查询任务状态
    ]
```

---

## 参考资源

| 组件 | 文档链接 |
|------|----------|
| mem0 | [官方文档](https://docs.mem0.ai/) |
| LightRAG | `third_party/LightRAG/README.md` |
| BullMQ | [官方文档](https://docs.bullmq.io/) |
| xiaozhi-esp32 | `third_party/xiaozhi-esp32/docs/` |
