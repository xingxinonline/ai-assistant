# Mem0 记忆服务设计

本文档详细说明 AI Assistant 的长短期记忆系统设计。

---

## 概述

为 AI 助手提供类人的记忆能力，让对话更加个性化和连贯。

### 核心问题

| 问题             | 挑战                       |
| ---------------- | -------------------------- |
| **记忆无限增长** | 存储成本、检索效率下降     |
| **遗忘机制**     | 哪些该忘、哪些该记         |
| **高效检索**     | 海量记忆中快速找到相关内容 |
| **隐私保护**     | 敏感信息的处理             |

---

## 记忆分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                        记忆系统                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  工作记忆    │  │  短期记忆    │  │  长期记忆           │ │
│  │  (会话内)    │  │  (24小时)   │  │  (持久化)           │ │
│  │             │  │             │  │                     │ │
│  │  • 当前对话  │  │  • 近期对话  │  │  • 核心事实         │ │
│  │  • 临时状态  │  │  • 情绪状态  │  │  • 用户偏好         │ │
│  │             │  │  • 待办事项  │  │  • 重要事件         │ │
│  │  Redis      │  │  Redis+TTL  │  │  Qdrant + 压缩      │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│         │                │                    │             │
│         └────────────────┼────────────────────┘             │
│                          ▼                                  │
│                   ┌─────────────┐                           │
│                   │  记忆整合    │                           │
│                   │  Consolidation                          │
│                   └─────────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

### 记忆类型定义

| 类型         | 生命周期 | 存储           | 示例               |
| ------------ | -------- | -------------- | ------------------ |
| **工作记忆** | 会话内   | Redis (无 TTL) | "用户刚才问了天气" |
| **短期记忆** | 24小时   | Redis (TTL)    | "今天用户心情不好" |
| **长期记忆** | 持久化   | Qdrant         | "用户喜欢咖啡"     |

---

## 遗忘曲线机制

### 艾宾浩斯遗忘曲线

记忆强度随时间衰减，但每次回忆都会强化：

```
记忆强度
    │
100%├──●
    │   ╲
 80%├────╲──●  (第1次回忆，强化)
    │      ╲  ╲
 60%├───────╲──╲──●  (第2次回忆)
    │         ╲   ╲   ╲
 40%├──────────╲───╲───╲──●
    │            ╲    ╲    ╲
 20%├─────────────╲────╲────╲──→ 稳定记忆
    │
    └────────────────────────────► 时间
        1天   3天   7天   30天
```

### 记忆强度计算

```python
class MemoryStrength:
    """记忆强度计算"""
    
    # 基础衰减参数
    DECAY_RATE = 0.1  # 每天衰减 10%
    RECALL_BOOST = 0.3  # 每次回忆增强 30%
    MIN_STRENGTH = 0.1  # 最低强度阈值
    
    def calculate(
        self,
        created_at: datetime,
        last_accessed: datetime,
        access_count: int,
        importance: float  # 0-1
    ) -> float:
        """
        计算当前记忆强度
        
        Args:
            created_at: 创建时间
            last_accessed: 最后访问时间
            access_count: 访问次数
            importance: 重要性权重
            
        Returns:
            记忆强度 (0-1)
        """
        days_since_access = (now() - last_accessed).days
        
        # 基础衰减
        decay = math.exp(-self.DECAY_RATE * days_since_access)
        
        # 回忆强化 (对数增长，避免无限增强)
        recall_factor = 1 + self.RECALL_BOOST * math.log(1 + access_count)
        
        # 重要性加权
        importance_factor = 0.5 + 0.5 * importance
        
        strength = decay * recall_factor * importance_factor
        return max(self.MIN_STRENGTH, min(1.0, strength))
```

---

## 记忆压缩策略

### 问题：记忆无限增长

随着对话增多，记忆会无限增长，导致：
- 存储成本增加
- 检索效率下降
- 噪音干扰有效信息

### 解决方案：分层压缩

```
原始记忆 (大量)
    │
    ▼ LLM 提取关键信息
关键事实 (精简)
    │
    ▼ 定期合并相似记忆
核心画像 (极简)
```

### 压缩规则

| 规则           | 触发条件                 | 操作                     |
| -------------- | ------------------------ | ------------------------ |
| **相似合并**   | 语义相似度 > 0.9         | 合并为一条，保留最新时间 |
| **过期清理**   | 强度 < 0.1 且 30天未访问 | 归档或删除               |
| **摘要提取**   | 同主题记忆 > 10条        | LLM 生成摘要替代原始记忆 |
| **重要性提升** | 访问次数 > 5             | 标记为核心记忆，永不删除 |

### 压缩流程

```python
class MemoryCompressor:
    """记忆压缩器"""
    
    async def compress_user_memories(self, user_id: str):
        """压缩用户记忆"""
        
        # 1. 获取所有记忆
        memories = await self.get_all_memories(user_id)
        
        # 2. 计算记忆强度，标记弱记忆
        weak_memories = [m for m in memories if m.strength < 0.1]
        
        # 3. 归档超过30天未访问的弱记忆
        for memory in weak_memories:
            if memory.days_since_access > 30:
                await self.archive_memory(memory)
        
        # 4. 合并相似记忆
        clusters = await self.cluster_similar_memories(memories)
        for cluster in clusters:
            if len(cluster) > 1:
                merged = await self.merge_memories(cluster)
                await self.replace_memories(cluster, merged)
        
        # 5. 生成用户画像摘要
        if len(memories) > 100:
            profile = await self.generate_profile_summary(memories)
            await self.store_profile(user_id, profile)
```

