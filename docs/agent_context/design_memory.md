# Mem0 记忆服务设计文档

> **定位**: 为 AI 助手提供类人记忆能力，让对话有温度、有延续。
> 
> **核心哲学**: 记忆不是数据库，而是**选择性遗忘与智能压缩的艺术**。

## 1. 设计目标与挑战

### 1.1 核心目标

| 目标 | 描述 | 度量指标 |
|------|------|----------|
| **有记忆** | AI 能记住用户偏好、历史对话 | 相关记忆召回率 > 85% |
| **会遗忘** | 自动衰减不重要的记忆 | 存储增长率 < 5%/月 |
| **能压缩** | 多次相似对话合并为一条 | 压缩比 > 3:1 |
| **快检索** | 毫秒级记忆召回 | P99 < 100ms |

### 1.2 核心挑战

```
┌─────────────────────────────────────────────────────────────┐
│                    记忆系统四大挑战                          │
├─────────────────────────────────────────────────────────────┤
│  ① 无限增长: 对话越多，记忆爆炸                              │
│     → 解法: 遗忘曲线 + 压缩合并                              │
│                                                             │
│  ② 检索效率: 海量记忆中找到相关的                            │
│     → 解法: 多路召回 + 重排序                                │
│                                                             │
│  ③ 记忆质量: 什么值得记？什么该忘？                          │
│     → 解法: 重要性评估 + 智能决策                            │
│                                                             │
│  ④ 一致性: 冲突记忆如何处理？                                │
│     → 解法: 时间戳优先 + 置信度评估                          │
└─────────────────────────────────────────────────────────────┘
```

## 2. 核心架构

### 2.1 三层记忆模型 (类人记忆)

```
┌──────────────────────────────────────────────────────────────────────┐
│                         三层记忆架构                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │                    工作记忆 (Working Memory)                  │    │
│   │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │    │
│   │  • 当前对话上下文 (最近 5-10 轮)                              │    │
│   │  • 存储: 内存/Redis                                          │    │
│   │  • 生命周期: 会话结束自动清空                                  │    │
│   │  • 容量: 每用户 ~4K tokens                                    │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                              ↓ 会话结束时提取                          │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │                   短期记忆 (Short-term Memory)                │    │
│   │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │    │
│   │  • 近期事件 (7 天内)                                         │    │
│   │  • 存储: Qdrant 向量库                                       │    │
│   │  • 生命周期: 按遗忘曲线衰减                                   │    │
│   │  • 容量: 每用户 ~500 条                                      │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                              ↓ 定期压缩/合并                          │
│   ┌─────────────────────────────────────────────────────────────┐    │
│   │                    长期记忆 (Long-term Memory)                │    │
│   │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │    │
│   │  • 压缩后的核心记忆、用户画像、稳定偏好                        │    │
│   │  • 存储: Qdrant + MySQL (结构化)                             │    │
│   │  • 生命周期: 永久 (但可被更新覆盖)                            │    │
│   │  • 容量: 每用户 ~100 条核心记忆                               │    │
│   └─────────────────────────────────────────────────────────────┘    │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 记忆数据结构

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Optional
import uuid


class MemoryType(Enum):
    """记忆类型"""
    EPISODIC = "episodic"       # 情景记忆: 具体事件 ("用户说他周五要出差")
    SEMANTIC = "semantic"       # 语义记忆: 抽象知识 ("用户是程序员")
    PREFERENCE = "preference"   # 偏好记忆: 个人喜好 ("喜欢简洁的回复")
    FACT = "fact"               # 事实记忆: 客观信息 ("用户住在北京")


class MemoryLayer(Enum):
    """记忆层级"""
    WORKING = "working"
    SHORT_TERM = "short_term"
    LONG_TERM = "long_term"


@dataclass
class Memory:
    """记忆实体"""
    id: str = field(default_factory=lambda: str(uuid.uuid4()))
    user_id: str = ""                          # 用户标识
    content: str = ""                          # 记忆内容 (自然语言)
    memory_type: MemoryType = MemoryType.EPISODIC
    layer: MemoryLayer = MemoryLayer.SHORT_TERM
    
    # 时间维度
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    last_accessed_at: datetime = field(default_factory=datetime.now)
    
    # 强度与衰减
    strength: float = 1.0                      # 记忆强度 [0, 1]
    access_count: int = 0                      # 访问次数
    importance: float = 0.5                    # 重要性评分 [0, 1]
    
    # 向量与元数据
    embedding: Optional[list[float]] = None   # 语义向量
    source_session_id: Optional[str] = None   # 来源会话
    related_memory_ids: list[str] = field(default_factory=list)  # 关联记忆
    tags: list[str] = field(default_factory=list)                # 标签
    
    # 压缩与合并
    is_compressed: bool = False               # 是否已压缩
    compression_sources: list[str] = field(default_factory=list)  # 压缩来源
    confidence: float = 1.0                   # 置信度 (冲突解决用)
```

## 3. 智能决策机制 (核心创新)

### 3.1 记忆提取决策器

> **核心问题**: 不是所有对话都值得记住。如何自动判断？

