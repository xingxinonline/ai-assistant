# Scheduler 定时提醒服务设计

本文档详细说明 AI Assistant 的定时提醒系统设计。

---

## 概述

为 AI 助手提供智能定时提醒能力，支持对话式交互和确认重试机制。

### 核心问题

| 问题         | 挑战                   |
| ------------ | ---------------------- |
| **可靠触发** | 确保提醒不丢失、不重复 |
| **用户确认** | 重要提醒需确保用户知晓 |
| **自然交互** | 提醒不是冷冰冰的通知   |
| **灵活调度** | 支持多种时间表达方式   |

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      定时提醒系统                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    MCP 接口层                        │   │
│  │                                                     │   │
│  │  reminder.create  reminder.list  reminder.cancel    │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    调度核心                          │   │
│  │                                                     │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐       │   │
│  │  │ 时间解析   │  │ Job 管理   │  │ 重试策略   │       │   │
│  │  └───────────┘  └───────────┘  └───────────┘       │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    BullMQ 队列                       │   │
│  │                                                     │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐       │   │
│  │  │ 提醒队列   │  │ 重试队列   │  │ 死信队列   │       │   │
│  │  │ reminders │  │ retries   │  │ dead      │       │   │
│  │  └───────────┘  └───────────┘  └───────────┘       │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Worker 层                         │   │
│  │                                                     │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐       │   │
│  │  │ 触发 Worker│  │ 确认 Worker│  │ 清理 Worker│       │   │
│  │  └───────────┘  └───────────┘  └───────────┘       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 提醒类型

### 两种提醒模式

| 类型         | 说明                 | 适用场景       |
| ------------ | -------------------- | -------------- |
| **普通提醒** | 播放后自动结束       | 一般提醒、通知 |
| **确认提醒** | 需用户确认，超时重试 | 吃药、重要事项 |

### 数据模型

```python
@dataclass
class Reminder:
    """提醒数据模型"""
    
    id: str                      # 唯一标识
    user_id: str                 # 用户 ID
    device_id: str               # 设备 ID
    
    # 内容
    content: str                 # 提醒内容
    context: dict                # 上下文信息
    
    # 时间
    trigger_at: datetime         # 触发时间
    created_at: datetime         # 创建时间
    
    # 确认配置
    confirmation: ConfirmationConfig
    
    # 状态
    status: ReminderStatus       # pending/triggered/confirmed/cancelled
    attempt: int = 0             # 当前尝试次数


@dataclass
class ConfirmationConfig:
    """确认配置"""
    
    required: bool = False       # 是否需要确认
    timeout: int = 300           # 会话超时 (秒)
    retry_interval: int = 60     # 重试间隔 (秒)
    max_retries: int = 3         # 最大重试次数
    confirm_keywords: list = None  # 确认关键词


class ReminderStatus(Enum):
    PENDING = "pending"          # 待触发
    TRIGGERED = "triggered"      # 已触发
    CONFIRMED = "confirmed"      # 已确认
    COMPLETED = "completed"      # 已完成 (普通提醒)
    TIMEOUT = "timeout"          # 超时未确认
    CANCELLED = "cancelled"      # 已取消
```

---

## 时间解析

### 自然语言时间解析

```python
class TimeParser:
    """自然语言时间解析器"""
    
    PATTERNS = {
        # 相对时间
        r"(\d+)(分钟|小时|天)后": "relative",
        r"明天(上午|下午|晚上)?(\d+)点": "tomorrow",
        r"后天(\d+)点": "day_after",
        r"(周|星期)(一|二|三|四|五|六|日)(\d+)点": "weekday",
        
        # 绝对时间
        r"(\d+)月(\d+)日(\d+)点": "absolute",
        r"(\d+):(\d+)": "time_only",
        
        # 周期性
        r"每天(\d+)点": "daily",
        r"每(周|星期)(一|二|三|四|五|六|日)": "weekly",
        r"每月(\d+)日": "monthly",
    }
    
    async def parse(self, text: str, base_time: datetime = None) -> datetime:
        """
        解析自然语言时间
        
        示例：
        - "5分钟后" -> now + 5min
        - "明天上午9点" -> tomorrow 09:00
        - "下周一下午3点" -> next Monday 15:00
        """
        base_time = base_time or datetime.now()
        
        # 正则匹配
        for pattern, time_type in self.PATTERNS.items():
            match = re.search(pattern, text)
            if match:
                return self.calculate_time(match, time_type, base_time)
        
        # 使用 LLM 理解复杂时间表达
        return await self.llm_parse(text, base_time)
    
    async def llm_parse(self, text: str, base_time: datetime) -> datetime:
        """LLM 解析复杂时间"""
        prompt = f"""
当前时间: {base_time.strftime('%Y-%m-%d %H:%M')}

解析以下时间表达:
"{text}"

输出 ISO 格式时间: YYYY-MM-DDTHH:MM:SS
"""
        result = await self.llm.generate(prompt)
        return datetime.fromisoformat(result.strip())
```

