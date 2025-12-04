# LightRAG 知识库服务设计

本文档详细说明 AI Assistant 的知识库系统设计。

---

## 概述

为 AI 助手提供私有知识检索能力，结合知识图谱实现精准问答。

### 核心问题

| 问题           | 挑战                       |
| -------------- | -------------------------- |
| **检索精度**   | 语义匹配 vs 精确匹配的平衡 |
| **知识更新**   | 增量更新而非全量重建       |
| **跨文档推理** | 信息分散在多个文档中       |
| **检索效率**   | 大规模知识库的响应速度     |

---

## 双模检索架构

```
┌─────────────────────────────────────────────────────────────┐
│                      知识库系统                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    查询层                            │   │
│  │                                                     │   │
│  │  用户查询 ──► 查询理解 ──► 多路检索 ──► 结果融合     │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│            ┌─────────────┼─────────────┐                   │
│            ▼             ▼             ▼                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │  向量检索    │ │  图谱检索    │ │  关键词检索  │          │
│  │  (语义)     │ │  (关系)     │ │  (精确)     │          │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘          │
│         │               │               │                  │
│         ▼               ▼               ▼                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐          │
│  │  Qdrant     │ │  知识图谱    │ │  全文索引    │          │
│  │  向量库     │ │  (实体+关系) │ │  (倒排)     │          │
│  └─────────────┘ └─────────────┘ └─────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 检索模式

### 四种检索模式

| 模式       | 说明         | 适用场景        |
| ---------- | ------------ | --------------- |
| **naive**  | 纯向量检索   | 简单语义匹配    |
| **local**  | 局部子图检索 | 实体关系查询    |
| **global** | 全局图谱检索 | 跨文档推理      |
| **hybrid** | 混合检索     | 通用场景 (推荐) |

### Hybrid 混合检索流程

```
用户查询: "ESP32 如何连接 MQTT 服务器?"
    │
    ├─► 向量检索
    │   • 召回语义相似的文档块
    │   • "ESP32 MQTT 连接配置..."
    │   • "WiFi 模块初始化..."
    │
    ├─► 图谱检索
    │   • 查找 ESP32 → 连接 → MQTT 路径
    │   • 扩展相关实体: WiFi, TLS, 认证
    │
    └─► 关键词检索
        • 精确匹配 "ESP32", "MQTT"
        • 召回代码示例
            │
            ▼
        结果融合
        • RRF (Reciprocal Rank Fusion)
        • 去重 + 重排序
            │
            ▼
        Rerank 精排
        • BGE-Reranker 重新评分
            │
            ▼
        Top-K 返回
```

---

## 知识图谱设计

### 实体-关系模型

```
┌─────────┐         ┌─────────┐         ┌─────────┐
│  ESP32  │──使用──►│  MQTT   │──需要──►│  WiFi   │
└─────────┘         └─────────┘         └─────────┘
     │                   │
     │包含               │支持
     ▼                   ▼
┌─────────┐         ┌─────────┐
│  GPIO   │         │  TLS    │
└─────────┘         └─────────┘
```

### 实体提取

```python
ENTITY_EXTRACTION_PROMPT = """
从以下文本中提取实体和关系。

文本：
{text}

实体类型：
- 技术/框架 (Technology)
- 组件/模块 (Component)
- 概念/术语 (Concept)
- 配置/参数 (Config)

输出 JSON：
{
  "entities": [
    {"name": "ESP32", "type": "Technology", "description": "..."},
    {"name": "MQTT", "type": "Technology", "description": "..."}
  ],
  "relations": [
    {"source": "ESP32", "relation": "使用", "target": "MQTT"},
    {"source": "MQTT", "relation": "需要", "target": "WiFi"}
  ]
}
"""
```

### 图谱存储

```python
# Redis Graph 或 Neo4j
# 这里使用 Redis + JSON 简化实现

# 实体存储
entity:{entity_id} = {
    "name": "ESP32",
    "type": "Technology",
    "description": "...",
    "embedding": [...],  # 向量
    "doc_ids": ["doc1", "doc2"]  # 来源文档
}

# 关系存储
relation:{relation_id} = {
    "source": "entity_id_1",
    "relation": "使用",
    "target": "entity_id_2",
    "weight": 0.8  # 关系强度
}

# 邻接表 (快速图遍历)
neighbors:{entity_id} = Set[entity_id]
```

---

## 知识入库流程

### 文档处理管线

```
原始文档 (PDF/MD/TXT/URL)
    │
    ▼