```python
from abc import ABC, abstractmethod


class MemoryExtractionDecider:
    """
    决定从对话中提取什么作为记忆。
    
    设计原则:
    1. 宁缺毋滥 - 低质量记忆污染检索
    2. 事件驱动 - 有意义的变化才触发
    3. 用户中心 - 关注用户信息而非泛泛对话
    """
    
    # 记忆提取规则
    EXTRACTION_PATTERNS = {
        # 高优先级 - 几乎必提取
        "explicit_preference": {
            "patterns": ["我喜欢", "我不喜欢", "我偏好", "我讨厌"],
            "importance": 0.9,
            "memory_type": MemoryType.PREFERENCE
        },
        "personal_fact": {
            "patterns": ["我是", "我叫", "我住在", "我在...工作"],
            "importance": 0.85,
            "memory_type": MemoryType.FACT
        },
        "future_plan": {
            "patterns": ["我打算", "我计划", "下周我要", "明天我会"],
            "importance": 0.8,
            "memory_type": MemoryType.EPISODIC
        },
        
        # 中优先级 - 需要上下文判断
        "emotional_event": {
            "patterns": ["开心", "难过", "生气", "兴奋", "担心"],
            "importance": 0.6,
            "memory_type": MemoryType.EPISODIC
        },
        
        # 低优先级 - 通常忽略
        "casual_chat": {
            "patterns": ["天气", "吃什么", "无聊"],
            "importance": 0.2,
            "memory_type": MemoryType.EPISODIC
        }
    }
    
    async def should_extract(
        self,
        conversation: list[dict],
        llm_client: "LLMClient"
    ) -> list[dict]:
        """
        分析对话，决定提取哪些记忆。
        
        Returns:
            [{"content": "用户是程序员", "importance": 0.85, "type": "fact"}, ...]
        """
        # 第一层: 规则快速过滤
        candidates = self._rule_based_filter(conversation)
        
        if not candidates:
            return []
        
        # 第二层: LLM 精细判断
        prompt = self._build_extraction_prompt(conversation, candidates)
        result = await llm_client.chat(prompt, model="glm-4.5-flash")
        
        return self._parse_extraction_result(result)
    
    def _rule_based_filter(self, conversation: list[dict]) -> list[str]:
        """规则快速筛选候选句子"""
        candidates = []
        for msg in conversation:
            if msg["role"] != "user":
                continue
            content = msg["content"]
            for pattern_info in self.EXTRACTION_PATTERNS.values():
                if any(p in content for p in pattern_info["patterns"]):
                    candidates.append(content)
                    break
        return candidates
    
    def _build_extraction_prompt(
        self,
        conversation: list[dict],
        candidates: list[str]
    ) -> str:
        """构建 LLM 提取 prompt"""
        return f"""分析以下对话，提取值得长期记住的用户信息。

## 对话内容
{self._format_conversation(conversation)}

## 候选句子
{candidates}

## 提取规则
1. 只提取关于用户本人的信息，忽略泛泛而谈
2. 将模糊表述转为明确事实 (例: "我老是加班" → "用户经常加班")  
3. 合并重复信息
4. 排除临时性、无意义内容

## 输出格式 (JSON)
[
  {{"content": "用户是后端程序员", "importance": 0.85, "type": "fact"}},
  {{"content": "用户喜欢简洁的回复风格", "importance": 0.9, "type": "preference"}}
]

如果没有值得记住的内容，返回空数组 []。
"""
```

### 3.2 重要性评估器

```python
class ImportanceEvaluator:
    """
    评估记忆的重要性，决定保留优先级。
    
    评分维度:
    1. 稀缺性 - 是否为独特信息
    2. 影响力 - 是否影响后续对话
    3. 时效性 - 是否仍然有效
    4. 确定性 - 表述是否明确
    """
    
    # 重要性权重
    WEIGHTS = {
        "scarcity": 0.25,      # 稀缺性
        "impact": 0.35,        # 影响力
        "recency": 0.20,       # 时效性  
        "certainty": 0.20      # 确定性
    }
    
    async def evaluate(
        self,
        memory: Memory,
        existing_memories: list[Memory],
        llm_client: "LLMClient"
    ) -> float:
        """评估记忆重要性，返回 [0, 1] 分数"""
        
        scores = {}
        
        # 1. 稀缺性: 与已有记忆的重复度
        scores["scarcity"] = self._calc_scarcity(memory, existing_memories)
        
        # 2. 影响力: 类型权重
        type_impact = {
            MemoryType.PREFERENCE: 0.9,  # 偏好影响每次对话
            MemoryType.FACT: 0.8,        # 事实是基础信息
            MemoryType.EPISODIC: 0.5,    # 事件可能一次性
            MemoryType.SEMANTIC: 0.7     # 抽象知识有参考价值
        }
        scores["impact"] = type_impact.get(memory.memory_type, 0.5)
        
        # 3. 时效性: 时间衰减
        age_days = (datetime.now() - memory.created_at).days
        scores["recency"] = max(0, 1 - age_days / 30)  # 30天衰减到0
        
        # 4. 确定性: LLM 评估表述的明确程度
        scores["certainty"] = await self._llm_certainty_check(
            memory.content, llm_client
        )
        
        # 加权求和
        final_score = sum(
            scores[k] * self.WEIGHTS[k] for k in self.WEIGHTS
        )
        
        return round(final_score, 3)
    
    def _calc_scarcity(
        self,
        memory: Memory,
        existing: list[Memory]
    ) -> float:
        """计算稀缺性 (与已有记忆的差异度)"""
        if not existing or memory.embedding is None:
            return 1.0
        
        # 找最相似的记忆
        max_similarity = 0
        for m in existing:
            if m.embedding is not None:
                sim = self._cosine_similarity(memory.embedding, m.embedding)
                max_similarity = max(max_similarity, sim)
        
        # 差异度 = 1 - 相似度
        return 1 - max_similarity
```

