# 系统架构说明

本文档详细说明 AI Assistant 项目的系统架构和部署模式。

---

## 架构概览

AI Assistant 是一个端到端的语音聊天助手项目，基于 **小智 AI** 开源生态构建。

```
用户语音 → ESP32采集 → 云端ASR → LLM对话 → TTS合成 → ESP32播放
```

---

## 整体架构

> **CRITICAL**: 默认采用网关模式 (MQTT+UDP → WebSocket)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              系统架构                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────────────────┐  │
│  │  设备层      │      │  网关层      │      │  后端服务层              │  │
│  │             │      │             │      │                         │  │
│  │  ESP32      │ MQTT │  Gateway    │  WS  │  ┌─────────────────┐   │  │
│  │  • 音频采集  │◄────►│  • 协议转换  │◄────►│  │  WebSocket Server│   │  │
│  │  • MCP 工具  │ UDP  │  • 音频加密  │      │  │  • ASR/LLM/TTS   │   │  │
│  │             │      │  • 设备认证  │      │  │  • 会话管理       │   │  │
│  └─────────────┘      └─────────────┘      │  └────────┬────────┘   │  │
│                                            │           │             │  │
│                                            │           ▼             │  │
│                                            │  ┌─────────────────┐   │  │
│                                            │  │  MCP 服务层      │   │  │
│                                            │  │                 │   │  │
│                                            │  │  • Mem0 记忆     │   │  │
│                                            │  │  • LightRAG 知识 │   │  │
│                                            │  │  • Scheduler 提醒│   │  │
│                                            │  └────────┬────────┘   │  │
│                                            │           │             │  │
│                                            │           ▼             │  │
│                                            │  ┌─────────────────┐   │  │
│                                            │  │  存储层          │   │  │
│                                            │  │  Redis + Qdrant  │   │  │
│                                            │  └─────────────────┘   │  │
│                                            └─────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 连接模式

### 网关模式 (默认)

**适用场景**: IoT 设备、低功耗场景、NAT 穿透、生产环境

```
ESP32 ──(MQTT+UDP)──► Gateway ──(WebSocket)──► Python Server
```

**设计决策**:
- **MQTT**: JSON 控制消息 (低带宽、可靠)
- **UDP**: 音频流 (低延迟、实时性)
- **网关**: 协议转换 + 音频加解密 + 设备认证

**特点**:
- 支持 NAT 穿透，设备无需公网 IP
- 设备可低功耗休眠
- HMAC-SHA256 设备认证
- AES-128-CTR 音频加密

### 直连模式 (仅开发调试)

```
ESP32 ──(WebSocket + Opus)──► Python Server
```

---

## MCP 服务层

三个核心 MCP 服务，为 LLM 提供工具调用能力：

| 服务 | 端口 | 用途 |
|------|------|------|
| **Mem0 MCP** | 3001 | 长期记忆 - 用户偏好、历史对话 |
| **LightRAG MCP** | 3002 | 知识库 - 私有知识、图谱检索 |
| **Scheduler MCP** | 3003 | 定时提醒 - 对话式交互、确认重试 |

### MCP 工具清单

**Mem0 (记忆)**:
- `memory.add` - 添加记忆
- `memory.search` - 搜索记忆
- `memory.get_context` - 获取对话上下文

**LightRAG (知识库)**:
- `knowledge.query` - 查询知识 (支持 hybrid 模式)
- `knowledge.insert` - 插入知识
- `knowledge.get_entities` - 获取图谱实体

**Scheduler (提醒)**:
- `reminder.create` - 创建提醒
- `reminder.list` - 列出提醒
- `reminder.confirm` - 确认提醒

---

## 核心流程

### 语音对话流程

```
1. 用户语音输入
   │
   ▼
2. ESP32 采集 → Opus 编码 → UDP 传输
   │
   ▼
3. Gateway 解密 → WebSocket 转发
   │
   ▼
4. ASR 语音识别
   │
   ├──► 并行: Mem0 获取记忆
   │
   ├──► 并行: LightRAG 查询知识
   │
   ▼
5. LLM 对话 (融合记忆+知识)
   │
   ▼
6. TTS 语音合成 → 音频回传
   │
   ▼
7. 保存新记忆到 Mem0
```

### 定时提醒流程

```
1. 用户: "明天8点提醒我吃药"
   │
   ▼
2. LLM 解析 → Scheduler.reminder.create
   │
   ▼
3. BullMQ Job 入队 (delay 到明天8点)
   │
   ▼
4. Job 触发 → Gateway 唤醒设备
   │
   ▼
5. 建立对话会话 → 播放提醒
   │
   ├──► 普通提醒: 30秒无响应自动关闭
   │
   └──► 确认提醒: 等待用户确认，超时重试 (最多3次)
```

---

## 服务端口分配

| 服务 | 端口 | 说明 |
|------|------|------|
| WebSocket Server | 8000 | Python 语音服务 |
| MQTT Broker | 1883 | 设备连接 |
| Gateway API | 8007 | 网关管理 |
| Mem0 MCP | 3001 | 记忆服务 |
| LightRAG MCP | 3002 | 知识库服务 |
| Scheduler MCP | 3003 | 提醒服务 |
| Redis | 6379 | 缓存/队列 |
| Qdrant | 6333 | 向量数据库 |

---

## 存储设计

| 存储 | 用途 |
|------|------|
| **Redis** | 会话缓存、任务队列 (BullMQ)、用户索引 |
| **Qdrant** | 向量存储 (记忆、知识库) |
| **MySQL** | 用户数据、设备绑定 (智控台) |

---

## 部署建议

### 开发环境

```bash
# 推荐: 本地网关模式 (与生产一致)
ESP32 ──(MQTT+UDP)──► Gateway ──(WS)──► Python Server
                                              │
                                        Redis + Qdrant
```

### 生产环境

```bash
# 完整部署
ESP32 ──(MQTT+UDP)──► Gateway ──(WS)──► Python Server
                                              │
                              ┌───────────────┼───────────────┐
                              │               │               │
                          Mem0 MCP      LightRAG MCP   Scheduler MCP
                              │               │               │
                              └───────────────┴───────────────┘
                                              │
                                        Redis + Qdrant
```

---

## 扩展性设计

### Provider 模式

所有 AI 模块采用 Provider 模式，便于替换：

```python
# ASR Provider
class ASRProviderBase:
    async def transcribe(self, audio: bytes) -> str: ...

# LLM Provider  
class LLMProviderBase:
    async def chat(self, messages: list) -> str: ...

# TTS Provider
class TTSProviderBase:
    async def synthesize(self, text: str) -> bytes: ...
```

### 水平扩展

- **Python Server**: 无状态设计，可多实例负载均衡
- **Gateway**: 可按设备分片部署
- **MCP 服务**: 独立部署，按需扩展
