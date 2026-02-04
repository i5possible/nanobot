# Tool System（工具系统）

## 概述

Tool System 允许代理通过函数调用与外部环境交互。所有工具都实现 `Tool` 接口，并通过 `ToolRegistry` 进行注册和管理。工具定义遵循 OpenAI 函数调用格式。

## 核心组件

### `Tool` 基类

所有工具必须继承 `Tool` 抽象基类。

**必需属性：**
- `name`：工具名称（用于函数调用）
- `description`：工具描述
- `parameters`：参数 JSON Schema

**必需方法：**
- `execute(**kwargs)`：执行工具

### `ToolRegistry`

工具注册表，管理所有可用工具。

**功能：**
- 注册/注销工具
- 获取工具定义（OpenAI 格式）
- 执行工具

## 内置工具

### 文件系统工具

#### `read_file`

读取文件内容。

**参数：**
- `path` (string)：文件路径

**示例：**
```python
result = await tools.execute("read_file", {"path": "/path/to/file.txt"})
```

#### `write_file`

写入文件（覆盖）。

**参数：**
- `path` (string)：文件路径
- `content` (string)：文件内容

**特性：**
- 自动创建父目录
- UTF-8 编码

#### `edit_file`

编辑文件（文本替换）。

**参数：**
- `path` (string)：文件路径
- `old_text` (string)：要替换的文本（必须完全匹配）
- `new_text` (string)：新文本

**特性：**
- 如果 `old_text` 出现多次，会警告并要求更多上下文
- 只替换第一次出现

#### `list_dir`

列出目录内容。

**参数：**
- `path` (string)：目录路径

**返回：**
格式化的目录列表（📁 表示目录，📄 表示文件）

### Shell 工具

#### `exec`

执行 shell 命令。

**参数：**
- `command` (string)：要执行的命令
- `working_dir` (string, 可选)：工作目录

**特性：**
- 超时保护（默认 60 秒）
- 捕获 stdout 和 stderr
- 输出截断（超过 10000 字符）
- 返回退出码

**安全注意事项：**
- 使用需谨慎
- 建议限制可执行的命令范围

### Web 工具

#### `web_search`

使用 Brave Search API 搜索网络。

**参数：**
- `query` (string)：搜索查询
- `count` (integer, 可选)：结果数量（1-10，默认 5）

**要求：**
- 需要配置 `BRAVE_API_KEY`

**返回：**
格式化的搜索结果（标题、URL、描述）

#### `web_fetch`

获取并提取网页内容。

**参数：**
- `url` (string)：要获取的 URL
- `extractMode` (string, 可选)：提取模式（"markdown" 或 "text"）
- `maxChars` (integer, 可选)：最大字符数（默认 50000）

**特性：**
- 使用 Readability 提取主要内容
- 支持 HTML → Markdown 转换
- 自动处理重定向（最多 5 次）
- URL 验证（仅允许 http/https）

**返回：**
JSON 格式的结果，包含：
- `url`：原始 URL
- `finalUrl`：最终 URL（考虑重定向）
- `status`：HTTP 状态码
- `extractor`：使用的提取器（readability/json/raw）
- `text`：提取的内容
- `truncated`：是否被截断

### 消息工具

#### `message`

发送消息到聊天渠道。

**参数：**
- `channel` (string)：目标渠道
- `chat_id` (string)：聊天 ID
- `content` (string)：消息内容

**特性：**
- 需要设置上下文（通过 `set_context()`）
- 用于在后台任务中发送消息

### 生成工具

#### `spawn`

创建子代理执行后台任务。

**参数：**
- `task` (string)：任务描述
- `label` (string, 可选)：任务标签（用于显示）

**特性：**
- 子代理在后台异步运行
- 完成后会通知主代理
- 子代理有独立的工具集（无 message 和 spawn 工具）

## 工具注册

### 默认工具注册

Agent Loop 在初始化时自动注册默认工具：

```python
# 文件工具
tools.register(ReadFileTool())
tools.register(WriteFileTool())
tools.register(EditFileTool())
tools.register(ListDirTool())

# Shell 工具
tools.register(ExecTool(working_dir=str(workspace)))

# Web 工具
tools.register(WebSearchTool(api_key=brave_api_key))
tools.register(WebFetchTool())

# 消息工具
tools.register(MessageTool(send_callback=bus.publish_outbound))

# 生成工具
tools.register(SpawnTool(manager=subagents))
```

### 自定义工具注册

```python
from nanobot.agent.tools.base import Tool
from nanobot.agent.tools.registry import ToolRegistry

class MyCustomTool(Tool):
    @property
    def name(self) -> str:
        return "my_tool"
    
    @property
    def description(self) -> str:
        return "My custom tool"
    
    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "param1": {"type": "string"}
            },
            "required": ["param1"]
        }
    
    async def execute(self, param1: str, **kwargs) -> str:
        return f"Result: {param1}"

# 注册
registry = ToolRegistry()
registry.register(MyCustomTool())
```

## 工具执行流程

```
LLM 返回工具调用
    ↓
Agent Loop 解析工具调用
    ↓
ToolRegistry.execute(tool_name, params)
    ↓
查找工具
    ↓
调用 tool.execute(**params)
    ↓
返回结果（字符串）
    ↓
添加到消息列表
    ↓
继续 LLM 调用
```

## 工具定义格式

工具定义遵循 OpenAI 函数调用格式：

```python
{
    "type": "function",
    "function": {
        "name": "read_file",
        "description": "Read the contents of a file",
        "parameters": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "The file path"
                }
            },
            "required": ["path"]
        }
    }
}
```

## 错误处理

所有工具都应该：
- 捕获异常并返回错误消息（字符串格式）
- 提供有意义的错误信息
- 不抛出未处理的异常

**示例：**
```python
async def execute(self, path: str, **kwargs) -> str:
    try:
        # 执行操作
        return "Success"
    except PermissionError:
        return f"Error: Permission denied: {path}"
    except Exception as e:
        return f"Error: {str(e)}"
```

## 工具上下文

某些工具需要上下文信息：

- **MessageTool**：需要知道发送消息的渠道和聊天 ID
- **SpawnTool**：需要知道原始渠道和聊天 ID（用于通知）

这些上下文在 Agent Loop 处理消息时自动设置。

## 最佳实践

1. **清晰的描述**：提供清晰的工具描述，帮助 LLM 理解何时使用
2. **参数验证**：在 `execute()` 中验证参数
3. **错误处理**：始终返回字符串结果，即使出错
4. **输出限制**：限制工具输出长度，避免 token 浪费
5. **安全性**：对危险操作（如 shell 命令）添加限制

## 相关文件

- `nanobot/agent/tools/base.py`：工具基类
- `nanobot/agent/tools/registry.py`：工具注册表
- `nanobot/agent/tools/filesystem.py`：文件系统工具
- `nanobot/agent/tools/shell.py`：Shell 工具
- `nanobot/agent/tools/web.py`：Web 工具
- `nanobot/agent/tools/message.py`：消息工具
- `nanobot/agent/tools/spawn.py`：生成工具