## 4. 遗忘曲线与记忆衰减

### 4.1 艾宾浩斯遗忘曲线实现

```python
import math


class ForgettingCurve:
    """
    基于艾宾浩斯遗忘曲线的记忆衰减模型。
    
    公式: R = e^(-t/S)
    - R: 保持率 (retention)
    - t: 时间 (天)
    - S: 记忆强度 (strength), 越高衰减越慢
    
    影响记忆强度的因素:
    1. 重要性 (importance)
    2. 访问频率 (access_count)  
    3. 初始强度 (base_strength)
    """
    
    # 最小保留阈值 (低于此值可被清理)
    MIN_RETENTION = 0.1
    
    # 强度计算参数
    BASE_STRENGTH = 1.0
    IMPORTANCE_MULTIPLIER = 2.0
    ACCESS_BONUS = 0.1  # 每次访问增加
    MAX_STRENGTH = 10.0
    
    def calculate_retention(self, memory: Memory) -> float:
        """计算当前记忆保持率"""
        age_days = (datetime.now() - memory.created_at).days
        if age_days <= 0:
            return 1.0
        
        # 计算有效强度
        strength = self._calculate_strength(memory)
        
        # 艾宾浩斯公式
        retention = math.exp(-age_days / strength)
        
        return round(max(retention, 0), 4)
    
    def _calculate_strength(self, memory: Memory) -> float:
        """
        计算记忆强度。
        
        强度越高，遗忘越慢。
        """
        strength = self.BASE_STRENGTH
        
        # 重要性加成
        strength += memory.importance * self.IMPORTANCE_MULTIPLIER
        
        # 访问频率加成 (复习效应)
        strength += memory.access_count * self.ACCESS_BONUS
        
        # 类型加成 (偏好类记忆更持久)
        if memory.memory_type == MemoryType.PREFERENCE:
            strength *= 1.5
        elif memory.memory_type == MemoryType.FACT:
            strength *= 1.3
        
        return min(strength, self.MAX_STRENGTH)
    
    def should_forget(self, memory: Memory) -> bool:
        """判断记忆是否应该被遗忘"""
        retention = self.calculate_retention(memory)
        return retention < self.MIN_RETENTION
    
    def refresh_memory(self, memory: Memory) -> Memory:
        """
        刷新记忆 (被访问时调用)。
        
        类似复习效应，增强记忆强度。
        """
        memory.last_accessed_at = datetime.now()
        memory.access_count += 1
        memory.strength = self._calculate_strength(memory)
        return memory
```

### 4.2 定时衰减任务

```python
class MemoryDecayScheduler:
    """
    定时运行记忆衰减任务。
    
    策略:
    1. 每日凌晨执行一次全量衰减计算
    2. 低保持率记忆进入候选清理队列
    3. 批量清理前做最后确认
    """
    
    def __init__(
        self,
        memory_store: "MemoryStore",
        forgetting_curve: ForgettingCurve
    ):
        self.store = memory_store
        self.curve = forgetting_curve
    
    async def run_decay_cycle(self, user_id: str) -> dict:
        """
        执行一轮衰减循环。
        
        Returns:
            {"processed": 100, "decayed": 20, "forgotten": 5}
        """
        stats = {"processed": 0, "decayed": 0, "forgotten": 0}
        
        # 获取用户所有短期记忆
        memories = await self.store.get_by_user(
            user_id, 
            layer=MemoryLayer.SHORT_TERM
        )
        
        forget_candidates = []
        
        for memory in memories:
            stats["processed"] += 1
            
            # 计算当前保持率
            retention = self.curve.calculate_retention(memory)
            
            if self.curve.should_forget(memory):
                forget_candidates.append(memory)
                stats["forgotten"] += 1
            elif retention < memory.strength:
                # 更新强度
                memory.strength = retention
                await self.store.update(memory)
                stats["decayed"] += 1
        
        # 批量清理
        if forget_candidates:
            await self._process_forget_candidates(forget_candidates)
        
        return stats
    
    async def _process_forget_candidates(
        self,
        candidates: list[Memory]
    ) -> None:
        """
        处理待遗忘记忆。
        
        不是直接删除，而是:
        1. 尝试压缩合并到长期记忆
        2. 无法合并的才真正删除
        """
        for memory in candidates:
            # 尝试找可合并的长期记忆
            merged = await self._try_merge_to_long_term(memory)
            
            if merged:
                # 成功合并，删除短期版本
                await self.store.delete(memory.id)
            else:
                # 无法合并，标记为可删除
                memory.layer = MemoryLayer.SHORT_TERM
                memory.tags.append("pending_deletion")
                await self.store.update(memory)
```

