# Context Builder（上下文构建器）

## 概述

Context Builder 负责组装代理的系统提示和消息列表，将引导文件、记忆、技能和对话历史整合成连贯的提示，供 LLM 使用。

## 核心职责

1. **系统提示构建**：组装完整的系统提示
2. **消息列表构建**：构建包含系统提示、历史和当前消息的完整消息列表
3. **引导文件加载**：从工作空间加载引导文件（AGENTS.md, SOUL.md, USER.md 等）
4. **记忆集成**：将长期记忆和今日笔记集成到上下文中
5. **技能管理**：加载和格式化技能内容

## 引导文件

默认加载的引导文件（按顺序）：

- `AGENTS.md`：代理指令和指南
- `SOUL.md`：代理个性和价值观
- `USER.md`：用户信息和偏好
- `TOOLS.md`：工具使用说明
- `IDENTITY.md`：身份定义（如果存在）

这些文件位于工作空间根目录，用于定义代理的行为和身份。

## 系统提示结构

系统提示由以下部分组成（用 `---` 分隔）：

1. **核心身份**：代理的基本信息和能力
2. **引导文件**：从引导文件加载的内容
3. **记忆上下文**：长期记忆和今日笔记
4. **活跃技能**：标记为 `always=true` 的技能
5. **可用技能摘要**：所有技能的 XML 摘要

### 核心身份部分

包含：
- 代理名称和描述
- 当前时间和日期
- 工作空间路径
- 可用工具列表
- 重要行为准则

### 技能加载策略

- **总是加载的技能**：标记为 `always=true` 的技能会完整加载到系统提示中
- **按需加载的技能**：其他技能只显示摘要，代理可以通过 `read_file` 工具按需加载

## 关键方法

### `build_system_prompt()`

构建完整的系统提示。

```python
def build_system_prompt(self, skill_names: list[str] | None = None) -> str:
    """构建系统提示"""
    parts = []
    parts.append(self._get_identity())
    parts.append(self._load_bootstrap_files())
    parts.append(self.memory.get_memory_context())
    # ... 技能加载
    return "\n\n---\n\n".join(parts)
```

### `build_messages()`

构建完整的消息列表，包括系统提示、历史记录和当前消息。

**参数：**
- `history`：对话历史（LLM 格式）
- `current_message`：当前用户消息
- `skill_names`：可选技能列表
- `media`：可选媒体文件路径列表

**返回：**
包含系统提示、历史和当前消息的完整消息列表。

### `add_tool_result()`

将工具执行结果添加到消息列表。

### `add_assistant_message()`

将助手消息（可能包含工具调用）添加到消息列表。

## 媒体支持

支持在用户消息中附加图片：

- 自动检测图片文件
- Base64 编码图片
- 使用 `data:image/...;base64,...` 格式
- 支持多图片附件

## 记忆集成

Context Builder 通过 `MemoryStore` 获取记忆上下文：

- **长期记忆**：`memory/MEMORY.md`
- **今日笔记**：`memory/YYYY-MM-DD.md`

这些内容会自动包含在系统提示中。

## 技能系统集成

通过 `SkillsLoader` 管理技能：

- 列出所有可用技能
- 检查技能可用性（依赖检查）
- 加载技能内容
- 构建技能摘要（XML 格式）

## 使用示例

```python
from nanobot.agent.context import ContextBuilder
from pathlib import Path

workspace = Path("~/.nanobot/workspace")
context = ContextBuilder(workspace)

# 构建系统提示
system_prompt = context.build_system_prompt()

# 构建消息列表
messages = context.build_messages(
    history=session.get_history(),
    current_message="Hello!",
    media=["/path/to/image.png"]
)
```

## 配置

引导文件位置：`workspace/` 目录

记忆文件位置：
- 长期记忆：`workspace/memory/MEMORY.md`
- 每日笔记：`workspace/memory/YYYY-MM-DD.md`

技能位置：
- 工作空间技能：`workspace/skills/{skill-name}/SKILL.md`
- 内置技能：`nanobot/skills/{skill-name}/SKILL.md`

## 相关文件

- `nanobot/agent/context.py`：主实现文件
- `nanobot/agent/memory.py`：记忆系统
- `nanobot/agent/skills.py`：技能加载器
