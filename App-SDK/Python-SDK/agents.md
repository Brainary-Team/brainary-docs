# 创建与运行 Agent

Brainary Python SDK 使用 `Thread` 表达一个 Agent 会话，使用 turn 表达一次完整的 Agent 运行，即一次完整的ReAct循环。

## 创建 Agent

```python
from pathlib import Path

from openai_codex import ApprovalMode, Codex, CodexConfig, Sandbox

config = CodexConfig(codex_bin="/absolute/path/to/brainary-codex")

with Codex(config=config) as codex:
    agent = codex.thread_start(
        cwd=str(Path("/absolute/path/to/workspace")),
        developer_instructions="你负责审查代码，只报告有证据的问题。",
        sandbox=Sandbox.read_only,
        approval_mode=ApprovalMode.deny_all,
    )
```

常用参数：

| 参数 | 作用 |
| --- | --- |
| `cwd` | Agent 的工作目录，也是仓库级配置、Skills 和 `AGENTS.md` 的发现起点 |
| `developer_instructions` | 本 Agent 的开发者指令 |
| `base_instructions` | 替换或指定更底层的基础指令；普通应用通常不需要设置 |
| `model` / `model_provider` | 覆盖运行时默认模型与 provider |
| `sandbox` | 文件系统访问级别 |
| `approval_mode` | 权限升级请求的处理方式 |
| `config` | 本 thread 的运行时配置覆盖，使用 Python `dict` 和 snake_case 键；详见[配置运行时](configuration.md) |
| `ephemeral` | 是否创建临时、不持久化的 thread |

SDK 当前提供两个审批模式：

- `ApprovalMode.auto_review`：默认值，由运行时的自动审批审查器处理请求。
- `ApprovalMode.deny_all`：不允许权限升级。

沙箱预设：

- `Sandbox.read_only`：可读，不允许写入。
- `Sandbox.workspace_write`：可写工作区和已配置的可写根目录。
- `Sandbox.full_access`：不限制文件系统访问，应只在明确需要时使用。

## 运行方式一：阻塞到完成

大多数脚本和后端任务直接使用 `run()`：

```python
result = agent.run("检查当前改动是否有回归风险。")

print(result.id)
print(result.status.value)
print(result.final_response)
print(result.items)
print(result.usage)
```

`run()` 内部会启动 turn、消费事件直到收到 `turn/completed`，然后组装 `TurnResult`。如果 turn 以失败状态结束，SDK 会抛出 `RuntimeError`，而不是返回一个可忽略的失败结果。

## 运行方式二：先拿句柄，再等待结果

需要在 turn 执行期间保留控制权时，先调用 `turn()`：

```python
handle = agent.turn("分析整个仓库并给出迁移计划。")
print("turn_id:", handle.id)

result = handle.run()
print(result.final_response)
```

`handle.run()` 与 `agent.run()` 最终返回相同形状的 `TurnResult`，区别是前者让调用方先拿到 `TurnHandle`。

## 运行方式三：流式消费事件

```python
handle = agent.turn("用三点解释这个项目。")

for event in handle.stream():
    if event.method == "item/agentMessage/delta":
        print(event.payload.delta, end="", flush=True)
    elif event.method == "turn/completed":
        print("\nstatus:", event.payload.turn.status.value)
```

`stream()` 只返回属于当前 turn 的通知，并在对应的 `turn/completed` 后结束。流式消费后不要再对同一个 handle 调用 `run()`，因为事件已经被消费。

## 运行方式四：异步执行

```python
import asyncio

from openai_codex import AsyncCodex, CodexConfig


async def main() -> None:
    config = CodexConfig(codex_bin="/absolute/path/to/brainary-codex")
    async with AsyncCodex(config=config) as codex:
        agent = await codex.thread_start()
        result = await agent.run("生成一份模块说明。")
        print(result.final_response)


asyncio.run(main())
```

异步流使用 `async for event in handle.stream()`。`AsyncCodex` 会在进入异步上下文或第一次 awaited API 调用时初始化。

## 在运行中追加指令或中断

```python
handle = agent.turn("输出一个详细重构方案。")

handle.steer("只保留三项最高优先级，并说明理由。")

for event in handle.stream():
    if event.method == "turn/completed":
        break
```

取消当前 turn：

```python
handle = agent.turn("执行一个耗时分析。")
handle.interrupt()

for event in handle.stream():
    if event.method == "turn/completed":
        print(event.payload.turn.status.value)
```

`steer()` 和 `interrupt()` 发出请求后，仍应继续消费事件直到 `turn/completed`，这样才能得到最终状态并释放 turn 的事件路由。

## 多轮对话

在同一个 `Thread` 上多次调用 `run()`，上下文会继续累积：

```python
agent.run("找出认证模块的入口。")
result = agent.run("继续检查它的错误处理。")
print(result.final_response)
```

需要跨进程继续时，保存 `agent.id`，以后恢复：

```python
agent = codex.thread_resume("thread-id")
result = agent.run("从上次的位置继续。")
```

需要从既有上下文创建独立分支时：

```python
branch = codex.thread_fork("thread-id")
result = branch.run("尝试另一套实现方案。")
```

还可以使用 `thread_list()`、`thread_archive()`、`thread_unarchive()` 管理保存的 thread；`agent.read(include_turns=True)` 读取历史，`agent.set_name(...)` 设置名称，`agent.compact()` 请求压缩上下文。

## 每个 turn 覆盖运行参数

模型、推理强度、工作目录、沙箱、审批模式和输出 schema 可以在 turn 上覆盖：

```python
result = agent.run(
    "只返回 JSON，包含 summary 和 risks。",
    sandbox=Sandbox.read_only,
    output_schema={
        "type": "object",
        "properties": {
            "summary": {"type": "string"},
            "risks": {"type": "array", "items": {"type": "string"}},
        },
        "required": ["summary", "risks"],
        "additionalProperties": False,
    },
)
```

turn 上的 `sandbox=` 会应用于该 turn，并成为这个 thread 后续 turn 的沙箱设置。

## 主 Agent 与子 Agent

Python SDK 没有 `spawn_agent()` 之类的直接公共方法。`thread_start()` 创建的是 Python 调用方直接控制的主 Agent。运行时是否向模型暴露多 Agent 工具，由 Brainary Codex 的配置和能力决定。

需要由 Python 同时控制多个独立 Agent 时，创建多个 thread；需要确定性的子 Agent 扇出、等待和汇总时，使用 [PoA](run-poa.md) 更合适。