## 5. 记忆压缩与合并

### 5.1 压缩策略

```python
class MemoryCompressor:
    """
    记忆压缩器 - 解决存储无限增长问题。
    
    压缩策略:
    1. 相似合并: 相似度 > 0.85 的记忆合并为一条
    2. 摘要提取: 多条情景记忆提取为语义记忆
    3. 模糊化: 具体细节泛化为模式 (见 5.2)
    """
    
    SIMILARITY_THRESHOLD = 0.85
    MIN_MERGE_COUNT = 3  # 至少3条才考虑合并
    
    async def compress_user_memories(
        self,
        user_id: str,
        memory_store: "MemoryStore",
        llm_client: "LLMClient"
    ) -> dict:
        """
        压缩用户记忆。
        
        Returns:
            {"merged": 10, "summarized": 5, "saved_bytes": 1024}
        """
        stats = {"merged": 0, "summarized": 0, "saved_bytes": 0}
        
        # 1. 相似记忆聚类
        memories = await memory_store.get_by_user(user_id)
        clusters = self._cluster_by_similarity(memories)
        
        for cluster in clusters:
            if len(cluster) >= self.MIN_MERGE_COUNT:
                # 合并为一条
                merged = await self._merge_memories(cluster, llm_client)
                
                # 保存合并后的记忆
                await memory_store.add(merged)
                
                # 删除原始记忆
                for m in cluster:
                    await memory_store.delete(m.id)
                    stats["saved_bytes"] += len(m.content.encode())
                
                stats["merged"] += len(cluster)
        
        # 2. 情景记忆摘要
        episodic = [m for m in memories if m.memory_type == MemoryType.EPISODIC]
        if len(episodic) > 20:
            summary = await self._summarize_episodic(episodic, llm_client)
            await memory_store.add(summary)
            stats["summarized"] += 1
        
        return stats
    
    async def _merge_memories(
        self,
        memories: list[Memory],
        llm_client: "LLMClient"
    ) -> Memory:
        """将多条相似记忆合并为一条"""
        contents = [m.content for m in memories]
        
        prompt = f"""合并以下相似的记忆为一条简洁的总结:

{chr(10).join(f'- {c}' for c in contents)}

要求:
1. 保留核心信息
2. 去除重复表述
3. 使用陈述句
4. 控制在50字以内

合并结果:"""
        
        merged_content = await llm_client.chat(prompt)
        
        # 取最高重要性和最新时间
        max_importance = max(m.importance for m in memories)
        latest_time = max(m.created_at for m in memories)
        
        return Memory(
            user_id=memories[0].user_id,
            content=merged_content.strip(),
            memory_type=memories[0].memory_type,
            layer=MemoryLayer.LONG_TERM,
            importance=max_importance,
            created_at=latest_time,
            is_compressed=True,
            compression_sources=[m.id for m in memories]
        )
```

### 5.2 模糊化层次 (核心创新)

> **灵感来源**: 人类记忆会随时间从具体细节泛化为抽象模式。
> 
> 例如: "周一买了咖啡" + "周三买了咖啡" + "周五买了咖啡" → "用户有喝咖啡的习惯"

```python
class MemoryAbstractor:
    """
    记忆模糊化/抽象化处理器。
    
    层次结构:
    L0 (具体): "2024-01-15 在星巴克买了拿铁"
    L1 (泛化): "用户经常买拿铁"  
    L2 (抽象): "用户喜欢喝咖啡"
    L3 (模式): "用户有规律的饮品消费习惯"
    """
    
    ABSTRACTION_LEVELS = {
        0: "具体事件 (时间+地点+行为)",
        1: "行为泛化 (去除时间)",
        2: "类别抽象 (去除具体对象)",
        3: "模式总结 (提取规律)"
    }
    
    async def abstract_memories(
        self,
        memories: list[Memory],
        target_level: int,
        llm_client: "LLMClient"
    ) -> Memory:
        """
        将记忆提升到更高抽象层次。
        
        Args:
            memories: 待抽象的记忆列表
            target_level: 目标抽象层级 (1-3)
            llm_client: LLM 客户端
        """
        contents = [m.content for m in memories]
        
        level_prompts = {
            1: "去除具体时间，保留行为模式",
            2: "将具体对象替换为类别 (如 '拿铁' → '咖啡')",
            3: "提取底层规律或习惯"
        }
        
        prompt = f"""将以下记忆抽象到更高层次。

## 原始记忆
{chr(10).join(f'- {c}' for c in contents)}

## 抽象要求
{level_prompts.get(target_level, level_prompts[1])}

## 输出格式
一句话总结，不超过30字。

抽象结果:"""
        
        abstract_content = await llm_client.chat(prompt)
        
        return Memory(
            user_id=memories[0].user_id,
            content=abstract_content.strip(),
            memory_type=MemoryType.SEMANTIC,  # 抽象后变为语义记忆
            layer=MemoryLayer.LONG_TERM,
            importance=max(m.importance for m in memories),
            is_compressed=True,
            compression_sources=[m.id for m in memories],
            tags=[f"abstraction_level_{target_level}"]
        )
```

