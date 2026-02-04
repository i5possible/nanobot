# Subagent Manager（子代理管理器）

## 概述

Subagent Manager 管理后台子代理的执行。子代理是轻量级的代理实例，用于处理复杂或耗时的任务，在后台异步运行，完成后通知主代理。

## 核心概念

### 子代理 vs 主代理

- **主代理**：处理用户交互，维护会话历史
- **子代理**：执行独立任务，有独立的上下文，完成后通知主代理

### 使用场景

- 长时间运行的任务（如文件处理、数据分析）
- 独立的研究任务（如网络搜索、信息收集）
- 可以并行执行的任务

## 核心类

### `SubagentManager`

子代理管理器主类。

**初始化：**
```python
from nanobot.agent.subagent import SubagentManager

manager = SubagentManager(
    provider=llm_provider,
    workspace=workspace_path,
    bus=message_bus,
    model="anthropic/claude-opus-4-5",
    brave_api_key="..."
)
```

## 关键方法

### `spawn()`

生成一个新的子代理来执行任务。

**参数：**
- `task` (string)：任务描述
- `label` (string, 可选)：任务标签（用于显示）
- `origin_channel` (string)：原始渠道（默认 "cli"）
- `origin_chat_id` (string)：原始聊天 ID（默认 "direct"）

**返回：**
状态消息，表示子代理已启动。

**示例：**
```python
result = await manager.spawn(
    task="分析这个项目的代码结构并生成报告",
    label="代码分析",
    origin_channel="telegram",
    origin_chat_id="123456789"
)
# 返回: "Subagent [代码分析] started (id: abc12345)..."
```

## 工作流程

### 1. 生成子代理

```
主代理调用 spawn 工具
    ↓
SubagentManager.spawn()
    ↓
创建后台任务
    ↓
返回启动消息
```

### 2. 子代理执行

```
子代理启动
    ↓
构建子代理系统提示
    ↓
创建工具集（无 message 和 spawn）
    ↓
执行代理循环（最多 15 次迭代）
    ↓
完成任务
    ↓
通知主代理
```

### 3. 结果通知

```
子代理完成任务
    ↓
构建通知消息
    ↓
通过系统消息发送到主代理
    ↓
主代理处理通知
    ↓
向用户报告结果
```

## 子代理系统提示

子代理使用专门设计的系统提示：

```
# Subagent

You are a subagent spawned by the main agent to complete a specific task.

## Your Task
{task}

## Rules
1. Stay focused - complete only the assigned task
2. Your final response will be reported back to the main agent
3. Do not initiate conversations or take on side tasks
4. Be concise but informative in your findings

## What You Can Do
- Read and write files in the workspace
- Execute shell commands
- Search the web and fetch web pages
- Complete the task thoroughly

## What You Cannot Do
- Send messages directly to users (no message tool)
- Spawn other subagents
- Access the main agent's conversation history
```

## 子代理工具集

子代理可以使用以下工具：

- ✅ `read_file`：读取文件
- ✅ `write_file`：写入文件
- ✅ `list_dir`：列出目录
- ✅ `exec`：执行命令
- ✅ `web_search`：网络搜索
- ✅ `web_fetch`：获取网页

**不可用：**
- ❌ `message`：发送消息（子代理不能直接与用户通信）
- ❌ `spawn`：生成子代理（防止递归）

## 结果通知格式

子代理完成后的通知消息格式：

```
[Subagent '{label}' completed successfully]

Task: {task}

Result:
{result}

Summarize this naturally for the user. Keep it brief (1-2 sentences). 
Do not mention technical details like "subagent" or task IDs.
```

主代理会收到这个系统消息，然后自然地总结给用户。

## 错误处理

如果子代理执行失败：

1. 捕获异常
2. 构建错误通知
3. 发送到主代理
4. 主代理向用户报告错误

**错误通知格式：**
```
[Subagent '{label}' failed]

Task: {task}

Result:
Error: {error_message}
```

## 任务管理

### 运行中的任务

`SubagentManager` 跟踪所有运行中的任务：

```python
count = manager.get_running_count()  # 返回运行中的子代理数量
```

### 任务清理

任务完成后自动从跟踪列表中移除（通过 `add_done_callback`）。

## 迭代限制

子代理有独立的迭代限制（默认 15 次），比主代理（20 次）更少，因为子代理应该专注于单一任务。

## 使用示例

### 在主代理中使用

```python
# 用户请求复杂任务
user_message = "分析这个项目的所有 Python 文件，找出潜在的 bug"

# 主代理决定使用 spawn 工具
tool_call = {
    "name": "spawn",
    "arguments": {
        "task": "分析项目中的所有 Python 文件，找出潜在的 bug 和安全问题",
        "label": "代码审查"
    }
}

# 执行后，子代理在后台运行
# 主代理立即回复："代码审查任务已启动，完成后会通知您"
```

### 子代理通知处理

```python
# 子代理完成后，发送系统消息
system_msg = InboundMessage(
    channel="system",
    sender_id="subagent",
    chat_id="telegram:123456789",  # 原始渠道和聊天 ID
    content="[Subagent '代码审查' completed successfully]..."
)

# 主代理处理系统消息
# 自然地总结结果给用户
```

## 设计优势

1. **非阻塞**：主代理可以继续处理其他请求
2. **隔离**：子代理有独立的上下文，不会干扰主会话
3. **可扩展**：可以同时运行多个子代理
4. **用户友好**：主代理会自然地总结结果

## 注意事项

1. **资源使用**：每个子代理都会消耗 LLM API 调用
2. **任务描述**：清晰的任务描述有助于子代理成功执行
3. **超时**：长时间运行的任务可能超时
4. **错误恢复**：子代理失败不会影响主代理

## 相关文件

- `nanobot/agent/subagent.py`：主实现文件
- `nanobot/agent/tools/spawn.py`：生成工具
- `nanobot/agent/loop.py`：主代理循环（处理系统消息）
