# SDK 参考实现详�?

本文档包�?AI Assistant 项目所�?SDK 组件的详细说明。AI Agent 在需要深入了解某�?SDK 组件时应阅读此文档�?

## 组件总览

| 组件                     | 路径                       | 语言            | 用�?             |
| ------------------------ | -------------------------- | --------------- | ----------------- |
| **xiaozhi-esp32**        | `third_party/xiaozhi-esp32`        | C++             | ESP32 硬件端固�? |
| **xiaozhi-esp32-server** | `third_party/xiaozhi-esp32-server` | Python/Java/Vue | 云端服务 + 智控�?|
| **xiaozhi-mqtt-gateway** | `third_party/xiaozhi-mqtt-gateway` | Node.js         | MQTT 协议网关     |
| **voiceprint-api**       | `third_party/voiceprint-api`       | Python          | 声纹识别服务      |
| **mcp-endpoint-server**  | `third_party/mcp-endpoint-server`  | Python          | MCP 工具注册中心  |
| **mem0** 🆕             | `third_party/mem0`                 | Python          | AI 记忆管理�?    |
| **LightRAG**             | `third_party/LightRAG`             | Python          | 知识图谱 RAG     |
| **BullMQ** 🆕            | `third_party/bullmq`               | Python/TS       | Redis 任务队列     |

---

## xiaozhi-esp32 (硬件�?

### 基本信息
- **路径**: `third_party/xiaozhi-esp32`
- **语言**: C++ (ESP-IDF 5.4+)
- **支持芯片**: ESP32-C3, ESP32-S3, ESP32-P4
- **参考文�?*: `third_party/xiaozhi-esp32/README_zh.md`

### 功能
- 音频采集/播放
- 唤醒词检�?
- 显示�?LED 控制
- MCP 设备控制

### 关键文件
- `third_party/xiaozhi-esp32/docs/websocket.md` - WebSocket 通信协议
- `third_party/xiaozhi-esp32/docs/mcp-protocol.md` - MCP 设备控制协议
- `third_party/xiaozhi-esp32/docs/mcp-usage.md` - MCP 使用示例

---

## xiaozhi-esp32-server (云端服务)

### xiaozhi-server (Python 服务�?
- **路径**: `third_party/xiaozhi-esp32-server/main/xiaozhi-server`
- **语言**: Python 3.10
- **功能**: 完整�?ASR/LLM/TTS 流程实现