## 6. 多路召回与检索

### 6.1 检索架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          多路召回架构                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   用户提问: "帮我推荐一家餐厅"                                            │
│        │                                                                 │
│        ▼                                                                 │
│   ┌──────────────────────────────────────────────────────────────┐      │
│   │                    并行多路召回                                │      │
│   ├──────────────────────────────────────────────────────────────┤      │
│   │                                                              │      │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │      │
│   │  │ 语义召回    │  │ 关键词召回  │  │ 规则召回    │             │      │
│   │  │ (向量)     │  │ (BM25)     │  │ (偏好标签)  │             │      │
│   │  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘             │      │
│   │         │              │               │                    │      │
│   │         └──────────────┼───────────────┘                    │      │
│   │                        ▼                                    │      │
│   │              ┌─────────────────┐                            │      │
│   │              │    去重 & 合并   │                            │      │
│   │              └────────┬────────┘                            │      │
│   │                       ▼                                     │      │
│   │              ┌─────────────────┐                            │      │
│   │              │  重排序 (Rerank) │  ← 基于当前上下文           │      │
│   │              └────────┬────────┘                            │      │
│   │                       ▼                                     │      │
│   │              ┌─────────────────┐                            │      │
│   │              │   Top-K 输出    │  → 3-5 条最相关记忆         │      │
│   │              └─────────────────┘                            │      │
│   │                                                              │      │
│   └──────────────────────────────────────────────────────────────┘      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 检索实现

```python
from dataclasses import dataclass


@dataclass
class RetrievalResult:
    """检索结果"""
    memory: Memory
    score: float
    source: str  # "semantic" | "keyword" | "rule"


class MemoryRetriever:
    """
    多路召回记忆检索器。
    """
    
    def __init__(
        self,
        vector_store: "QdrantClient",
        embedding_model: "EmbeddingModel",
        reranker: "Reranker"
    ):
        self.vector_store = vector_store
        self.embedding = embedding_model
        self.reranker = reranker
    
    async def retrieve(
        self,
        query: str,
        user_id: str,
        top_k: int = 5,
        include_working: bool = True
    ) -> list[RetrievalResult]:
        """
        多路召回检索。
        
        Args:
            query: 用户查询
            user_id: 用户ID
            top_k: 返回数量
            include_working: 是否包含工作记忆
        """
        # 并行执行多路召回
        semantic_task = self._semantic_recall(query, user_id)
        keyword_task = self._keyword_recall(query, user_id)
        rule_task = self._rule_recall(query, user_id)
        
        semantic_results, keyword_results, rule_results = await asyncio.gather(
            semantic_task, keyword_task, rule_task
        )
        
        # 合并去重
        all_results = self._merge_and_dedupe(
            semantic_results, keyword_results, rule_results
        )
        
        # 重排序
        reranked = await self._rerank(query, all_results)
        
        # 刷新被访问的记忆
        for r in reranked[:top_k]:
            await self._refresh_accessed(r.memory)
        
        return reranked[:top_k]
    
    async def _semantic_recall(
        self,
        query: str,
        user_id: str,
        limit: int = 20
    ) -> list[RetrievalResult]:
        """语义向量召回"""
        query_embedding = await self.embedding.embed(query)
        
        results = await self.vector_store.search(
            collection_name="memories",
            query_vector=query_embedding,
            query_filter={"user_id": user_id},
            limit=limit
        )
        
        return [
            RetrievalResult(
                memory=self._to_memory(r),
                score=r.score,
                source="semantic"
            )
            for r in results
        ]
    
    async def _keyword_recall(
        self,
        query: str,
        user_id: str,
        limit: int = 10
    ) -> list[RetrievalResult]:
        """关键词召回 (BM25)"""
        # 提取关键词
        keywords = self._extract_keywords(query)
        
        # 全文搜索
        results = await self.vector_store.search(
            collection_name="memories",
            query_filter={
                "user_id": user_id,
                "content": {"$contains": keywords}
            },
            limit=limit
        )
        
        return [
            RetrievalResult(
                memory=self._to_memory(r),
                score=0.5,  # 关键词匹配给固定分
                source="keyword"
            )
            for r in results
        ]
    
    async def _rule_recall(
        self,
        query: str,
        user_id: str
    ) -> list[RetrievalResult]:
        """规则召回 (偏好类记忆)"""
        # 某些查询总是需要偏好记忆
        need_preference = any(
            kw in query for kw in ["推荐", "建议", "喜欢", "偏好"]
        )
        
        if not need_preference:
            return []
        
        # 获取用户所有偏好记忆
        preferences = await self.vector_store.search(
            collection_name="memories",
            query_filter={
                "user_id": user_id,
                "memory_type": MemoryType.PREFERENCE.value
            },
            limit=5
        )
        
        return [
            RetrievalResult(
                memory=self._to_memory(r),
                score=0.8,  # 偏好记忆高权重
                source="rule"
            )
            for r in preferences
        ]
    
    async def _rerank(
        self,
        query: str,
        results: list[RetrievalResult]
    ) -> list[RetrievalResult]:
        """重排序"""
        if not results:
            return []
        
        # 使用 Reranker 模型
        texts = [r.memory.content for r in results]
        scores = await self.reranker.rerank(query, texts)
        
        # 更新分数
        for i, result in enumerate(results):
            result.score = scores[i]
        
        # 按分数降序
        return sorted(results, key=lambda x: x.score, reverse=True)
```