┌─────────────┐
│  文档解析    │  • PDF 提取文本
│             │  • Markdown 解析
│             │  • HTML 清洗
└─────────────┘
    │
    ▼
┌─────────────┐
│  文本分块    │  • 语义分块 (非固定长度)
│             │  • 保留上下文重叠
│             │  • 代码块特殊处理
└─────────────┘
    │
    ▼
┌─────────────┐
│  实体提取    │  • LLM 提取实体
│             │  • 构建关系三元组
│             │  • 去重合并
└─────────────┘
    │
    ├─────────────────────────┐
    ▼                         ▼
┌─────────────┐       ┌─────────────┐
│  向量化      │       │  图谱构建    │
│             │       │             │
│  Embedding  │       │  实体入库    │
│  Qdrant存储  │       │  关系入库    │
└─────────────┘       └─────────────┘
```

### 语义分块策略

```python
class SemanticChunker:
    """语义分块器"""
    
    def __init__(
        self,
        max_chunk_size: int = 512,
        overlap_size: int = 50,
        min_chunk_size: int = 100
    ):
        self.max_chunk_size = max_chunk_size
        self.overlap_size = overlap_size
        self.min_chunk_size = min_chunk_size
    
    def chunk(self, text: str) -> list[str]:
        """
        语义分块
        
        策略：
        1. 按段落分割
        2. 段落过长则按句子分割
        3. 保留上下文重叠
        4. 代码块不分割
        """
        chunks = []
        paragraphs = self.split_paragraphs(text)
        
        current_chunk = ""
        for para in paragraphs:
            # 代码块特殊处理
            if self.is_code_block(para):
                if current_chunk:
                    chunks.append(current_chunk)
                    current_chunk = ""
                chunks.append(para)  # 代码块单独成块
                continue
            
            # 检查是否超长
            if len(current_chunk) + len(para) > self.max_chunk_size:
                if current_chunk:
                    chunks.append(current_chunk)
                # 保留重叠
                current_chunk = self.get_overlap(current_chunk) + para
            else:
                current_chunk += para
        
        if current_chunk and len(current_chunk) >= self.min_chunk_size:
            chunks.append(current_chunk)
        
        return chunks
```

---

## 增量更新策略

### 问题：全量重建成本高

每次知识更新都重建索引：
- 耗时长
- 资源消耗大
- 服务不可用时间长

### 解决方案：增量更新

```
新文档/更新文档
    │
    ▼
┌─────────────┐
│  变更检测    │  • 计算文档 hash
│             │  • 对比历史版本
└─────────────┘
    │
    ├── 无变化 ──► 跳过
    │
    ├── 新增 ──► 直接入库
    │
    └── 更新 ──► 差异更新
            │
            ▼
      ┌─────────────┐
      │  删除旧块    │
      │  插入新块    │
      │  更新图谱    │
      └─────────────┘
```

### 增量更新实现

```python
class IncrementalUpdater:
    """增量更新器"""
    
    async def update_document(
        self,
        doc_id: str,
        new_content: str
    ):
        """增量更新文档"""
        
        # 1. 获取旧文档信息
        old_doc = await self.get_document(doc_id)
        if not old_doc:
            # 新文档，直接入库
            return await self.insert_document(doc_id, new_content)
        
        # 2. 计算变更
        old_hash = old_doc.content_hash
        new_hash = self.hash_content(new_content)
        
        if old_hash == new_hash:
            return  # 无变化
        
        # 3. 差异分析
        old_chunks = old_doc.chunks
        new_chunks = self.chunk(new_content)
        
        to_delete = []
        to_insert = []
        
        for chunk in new_chunks:
            chunk_hash = self.hash_content(chunk)
            if chunk_hash not in old_doc.chunk_hashes:
                to_insert.append(chunk)
        
        for old_chunk in old_chunks:
            if old_chunk.hash not in [self.hash_content(c) for c in new_chunks]:
                to_delete.append(old_chunk.id)
        
        # 4. 执行更新
        await self.delete_chunks(to_delete)
        await self.insert_chunks(doc_id, to_insert)
        
        # 5. 更新图谱 (增量)
        await self.update_graph(doc_id, new_content)