### 周期性提醒

```python
class RecurringReminder:
    """周期性提醒"""
    
    def __init__(
        self,
        pattern: str,           # cron 表达式
        content: str,
        user_id: str,
        device_id: str,
        end_date: datetime = None
    ):
        self.pattern = pattern
        self.content = content
        self.user_id = user_id
        self.device_id = device_id
        self.end_date = end_date
    
    def get_next_trigger(self, after: datetime = None) -> datetime:
        """获取下次触发时间"""
        after = after or datetime.now()
        cron = croniter(self.pattern, after)
        next_time = cron.get_next(datetime)
        
        if self.end_date and next_time > self.end_date:
            return None
        
        return next_time


# Cron 表达式示例
CRON_EXAMPLES = {
    "每天9点": "0 9 * * *",
    "每周一9点": "0 9 * * 1",
    "每月1日9点": "0 9 1 * *",
    "工作日9点": "0 9 * * 1-5",
    "每小时": "0 * * * *",
}
```

---

## 触发流程

### 普通提醒

```
Job 触发
    │
    ▼
┌─────────────┐
│ 查找设备    │  设备不在线 → 延迟重试
└─────────────┘
    │ 在线
    ▼
┌─────────────┐
│ 唤醒设备    │  Gateway → ESP32: wakeup
└─────────────┘
    │
    ▼
┌─────────────┐
│ 建立会话    │  ESP32 → Server: WebSocket
└─────────────┘
    │
    ▼
┌─────────────┐
│ LLM 生成    │  生成自然的提醒话术
│ 提醒话术    │
└─────────────┘
    │
    ▼
┌─────────────┐
│ TTS 播放    │  Server → ESP32: 语音
└─────────────┘
    │
    ▼
┌─────────────┐
│ 等待响应    │  30秒超时
└─────────────┘
    │
    ├── 用户响应 → 进入正常对话
    │
    └── 超时 → 标记完成，关闭会话
```

### 确认提醒 (含重试)

```
Job 触发 (attempt: 1)
    │
    ▼
┌─────────────┐
│ 唤醒 + 播放 │
└─────────────┘
    │
    ▼
┌─────────────┐
│ 等待确认    │  5分钟会话超时
└─────────────┘
    │
    ├── 用户确认 ("好的/知道了") 
    │       │
    │       ▼
    │   ┌─────────────┐
    │   │ 标记确认    │
    │   │ 取消重试    │
    │   └─────────────┘
    │       │
    │       ▼
    │   进入日常聊天
    │
    └── 会话超时
            │
            ▼
    ┌─────────────┐
    │ attempt < 3 │
    └─────────────┘
            │
            ├── 是 → 安排重试 (1分钟后)
            │       │
            │       ▼
            │   Job 触发 (attempt: 2)
            │       │
            │       ... 重复流程 ...
            │
            └── 否 → 标记未确认
                    │
                    ▼
                ┌─────────────┐
                │ 升级处理    │  可选: 通知紧急联系人
                └─────────────┘
```

---

## 确认检测

### 确认意图识别

