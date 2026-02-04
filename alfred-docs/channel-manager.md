# Channel Manager（渠道管理器）

## 概述

Channel Manager 负责管理和协调所有聊天渠道（Telegram、WhatsApp 等）。它初始化启用的渠道，启动/停止渠道服务，并路由出站消息到相应的渠道。

## 核心职责

1. **渠道初始化**：根据配置初始化启用的渠道
2. **渠道启动/停止**：管理渠道的生命周期
3. **消息路由**：将出站消息路由到正确的渠道
4. **状态管理**：跟踪渠道运行状态

## 架构设计

```
ChannelManager
    ├── TelegramChannel
    ├── WhatsAppChannel
    └── (其他渠道...)
         │
         └──> MessageBus
```

## 核心类

### `ChannelManager`

渠道管理器主类。

**初始化：**
```python
from nanobot.channels.manager import ChannelManager
from nanobot.config.schema import Config
from nanobot.bus.queue import MessageBus

config = load_config()
bus = MessageBus()
manager = ChannelManager(config, bus)
```

**关键方法：**

#### `start_all()`

启动所有启用的渠道和出站分发器。

```python
await manager.start_all()
```

**流程：**
1. 启动出站消息分发器
2. 启动所有启用的渠道
3. 等待所有渠道运行（它们应该一直运行）

#### `stop_all()`

停止所有渠道和分发器。

```python
await manager.stop_all()
```

#### `get_channel()`

获取指定名称的渠道。

```python
telegram = manager.get_channel("telegram")
```

#### `get_status()`

获取所有渠道的状态。

```python
status = manager.get_status()
# 返回: {
#     "telegram": {"enabled": True, "running": True},
#     "whatsapp": {"enabled": True, "running": False}
# }
```

## 渠道初始化

### Telegram 渠道

如果配置中启用了 Telegram：

```python
if config.channels.telegram.enabled:
    from nanobot.channels.telegram import TelegramChannel
    self.channels["telegram"] = TelegramChannel(
        config.channels.telegram,
        bus,
        groq_api_key=config.providers.groq.api_key  # 用于语音转录
    )
```

### WhatsApp 渠道

如果配置中启用了 WhatsApp：

```python
if config.channels.whatsapp.enabled:
    from nanobot.channels.whatsapp import WhatsAppChannel
    self.channels["whatsapp"] = WhatsAppChannel(
        config.channels.whatsapp,
        bus
    )
```

## 消息路由

### 出站消息分发

Channel Manager 运行一个后台任务来分发出站消息：

```python
async def _dispatch_outbound(self):
    """分发出站消息到相应渠道"""
    while True:
        msg = await self.bus.consume_outbound()
        channel = self.channels.get(msg.channel)
        if channel:
            await channel.send(msg)
```

**流程：**
1. 从消息总线消费出站消息
2. 根据 `msg.channel` 查找对应渠道
3. 调用渠道的 `send()` 方法发送消息

## 渠道接口

所有渠道必须实现 `BaseChannel` 接口：

### `BaseChannel`

抽象基类，定义渠道接口。

**必需方法：**

- `start()`：启动渠道，开始监听消息
- `stop()`：停止渠道
- `send(msg: OutboundMessage)`：发送消息
- `is_allowed(sender_id: str)`：检查发送者是否被允许

**消息处理：**

渠道应该使用 `_handle_message()` 方法处理入站消息：

```python
await self._handle_message(
    sender_id="123",
    chat_id="456",
    content="Hello",
    media=None,
    metadata={}
)
```

这个方法会：
1. 检查权限（`is_allowed()`）
2. 创建 `InboundMessage`
3. 发布到消息总线

## 配置示例

### Telegram 配置

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "123456:ABC...",
      "allowFrom": ["123456789"]
    }
  }
}
```

### WhatsApp 配置

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "bridge_url": "ws://localhost:3001",
      "allowFrom": ["+1234567890"]
    }
  }
}
```

## 使用示例

### 启动网关

```python
from nanobot.channels.manager import ChannelManager
from nanobot.config.loader import load_config
from nanobot.bus.queue import MessageBus

config = load_config()
bus = MessageBus()
manager = ChannelManager(config, bus)

# 启动所有渠道
await manager.start_all()
```

### 检查渠道状态

```python
status = manager.get_status()
for name, info in status.items():
    print(f"{name}: {'running' if info['running'] else 'stopped'}")
```

### 获取启用的渠道

```python
enabled = manager.enabled_channels
# 返回: ["telegram", "whatsapp"]
```

## 错误处理

- 渠道初始化失败会被记录为警告，不会阻止其他渠道启动
- 发送消息时的异常会被捕获和记录
- 未知渠道的消息会被记录为警告

## 扩展新渠道

要添加新渠道：

1. **创建渠道类**：继承 `BaseChannel`
2. **实现必需方法**：`start()`, `stop()`, `send()`
3. **在 ChannelManager 中注册**：在 `_init_channels()` 中添加初始化逻辑
4. **添加配置**：在 `config/schema.py` 中添加配置类

## 相关文件

- `nanobot/channels/manager.py`：主实现文件
- `nanobot/channels/base.py`：基础渠道接口
- `nanobot/channels/telegram.py`：Telegram 实现
- `nanobot/channels/whatsapp.py`：WhatsApp 实现