#### 核心目录
- `core/providers/` - ASR/LLM/TTS/VAD/VLLM 提供者实�?
- `core/handle/` - 连接处理和消息分�?
- `plugins_func/` - 插件函数 (天气/新闻/音乐�?

#### 配置参�?
- `config.yaml` - 模块配置参�?

#### 支持的模�?

| 模块       | 支持平台                                                            |
| ---------- | ------------------------------------------------------------------- |
| **ASR**    | FunASR(本地), 火山引擎, 讯飞, 阿里�? 腾讯, 百度, OpenAI            |
| **LLM**    | 智谱GLM, 阿里百炼, 电豆Doubao, DeepSeek, Gemini, Ollama, Dify, Coze |
| **TTS**    | EdgeTTS(免费), 火山引擎, 阿里�? 腾讯, 讯飞, FishSpeech, GPT-SoVITS |
| **VAD**    | SileroVAD(本地)                                                     |
| **VLLM**   | 智谱glm-4v-flash, 千问qwen2.5-vl                                    |
| **Memory** | mem0ai, 本地短期记忆                                                |
| **Intent** | function_call, intent_llm                                           |

### manager-api (智控台后�?
- **路径**: `third_party/xiaozhi-esp32-server/main/manager-api`
- **语言**: Java (Spring Boot)
- **功能**: 用户管理、设备绑定、模型配置、OTA 升级

#### 核心模块
- `modules/agent/` - 智能�?角色管理
- `modules/device/` - 设备管理
- `modules/model/` - 模型配置 (ASR/LLM/TTS)
- `modules/knowledge/` - 知识库管�?
- `modules/timbre/` - 音色管理
- `modules/voiceclone/` - 语音克隆
- `modules/security/` - 安全认证
- `modules/sys/` - 系统配置

#### 国际�?
支持�?�?�?越南�?

### manager-web (智控台前�?
- **路径**: `third_party/xiaozhi-esp32-server/main/manager-web`
- **语言**: Vue.js
- **功能**: Web 管理界面

#### 核心页面
- `DeviceManagement.vue` - 设备管理
- `ModelConfig.vue` - 模型配置
- `AgentTemplateManagement.vue` - 智能体模�?
- `KnowledgeBaseManagement.vue` - 知识库管�?
- `VoiceCloneManagement.vue` - 语音克隆
- `VoicePrint.vue` - 声纹管理
- `OtaManagement.vue` - OTA 升级
- `UserManagement.vue` - 用户管理

### manager-mobile (智控台移动版)
- **路径**: `third_party/xiaozhi-esp32-server/main/manager-mobile`
- **语言**: Vue 3 + TypeScript (uni-app)
- **功能**: 跨端移动管理�?(App/H5/小程�?

#### 平台兼容�?
| H5  | iOS | Android | 微信小程�?|
| --- | --- | ------- | ---------- |
| �?  | �?  | �?      | �?         |

#### 核心功能
- `pages/agent/` - 智能体管�?
- `pages/device/` - 设备管理
- `pages/device-config/` - 设备配网 (WiFi/超声�?
- `pages/voiceprint/` - 声纹管理
- `pages/chat-history/` - 聊天记录

#### 技术栈
alova (请求�? + pinia (状�? + wot-design-uni (UI)

#### 开发命�?
```bash
pnpm dev:h5          # H5 开�?
pnpm dev:mp          # 微信小程�?
pnpm build:app       # App 构建
```

---

## xiaozhi-mqtt-gateway (MQTT 网关)

### 基本信息
- **路径**: `third_party/xiaozhi-mqtt-gateway`
- **语言**: Node.js
- **功能**: MQTT+UDP �?WebSocket 桥接，设备指令下�?

### 核心文件
- `app.js` - 主入口，MQTT/UDP 服务器，WebSocket 桥接
- `mqtt-protocol.js` - MQTT 3.1.1 协议解析与封�?
- `utils/mqtt_config_v2.js` - 设备认证签名生成与验�?

---

## voiceprint-api (声纹识别服务)

### 基本信息
- **路径**: `third_party/voiceprint-api`
- **语言**: Python 3.10 + FastAPI
- **功能**: 基于 3D-Speaker 模型的声纹识�?

### 核心文件
- `app/services/voiceprint_service.py` - 声纹提取/注册/识别服务
- `app/api/v1/voiceprint.py` - REST API 接口
- `app/database/voiceprint_db.py` - MySQL 声纹存储
- `voiceprint.yaml` - 配置模板

### API 接口
| 方法     | 路径                       | 功能                                |
| -------- | -------------------------- | ----------------------------------- |
| `POST`   | `/voiceprint/register`     | 注册声纹 (speaker_id + WAV音频)     |
| `POST`   | `/voiceprint/identify`     | 识别声纹 (返回 speaker_id + 相似�? |
| `DELETE` | `/voiceprint/{speaker_id}` | 删除声纹                            |

### 技术特�?
- **模型**: `iic/speech_campplus_sv_zh-cn_3dspeaker_16k` (ModelScope)
- **特征计算**: 余弦相似�?
- **音频处理**: 自动重采样到 16kHz

---

## mcp-endpoint-server (MCP 接入点服�?

### 基本信息
- **路径**: `third_party/mcp-endpoint-server`
- **语言**: Python 3.10 + FastAPI
- **功能**: MCP 工具注册中心，转发小智端和工具端消息

### 核心文件
- `src/server.py` - FastAPI 主服务，WebSocket 端点
- `src/core/connection_manager.py` - 连接管理和消息转�?
- `src/handlers/websocket_handler.py` - 工具�?小智端消息处�?
- `src/utils/aes_utils.py` - Token 加解�?
- `mcp-endpoint-server.cfg` - 配置文件

### WebSocket 端点
| 端点                       | 用�?                      |
| -------------------------- | -------------------------- |
| `WS /mcp_endpoint/mcp/`    | 工具端连�?(注册 MCP 工具) |
| `WS /mcp_endpoint/call/`   | 小智端连�?(调用 MCP 工具) |
| `GET /mcp_endpoint/health` | 健康检查和连接统计         |

### 架构设计
```
工具�?(MCP Server) ──WS──�?MCP Endpoint ◄──WS── 小智�?(ESP32/Cloud)
     �?                       �?                     �?
     �? 注册工具列表            �? 转发 JSON-RPC        �? 调用工具
     └──────────────────────►◄──────────────────────�?
```

---

## mem0 (AI 记忆管理�?

### 基本信息
- **路径**: `third_party/mem0`
- **语言**: Python 3.10+
- **用�?*: �?AI 助手提供智能记忆层，支持用户偏好、会话上下文、长期记�?
- **文档**: `third_party/mem0/README.md`, `third_party/mem0/docs/`

### 核心 API

| 方法 | 功能 | 说明 |
|------|------|------|
| `memory.add()` | 添加记忆 | 自动提取事实并去�?|
| `memory.search()` | 搜索记忆 | 语义搜索 + Rerank |
| `memory.get()` | 获取记忆 | �?ID 获取 |
| `memory.get_all()` | 获取所�?| �?user_id/agent_id 过滤 |
| `memory.update()` | 更新记忆 | �?ID 更新 |
| `memory.delete()` | 删除记忆 | �?ID 删除 |
| `memory.history()` | 记忆历史 | 查看变更历史 |

### 记忆类型
- **User Memory**: 用户偏好和个人信�?
- **Session Memory**: 会话上下�?
- **Agent Memory**: 智能体状�?
- **Procedural Memory**: 操作流程记忆

### 关键特�?
- �?自动事实提取 (LLM 驱动)
- �?记忆去重与合�?
- �?支持 Graph Store (关系提取)
- �?异步 API (AsyncMemory)
- �?多种向量库支�?(Qdrant/Faiss/Chroma)

### 代码示例
```python
from mem0 import Memory

memory = Memory()

# 添加记忆 (自动提取事实)
memory.add(
    messages=[{"role": "user", "content": "我喜欢吃四川菜，但不能吃�?}],
    user_id="user_123"
)

# 搜索相关记忆
results = memory.search(query="用户的饮食偏�?, user_id="user_123")
```

---

## LightRAG (知识图谱 RAG)

### 基本信息
- **路径**: `third_party/LightRAG`
- **语言**: Python 3.10+
- **用�?*: 基于知识图谱�?RAG 系统，支持复杂多跳查�?
- **文档**: `third_party/LightRAG/README.md`, `third_party/LightRAG/docs/`

### 核心 API

| 方法 | 功能 | 说明 |
|------|------|------|
| `rag.ainsert()` | 插入文档 | 自动提取实体关系构建知识图谱 |
| `rag.aquery()` | 查询 | 支持多种检索模�?|
| `rag.adelete_by_doc_id()` | 删除文档 | 自动重建 KG |
| `rag.get_knowledge_graph()` | 获取知识图谱 | 返回节点和边 |
| `rag.get_graph_labels()` | 获取实体标签 | 列出所有实体类�?|

### 查询模式 (QueryParam.mode)

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `naive` | 向量相似度搜�?| 简单问�?|
| `local` | 局部知识图谱检�?| 实体相关问题 |
| `global` | 全局知识图谱检�?| 跨实体关系问�?|
| `hybrid` | 混合检�?| 复杂问题 (推荐) |
| `mix` | 混合 + Rerank | 最高质�?(默认) |

### 关键特�?
- �?知识图谱自动构建 (实体/关系提取)
- �?多跳推理能力
- �?增量更新知识
- �?支持 Reranker 提升精度
- �?多种存储后端 (NetworkX/Neo4j/PostgreSQL)

### 代码示例
```python
from lightrag import LightRAG, QueryParam

rag = LightRAG(
    working_dir="./rag_storage",
    llm_model_func=llm_func,
    embedding_func=embed_func,
)

await rag.initialize_storages()

# 插入文档 (自动构建知识图谱)
await rag.ainsert("文档内容...")

# 查询 (使用混合模式)
result = await rag.aquery(
    "问题内容",
    param=QueryParam(mode="hybrid")
)

await rag.finalize_storages()
```

---

## BullMQ (Redis 任务队列)

### 基本信息
- **路径**: `third_party/bullmq`
- **语言**: Python / TypeScript
- **用�?*: 定时任务、延迟执行、任务队列、重试机�?
- **依赖**: Redis
- **文档**: `third_party/bullmq/README.md`, https://docs.bullmq.io

### 核心概念

| 概念 | 说明 |
|------|------|
| **Queue** | 任务队列，用于添加和管理任务 |
| **Job** | 任务实例，包含数据和状�?|
| **Worker** | 工作进程，消费并执行任务 |

### 核心 API (Python)

| 方法 | 功能 | 说明 |
|------|------|------|
| `queue.add()` | 添加任务 | 支持延迟、重试等选项 |
| `queue.addBulk()` | 批量添加 | 高效批量操作 |
| `queue.pause()` | 暂停队列 | 停止处理新任�?|
| `queue.resume()` | 恢复队列 | 继续处理任务 |
| `job.remove()` | 删除任务 | 删除指定任务 |
| `job.promote()` | 立即执行 | 将延迟任务移到队列前�?|
| `job.getState()` | 获取状�?| waiting/active/completed/failed |

### 任务选项 (JobOptions)

| 选项 | 类型 | 说明 |
|------|------|------|
| `delay` | int (ms) | 延迟执行时间 |
| `attempts` | int | 重试次数 |
| `backoff` | dict | 重试策略 (fixed/exponential) |
| `removeOnComplete` | bool | 完成后自动删�?|
| `removeOnFail` | bool | 失败后自动删�?|
| `priority` | int | 优先�?(越小越高) |
| `repeat` | dict | 重复任务配置 (cron/every) |

### 应用场景
- �?**定时提醒**: 语音助手定时提醒用户
- �?**IoT 定时控制**: 定时开关灯、空调等
- �?**延迟任务**: �?分钟后提醒我�?
- �?**周期任务**: 每天早上播报天气
- �?**重试机制**: 失败自动重试

### 代码示例
```python
from bullmq import Queue, Worker, Job

# 创建队列
queue = Queue("reminders", {"connection": {"host": "localhost", "port": 6379}})

# 添加延迟任务 (5分钟后执�?
await queue.add(
    "reminder",
    {"user_id": "user_123", "message": "该吃药了"},
    {"delay": 5 * 60 * 1000}  # 5分钟
)

# 添加重复任务 (每天早上8�?
await queue.add(
    "daily_weather",
    {"user_id": "user_123"},
    {"repeat": {"cron": "0 8 * * *"}}
)

# 创建 Worker 处理任务
async def process_reminder(job: Job, token: str):
    user_id = job.data["user_id"]
    message = job.data["message"]
    # 通过 MQTT 发送语音提醒到 ESP32
    await send_voice_reminder(user_id, message)
    return {"status": "sent"}

worker = Worker("reminders", process_reminder, {"connection": {...}})
```

---

## 全模块部署端�?

| 服务                | 端口      | 说明              |
| ------------------- | --------- | ----------------- |
| xiaozhi-server      | 8000      | WebSocket 服务    |
| manager-api         | 8002      | 智控台后�?(Java) |
| manager-web         | 8001      | 智控台前�?(Vue)  |
| mcp-endpoint-server | 8004      | MCP 接入�?       |
| voiceprint-api      | 8005      | 声纹识别服务      |
| MySQL               | 3306      | 数据�?           |
| Redis               | 6379      | 缓存              |
| MQTT Gateway        | 1883/8007 | MQTT/HTTP         |