```python
class ConfirmationDetector:
    """确认意图检测器"""
    
    # 确认关键词
    CONFIRM_KEYWORDS = [
        "好的", "知道了", "好", "嗯", "OK", "收到",
        "我知道了", "明白", "了解", "可以",
        "吃过了", "吃了", "做了", "完成了"
    ]
    
    # 否认关键词
    DENY_KEYWORDS = [
        "等一下", "稍后", "待会", "一会儿",
        "还没", "没有", "不行"
    ]
    
    async def detect(
        self,
        user_response: str,
        reminder_context: dict
    ) -> ConfirmResult:
        """
        检测用户响应是否为确认
        
        Returns:
            ConfirmResult: confirmed/denied/unclear
        """
        # 1. 关键词匹配
        if any(kw in user_response for kw in self.CONFIRM_KEYWORDS):
            return ConfirmResult.CONFIRMED
        
        if any(kw in user_response for kw in self.DENY_KEYWORDS):
            return ConfirmResult.DENIED
        
        # 2. LLM 意图识别 (复杂情况)
        prompt = f"""
提醒内容: {reminder_context['content']}
用户回复: {user_response}

判断用户是否确认了提醒:
- CONFIRMED: 用户确认已完成或知晓
- DENIED: 用户明确拒绝或推迟
- UNCLEAR: 无法判断

只输出一个词: CONFIRMED / DENIED / UNCLEAR
"""
        result = await self.llm.generate(prompt)
        return ConfirmResult(result.strip())
```

### 对话式确认

```python
class ConversationalConfirmation:
    """对话式确认"""
    
    async def handle_unclear_response(
        self,
        reminder: Reminder,
        user_response: str
    ) -> str:
        """处理不明确的响应"""
        
        prompt = f"""
你是一个贴心的AI助手，正在提醒用户: {reminder.content}

用户回复: {user_response}

用户的回复不太明确。请生成一个追问，确认用户是否已完成提醒的事项。
要求：
1. 语气亲切自然
2. 直接询问是否完成
3. 不超过20字
"""
        return await self.llm.generate(prompt)
```

---

## 话术生成

### 自然提醒话术

```python
class ReminderSpeechGenerator:
    """提醒话术生成器"""
    
    async def generate(
        self,
        reminder: Reminder,
        user_context: dict,      # 用户记忆上下文
        attempt: int = 1
    ) -> str:
        """生成自然的提醒话术"""
        
        # 根据尝试次数调整语气
        if attempt == 1:
            tone = "温和提醒"
        elif attempt == 2:
            tone = "友好催促"
        else:
            tone = "认真强调"
        
        prompt = f"""
生成一段提醒话术。

提醒内容: {reminder.content}
用户信息: {user_context}
尝试次数: 第{attempt}次
语气要求: {tone}

要求：
1. 像朋友一样自然
2. 结合用户习惯
3. 需要确认的提醒要让用户回复
4. 不超过50字

示例：
- "主人~该吃药啦，吃完告诉我一声哦~"
- "午饭时间到了！今天想吃什么呀？"
"""
        return await self.llm.generate(prompt)
```

---

## MCP 工具接口

```python
class SchedulerMCPService:
    """Scheduler MCP 服务"""
    
    tools = {
        # 创建
        "reminder.create": "创建提醒 (指定时间)",
        "reminder.create_natural": "自然语言创建提醒",
        "reminder.create_recurring": "创建周期性提醒",
        
        # 查询
        "reminder.list": "列出用户提醒",
        "reminder.get": "获取提醒详情",
        
        # 管理
        "reminder.update": "更新提醒",
        "reminder.cancel": "取消提醒",
        "reminder.snooze": "延迟提醒",
        
        # 确认
        "reminder.confirm": "确认提醒",
    }
```

### 工具示例

```python
# 自然语言创建
await reminder.create_natural(
    user_id="user123",
    device_id="device456",
    text="明天早上8点提醒我吃药，要确认",
)
# 解析结果:
# - trigger_at: 明天 08:00
# - content: "吃药"
# - confirmation.required: True

# 创建周期性提醒
await reminder.create_recurring(
    user_id="user123",
    device_id="device456",
    content="吃降压药",
    pattern="0 8 * * *",  # 每天8点
    confirmation={
        "required": True,
        "retry_interval": 60,
        "max_retries": 3
    }
)

# 列出提醒
reminders = await reminder.list(
    user_id="user123",
    status=["pending", "triggered"]
)

# 延迟提醒
await reminder.snooze(
    reminder_id="rem_xxx",
    duration=300  # 5分钟后
)
```

---

## BullMQ 队列设计

### 队列配置