---

## 记忆检索策略

### 多路召回

```
用户查询
    │
    ├─► 向量检索 (语义相似)
    │       │
    ├─► 关键词检索 (精确匹配)
    │       │
    ├─► 时间检索 (近期优先)
    │       │
    └─► 重要性检索 (核心记忆)
            │
            ▼
        结果融合 + 重排序
            │
            ▼
        Top-K 返回
```

### 检索优化

```python
class MemoryRetriever:
    """记忆检索器"""
    
    async def retrieve(
        self,
        user_id: str,
        query: str,
        top_k: int = 5
    ) -> list[Memory]:
        """
        检索相关记忆
        """
        # 1. 向量检索
        vector_results = await self.vector_search(
            query=query,
            user_id=user_id,
            top_k=top_k * 2  # 召回更多用于重排
        )
        
        # 2. 时间加权 (近期记忆权重更高)
        for memory in vector_results:
            recency_boost = self.calculate_recency_boost(memory)
            memory.score *= recency_boost
        
        # 3. 重要性加权
        for memory in vector_results:
            memory.score *= (0.5 + 0.5 * memory.importance)
        
        # 4. 强度加权 (遗忘曲线)
        for memory in vector_results:
            memory.score *= memory.strength
        
        # 5. 排序返回
        vector_results.sort(key=lambda m: m.score, reverse=True)
        return vector_results[:top_k]
```

---

## 记忆重要性评估

### LLM 自动评估

```python
IMPORTANCE_PROMPT = """
分析以下对话内容，提取值得长期记忆的信息。

对话内容：
{conversation}

评估标准：
1. 用户偏好 (喜好、习惯) - 重要性: 高
2. 个人信息 (姓名、生日、职业) - 重要性: 高
3. 情感表达 (开心、难过的原因) - 重要性: 中
4. 临时事项 (今天要做的事) - 重要性: 低
5. 闲聊内容 (天气讨论) - 重要性: 极低

输出 JSON 格式：
{
  "memories": [
    {"content": "记忆内容", "importance": 0.9, "category": "preference"},
    ...
  ]
}
"""
```

### 重要性分级

| 级别     | 分数范围 | 类别       | 处理策略     |
| -------- | -------- | ---------- | ------------ |
| **核心** | 0.8-1.0  | 偏好、身份 | 永不删除     |
| **重要** | 0.5-0.8  | 情感、事件 | 正常遗忘曲线 |
| **一般** | 0.2-0.5  | 对话上下文 | 加速遗忘     |
| **临时** | 0-0.2    | 闲聊       | 24小时后清理 |

---

## MCP 工具接口

```python
class MemoryMCPService:
    """Mem0 MCP 服务"""
    
    tools = {
        # 写入
        "memory.add": "添加记忆 (自动评估重要性)",
        "memory.update": "更新记忆内容",
        "memory.delete": "删除指定记忆",
        
        # 读取
        "memory.search": "语义搜索相关记忆",
        "memory.get_context": "获取对话上下文记忆",
        "memory.get_profile": "获取用户画像摘要",
        "memory.list": "列出用户所有记忆",
        
        # 管理
        "memory.compress": "触发记忆压缩",
        "memory.stats": "获取记忆统计信息",
    }
```

### 工具示例

```python
# 添加记忆
await memory.add(
    user_id="user123",
    content="用户说他喜欢拿铁咖啡，每天早上都会喝一杯",
    metadata={"source": "conversation", "session_id": "xxx"}
)
# 自动提取: importance=0.8, category="preference"

# 搜索记忆
results = await memory.search(
    user_id="user123",
    query="用户的饮品偏好",
    top_k=5
)

# 获取对话上下文
context = await memory.get_context(
    user_id="user123",
    current_query="推荐一家咖啡店"
)
# 返回: "用户喜欢拿铁咖啡，常去星巴克Reserve"
```

---

## 存储设计

### Qdrant Collection

```python
# 记忆向量存储
memories_collection = {
    "name": "memories",
    "vectors": {
        "size": 1024,  # bge-m3 维度
        "distance": "Cosine"
    },
    "payload_schema": {
        "user_id": "keyword",
        "content": "text",
        "importance": "float",
        "strength": "float",
        "category": "keyword",
        "created_at": "datetime",
        "last_accessed": "datetime",
        "access_count": "integer",
        "metadata": "json"
    }
}
```

### Redis 索引

```python
# 用户记忆索引
user_memories:{user_id} = Set[memory_id]

# 工作记忆 (会话内)
working_memory:{session_id} = List[message]

# 短期记忆 (TTL 24小时)
short_term:{user_id}:{key} = value  # TTL: 86400s
```

---

## 隐私与安全

### 敏感信息处理

| 信息类型         | 处理方式            |
| ---------------- | ------------------- |
| 身份证号、银行卡 | 不存储              |
| 手机号、邮箱     | 脱敏存储            |
| 地址             | 模糊化 (只保留城市) |
| 健康信息         | 加密存储            |

### 用户控制

- 用户可查看所有记忆
- 用户可删除任意记忆
- 用户可导出记忆数据
- 用户可关闭记忆功能

---

## 监控指标

| 指标                | 说明         | 告警阈值   |
| ------------------- | ------------ | ---------- |
| `memory_count`      | 用户记忆数量 | > 10000    |
| `compression_rate`  | 压缩率       | < 0.3      |
| `retrieval_latency` | 检索延迟     | > 500ms    |
| `storage_size`      | 存储大小     | > 1GB/用户 |
