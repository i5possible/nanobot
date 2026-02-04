# Heartbeat Service（心跳服务）

## 概述

Heartbeat Service 提供周期性唤醒功能，定期检查工作空间中的 `HEARTBEAT.md` 文件，如果发现待办任务，则唤醒代理执行。这允许代理主动处理任务，而无需用户直接交互。

## 核心功能

1. **周期性检查**：定期检查 `HEARTBEAT.md` 文件
2. **任务检测**：识别文件中的待办任务
3. **代理唤醒**：唤醒代理执行任务
4. **空任务跳过**：如果文件为空或只有已完成任务，则跳过

## 工作原理

```
每 30 分钟（默认）
    ↓
读取 HEARTBEAT.md
    ↓
检查是否有待办任务
    ↓
如果有任务：
    - 唤醒代理
    - 代理读取并执行任务
    - 代理更新文件（标记完成等）
如果没有任务：
    - 跳过本次心跳
```

## 核心类

### `HeartbeatService`

心跳服务主类。

**初始化：**
```python
from nanobot.heartbeat.service import HeartbeatService
from pathlib import Path

async def on_heartbeat(prompt: str) -> str:
    """心跳回调，通过代理执行"""
    return await agent.process_direct(prompt, session_key="heartbeat")

service = HeartbeatService(
    workspace=Path("~/.nanobot/workspace"),
    on_heartbeat=on_heartbeat,
    interval_s=30 * 60,  # 30 分钟
    enabled=True
)
```

## 关键方法

### `start()`

启动心跳服务。

```python
await service.start()
```

### `stop()`

停止心跳服务。

```python
service.stop()
```

### `trigger_now()`

手动触发一次心跳。

```python
response = await service.trigger_now()
```

## HEARTBEAT.md 格式

### 基本格式

```markdown
# 待办任务

- [ ] 检查邮件
- [ ] 更新项目文档
- [ ] 准备会议材料

# 已完成

- [x] 完成代码审查
```

### 空文件检测

以下情况被视为"空"（无待办任务）：

- 文件不存在
- 文件为空
- 只有标题行（`#` 开头）
- 只有 HTML 注释（`<!-- -->`）
- 只有空的复选框（`- [ ]` 或 `* [ ]`）
- 只有已完成的复选框（`- [x]` 或 `* [x]`）

### 待办任务检测

任何非空、非标题、非注释、非空复选框的行都被视为待办任务。

## 心跳提示

代理收到的提示：

```
Read HEARTBEAT.md in your workspace (if it exists).
Follow any instructions or tasks listed there.
If nothing needs attention, reply with just: HEARTBEAT_OK
```

## 响应处理

### 空任务响应

如果代理返回包含 `HEARTBEAT_OK`（忽略下划线），表示没有需要处理的任务：

```python
if "HEARTBEAT_OK" in response.upper().replace("_", ""):
    logger.info("Heartbeat: OK (no action needed)")
```

### 任务完成响应

如果代理返回其他内容，表示执行了任务：

```python
else:
    logger.info("Heartbeat: completed task")
```

## 配置

### 默认间隔

默认每 30 分钟检查一次：

```python
DEFAULT_HEARTBEAT_INTERVAL_S = 30 * 60  # 30 分钟
```

### 自定义间隔

```python
service = HeartbeatService(
    workspace=workspace,
    on_heartbeat=on_heartbeat,
    interval_s=60 * 60,  # 1 小时
    enabled=True
)
```

### 禁用心跳

```python
service = HeartbeatService(
    workspace=workspace,
    on_heartbeat=on_heartbeat,
    enabled=False  # 禁用
)
```

## 使用场景

### 1. 每日待办清单

在 `HEARTBEAT.md` 中维护每日待办：

```markdown
# 今日待办

- [ ] 检查 GitHub issues
- [ ] 回复重要邮件
- [ ] 更新项目进度
```

### 2. 定期检查

```markdown
# 定期检查

- [ ] 检查系统日志是否有错误
- [ ] 备份重要文件
- [ ] 更新依赖包
```

### 3. 提醒事项

```markdown
# 提醒

- [ ] 提醒用户：明天有重要会议
- [ ] 检查待办事项列表
```

## 工作流程示例

### 场景：每日待办清单

1. **用户创建 HEARTBEAT.md**：
   ```markdown
   - [ ] 检查邮件
   - [ ] 生成日报
   ```

2. **心跳触发**（30 分钟后）：
   - 服务读取文件
   - 检测到待办任务
   - 唤醒代理

3. **代理执行**：
   - 读取 `HEARTBEAT.md`
   - 执行任务（检查邮件、生成日报）
   - 更新文件，标记完成：
     ```markdown
     - [x] 检查邮件
     - [x] 生成日报
     ```

4. **下次心跳**：
   - 文件只有已完成任务
   - 跳过执行

## 最佳实践

1. **清晰的任务描述**：使用明确、可执行的任务描述
2. **及时更新**：完成任务后及时标记
3. **合理间隔**：根据任务频率调整间隔
4. **避免冲突**：确保任务可以独立执行，不依赖顺序

## 与 Cron Service 的区别

| 特性 | Heartbeat Service | Cron Service |
|------|-------------------|--------------|
| 触发方式 | 周期性检查文件 | 精确时间调度 |
| 任务定义 | HEARTBEAT.md 文件 | 配置中的任务 |
| 灵活性 | 用户可编辑文件 | 需要 CLI/API |
| 适用场景 | 待办清单、提醒 | 定时报告、自动化 |

## 相关文件

- `nanobot/heartbeat/service.py`：主实现文件
- `workspace/HEARTBEAT.md`：心跳任务文件
