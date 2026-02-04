# Session Manager（会话管理器）

## 概述

Session Manager 负责管理对话会话，为每个聊天（渠道+聊天ID）维护独立的对话历史。会话以 JSONL 格式存储在磁盘上，支持持久化和恢复。

## 核心职责

1. **会话创建**：为新的聊天创建会话
2. **会话加载**：从磁盘加载现有会话
3. **历史管理**：维护对话消息历史
4. **会话持久化**：将会话保存到磁盘

## 文件结构

会话文件存储在：`~/.nanobot/sessions/{session_key}.jsonl`

**会话键格式**：`{channel}:{chat_id}`

例如：
- `telegram:123456789.jsonl`
- `whatsapp:+1234567890.jsonl`
- `cli:default.jsonl`

## 文件格式

JSONL（JSON Lines）格式，每行一个 JSON 对象：

```jsonl
{"_type":"metadata","created_at":"2026-01-15T10:00:00","updated_at":"2026-01-15T10:05:00","metadata":{}}
{"role":"user","content":"Hello","timestamp":"2026-01-15T10:00:00"}
{"role":"assistant","content":"Hi there!","timestamp":"2026-01-15T10:00:01"}
{"role":"user","content":"How are you?","timestamp":"2026-01-15T10:00:05"}
{"role":"assistant","content":"I'm doing well!","timestamp":"2026-01-15T10:00:06"}
```

第一行是元数据，后续行是消息。

## 核心类

### `Session`

会话数据类。

**字段：**
- `key`：会话键（`channel:chat_id`）
- `messages`：消息列表
- `created_at`：创建时间
- `updated_at`：更新时间
- `metadata`：元数据字典

**关键方法：**

#### `add_message()`

添加消息到会话。

```python
session.add_message("user", "Hello")
session.add_message("assistant", "Hi there!")
```

#### `get_history()`

获取 LLM 格式的历史记录。

```python
history = session.get_history(max_messages=50)
# 返回: [
#     {"role": "user", "content": "Hello"},
#     {"role": "assistant", "content": "Hi there!"}
# ]
```

#### `clear()`

清空所有消息。

```python
session.clear()
```

### `SessionManager`

会话管理器主类。

**初始化：**
```python
from nanobot.session.manager import SessionManager
from pathlib import Path

workspace = Path("~/.nanobot/workspace")
manager = SessionManager(workspace)
```

**关键方法：**

#### `get_or_create()`

获取或创建会话。

```python
session = manager.get_or_create("telegram:123456789")
```

**流程：**
1. 检查内存缓存
2. 如果不存在，尝试从磁盘加载
3. 如果仍不存在，创建新会话
4. 缓存会话并返回

#### `save()`

保存会话到磁盘。

```python
manager.save(session)
```

**格式：**
- 第一行：元数据（`_type: "metadata"`）
- 后续行：消息（每行一个 JSON 对象）

#### `delete()`

删除会话。

```python
deleted = manager.delete("telegram:123456789")
```

#### `list_sessions()`

列出所有会话。

```python
sessions = manager.list_sessions()
# 返回: [
#     {
#         "key": "telegram:123456789",
#         "created_at": "2026-01-15T10:00:00",
#         "updated_at": "2026-01-15T10:05:00",
#         "path": "/path/to/file.jsonl"
#     },
#     ...
# ]
```

## 使用场景

### 1. 在代理循环中使用

```python
# 获取或创建会话
session = self.sessions.get_or_create(msg.session_key)

# 添加消息
session.add_message("user", msg.content)
session.add_message("assistant", response.content)

# 保存会话
self.sessions.save(session)
```

### 2. 获取历史记录

```python
session = manager.get_or_create("telegram:123456789")
history = session.get_history(max_messages=50)

# 用于构建 LLM 消息列表
messages = context.build_messages(
    history=history,
    current_message="New message"
)
```

### 3. 清理旧会话

```python
# 列出所有会话
sessions = manager.list_sessions()

# 删除特定会话
manager.delete("telegram:123456789")
```

## 会话键格式

会话键用于唯一标识一个对话：

- **格式**：`{channel}:{chat_id}`
- **示例**：
  - `telegram:123456789`
  - `whatsapp:+1234567890`
  - `cli:default`
  - `cli:direct`

## 历史记录限制

`get_history()` 方法支持限制返回的消息数量：

```python
# 只返回最近 50 条消息
history = session.get_history(max_messages=50)
```

这有助于：
- 控制 token 使用
- 保持上下文窗口大小
- 提高响应速度

## 缓存机制

SessionManager 使用内存缓存来避免频繁的磁盘 I/O：

- 已加载的会话会缓存在 `_cache` 字典中
- `get_or_create()` 首先检查缓存
- `save()` 会更新缓存

## 文件安全

会话键中的特殊字符会被转换为安全的文件名：

```python
safe_key = safe_filename(key.replace(":", "_"))
# "telegram:123456789" -> "telegram_123456789"
```

## 错误处理

- 加载会话失败会返回 `None`，然后创建新会话
- 保存失败会记录警告，但不会抛出异常
- 损坏的会话文件会被忽略

## 最佳实践

1. **定期保存**：在处理完消息后立即保存会话
2. **限制历史**：使用 `max_messages` 限制历史记录长度
3. **清理旧会话**：定期清理不再使用的会话
4. **备份重要会话**：对重要对话进行备份

## 相关文件

- `nanobot/session/manager.py`：主实现文件
- `nanobot/utils/helpers.py`：工具函数（`safe_filename`）
