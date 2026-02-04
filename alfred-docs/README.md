# nanobot 技术文档

本文档目录包含 nanobot 项目所有关键实现模块的详细技术文档。

## 文档索引

### 核心架构

- **[Agent Loop](./agent-loop.md)** - 代理循环，核心处理引擎
- **[Context Builder](./context-builder.md)** - 上下文构建器，组装系统提示和消息
- **[Message Bus](./message-bus.md)** - 消息总线，解耦渠道和代理

### 数据管理

- **[Memory System](./memory-system.md)** - 记忆系统，持久化存储
- **[Session Manager](./session-manager.md)** - 会话管理器，对话历史管理

### 能力扩展

- **[Skills System](./skills-system.md)** - 技能系统，扩展代理能力
- **[Tool System](./tool-system.md)** - 工具系统，函数调用接口
- **[Subagent Manager](./subagent-manager.md)** - 子代理管理器，后台任务执行

### 通信与集成

- **[Channel Manager](./channel-manager.md)** - 渠道管理器，管理聊天渠道
- **[LLM Provider](./llm-provider.md)** - LLM 提供者，统一模型接口

### 自动化服务

- **[Cron Service](./cron-service.md)** - 定时任务服务，任务调度
- **[Heartbeat Service](./heartbeat-service.md)** - 心跳服务，周期性检查

### 配置

- **[Configuration](./configuration.md)** - 配置系统，类型安全的配置管理

## 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                    User / Channels                      │
│              (Telegram, WhatsApp, CLI)                   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                  Channel Manager                        │
│              (管理渠道，路由消息)                        │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│                    Message Bus                          │
│          (异步消息队列，解耦通信)                        │
└────────────┬───────────────────────┬───────────────────┘
             │                       │
             ▼                       ▼
    ┌──────────────┐        ┌──────────────┐
    │ Agent Loop   │        │  Subagent    │
    │ (主代理)     │        │  Manager     │
    └──────┬───────┘        └──────────────┘
           │
           ▼
    ┌──────────────┐
    │   Context    │
    │   Builder    │
    └──────┬───────┘
           │
    ┌──────┴───────┐
    │              │
    ▼              ▼
┌─────────┐  ┌──────────┐
│ Memory  │  │  Skills  │
│ System  │  │  System  │
└─────────┘  └──────────┘
```

## 数据流

### 消息处理流程

```
用户消息
    ↓
Channel (Telegram/WhatsApp)
    ↓
MessageBus.publish_inbound()
    ↓
AgentLoop._process_message()
    ↓
ContextBuilder.build_messages()
    ├─ 系统提示（身份 + 引导文件 + 记忆 + 技能）
    ├─ 会话历史
    └─ 当前消息
    ↓
LLMProvider.chat()
    ↓
工具调用（如果有）
    ↓
ToolRegistry.execute()
    ↓
结果添加到消息列表
    ↓
继续 LLM 调用（循环）
    ↓
最终响应
    ↓
SessionManager.save()
    ↓
MessageBus.publish_outbound()
    ↓
ChannelManager 分发
    ↓
用户收到响应
```

### 子代理流程

```
主代理调用 spawn 工具
    ↓
SubagentManager.spawn()
    ↓
创建后台任务
    ↓
子代理执行（独立上下文）
    ↓
完成任务
    ↓
通过系统消息通知主代理
    ↓
主代理总结结果
    ↓
用户收到通知
```

## 关键概念

### 会话键（Session Key）

格式：`{channel}:{chat_id}`

用于唯一标识一个对话会话：
- `telegram:123456789`
- `whatsapp:+1234567890`
- `cli:default`

### 工具调用

代理通过函数调用使用工具：
1. LLM 返回工具调用请求
2. Agent Loop 执行工具
3. 结果添加到消息列表
4. 继续 LLM 调用

### 技能加载策略

- **总是加载**：`always=true` 的技能完整加载到系统提示
- **按需加载**：其他技能只显示摘要，代理按需读取

### 记忆类型

- **每日笔记**：`memory/YYYY-MM-DD.md` - 临时记忆
- **长期记忆**：`memory/MEMORY.md` - 持久化信息

## 扩展指南

### 添加新工具

1. 继承 `Tool` 基类
2. 实现 `name`, `description`, `parameters`, `execute`
3. 在 Agent Loop 中注册

详见：[Tool System](./tool-system.md)

### 添加新渠道

1. 继承 `BaseChannel`
2. 实现 `start()`, `stop()`, `send()`
3. 在 ChannelManager 中注册

详见：[Channel Manager](./channel-manager.md)

### 添加新技能

1. 创建 `workspace/skills/{name}/SKILL.md`
2. 编写技能说明
3. 可选：添加元数据和依赖要求

详见：[Skills System](./skills-system.md)

## 相关资源

- [项目 README](../README.md) - 项目概览和快速开始
- [工作空间文档](../workspace/) - 工作空间结构说明

## 贡献

欢迎贡献文档改进！如果发现文档有误或需要补充，请提交 Issue 或 PR。
