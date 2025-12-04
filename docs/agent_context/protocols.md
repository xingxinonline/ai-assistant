# 通信协议详解

本文档详细说明 AI Assistant 项目使用的所有通信协议。AI Agent 在需要实现或调试通信功能时应阅读此文档。

---

## 协议总览

> **CRITICAL**: 默认采用 MQTT+UDP 网关模式，ESP32 本质上是 IoT 设备。

| 协议              | 用途                  | 优先级     | 文档位置                                 |
| ----------------- | --------------------- | ---------- | ---------------------------------------- |
| **MQTT+UDP** ✅   | IoT 设备网关接入 (默认) | 默认       | 本文档                                   |
| **WebSocket**     | 开发调试直连          | 仅开发调试 | `SDK/xiaozhi-esp32/docs/websocket.md`    |
| **MCP**           | 设备控制              | 两种模式通用 | `SDK/xiaozhi-esp32/docs/mcp-protocol.md` |

---

## WebSocket 协议 (直连模式 - 仅开发调试)

> ⚠️ **注意**: 此模式仅建议用于开发调试。生产环境请使用 MQTT+UDP 网关模式。

### 握手流程

1. **设备连接** → 发送 `hello` 消息 (含 audio_params)
2. **服务端回复** `hello` 消息 (含 session_id)
3. **双向通信** → Opus 音频流 + JSON 控制消息

### 消息类型

| 类型      | 方向        | 用途          |
| --------- | ----------- | ------------- |
| `hello`   | 双向        | 握手消息      |
| `listen`  | 双向        | 开始/停止监听 |
| `stt`     | 服务端→设备 | 语音识别结果  |
| `tts`     | 服务端→设备 | 语音合成控制  |
| `llm`     | 服务端→设备 | 表情/情绪控制 |
| `mcp`     | 双向        | 设备控制协议  |
| `goodbye` | 双向        | 断开连接      |

### 音频传输

- **编码**: Opus
- **帧类型**: Binary
- **采样率**: 16kHz (通常)

---

## MCP 协议 (设备控制)

基于 **JSON-RPC 2.0**，用于服务端控制 ESP32 设备。

### 方法列表

| 方法         | 用途             |
| ------------ | ---------------- |
| `tools/list` | 获取设备能力列表 |
| `tools/call` | 调用设备功能     |

### 请求示例

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "set_volume",
    "arguments": {
      "volume": 80
    }
  }
}
```

### 响应示例

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "success": true,
    "message": "Volume set to 80"
  }
}
```

### 常用工具

- 音量控制
- 灯光/LED 控制
- GPIO 操作
- 系统信息获取

---

## MQTT 网关协议 (IoT 设备接入) - ✅ 默认模式

> **CRITICAL**: 这是项目的默认通信架构。ESP32 本质上是 IoT 设备，网关模式更符合 IoT 设备的通信范式。

网关将 IoT 设备常用的 MQTT+UDP 协议转换为 WebSocket 协议。

**设计决策**:
- **MQTT**: 用于 JSON 控制消息 (低带宽、可靠传输、消息队列)
- **UDP**: 用于音频流 (低延迟、实时性、丢包可容忍)
- **网关**: 协议转换 + 音频加解密 + 设备认证

### 数据流

```
┌─────────────────── 上行数据流 (用户语音 → AI) ───────────────────┐
│                                                                  │
│  ESP32 ──MQTT──► Gateway ──WebSocket──► Cloud                   │
│         (JSON控制)        (JSON控制)                              │
│                                                                  │
│  ESP32 ──UDP───► Gateway ──WebSocket──► Cloud                   │
│      (加密Opus)     (解密)    (Binary)                           │
│                                                                  │
├─────────────────── 下行数据流 (AI回复 → 用户) ───────────────────┤
│                                                                  │
│  Cloud ──WebSocket──► Gateway ──MQTT──► ESP32                   │
│        (JSON控制)            (JSON控制)                          │
│                                                                  │
│  Cloud ──WebSocket──► Gateway ──UDP───► ESP32                   │
│        (Binary)        (加密)   (加密Opus)                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### MQTT 设备认证

| 字段       | 格式                             | 说明                          |
| ---------- | -------------------------------- | ----------------------------- |
| `clientId` | `{GroupID}@@@{MAC地址}@@@{UUID}` | 设备唯一标识                  |
| `username` | Base64 编码的 JSON               | 用户数据                      |
| `password` | `HMAC-SHA256(clientId + "        | " + username, SIGNATURE_KEY)` | 签名 |

### UDP 音频加密

| 项目       | 说明                           |
| ---------- | ------------------------------ |
| **算法**   | AES-128-CTR                    |
| **密钥**   | 每次会话生成随机 16 字节       |
| **帧格式** | 16字节 Header + 加密 Opus 数据 |

### UDP Header 格式 (16字节)

| Offset | Size | Field         | 说明             |
| ------ | ---- | ------------- | ---------------- |
| 0      | 1    | type          | 1=音频           |
| 2      | 2    | payloadLength | 负载长度         |
| 4      | 4    | connectionId  | 连接ID           |
| 8      | 4    | timestamp     | 时间戳 (60ms/帧) |
| 12     | 4    | sequence      | 序列号           |

### 网关管理 API

#### 设备指令下发 (MCP)

```http
POST /api/commands/:clientId
Content-Type: application/json

{
  "type": "mcp",
  "payload": {
    "method": "tools/call",
    "params": {
      "name": "set_volume",
      "arguments": {"volume": 80}
    }
  }
}
```

#### 批量设备状态查询

```http
POST /api/devices/status
Content-Type: application/json

{
  "clientIds": ["GID@@@MAC1@@@UUID1", "GID@@@MAC2@@@UUID2"]
}
```

---

## 协议实现参考

### 直连模式实现

参考 `SDK/xiaozhi-esp32-server/main/xiaozhi-server/core/` 目录：
- `handle/` - 连接处理和消息分发
- `providers/` - ASR/LLM/TTS 提供者

### 网关模式实现

参考 `SDK/xiaozhi-mqtt-gateway/` 目录：
- `app.js` - 主入口，MQTT/UDP/WebSocket 桥接
- `mqtt-protocol.js` - MQTT 协议解析
- `utils/mqtt_config_v2.js` - 设备认证

### MCP 实现

参考 `SDK/mcp-endpoint-server/src/` 目录：
- `server.py` - FastAPI 主服务
- `core/connection_manager.py` - 连接管理
- `handlers/websocket_handler.py` - 消息处理
