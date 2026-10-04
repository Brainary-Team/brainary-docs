# 创建与运行 Agent

Brainary Python SDK 使用 `Thread` 表达一个 Agent 会话，使用 turn 表达一次完整的 Agent 运行，即一次完整的ReAct循环。

SDK 分发包名为 `brainary-sdk`，导入模块为 `brainary`。同步客户端是 `Brainary`，
异步客户端是 `AsyncBrainary`；通过 `BrainaryConfig(brainary_bin=...)` 指定运行时。
运行时的可执行文件仍叫 `codex`，配置目录与环境变量仍使用 `.codex` 和 `CODEX_HOME`。

## 创建 Agent

下面的示例同时注册 model provider、选择模型，并配置 Agent 的工作目录、指令和权限：

```python
import os
from pathlib import Path

from brainary import ApprovalMode, Brainary, BrainaryConfig, Sandbox

config = BrainaryConfig(
    brainary_bin="/absolute/path/to/brainary-codex",
    env={"ANTHROPIC_API_KEY": os.environ["ANTHROPIC_API_KEY"]},
)

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(
        cwd=str(Path("/absolute/path/to/workspace")),
        developer_instructions="你负责审查代码，只报告有证据的问题。",
        sandbox=Sandbox.read_only,
        approval_mode=ApprovalMode.deny_all,
        model="<anthropic-model-id>",
        model_provider="anthropic-runtime",
        config={
            "model_providers": {
                "anthropic-runtime": {
                    "name": "Runtime Anthropic provider",
                    "base_url": "https://api.anthropic.com/v1",
                    "env_key": "ANTHROPIC_API_KEY",
                    "wire_api": "anthropic",
                    "request_max_retries": 4,
                    "stream_max_retries": 5,
                    "stream_idle_timeout_ms": 300_000,
                }
            }
        },
    )
    result = agent.run("检查当前改动。")
    print(result.final_response)
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
| `config` | 本 thread 的运行时配置覆盖，使用 Python `dict` 和 snake_case 键 |
| `ephemeral` | 是否创建临时、不持久化的 thread |

## 配置并选择模型

模型配置分成三个彼此独立的概念：

| 概念 | 配置位置 | 含义 |
| --- | --- | --- |
| 模型 | `model` | provider 接受的模型 ID，例如企业网关中注册的模型名 |
| provider | `model_provider` | `model_providers` 表中的键，决定请求发往哪里、如何认证 |
| 模型协议 | `model_providers.<id>.wire_api` | provider 使用的 HTTP 请求与流式响应协议 |

不要把 provider 名当作模型名。`model="..."` 只选择模型；它不会自动改变 endpoint、认证
方式或协议。要切换到另一个协议，必须同时选择对应的 `model_provider`。

### 当前支持的协议

当前运行时只接受以下两个 `wire_api` 值：

| `wire_api` | 请求端点 | 适用场景 |
| --- | --- | --- |
| `responses` | `POST <base_url>/responses` | OpenAI Responses API 或兼容网关；未填写 `wire_api` 时也是此默认值 |
| `anthropic` | `POST <base_url>/messages` | Anthropic Messages API 或兼容网关 |

`chat`（Chat Completions）已经移除，配置后会直接报错，不会自动降级为其他协议。对于
`anthropic`，运行时负责在内部 Prompt/ResponseItem 与 Messages 协议之间转换，并自动发送
`anthropic-version: 2023-06-01`；`env_key` 提供的 key 还会以 `x-api-key` 发送。

协议兼容不代表 provider 能力完全相同。当前 Anthropic 适配器支持消息、流式输出、普通
function tools 和 freeform tools；Responses 中的 provider-side Web Search、Tool Search 和
Namespace tools 没有 Anthropic 等价声明，会被省略。应用依赖这些能力时，应选用
`responses` provider 或在部署前做能力验证。

### 在 Python 中注册并应用 provider

开头的[创建 Agent 示例](#创建-agent)将 provider 定义放入 `thread_start(config=...)`，
再用 `model_provider` 选择其 ID。配置键使用 snake_case，嵌套配置使用 Python `dict`。

`base_url` 是 API 根地址，运行时会按协议追加 `responses` 或 `messages`。
`env_key` 的值是环境变量名，不是 API key 本身。`BrainaryConfig.env` 会在父进程环境
基础上更新变量；部署环境已导出 key 时，可以省略示例中的 `env`。

网关要求自定义认证 header 时，在 provider 定义中添加
`"env_http_headers": {"X-Gateway-Key": "CORP_GATEWAY_KEY"}`，并在进程环境中提供
`CORP_GATEWAY_KEY`。固定且不敏感的 header 使用 `http_headers`。

多个 Agent 可以复用同一个 Python 配置字典，并分别传入 `thread_start(config=...)`。
这些覆盖不会写入配置文件；跨进程调用 `thread_resume()` 或 `thread_fork()` 时，
仍需传入同一 provider 定义。配置文件的注册和加载方式见[配置文件使用说明](configuration.md)。

自定义 provider ID 不得覆盖内置 ID；当前内置项包括 `openai`、`amazon-bedrock`、
`ollama` 和 `lmstudio`，其中 `amazon-bedrock` 只开放源码限定的少量覆盖字段。

### thread 与 turn 的模型覆盖边界

`thread_start()`、`thread_resume()` 和 `thread_fork()` 都同时接受 `model=` 与
`model_provider=`，因此可以选择完整的“模型 + provider + 协议”组合。`run()` 和 `turn()`
只有 `model=`，适合在同一个 provider 内切换模型。在创建 Agent 示例的 `with` 块内调用：

```python
# 接创建 Agent 示例中的 agent，仍使用 anthropic-runtime 及其 anthropic 协议。
# 这里只覆盖模型 ID。
result = agent.run(
    "进行更深入的分析。",
    model="<strong-model-id>",
)
```

单次 turn 不能通过高层 Python API 改变 `model_provider`。需要从 `responses` 切换到
`anthropic`（或反向切换）时，应创建新 thread，或使用带 `model_provider=` 的
`thread_resume()` / `thread_fork()`。同步与异步 API 的参数和规则相同。

可以用运行时模型目录检查其报告的模型 ID、推理强度和 service tier：

```python
with Brainary(config=config) as brainary:
    for item in brainary.models().data:
        print(item.model, item.supported_reasoning_efforts, item.service_tiers)
```

`brainary.models()` 返回运行时加载的模型目录，不保证列出自定义 provider 接受的所有模型；
最终可用的模型 ID 仍由 provider 契约决定。不要在框架代码里假设固定模型 ID。

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

from brainary import AsyncBrainary, BrainaryConfig


async def main() -> None:
    config = BrainaryConfig(brainary_bin="/absolute/path/to/brainary-codex")
    async with AsyncBrainary(config=config) as brainary:
        agent = await brainary.thread_start()
        result = await agent.run("生成一份模块说明。")
        print(result.final_response)


asyncio.run(main())
```

异步流使用 `async for event in handle.stream()`。`AsyncBrainary` 会在进入异步上下文或第一次 awaited API 调用时初始化。

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
agent = brainary.thread_resume("thread-id")
result = agent.run("从上次的位置继续。")
```

需要从既有上下文创建独立分支时：

```python
branch = brainary.thread_fork("thread-id")
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
