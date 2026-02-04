# Skills System（技能系统）

## 概述

Skills System 允许扩展代理的能力，通过 Markdown 文件（`SKILL.md`）定义新技能。技能可以包含工具使用说明、工作流程、最佳实践等，代理可以按需加载和使用这些技能。

## 核心概念

### 技能定义

技能是一个包含 `SKILL.md` 文件的目录，位于：
- **工作空间技能**：`workspace/skills/{skill-name}/SKILL.md`（优先级高）
- **内置技能**：`nanobot/skills/{skill-name}/SKILL.md`

### 技能元数据

技能可以通过 YAML frontmatter 定义元数据：

```markdown
---
description: 技能描述
always: true  # 是否总是加载
metadata: |
  {
    "nanobot": {
      "always": false,
      "requires": {
        "bins": ["git", "docker"],
        "env": ["GITHUB_TOKEN"]
      }
    }
  }
---

# 技能内容

技能的使用说明...
```

## 核心类

### `SkillsLoader`

技能加载器，负责发现、加载和管理技能。

**初始化：**
```python
loader = SkillsLoader(workspace=Path("~/.nanobot/workspace"))
```

**关键方法：**

#### `list_skills()`

列出所有可用技能。

```python
skills = loader.list_skills(filter_unavailable=True)
# 返回: [{"name": "github", "path": "...", "source": "builtin"}, ...]
```

#### `load_skill()`

加载指定技能的内容。

```python
content = loader.load_skill("github")
```

#### `load_skills_for_context()`

加载多个技能用于上下文。

```python
content = loader.load_skills_for_context(["github", "weather"])
```

#### `build_skills_summary()`

构建所有技能的 XML 摘要，用于渐进式加载。

**返回格式：**
```xml
<skills>
  <skill available="true">
    <name>github</name>
    <description>GitHub 操作技能</description>
    <location>/path/to/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>docker</name>
    <description>Docker 操作技能</description>
    <location>/path/to/SKILL.md</location>
    <requires>CLI: docker, ENV: DOCKER_HOST</requires>
  </skill>
</skills>
```

#### `get_always_skills()`

获取标记为 `always=true` 的技能列表。

## 技能加载策略

### 1. 总是加载（Always Load）

标记为 `always=true` 的技能会在每次构建系统提示时完整加载。

**使用场景：**
- 核心功能技能
- 频繁使用的技能
- 基础工具使用说明

### 2. 按需加载（On-Demand Load）

其他技能只显示摘要，代理可以通过 `read_file` 工具按需加载完整内容。

**优势：**
- 减少系统提示长度
- 提高响应速度
- 降低 token 消耗

## 技能可用性检查

技能可以声明依赖要求：

```json
{
  "nanobot": {
    "requires": {
      "bins": ["git", "docker"],  // 需要这些命令行工具
      "env": ["GITHUB_TOKEN"]      // 需要这些环境变量
    }
  }
}
```

`SkillsLoader` 会自动检查：
- 命令行工具是否在 PATH 中
- 环境变量是否已设置

不可用的技能会标记为 `available="false"`，并显示缺失的依赖。

## 内置技能

nanobot 包含以下内置技能：

- **github**：GitHub 操作（创建 issue、PR 等）
- **weather**：天气查询
- **tmux**：tmux 会话管理
- **summarize**：文本摘要
- **skill-creator**：创建新技能

## 创建自定义技能

### 1. 创建技能目录

```bash
mkdir -p ~/.nanobot/workspace/skills/my-skill
```

### 2. 创建 SKILL.md

```markdown
---
description: 我的自定义技能
---

# My Skill

这个技能用于...

## 使用方法

1. 步骤一
2. 步骤二

## 示例

```bash
# 示例命令
```

## 注意事项

- 注意点1
- 注意点2
```

### 3. 可选：添加依赖检查

```markdown
---
metadata: |
  {
    "nanobot": {
      "requires": {
        "bins": ["my-tool"],
        "env": ["MY_API_KEY"]
      }
    }
  }
---
```

## 技能内容最佳实践

1. **清晰的结构**：使用 Markdown 标题组织内容
2. **具体示例**：提供实际可用的示例
3. **工具说明**：说明需要使用的工具和参数
4. **错误处理**：说明常见错误和解决方法
5. **工作流程**：描述完整的工作流程

## 技能在系统提示中的位置

技能内容会出现在系统提示的以下位置：

1. **活跃技能部分**：`always=true` 的技能完整内容
2. **技能摘要部分**：所有技能的 XML 摘要

代理可以根据摘要决定是否需要加载某个技能。

## 使用示例

### 在代理中使用技能

代理会自动看到技能摘要，可以这样使用：

```
用户：帮我创建一个 GitHub issue

代理：
1. 看到 github 技能摘要
2. 使用 read_file 加载完整技能内容
3. 按照技能说明执行操作
```

### 编程方式加载技能

```python
from nanobot.agent.skills import SkillsLoader
from pathlib import Path

loader = SkillsLoader(workspace=Path("~/.nanobot/workspace"))

# 列出所有技能
skills = loader.list_skills()

# 加载特定技能
content = loader.load_skill("github")

# 获取总是加载的技能
always_skills = loader.get_always_skills()
```

## 相关文件

- `nanobot/agent/skills.py`：主实现文件
- `nanobot/skills/`：内置技能目录
- `workspace/skills/`：用户自定义技能目录
