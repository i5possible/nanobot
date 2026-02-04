# Configuration（配置系统）

## 概述

Configuration 系统使用 Pydantic 提供类型安全的配置管理。配置文件位于 `~/.nanobot/config.json`，支持环境变量覆盖。

## 配置文件位置

默认配置文件：`~/.nanobot/config.json`

## 配置结构

### 完整配置示例

```json
{
  "agents": {
    "defaults": {
      "workspace": "~/.nanobot/workspace",
      "model": "anthropic/claude-opus-4-5",
      "max_tokens": 8192,
      "temperature": 0.7,
      "max_tool_iterations": 20
    }
  },
  "providers": {
    "openrouter": {
      "apiKey": "sk-or-v1-xxx"
    },
    "anthropic": {
      "apiKey": "sk-ant-xxx"
    },
    "openai": {
      "apiKey": "sk-xxx"
    },
    "groq": {
      "apiKey": "gsk_xxx"
    },
    "gemini": {
      "apiKey": "xxx"
    },
    "zhipu": {
      "apiKey": "xxx"
    },
    "vllm": {
      "apiKey": "dummy",
      "apiBase": "http://localhost:8000/v1"
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "token": "123456:ABC...",
      "allowFrom": ["123456789"]
    },
    "whatsapp": {
      "enabled": true,
      "bridge_url": "ws://localhost:3001",
      "allowFrom": ["+1234567890"]
    }
  },
  "gateway": {
    "host": "0.0.0.0",
    "port": 18790
  },
  "tools": {
    "web": {
      "search": {
        "apiKey": "BSA...",
        "max_results": 5
      }
    }
  }
}
```

## 配置类

### `Config`

根配置类。

**属性：**
- `agents`：代理配置
- `channels`：渠道配置
- `providers`：LLM 提供商配置
- `gateway`：网关配置
- `tools`：工具配置

**方法：**
- `workspace_path`：获取展开的工作空间路径
- `get_api_key()`：按优先级获取 API 密钥
- `get_api_base()`：获取 API 基础 URL

### `AgentsConfig`

代理配置。

**属性：**
- `defaults`：默认代理配置

### `AgentDefaults`

默认代理配置。

**字段：**
- `workspace`：工作空间路径（默认 `~/.nanobot/workspace`）
- `model`：默认模型（默认 `anthropic/claude-opus-4-5`）
- `max_tokens`：最大 token 数（默认 8192）
- `temperature`：采样温度（默认 0.7）
- `max_tool_iterations`：最大工具迭代次数（默认 20）

### `ProvidersConfig`

LLM 提供商配置。

**支持的提供商：**
- `openrouter`：OpenRouter
- `anthropic`：Anthropic (Claude)
- `openai`：OpenAI (GPT)
- `groq`：Groq
- `gemini`：Google Gemini
- `zhipu`：智谱 AI
- `vllm`：vLLM（本地模型）

### `ProviderConfig`

单个提供商配置。

**字段：**
- `api_key`：API 密钥
- `api_base`：API 基础 URL（可选）

### `ChannelsConfig`

渠道配置。

**属性：**
- `telegram`：Telegram 配置
- `whatsapp`：WhatsApp 配置

### `TelegramConfig`

Telegram 渠道配置。

**字段：**
- `enabled`：是否启用（默认 false）
- `token`：Bot Token（从 @BotFather 获取）
- `allow_from`：允许的用户 ID 列表（空列表表示允许所有人）

### `WhatsAppConfig`

WhatsApp 渠道配置。

**字段：**
- `enabled`：是否启用（默认 false）
- `bridge_url`：Bridge WebSocket URL（默认 `ws://localhost:3001`）
- `allow_from`：允许的电话号码列表

### `GatewayConfig`

网关配置。

**字段：**
- `host`：监听地址（默认 `0.0.0.0`）
- `port`：监听端口（默认 18790）

### `ToolsConfig`

工具配置。

**属性：**
- `web`：Web 工具配置

### `WebToolsConfig`

Web 工具配置。

**属性：**
- `search`：搜索工具配置

### `WebSearchConfig`

Web 搜索配置。

**字段：**
- `api_key`：Brave Search API 密钥
- `max_results`：最大结果数（默认 5）

## API 密钥优先级

`get_api_key()` 方法按以下优先级返回 API 密钥：

1. OpenRouter
2. Anthropic
3. OpenAI
4. Gemini
5. Zhipu
6. Groq
7. vLLM

## API 基础 URL

`get_api_base()` 方法返回：

- OpenRouter：`https://openrouter.ai/api/v1`（如果配置了 OpenRouter）
- Zhipu：Zhipu 的 `api_base`（如果配置了）
- vLLM：vLLM 的 `api_base`（如果配置了）
- 其他：`None`

## 环境变量支持

配置支持通过环境变量覆盖，使用 `NANOBOT_` 前缀：

```bash
# 设置 OpenRouter API 密钥
export NANOBOT_PROVIDERS__OPENROUTER__APIKEY="sk-or-v1-xxx"

# 设置模型
export NANOBOT_AGENTS__DEFAULTS__MODEL="anthropic/claude-sonnet-4-5"

# 启用 Telegram
export NANOBOT_CHANNELS__TELEGRAM__ENABLED="true"
export NANOBOT_CHANNELS__TELEGRAM__TOKEN="123456:ABC..."
```

**注意：** 嵌套配置使用双下划线（`__`）分隔。

## 配置加载

### 加载配置

```python
from nanobot.config.loader import load_config

config = load_config()
```

### 保存配置

```python
from nanobot.config.loader import save_config

save_config(config)
```

### 获取配置路径

```python
from nanobot.config.loader import get_config_path

config_path = get_config_path()
# 返回: Path("~/.nanobot/config.json")
```

## 初始化配置

使用 CLI 命令初始化：

```bash
nanobot onboard
```

这会创建：
1. 默认配置文件
2. 工作空间目录
3. 引导文件模板（AGENTS.md, SOUL.md, USER.md）
4. 记忆目录和 MEMORY.md

## 配置验证

Pydantic 自动验证配置：

- **类型检查**：确保字段类型正确
- **必需字段**：检查必需字段是否存在
- **默认值**：为缺失字段提供默认值
- **验证错误**：提供清晰的错误信息

## 使用示例

### 读取配置

```python
from nanobot.config.loader import load_config

config = load_config()

# 获取工作空间路径
workspace = config.workspace_path

# 获取 API 密钥
api_key = config.get_api_key()

# 获取模型
model = config.agents.defaults.model

# 检查渠道是否启用
if config.channels.telegram.enabled:
    token = config.channels.telegram.token
```

### 修改配置

```python
from nanobot.config.loader import load_config, save_config

config = load_config()

# 修改模型
config.agents.defaults.model = "anthropic/claude-sonnet-4-5"

# 启用 Telegram
config.channels.telegram.enabled = True
config.channels.telegram.token = "123456:ABC..."

# 保存
save_config(config)
```

## 相关文件

- `nanobot/config/schema.py`：配置模式定义
- `nanobot/config/loader.py`：配置加载器
