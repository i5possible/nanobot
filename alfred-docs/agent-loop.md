# Agent Loop（代理循环）

## 概述

Agent Loop 是 nanobot 的核心处理引擎，负责协调 LLM 调用、工具执行和消息处理。它实现了经典的 ReAct（Reasoning + Acting）模式，通过迭代调用 LLM 和执行工具来完成复杂任务。

## 核心职责

1. **消息处理**：从消息总线接收用户消息
2. **上下文构建**：组装系统提示、历史记录、记忆和技能
3. **LLM 调用**：与语言模型交互获取响应
4. **工具执行**：执行 LLM 请求的工具调用
5. **响应发送**：将最终响应发送回消息总线

## 架构设计

### 主要组件

- **MessageBus**：消息队列，解耦渠道和代理
- **LLMProvider**：LLM 提供者接口
- **ContextBuilder**：构建系统提示和消息列表
- **SessionManager**：管理对话会话
- **ToolRegistry**：工具注册表
- **SubagentManager**：子代理管理器

### 工作流程

```
用户消息 → MessageBus (inbound) 
    ↓
AgentLoop._process_message()
    ↓
构建上下文（系统提示 + 历史 + 记忆 + 技能）
    ↓
[循环开始]
    ↓
调用 LLM（带工具定义）
    ↓
LLM 返回：文本响应 或 工具调用
    ↓
如果有工具调用：
    - 执行工具
    - 将结果添加到消息列表
    - 继续循环
否则：
    - 保存会话
    - 发送响应到 MessageBus (outbound)
    ↓
[循环结束]
```

## 关键方法

### `run()`

启动代理循环，持续从消息总线消费消息并处理。

```python
async def run(self) -> None:
    """运行代理循环，处理来自总线的消息"""
    while self._running:
        msg = await self.bus.consume_inbound()
        response = await self._process_message(msg)
        if response:
            await self.bus.publish_outbound(response)
```

### `_process_message()`

处理单个入站消息的核心方法。

**流程：**
1. 处理系统消息（子代理通知）
2. 获取或创建会话
3. 更新工具上下文（消息工具、生成工具）
4. 构建消息列表
5. 执行代理循环（最多 `max_iterations` 次）
6. 保存会话并返回响应

### `process_direct()`

直接处理消息（用于 CLI 模式），不通过消息总线。

## 工具注册

Agent Loop 在初始化时注册默认工具集：

- **文件工具**：`read_file`, `write_file`, `edit_file`, `list_dir`
- **Shell 工具**：`exec`（执行命令）
- **Web 工具**：`web_search`, `web_fetch`
- **消息工具**：`message`（发送消息到渠道）
- **生成工具**：`spawn`（创建子代理）

## 迭代限制

为防止无限循环，设置了 `max_iterations`（默认 20）限制。如果达到限制，循环会停止并返回当前结果。

## 错误处理

- 捕获处理过程中的异常
- 向用户发送错误响应
- 记录错误日志

## 系统消息处理

系统消息（`channel="system"`）用于子代理通知主代理。`chat_id` 字段包含原始渠道和聊天 ID（格式：`"channel:chat_id"`），用于路由响应。

## 配置参数

- `max_iterations`：最大迭代次数（默认 20）
- `model`：使用的 LLM 模型
- `workspace`：工作空间路径
- `brave_api_key`：Brave Search API 密钥（用于网络搜索）

## 使用示例

```python
from nanobot.agent.loop import AgentLoop
from nanobot.bus.queue import MessageBus
from nanobot.providers.litellm_provider import LiteLLMProvider

bus = MessageBus()
provider = LiteLLMProvider(api_key="...", default_model="anthropic/claude-opus-4-5")
agent = AgentLoop(
    bus=bus,
    provider=provider,
    workspace=Path("~/.nanobot/workspace"),
    max_iterations=20
)

# 启动循环
await agent.run()
```

## 相关文件

- `nanobot/agent/loop.py`：主实现文件
- `nanobot/agent/context.py`：上下文构建器
- `nanobot/agent/tools/`：工具实现
- `nanobot/bus/queue.py`：消息总线
