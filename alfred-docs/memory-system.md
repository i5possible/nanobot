# Memory System（记忆系统）

## 概述

Memory System 提供持久化的记忆存储，支持两种类型的记忆：每日笔记和长期记忆。这允许代理在不同会话之间保持上下文和重要信息。

## 核心功能

1. **每日笔记**：按日期组织的临时记忆（`memory/YYYY-MM-DD.md`）
2. **长期记忆**：持久化的关键信息（`memory/MEMORY.md`）
3. **记忆检索**：获取最近 N 天的记忆
4. **记忆上下文**：为代理提供格式化的记忆上下文

## 文件结构

```
workspace/
└── memory/
    ├── MEMORY.md          # 长期记忆
    ├── 2026-01-15.md      # 2026年1月15日的笔记
    ├── 2026-01-16.md      # 2026年1月16日的笔记
    └── ...
```

## 核心类

### `MemoryStore`

记忆存储的主要类。

**初始化：**
```python
memory = MemoryStore(workspace=Path("~/.nanobot/workspace"))
```

**关键方法：**

#### `read_today()`

读取今天的记忆笔记。

```python
content = memory.read_today()
```

#### `append_today()`

追加内容到今天的笔记。如果文件不存在，会自动创建并添加日期标题。

```python
memory.append_today("用户喜欢喝咖啡")
```

#### `read_long_term()`

读取长期记忆文件。

```python
content = memory.read_long_term()
```

#### `write_long_term()`

写入长期记忆文件（覆盖）。

```python
memory.write_long_term("# 长期记忆\n\n重要信息...")
```

#### `get_recent_memories()`

获取最近 N 天的记忆。

```python
recent = memory.get_recent_memories(days=7)  # 最近7天
```

#### `get_memory_context()`

获取格式化的记忆上下文，用于代理系统提示。

**返回格式：**
```
## Long-term Memory

[长期记忆内容]

## Today's Notes

[今日笔记内容]
```

## 使用场景

### 1. 记录用户偏好

```python
memory.append_today("用户偏好：喜欢简洁的回答，不喜欢冗长的解释")
```

### 2. 保存重要信息

```python
memory.write_long_term("""
# 用户信息

- 姓名：张三
- 时区：UTC+8
- 语言：中文
""")
```

### 3. 获取记忆上下文

在代理循环中，记忆上下文会自动包含在系统提示中：

```python
context = ContextBuilder(workspace)
system_prompt = context.build_system_prompt()  # 自动包含记忆
```

## 文件格式

### 每日笔记格式

```markdown
# 2026-01-15

- 用户询问了关于 Python 的问题
- 推荐了使用 type hints
- 用户反馈很有帮助
```

### 长期记忆格式

```markdown
# Long-term Memory

## 用户信息

- 职业：软件工程师
- 主要使用语言：Python, TypeScript

## 偏好

- 喜欢简洁的代码
- 偏好函数式编程风格

## 重要事项

- 每周三有团队会议
```

## 自动集成

记忆系统通过 `ContextBuilder` 自动集成到代理上下文中：

1. 每次构建系统提示时，会自动加载：
   - 长期记忆（`MEMORY.md`）
   - 今日笔记（`YYYY-MM-DD.md`）

2. 代理可以通过工具直接操作记忆文件：
   - `read_file`：读取记忆文件
   - `write_file`：写入记忆文件
   - `edit_file`：编辑记忆文件

## 最佳实践

1. **每日笔记**：用于记录临时信息、对话摘要、任务清单
2. **长期记忆**：用于存储用户信息、偏好、重要事实
3. **定期整理**：可以定期将每日笔记中的重要信息提取到长期记忆
4. **结构化存储**：使用 Markdown 标题组织信息，便于检索

## 相关文件

- `nanobot/agent/memory.py`：主实现文件
- `nanobot/agent/context.py`：上下文构建器（集成记忆）
