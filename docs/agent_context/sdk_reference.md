# SDK 参考实现详解

本文档包含 AI Assistant 项目所有 SDK 组件的详细说明。AI Agent 在需要深入了解某个 SDK 组件时应阅读此文档。

## 组件总览

| 组件                     | 路径                       | 语言            | 用途              |
| ------------------------ | -------------------------- | --------------- | ----------------- |
| **xiaozhi-esp32**        | `SDK/xiaozhi-esp32`        | C++             | ESP32 硬件端固件  |
| **xiaozhi-esp32-server** | `SDK/xiaozhi-esp32-server` | Python/Java/Vue | 云端服务 + 智控台 |
| **xiaozhi-mqtt-gateway** | `SDK/xiaozhi-mqtt-gateway` | Node.js         | MQTT 协议网关     |
| **voiceprint-api**       | `SDK/voiceprint-api`       | Python          | 声纹识别服务      |
| **mcp-endpoint-server**  | `SDK/mcp-endpoint-server`  | Python          | MCP 工具注册中心  |

---

## xiaozhi-esp32 (硬件端)

### 基本信息
- **路径**: `SDK/xiaozhi-esp32`
- **语言**: C++ (ESP-IDF 5.4+)
- **支持芯片**: ESP32-C3, ESP32-S3, ESP32-P4
- **参考文档**: `SDK/xiaozhi-esp32/README_zh.md`

### 功能
- 音频采集/播放
- 唤醒词检测
- 显示屏/LED 控制
- MCP 设备控制

### 关键文件
- `SDK/xiaozhi-esp32/docs/websocket.md` - WebSocket 通信协议
- `SDK/xiaozhi-esp32/docs/mcp-protocol.md` - MCP 设备控制协议
- `SDK/xiaozhi-esp32/docs/mcp-usage.md` - MCP 使用示例

---

## xiaozhi-esp32-server (云端服务)

### xiaozhi-server (Python 服务端)
- **路径**: `SDK/xiaozhi-esp32-server/main/xiaozhi-server`
- **语言**: Python 3.10
- **功能**: 完整的 ASR/LLM/TTS 流程实现

#### 核心目录
- `core/providers/` - ASR/LLM/TTS/VAD/VLLM 提供者实现
- `core/handle/` - 连接处理和消息分发
- `plugins_func/` - 插件函数 (天气/新闻/音乐等)

#### 配置参考
- `config.yaml` - 模块配置参考

#### 支持的模块

| 模块       | 支持平台                                                            |
| ---------- | ------------------------------------------------------------------- |
| **ASR**    | FunASR(本地), 火山引擎, 讯飞, 阿里云, 腾讯, 百度, OpenAI            |
| **LLM**    | 智谱GLM, 阿里百炼, 电豆Doubao, DeepSeek, Gemini, Ollama, Dify, Coze |
| **TTS**    | EdgeTTS(免费), 火山引擎, 阿里云, 腾讯, 讯飞, FishSpeech, GPT-SoVITS |
| **VAD**    | SileroVAD(本地)                                                     |
| **VLLM**   | 智谱glm-4v-flash, 千问qwen2.5-vl                                    |
| **Memory** | mem0ai, 本地短期记忆                                                |
| **Intent** | function_call, intent_llm                                           |

### manager-api (智控台后端)
- **路径**: `SDK/xiaozhi-esp32-server/main/manager-api`
- **语言**: Java (Spring Boot)
- **功能**: 用户管理、设备绑定、模型配置、OTA 升级

#### 核心模块
- `modules/agent/` - 智能体/角色管理
- `modules/device/` - 设备管理
- `modules/model/` - 模型配置 (ASR/LLM/TTS)
- `modules/knowledge/` - 知识库管理
- `modules/timbre/` - 音色管理
- `modules/voiceclone/` - 语音克隆
- `modules/security/` - 安全认证
- `modules/sys/` - 系统配置

#### 国际化
支持中/英/德/越南语

### manager-web (智控台前端)
- **路径**: `SDK/xiaozhi-esp32-server/main/manager-web`
- **语言**: Vue.js
- **功能**: Web 管理界面

#### 核心页面
- `DeviceManagement.vue` - 设备管理
- `ModelConfig.vue` - 模型配置
- `AgentTemplateManagement.vue` - 智能体模板
- `KnowledgeBaseManagement.vue` - 知识库管理
- `VoiceCloneManagement.vue` - 语音克隆
- `VoicePrint.vue` - 声纹管理
- `OtaManagement.vue` - OTA 升级
- `UserManagement.vue` - 用户管理