## 7. MCP 工具接口

### 7.1 工具定义

```python
from typing import Literal

# MCP 工具 Schema
MCP_MEMORY_TOOLS = [
    {
        "name": "memory.add",
        "description": "添加一条记忆到记忆库",
        "input_schema": {
            "type": "object",
            "properties": {
                "content": {
                    "type": "string",
                    "description": "记忆内容 (自然语言描述)"
                },
                "memory_type": {
                    "type": "string",
                    "enum": ["episodic", "semantic", "preference", "fact"],
                    "description": "记忆类型"
                },
                "importance": {
                    "type": "number",
                    "minimum": 0,
                    "maximum": 1,
                    "description": "重要性评分"
                }
            },
            "required": ["content"]
        }
    },
    {
        "name": "memory.search",
        "description": "搜索相关记忆",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索查询"
                },
                "top_k": {
                    "type": "integer",
                    "default": 5,
                    "description": "返回数量"
                },
                "memory_types": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "过滤记忆类型"
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "memory.get_context",
        "description": "获取当前对话的记忆上下文",
        "input_schema": {
            "type": "object",
            "properties": {
                "max_tokens": {
                    "type": "integer",
                    "default": 1000,
                    "description": "最大 token 数"
                }
            }
        }
    },
    {
        "name": "memory.update",
        "description": "更新已有记忆",
        "input_schema": {
            "type": "object",
            "properties": {
                "memory_id": {
                    "type": "string",
                    "description": "记忆ID"
                },
                "content": {
                    "type": "string",
                    "description": "新内容"
                }
            },
            "required": ["memory_id", "content"]
        }
    },
    {
        "name": "memory.forget",
        "description": "主动遗忘一条记忆 (用户要求或隐私保护)",
        "input_schema": {
            "type": "object",
            "properties": {
                "memory_id": {
                    "type": "string",
                    "description": "记忆ID"
                },
                "reason": {
                    "type": "string",
                    "description": "遗忘原因"
                }
            },
            "required": ["memory_id"]
        }
    }
]
```

### 7.2 工具实现

```python
class MemoryMCPHandler:
    """
    MCP 记忆工具处理器。
    """
    
    def __init__(
        self,
        memory_store: "MemoryStore",
        retriever: MemoryRetriever,
        extractor: MemoryExtractionDecider
    ):
        self.store = memory_store
        self.retriever = retriever
        self.extractor = extractor
    
    async def handle_tool_call(
        self,
        tool_name: str,
        arguments: dict,
        user_id: str
    ) -> dict:
        """处理 MCP 工具调用"""
        handlers = {
            "memory.add": self._handle_add,
            "memory.search": self._handle_search,
            "memory.get_context": self._handle_get_context,
            "memory.update": self._handle_update,
            "memory.forget": self._handle_forget
        }
        
        handler = handlers.get(tool_name)
        if not handler:
            return {"error": f"Unknown tool: {tool_name}"}
        
        return await handler(arguments, user_id)
    
    async def _handle_add(self, args: dict, user_id: str) -> dict:
        """添加记忆"""
        memory = Memory(
            user_id=user_id,
            content=args["content"],
            memory_type=MemoryType(args.get("memory_type", "episodic")),
            importance=args.get("importance", 0.5)
        )
        
        # 生成 embedding
        memory.embedding = await self.retriever.embedding.embed(memory.content)
        
        # 保存
        await self.store.add(memory)
        
        return {
            "success": True,
            "memory_id": memory.id,
            "message": f"记忆已保存: {memory.content[:50]}..."
        }
    
    async def _handle_search(self, args: dict, user_id: str) -> dict:
        """搜索记忆"""
        results = await self.retriever.retrieve(
            query=args["query"],
            user_id=user_id,
            top_k=args.get("top_k", 5)
        )
        
        return {
            "memories": [
                {
                    "id": r.memory.id,
                    "content": r.memory.content,
                    "type": r.memory.memory_type.value,
                    "score": r.score,
                    "created_at": r.memory.created_at.isoformat()
                }
                for r in results
            ]
        }
    
    async def _handle_get_context(self, args: dict, user_id: str) -> dict:
        """
        获取对话上下文。
        
        这是最常用的接口，用于在对话开始时获取用户相关记忆。
        """
        max_tokens = args.get("max_tokens", 1000)
        
        # 获取用户画像 (长期记忆)
        profile = await self._get_user_profile(user_id)
        
        # 获取最近记忆 (短期)
        recent = await self.store.get_by_user(
            user_id,
            layer=MemoryLayer.SHORT_TERM,
            limit=10,
            order_by="last_accessed_at"
        )
        
        # 组装上下文
        context = self._format_context(profile, recent, max_tokens)
        
        return {
            "context": context,
            "profile_summary": profile.get("summary", ""),
            "recent_topics": [m.content for m in recent[:3]]
        }
    
    async def _handle_forget(self, args: dict, user_id: str) -> dict:
        """主动遗忘"""
        memory_id = args["memory_id"]
        reason = args.get("reason", "user_request")
        
        # 验证所有权
        memory = await self.store.get(memory_id)
        if not memory or memory.user_id != user_id:
            return {"error": "Memory not found or access denied"}
        
        # 软删除 (保留审计记录)
        await self.store.soft_delete(memory_id, reason)
        
        return {
            "success": True,
            "message": "记忆已遗忘"
        }
```

