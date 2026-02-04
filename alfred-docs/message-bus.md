# Message Bus（消息总线）

## 概述

Message Bus 是一个异步消息队列系统，用于解耦聊天渠道和代理核心。它提供了两个队列：入站队列（inbound）和出站队列（outbound），实现了发布-订阅模式。

## 核心职责

1. **消息路由**：在渠道和代理之间路由消息
2. **异步通信**：提供非阻塞的消息传递
3. **订阅管理**：支持渠道订阅出站消息
4. **消息分发**：将代理响应分发到相应渠道

## 架构设计

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Channel   │ ──────> │ Message Bus  │ <────── │    Agent    │
│ (Telegram)  │ inbound │              │ outbound │    Loop     │
└─────────────┘         └──────────────┘         └─────────────┘
                              │
                              │ dispatch
                              ▼
                        ┌─────────────┐
                        │  Subscribers │
                        │  (Channels)  │
                        └─────────────┘
```

## 核心类

### `MessageBus`

消息总线主类。

**初始化：**
```python
bus = MessageBus()
```

**关键方法：**

#### `publish_inbound()`

发布入站消息（从渠道到代理）。

```python
msg = InboundMessage(
    channel="telegram",
    sender_id="123456",
    chat_id="789",
    content="Hello!"
)
await bus.publish_inbound(msg)
```

#### `consume_inbound()`

消费入站消息（代理使用）。

```python
msg = await bus.consume_inbound()  # 阻塞直到有消息
```

#### `publish_outbound()`

发布出站消息（从代理到渠道）。

```python
response = OutboundMessage(
    channel="telegram",
    chat_id="789",
    content="Hi there!"
)
await bus.publish_outbound(response)
```

#### `consume_outbound()`

消费出站消息（渠道管理器使用）。

```python
msg = await bus.consume_outbound()
```

#### `subscribe_outbound()`

订阅特定渠道的出站消息。

```python
async def handle_message(msg: OutboundMessage):
    await channel.send(msg)

bus.subscribe_outbound("telegram", handle_message)
```

#### `dispatch_outbound()`

分发出站消息到订阅者（后台任务）。

```python
# 在后台运行
asyncio.create_task(bus.dispatch_outbound())
```

## 消息类型

### `InboundMessage`

入站消息（从渠道到代理）。

**字段：**
- `channel`：渠道名称（telegram, whatsapp 等）
- `sender_id`：发送者 ID
- `chat_id`：聊天/频道 ID
- `content`：消息文本
- `timestamp`：时间戳
- `media`：媒体 URL 列表
- `metadata`：渠道特定元数据

**属性：**
- `session_key`：会话键（`channel:chat_id`）

### `OutboundMessage`

出站消息（从代理到渠道）。

**字段：**
- `channel`：目标渠道
- `chat_id`：目标聊天 ID
- `content`：消息内容
- `reply_to`：回复的消息 ID（可选）
- `media`：媒体文件列表（可选）
- `metadata`：渠道特定元数据（可选）

## 工作流程

### 1. 消息接收流程

```
用户发送消息
    ↓
Channel 接收消息
    ↓
创建 InboundMessage
    ↓
bus.publish_inbound(msg)
    ↓
Agent Loop 调用 bus.consume_inbound()
    ↓
处理消息
```

### 2. 消息发送流程

```
Agent Loop 生成响应
    ↓
创建 OutboundMessage
    ↓
bus.publish_outbound(response)
    ↓
Channel Manager 调用 bus.consume_outbound()
    ↓
分发到相应 Channel
    ↓
Channel.send(msg)
```

## 使用示例

### 基本使用

```python
from nanobot.bus.queue import MessageBus
from nanobot.bus.events import InboundMessage, OutboundMessage

bus = MessageBus()

# 发布入站消息
await bus.publish_inbound(InboundMessage(
    channel="cli",
    sender_id="user",
    chat_id="direct",
    content="Hello"
))

# 消费入站消息（在代理循环中）
msg = await bus.consume_inbound()

# 发布出站消息
await bus.publish_outbound(OutboundMessage(
    channel="cli",
    chat_id="direct",
    content="Hi there!"
))
```

### 订阅模式

```python
# 渠道订阅出站消息
async def telegram_handler(msg: OutboundMessage):
    if msg.channel == "telegram":
        await telegram_channel.send(msg)

bus.subscribe_outbound("telegram", telegram_handler)

# 启动分发器
asyncio.create_task(bus.dispatch_outbound())
```

## 队列特性

- **异步队列**：使用 `asyncio.Queue` 实现
- **阻塞消费**：`consume_*` 方法会阻塞直到有消息
- **无界队列**：队列大小无限制（注意内存使用）
- **线程安全**：所有操作都是异步的，线程安全

## 监控和调试

### 队列大小

```python
inbound_size = bus.inbound_size  # 入站队列大小
outbound_size = bus.outbound_size  # 出站队列大小
```

### 停止分发器

```python
bus.stop()  # 停止 dispatch_outbound 循环
```

## 错误处理

- 分发消息时的异常会被捕获和记录
- 不会影响其他订阅者的消息处理
- 错误不会传播到消息发布者

## 设计优势

1. **解耦**：渠道和代理完全解耦
2. **可扩展**：易于添加新渠道
3. **异步**：非阻塞的消息传递
4. **灵活**：支持多种消息路由模式

## 相关文件

- `nanobot/bus/queue.py`：主实现文件
- `nanobot/bus/events.py`：消息类型定义