### manager-mobile (智控台移动版)
- **路径**: `SDK/xiaozhi-esp32-server/main/manager-mobile`
- **语言**: Vue 3 + TypeScript (uni-app)
- **功能**: 跨端移动管理端 (App/H5/小程序)

#### 平台兼容性
| H5  | iOS | Android | 微信小程序 |
| --- | --- | ------- | ---------- |
| ✓   | ✓   | ✓       | ✓          |

#### 核心功能
- `pages/agent/` - 智能体管理
- `pages/device/` - 设备管理
- `pages/device-config/` - 设备配网 (WiFi/超声波)
- `pages/voiceprint/` - 声纹管理
- `pages/chat-history/` - 聊天记录

#### 技术栈
alova (请求库) + pinia (状态) + wot-design-uni (UI)

#### 开发命令
```bash
pnpm dev:h5          # H5 开发
pnpm dev:mp          # 微信小程序
pnpm build:app       # App 构建
```

---

## xiaozhi-mqtt-gateway (MQTT 网关)

### 基本信息
- **路径**: `SDK/xiaozhi-mqtt-gateway`
- **语言**: Node.js
- **功能**: MQTT+UDP 到 WebSocket 桥接，设备指令下发

### 核心文件
- `app.js` - 主入口，MQTT/UDP 服务器，WebSocket 桥接
- `mqtt-protocol.js` - MQTT 3.1.1 协议解析与封装
- `utils/mqtt_config_v2.js` - 设备认证签名生成与验证

---

## voiceprint-api (声纹识别服务)

### 基本信息
- **路径**: `SDK/voiceprint-api`
- **语言**: Python 3.10 + FastAPI
- **功能**: 基于 3D-Speaker 模型的声纹识别

### 核心文件
- `app/services/voiceprint_service.py` - 声纹提取/注册/识别服务
- `app/api/v1/voiceprint.py` - REST API 接口
- `app/database/voiceprint_db.py` - MySQL 声纹存储
- `voiceprint.yaml` - 配置模板

### API 接口
| 方法     | 路径                       | 功能                                |
| -------- | -------------------------- | ----------------------------------- |
| `POST`   | `/voiceprint/register`     | 注册声纹 (speaker_id + WAV音频)     |
| `POST`   | `/voiceprint/identify`     | 识别声纹 (返回 speaker_id + 相似度) |
| `DELETE` | `/voiceprint/{speaker_id}` | 删除声纹                            |

### 技术特点
- **模型**: `iic/speech_campplus_sv_zh-cn_3dspeaker_16k` (ModelScope)
- **特征计算**: 余弦相似度
- **音频处理**: 自动重采样到 16kHz

---

## mcp-endpoint-server (MCP 接入点服务)

### 基本信息
- **路径**: `SDK/mcp-endpoint-server`
- **语言**: Python 3.10 + FastAPI
- **功能**: MCP 工具注册中心，转发小智端和工具端消息

### 核心文件
- `src/server.py` - FastAPI 主服务，WebSocket 端点
- `src/core/connection_manager.py` - 连接管理和消息转发
- `src/handlers/websocket_handler.py` - 工具端/小智端消息处理
- `src/utils/aes_utils.py` - Token 加解密
- `mcp-endpoint-server.cfg` - 配置文件

### WebSocket 端点
| 端点                       | 用途                       |
| -------------------------- | -------------------------- |
| `WS /mcp_endpoint/mcp/`    | 工具端连接 (注册 MCP 工具) |
| `WS /mcp_endpoint/call/`   | 小智端连接 (调用 MCP 工具) |
| `GET /mcp_endpoint/health` | 健康检查和连接统计         |

### 架构设计
```
工具端 (MCP Server) ──WS──► MCP Endpoint ◄──WS── 小智端 (ESP32/Cloud)
     │                        │                      │
     │  注册工具列表            │  转发 JSON-RPC        │  调用工具
     └──────────────────────►◄──────────────────────┘
```

---

## 全模块部署端口

| 服务                | 端口      | 说明              |
| ------------------- | --------- | ----------------- |
| xiaozhi-server      | 8000      | WebSocket 服务    |
| manager-api         | 8002      | 智控台后端 (Java) |
| manager-web         | 8001      | 智控台前端 (Vue)  |
| mcp-endpoint-server | 8004      | MCP 接入点        |
| voiceprint-api      | 8005      | 声纹识别服务      |
| MySQL               | 3306      | 数据库            |
| Redis               | 6379      | 缓存              |
| MQTT Gateway        | 1883/8007 | MQTT/HTTP         |