## 8. 事件驱动架构

### 8.1 记忆事件流

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         事件驱动记忆流                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────┐                                                   │
│   │   对话结束事件   │                                                   │
│   │ SessionEndEvent │                                                   │
│   └────────┬────────┘                                                   │
│            │                                                             │
│            ▼                                                             │
│   ┌─────────────────┐     ┌─────────────────┐                          │
│   │  提取决策器      │────►│   记忆存储       │                          │
│   │ ExtractionDecider│     │  MemoryStore    │                          │
│   └─────────────────┘     └────────┬────────┘                          │
│                                    │                                     │
│                                    ▼                                     │
│   ┌─────────────────────────────────────────────────────────────┐      │
│   │                    后台任务队列 (BullMQ)                      │      │
│   ├─────────────────────────────────────────────────────────────┤      │
│   │                                                              │      │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────┐             │      │
│   │  │ 衰减任务    │  │ 压缩任务    │  │ 抽象任务    │             │      │
│   │  │ DecayJob   │  │ CompressJob│  │ AbstractJob│             │      │
│   │  └────────────┘  └────────────┘  └────────────┘             │      │
│   │                                                              │      │
│   └─────────────────────────────────────────────────────────────┘      │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 事件处理器

```python
from dataclasses import dataclass
from typing import Callable


@dataclass
class MemoryEvent:
    """记忆事件"""
    event_type: str
    user_id: str
    payload: dict
    timestamp: datetime = None
    
    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()


class MemoryEventBus:
    """
    记忆事件总线。
    
    支持的事件:
    - session.end: 会话结束，触发记忆提取
    - memory.added: 记忆添加，触发向量索引
    - memory.accessed: 记忆访问，触发强度刷新
    - schedule.daily: 每日定时，触发衰减/压缩
    """
    
    def __init__(self):
        self._handlers: dict[str, list[Callable]] = {}
    
    def on(self, event_type: str, handler: Callable):
        """注册事件处理器"""
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)
    
    async def emit(self, event: MemoryEvent):
        """触发事件"""
        handlers = self._handlers.get(event.event_type, [])
        for handler in handlers:
            try:
                await handler(event)
            except Exception as e:
                logging.error(f"Handler error for {event.event_type}: {e}")


# 事件处理器示例
async def on_session_end(event: MemoryEvent):
    """会话结束时提取记忆"""
    conversation = event.payload.get("conversation", [])
    user_id = event.user_id
    
    # 提取记忆
    extractor = MemoryExtractionDecider()
    memories_to_add = await extractor.should_extract(conversation, llm_client)
    
    # 添加到存储
    for mem_data in memories_to_add:
        memory = Memory(
            user_id=user_id,
            content=mem_data["content"],
            importance=mem_data["importance"],
            memory_type=MemoryType(mem_data["type"])
        )
        await memory_store.add(memory)


async def on_daily_schedule(event: MemoryEvent):
    """每日定时任务"""
    user_id = event.user_id
    
    # 1. 执行衰减
    decay_scheduler = MemoryDecayScheduler(memory_store, ForgettingCurve())
    await decay_scheduler.run_decay_cycle(user_id)
    
    # 2. 执行压缩
    compressor = MemoryCompressor()
    await compressor.compress_user_memories(user_id, memory_store, llm_client)
```

## 9. 冲突解决机制

### 9.1 冲突检测

