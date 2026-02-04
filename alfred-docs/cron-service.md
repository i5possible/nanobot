# Cron Service（定时任务服务）

## 概述

Cron Service 提供定时任务调度功能，允许代理在指定时间执行任务。支持三种调度方式：一次性执行、间隔执行和 Cron 表达式。

## 核心功能

1. **任务调度**：支持多种调度方式
2. **任务执行**：通过代理执行任务
3. **结果交付**：可选地将结果发送到聊天渠道
4. **持久化**：任务存储在磁盘上

## 调度类型

### 1. 一次性执行（`at`）

在指定时间执行一次。

**示例：**
```bash
nanobot cron add --name "reminder" --message "重要会议" --at "2026-01-20T14:00:00"
```

### 2. 间隔执行（`every`）

每隔 N 秒执行一次。

**示例：**
```bash
nanobot cron add --name "hourly" --message "检查状态" --every 3600
```

### 3. Cron 表达式（`cron`）

使用标准 Cron 表达式。

**示例：**
```bash
nanobot cron add --name "daily" --message "每日报告" --cron "0 9 * * *"
```

**Cron 格式：**
```
分钟 小时 日 月 星期
0    9   *  *  *    # 每天 9:00
0    */6 *  *  *    # 每 6 小时
0    0   *  *  1    # 每周一 0:00
```

## 核心类

### `CronService`

定时任务服务主类。

**初始化：**
```python
from nanobot.cron.service import CronService
from pathlib import Path

async def on_job(job: CronJob) -> str | None:
    """执行任务的回调"""
    # 通过代理执行任务
    response = await agent.process_direct(job.payload.message)
    return response

service = CronService(
    store_path=Path("~/.nanobot/cron/jobs.json"),
    on_job=on_job
)
```

## 关键方法

### `start()`

启动定时任务服务。

```python
await service.start()
```

**流程：**
1. 加载任务存储
2. 重新计算下次运行时间
3. 保存存储
4. 启动定时器

### `add_job()`

添加新任务。

**参数：**
- `name` (string)：任务名称
- `schedule` (CronSchedule)：调度配置
- `message` (string)：发送给代理的消息
- `deliver` (bool)：是否交付结果到渠道
- `channel` (string, 可选)：目标渠道
- `to` (string, 可选)：接收者 ID
- `delete_after_run` (bool)：执行后删除（仅一次性任务）

**示例：**
```python
from nanobot.cron.types import CronSchedule

schedule = CronSchedule(
    kind="cron",
    expr="0 9 * * *"  # 每天 9:00
)

job = service.add_job(
    name="每日报告",
    schedule=schedule,
    message="生成今日工作摘要",
    deliver=True,
    channel="telegram",
    to="123456789"
)
```

### `list_jobs()`

列出所有任务。

```python
jobs = service.list_jobs(include_disabled=False)
```

### `remove_job()`

删除任务。

```python
deleted = service.remove_job(job_id)
```

### `enable_job()`

启用或禁用任务。

```python
job = service.enable_job(job_id, enabled=True)
```

### `run_job()`

手动执行任务。

```python
success = await service.run_job(job_id, force=False)
```

## 任务执行流程

```
定时器触发
    ↓
检查到期的任务
    ↓
调用 on_job 回调
    ↓
通过代理执行任务
    ↓
更新任务状态
    ↓
计算下次运行时间
    ↓
保存任务存储
    ↓
重新设置定时器
```

## 任务存储格式

任务存储在 JSON 文件中：

```json
{
  "version": "1.0",
  "jobs": [
    {
      "id": "abc12345",
      "name": "每日报告",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * *",
        "tz": null
      },
      "payload": {
        "kind": "agent_turn",
        "message": "生成今日工作摘要",
        "deliver": true,
        "channel": "telegram",
        "to": "123456789"
      },
      "state": {
        "nextRunAtMs": 1737352800000,
        "lastRunAtMs": 1737266400000,
        "lastStatus": "ok",
        "lastError": null
      },
      "createdAtMs": 1737180000000,
      "updatedAtMs": 1737266400000,
      "deleteAfterRun": false
    }
  ]
}
```

## 任务状态

每个任务维护以下状态：

- `nextRunAtMs`：下次运行时间（毫秒时间戳）
- `lastRunAtMs`：上次运行时间
- `lastStatus`：上次执行状态（"ok" 或 "error"）
- `lastError`：错误消息（如果有）

## 结果交付

如果任务配置了 `deliver=true`，执行结果会自动发送到指定渠道：

```python
# 在 on_job 回调中
if job.payload.deliver and job.payload.to:
    await bus.publish_outbound(OutboundMessage(
        channel=job.payload.channel or "whatsapp",
        chat_id=job.payload.to,
        content=response or ""
    ))
```

## 使用示例

### 添加每日任务

```bash
nanobot cron add \
  --name "每日摘要" \
  --message "总结今天的重要事项" \
  --cron "0 22 * * *" \
  --deliver \
  --channel telegram \
  --to "123456789"
```

### 添加每小时任务

```bash
nanobot cron add \
  --name "状态检查" \
  --message "检查系统状态" \
  --every 3600
```

### 列出任务

```bash
nanobot cron list
```

### 删除任务

```bash
nanobot cron remove abc12345
```

### 禁用任务

```bash
nanobot cron enable abc12345 --disable
```

### 手动执行任务

```bash
nanobot cron run abc12345
```

## 定时器机制

Cron Service 使用异步定时器：

1. 计算所有任务的下次运行时间
2. 找到最早的时间
3. 设置定时器等待到该时间
4. 触发时执行到期任务
5. 重新计算并设置下一个定时器

**优势：**
- 高效：只在需要时唤醒
- 精确：基于毫秒时间戳
- 低资源：不使用轮询

## 一次性任务

对于 `kind="at"` 的任务：

- 执行后自动禁用（如果 `delete_after_run=false`）
- 或执行后删除（如果 `delete_after_run=true`）

## 错误处理

- 任务执行失败会记录错误状态
- 不会阻止其他任务执行
- 错误信息保存在 `lastError` 字段

## 时区支持

Cron 表达式支持时区配置（通过 `tz` 字段），但当前实现使用系统时区。

## 相关文件

- `nanobot/cron/service.py`：主实现文件
- `nanobot/cron/types.py`：类型定义
- `nanobot/cli/commands.py`：CLI 命令
