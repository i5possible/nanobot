# LLM Provider（LLM 提供者）

## 概述

LLM Provider 提供了统一的接口来访问各种大语言模型。nanobot 使用 LiteLLM 作为底层库，支持多种 LLM 提供商，包括 OpenRouter、Anthropic、OpenAI、Gemini 等。

## 核心组件

### `LLMProvider` 基类

抽象基类，定义 LLM 提供者接口。

**必需方法：**
- `chat()`：发送聊天完成请求
- `get_default_model()`：获取默认模型

### `LiteLLMProvider`

使用 LiteLLM 的实现，支持多提供商。

## 支持的提供商

### OpenRouter（推荐）

- **优势**：访问所有主要模型，统一接口
- **配置**：
  ```json
  {
    "providers": {
      "openrouter": {
        "apiKey": "sk-or-v1-xxx"
      }
    }
  }
  ```
- **模型格式**：`anthropic/claude-opus-4-5` 或 `openrouter/anthropic/claude-opus-4-5`

### Anthropic（Claude）

- **配置**：
  ```json
  {
    "providers": {
      "anthropic": {
        "apiKey": "sk-ant-xxx"
      }
    }
  }
  ```
- **模型格式**：`anthropic/claude-sonnet-4-5`

### OpenAI（GPT）

- **配置**：
  ```json
  {
    "providers": {
      "openai": {
        "apiKey": "sk-xxx"
      }
    }
  }
  ```
- **模型格式**：`gpt-4`, `gpt-3.5-turbo`

### Gemini

- **配置**：
  ```json
  {
    "providers": {
      "gemini": {
        "apiKey": "xxx"
      }
    }
  }
  ```
- **模型格式**：`gemini/gemini-pro`

### Groq

- **配置**：
  ```json
  {
    "providers": {
      "groq": {
        "apiKey": "gsk_xxx"
      }
    }
  }
  ```
- **特性**：还支持 Whisper 语音转录

### vLLM（本地模型）

- **配置**：
  ```json
  {
    "providers": {
      "vllm": {
        "apiKey": "dummy",
        "apiBase": "http://localhost:8000/v1"
      }
    }
  }
  ```
- **用途**：运行本地部署的模型

## 核心方法

### `chat()`

发送聊天完成请求。

**参数：**
- `messages` (list[dict])：消息列表
- `tools` (list[dict], 可选)：工具定义
- `model` (string, 可选)：模型标识符
- `max_tokens` (int)：最大 token 数（默认 4096）
- `temperature` (float)：采样温度（默认 0.7）

**返回：**
`LLMResponse` 对象，包含：
- `content`：文本响应
- `tool_calls`：工具调用列表
- `finish_reason`：完成原因
- `usage`：token 使用统计

**示例：**
```python
response = await provider.chat(
    messages=[
        {"role": "system", "content": "You are a helpful assistant"},
        {"role": "user", "content": "Hello!"}
    ],
    tools=tool_definitions,
    model="anthropic/claude-opus-4-5"
)
```

## 响应格式

### `LLMResponse`

```python
@dataclass
class LLMResponse:
    content: str | None  # 文本响应
    tool_calls: list[ToolCallRequest]  # 工具调用
    finish_reason: str  # "stop", "length", "tool_calls", "error"
    usage: dict[str, int]  # {"prompt_tokens": 100, "completion_tokens": 50, ...}
    
    @property
    def has_tool_calls(self) -> bool:
        """检查是否包含工具调用"""
```

### `ToolCallRequest`

```python
@dataclass
class ToolCallRequest:
    id: str  # 工具调用 ID
    name: str  # 工具名称
    arguments: dict[str, Any]  # 工具参数
```

## 模型名称处理

`LiteLLMProvider` 会自动处理不同提供商的模型名称格式：

### OpenRouter

```python
# 自动添加 openrouter/ 前缀（如果需要）
model = "anthropic/claude-opus-4-5"
# 转换为: "openrouter/anthropic/claude-opus-4-5"
```

### Zhipu

```python
# 自动添加 zhipu/ 前缀
model = "glm-4.7-flash"
# 转换为: "zhipu/glm-4.7-flash"
```

### vLLM

```python
# 使用 hosted_vllm/ 前缀
model = "meta-llama/Llama-3.1-8B-Instruct"
# 转换为: "hosted_vllm/meta-llama/Llama-3.1-8B-Instruct"
```

### Gemini

```python
# 自动添加 gemini/ 前缀
model = "gemini-pro"
# 转换为: "gemini/gemini-pro"
```

## API 密钥优先级

配置加载器按以下优先级选择 API 密钥：

1. OpenRouter
2. Anthropic
3. OpenAI
4. Gemini
5. Zhipu
6. Groq
7. vLLM

## 工具调用支持

所有支持的提供商都支持函数调用（工具调用）：

```python
response = await provider.chat(
    messages=messages,
    tools=[
        {
            "type": "function",
            "function": {
                "name": "read_file",
                "description": "Read a file",
                "parameters": {...}
            }
        }
    ],
    tool_choice="auto"  # 自动决定是否使用工具
)
```

## 错误处理

如果 LLM 调用失败：

```python
try:
    response = await provider.chat(...)
except Exception as e:
    # 返回错误响应
    return LLMResponse(
        content=f"Error calling LLM: {str(e)}",
        finish_reason="error"
    )
```

## 使用示例

### 基本使用

```python
from nanobot.providers.litellm_provider import LiteLLMProvider

provider = LiteLLMProvider(
    api_key="sk-or-v1-xxx",
    api_base="https://openrouter.ai/api/v1",
    default_model="anthropic/claude-opus-4-5"
)

response = await provider.chat(
    messages=[
        {"role": "user", "content": "Hello!"}
    ]
)

print(response.content)
```

### 带工具调用

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Read a file",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string"}
                },
                "required": ["path"]
            }
        }
    }
]

response = await provider.chat(
    messages=messages,
    tools=tools
)

if response.has_tool_calls:
    for tool_call in response.tool_calls:
        print(f"Tool: {tool_call.name}")
        print(f"Args: {tool_call.arguments}")
```

### 本地模型（vLLM）

```python
provider = LiteLLMProvider(
    api_key="dummy",
    api_base="http://localhost:8000/v1",
    default_model="meta-llama/Llama-3.1-8B-Instruct"
)
```

## 配置示例

### 完整配置

```json
{
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-xxx"
    },
    "groq": {
      "apiKey": "gsk_xxx"
    }
  },
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-4-5"
    }
  }
}
```

## 相关文件

- `nanobot/providers/base.py`：基类定义
- `nanobot/providers/litellm_provider.py`：LiteLLM 实现
- `nanobot/config/schema.py`：配置模式