```python
class MemoryConflictResolver:
    """
    记忆冲突检测与解决。
    
    冲突类型:
    1. 直接矛盾: "用户住在北京" vs "用户住在上海"
    2. 时间过期: "用户在A公司工作" (但用户已跳槽)
    3. 部分重叠: "用户喜欢咖啡" vs "用户喜欢拿铁"
    """
    
    CONFLICT_THRESHOLD = 0.8  # 相似度超过此值需要检查冲突
    
    async def check_and_resolve(
        self,
        new_memory: Memory,
        existing_memories: list[Memory],
        llm_client: "LLMClient"
    ) -> tuple[Memory, list[str]]:
        """
        检查并解决冲突。
        
        Returns:
            (最终记忆, 需要删除的旧记忆ID列表)
        """
        # 找相似记忆
        similar = [
            m for m in existing_memories
            if self._similarity(new_memory, m) > self.CONFLICT_THRESHOLD
        ]
        
        if not similar:
            return new_memory, []
        
        # 检查是否真的冲突
        conflicts = await self._detect_conflicts(
            new_memory, similar, llm_client
        )
        
        if not conflicts:
            return new_memory, []
        
        # 解决冲突
        return await self._resolve_conflicts(
            new_memory, conflicts, llm_client
        )
    
    async def _detect_conflicts(
        self,
        new: Memory,
        candidates: list[Memory],
        llm_client: "LLMClient"
    ) -> list[Memory]:
        """检测实际冲突"""
        prompt = f"""判断新记忆是否与已有记忆存在冲突。

新记忆: {new.content}

已有记忆:
{chr(10).join(f'{i+1}. {m.content}' for i, m in enumerate(candidates))}

冲突判断规则:
1. 直接矛盾 (如地点、事实不一致) → 冲突
2. 时间更新 (如工作变动) → 冲突 (新的覆盖旧的)
3. 补充细化 (如 "喜欢咖啡" → "喜欢拿铁") → 不冲突，可共存

输出格式:
返回冲突的记忆序号，用逗号分隔。如果没有冲突，返回 "无"。
"""
        
        result = await llm_client.chat(prompt)
        
        if "无" in result:
            return []
        
        # 解析冲突序号
        conflict_indices = [int(x.strip()) - 1 for x in result.split(",") if x.strip().isdigit()]
        return [candidates[i] for i in conflict_indices if i < len(candidates)]
    
    async def _resolve_conflicts(
        self,
        new: Memory,
        conflicts: list[Memory],
        llm_client: "LLMClient"
    ) -> tuple[Memory, list[str]]:
        """
        解决冲突策略:
        1. 时间优先: 新记忆覆盖旧记忆
        2. 置信度: 高置信度优先
        3. 合并: 尝试合并为一条
        """
        to_delete = []
        
        for old in conflicts:
            # 时间优先策略
            if new.created_at > old.created_at:
                to_delete.append(old.id)
            # 如果新的更早但置信度更高
            elif new.confidence > old.confidence + 0.2:
                to_delete.append(old.id)
            else:
                # 尝试合并
                merged_content = await self._merge_conflict(
                    new.content, old.content, llm_client
                )
                new.content = merged_content
                to_delete.append(old.id)
        
        return new, to_delete
```

## 10. 部署与监控

### 10.1 Docker 部署

```yaml
# docker-compose.mem0.yml
services:
  mem0-service:
    build:
      context: .
      dockerfile: Dockerfile.mem0
    ports:
      - "3001:3001"
    environment:
      - QDRANT_HOST=qdrant
      - QDRANT_PORT=6333
      - REDIS_URL=redis://redis:6379
      - LLM_API_KEY=${GLM_API_KEY}
      - EMBEDDING_API_KEY=${GITEE_API_KEY}
    depends_on:
      - qdrant
      - redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"
    volumes:
      - qdrant_data:/qdrant/storage

  redis:
    image: redis:alpine
    ports:
      - "6379:6379"

volumes:
  qdrant_data:
```

### 10.2 监控指标

```python
from prometheus_client import Counter, Histogram, Gauge

# 核心指标
MEMORY_ADD_TOTAL = Counter(
    "memory_add_total",
    "Total memories added",
    ["user_id", "memory_type"]
)

MEMORY_SEARCH_LATENCY = Histogram(
    "memory_search_latency_seconds",
    "Memory search latency",
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0]
)

MEMORY_STORE_SIZE = Gauge(
    "memory_store_size",
    "Total memories in store",
    ["layer"]
)

MEMORY_COMPRESSION_RATIO = Gauge(
    "memory_compression_ratio",
    "Memory compression ratio"
)

MEMORY_RETENTION_RATE = Gauge(
    "memory_retention_rate",
    "Average memory retention rate",
    ["user_id"]
)
```

### 10.3 告警规则

```yaml
# alerts.yml
groups:
  - name: mem0_alerts
    rules:
      - alert: HighMemorySearchLatency
        expr: histogram_quantile(0.99, memory_search_latency_seconds) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "记忆搜索延迟过高"
          
      - alert: MemoryStoreGrowthTooFast
        expr: rate(memory_store_size{layer="short_term"}[1h]) > 100
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "短期记忆增长过快，检查压缩任务"
          
      - alert: LowRetentionRate
        expr: memory_retention_rate < 0.3
        for: 24h
        labels:
          severity: info
        annotations:
          summary: "记忆保持率过低，可能需要调整衰减参数"
```

## 11. 未来演进

### 11.1 路线图

| 阶段 | 功能 | 预计时间 |
|------|------|----------|
| v1.0 | 基础记忆 CRUD + 简单检索 | Month 1 |
| v1.5 | 遗忘曲线 + 压缩合并 | Month 2 |
| v2.0 | 智能提取决策 + 多路召回 | Month 3 |
| v2.5 | 模糊化层次 + 冲突解决 | Month 4 |
| v3.0 | 多模态记忆 (图片/语音) | Month 6 |

### 11.2 技术债务

- [ ] 支持跨用户记忆隔离的多租户架构
- [ ] 实现记忆导出/导入 (用户数据可携带)
- [ ] 添加记忆解释性 (为什么记住这个)
- [ ] 优化冷启动 (新用户无记忆时的体验)

---

> **设计哲学**: 好的记忆系统不是记住一切，而是在对的时间想起对的事。