```python
# 提醒队列
reminders_queue = Queue(
    "reminders",
    connection=redis,
    defaultJobOptions={
        "attempts": 1,              # Worker 内部处理重试
        "backoff": {"type": "fixed", "delay": 60000},
        "removeOnComplete": 100,    # 保留最近 100 个完成的
        "removeOnFail": 1000        # 保留最近 1000 个失败的
    }
)

# 重试队列 (确认提醒专用)
retries_queue = Queue(
    "reminders:retries",
    connection=redis
)

# 死信队列
dead_queue = Queue(
    "reminders:dead",
    connection=redis
)
```

### Worker 实现

```python
class ReminderWorker:
    """提醒 Worker"""
    
    async def process(self, job: Job) -> dict:
        """处理提醒任务"""
        reminder = Reminder.from_dict(job.data)
        
        try:
            # 1. 触发提醒
            result = await self.trigger_reminder(reminder)
            
            # 2. 根据结果处理
            if result.status == "confirmed":
                await self.mark_confirmed(reminder)
                return {"status": "confirmed"}
            
            if result.status == "timeout":
                if reminder.confirmation.required:
                    if reminder.attempt < reminder.confirmation.max_retries:
                        # 安排重试
                        await self.schedule_retry(reminder)
                        return {"status": "retry_scheduled"}
                    else:
                        # 达到最大重试
                        await self.mark_unconfirmed(reminder)
                        await self.escalate(reminder)  # 升级处理
                        return {"status": "unconfirmed"}
                else:
                    # 普通提醒完成
                    await self.mark_completed(reminder)
                    return {"status": "completed"}
            
            return {"status": result.status}
            
        except Exception as e:
            # 移入死信队列
            await self.move_to_dead(reminder, str(e))
            raise
```

---

## 存储设计

### Redis 数据结构

```python
# 提醒详情
reminder:{reminder_id} = {
    "id": "rem_xxx",
    "user_id": "user123",
    "device_id": "device456",
    "content": "吃药",
    "trigger_at": "2024-12-05T08:00:00",
    "confirmation": {...},
    "status": "pending",
    "attempt": 0,
    "created_at": "..."
}

# 用户提醒索引
user_reminders:{user_id} = SortedSet[
    (trigger_timestamp, reminder_id)
]

# 周期性提醒
recurring:{recurring_id} = {
    "pattern": "0 8 * * *",
    "content": "...",
    "next_trigger": "...",
    "enabled": True
}

# 设备活跃状态
device_status:{device_id} = {
    "online": True,
    "last_seen": "..."
}
```

---

## 可靠性保障

### 幂等性

```python
class IdempotentTrigger:
    """幂等触发"""
    
    async def trigger(self, reminder_id: str) -> bool:
        """
        确保同一提醒不会重复触发
        """
        lock_key = f"trigger_lock:{reminder_id}"
        
        # 尝试获取锁
        acquired = await self.redis.set(
            lock_key, "1",
            nx=True,  # 不存在才设置
            ex=300    # 5分钟过期
        )
        
        if not acquired:
            logger.info(f"提醒 {reminder_id} 已在处理中")
            return False
        
        return True
```

### 设备离线处理

```python
class OfflineHandler:
    """设备离线处理"""
    
    async def handle_offline_device(
        self,
        reminder: Reminder
    ):
        """设备离线时的处理策略"""
        
        # 检查设备状态
        device = await self.get_device(reminder.device_id)
        
        if not device.online:
            # 延迟重试 (1分钟后)
            await self.reschedule(reminder, delay=60)
            
            # 如果持续离线超过30分钟，标记失败
            if device.offline_duration > 1800:
                await self.mark_failed(
                    reminder,
                    reason="device_offline_too_long"
                )
```

---

## 监控指标

| 指标                   | 说明         | 告警阈值   |
| ---------------------- | ------------ | ---------- |
| `reminder_pending`     | 待触发提醒数 | > 10000    |
| `trigger_success_rate` | 触发成功率   | < 95%      |
| `confirm_rate`         | 确认率       | < 80%      |
| `retry_count`          | 重试次数     | > 3次/提醒 |
| `trigger_latency`      | 触发延迟     | > 10s      |
| `dead_queue_size`      | 死信队列大小 | > 100      |