```

---

## 检索优化

### 查询理解

```python
class QueryUnderstanding:
    """查询理解"""
    
    async def analyze(self, query: str) -> QueryIntent:
        """
        分析查询意图
        
        返回：
        - 查询类型 (事实/操作/比较/推理)
        - 关键实体
        - 扩展词
        """
        prompt = f"""
分析用户查询意图：

查询：{query}

输出 JSON：
{{
  "type": "fact|how-to|compare|reason",
  "entities": ["ESP32", "MQTT"],
  "expansions": ["连接", "配置", "示例"]
}}
"""
        result = await self.llm.generate(prompt)
        return QueryIntent.from_json(result)
```

### 结果融合 (RRF)

```python
def reciprocal_rank_fusion(
    results: list[list[Document]],
    k: int = 60
) -> list[Document]:
    """
    RRF 融合多路检索结果
    
    score = sum(1 / (k + rank_i))
    """
    scores = defaultdict(float)
    doc_map = {}
    
    for result_list in results:
        for rank, doc in enumerate(result_list):
            scores[doc.id] += 1 / (k + rank + 1)
            doc_map[doc.id] = doc
    
    sorted_ids = sorted(scores.keys(), key=lambda x: scores[x], reverse=True)
    return [doc_map[doc_id] for doc_id in sorted_ids]
```

### Rerank 精排

```python
async def rerank(
    query: str,
    documents: list[Document],
    top_k: int = 5
) -> list[Document]:
    """
    使用 BGE-Reranker 重排序
    """
    pairs = [(query, doc.content) for doc in documents]
    
    # 调用 Rerank API
    response = await rerank_api.rerank(
        model="bge-reranker-v2-m3",
        query=query,
        documents=[doc.content for doc in documents],
        top_n=top_k
    )
    
    # 按新分数排序
    reranked = []
    for item in response.results:
        doc = documents[item.index]
        doc.rerank_score = item.relevance_score
        reranked.append(doc)
    
    return reranked
```

---

## MCP 工具接口

```python
class KnowledgeMCPService:
    """LightRAG MCP 服务"""
    
    tools = {
        # 查询
        "knowledge.query": "查询知识库 (支持 naive/local/global/hybrid)",
        "knowledge.get_entities": "获取相关实体",
        "knowledge.get_relations": "获取实体关系",
        
        # 写入
        "knowledge.insert": "插入单个文档",
        "knowledge.batch_insert": "批量插入文档",
        "knowledge.update": "更新文档",
        "knowledge.delete": "删除文档",
        
        # 管理
        "knowledge.stats": "知识库统计",
        "knowledge.rebuild_index": "重建索引",
    }
```

### 工具示例

```python
# 混合检索
results = await knowledge.query(
    query="ESP32 如何连接 MQTT?",
    mode="hybrid",
    top_k=5
)

# 获取实体关系
relations = await knowledge.get_relations(
    entity="ESP32",
    depth=2  # 2跳关系
)

# 插入文档
await knowledge.insert(
    content="ESP32 是乐鑫公司推出的...",
    metadata={"source": "docs", "category": "hardware"}
)
```

---

## 存储设计

### Qdrant Collection

```python
# 知识块向量存储
knowledge_chunks = {
    "name": "knowledge_chunks",
    "vectors": {
        "size": 1024,
        "distance": "Cosine"
    },
    "payload_schema": {
        "doc_id": "keyword",
        "content": "text",
        "chunk_index": "integer",
        "metadata": "json",
        "entities": "keyword[]",
        "created_at": "datetime",
        "updated_at": "datetime"
    }
}

# 实体向量存储 (用于实体链接)
entities = {
    "name": "entities",
    "vectors": {
        "size": 1024,
        "distance": "Cosine"
    },
    "payload_schema": {
        "name": "keyword",
        "type": "keyword",
        "description": "text",
        "doc_ids": "keyword[]"
    }
}
```

### Redis 索引

```python
# 文档元数据
doc:{doc_id} = {
    "title": "...",
    "source": "...",
    "content_hash": "...",
    "chunk_ids": [...],
    "created_at": "...",
    "updated_at": "..."
}

# 实体邻接表
entity_neighbors:{entity_id} = Set[entity_id]

# 关系索引
relations:{source_id}:{relation_type} = Set[target_id]
```

---

## 监控指标

| 指标                | 说明         | 告警阈值 |
| ------------------- | ------------ | -------- |
| `doc_count`         | 文档数量     | -        |
| `chunk_count`       | 分块数量     | -        |
| `entity_count`      | 实体数量     | -        |
| `query_latency_p99` | 查询延迟 P99 | > 2s     |
| `rerank_latency`    | 重排延迟     | > 500ms  |
| `index_freshness`   | 索引新鲜度   | > 1h     |
