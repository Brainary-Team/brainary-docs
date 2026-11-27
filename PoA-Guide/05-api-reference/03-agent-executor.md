# AgentExecutor

AgentExecutor 创建、派发、等待、查询、追加消息、打断和关闭 agent，kind 为 "agent"。

- [create_agent](#create_agentrole-nickname-description-agent_config)：按 [AgentConfig](02-agent-config.md) 创建 agent，返回可以多次运行的 [Agent](01-agent.md)。
- [spawn_agent](#spawn_agentspawn_spec)：沿用当前程序的配置派出 agent 并立即开始任务，再用 [acquire_agent_result](#acquire_agent_resultagent_id-timeout_ms) 获取结果。

按 ID 操作的方法（acquire_agent_result、get_status、append_message、interrupt、close）可以用于任何 agent，不限于本 AgentExecutor 创建的。使用 AgentExecutor 需要在 config.toml 中开启多 agent，见 [开启多 agent](../02-configuration.md#开启多-agent)。

以下示例中的 handle 是 spawn_agent 返回的句柄：

~~~js
import { AgentExecutor, text } from "@brainary/poa-sdk";

const agentExecutor = new AgentExecutor();
const handle = await agentExecutor.spawn_agent({ message: "检查 README", nickname: "reviewer" });
~~~

## 类型

### AgentStatus

**作用**：agent 的状态。

**类型定义**：

~~~ts
type AgentStatus =
  | "pending_init"
  | "running"
  | "interrupted"
  | "completed"
  | "errored"
  | "shutdown"
  | "not_found";
~~~

| 状态 | 含义 |
| --- | --- |
| pending_init | 已创建，尚未开始第一轮 |
| running | 正在运行一轮任务 |
| interrupted | 上一轮被打断 |
| completed | 上一轮已完成 |
| errored | 上一轮出错 |
| shutdown | 已停止 |
| not_found | 不存在或已关闭 |

处于 pending_init、interrupted、completed、errored 时，可以用 [Agent.run](01-agent.md#runmessage-message_items-timeout_ms) 开始新一轮。
completed 的最终回复和 errored 的错误信息用 [acquire_agent_result](#acquire_agent_resultagent_id-timeout_ms) 获取。

**示例**：

~~~js
if ((await agentExecutor.get_status(handle.agent_id)) === "completed") {
  text("finished");
}
~~~

### Handle

**作用**：spawn_agent 和 append_message 返回的句柄。

**类型定义**：

~~~ts
interface Handle {
  agent_id: string;
  agent_name: string;
  meta?: Record<string, unknown>;
  agent_status: AgentStatus;
}
~~~

| 字段 | 说明 |
| --- | --- |
| agent_id | agent 的 ID，传给 acquire_agent_result、get_status、append_message、interrupt 和 close |
| agent_name | spawn_agent：agent 的 nickname，即传入的 nickname 或 Brainary 分配的昵称；AgentExecutor.append_message：agent_id；[Agent.append_message](01-agent.md#append_messagemessage-message_items-immediately)：agent 的 nickname，没有时为 role |
| meta | Brainary 返回的信息，包含 thread_id（即 agent_id）、path（agent 的路径，如 `/root/default_agent_1`）和 status；spawn_agent 还包含 nickname 和 role |
| agent_status | 调用时 agent 的状态；之后的状态用 [get_status](#get_statusagent_id) 获取 |

**示例**：

~~~js
text({ id: handle.agent_id, name: handle.agent_name, path: handle.meta?.path, status: handle.agent_status });
~~~

### SpawnSpec

**作用**：spawn_agent 的参数。

**类型定义**：

~~~ts
type SpawnSpec = {
  nickname?: string;
  ephemeral?: boolean;
  // 以下字段尚未实现
  description?: string;
  custom_config?: AgentConfig;
} & (
  | { message: string; message_items?: readonly MessageItem[] }
  | { message?: string; message_items: readonly MessageItem[] }
);
~~~

| 字段 | 说明 |
| --- | --- |
| message | 任务的文本消息 |
| message_items | 任务的消息项，见 [MessageItem](04-message-item.md)。同时传入 message 时，message 作为第一个文本项排在消息项之前 |
| nickname | agent 的昵称，原样使用，不能与打开的 agent 重名，最长 64 字节；未传入时由 Brainary 分配 |
| ephemeral | 为 true 时不保存这个 agent 的会话记录；默认为 false，即使本次运行不保存会话记录，派出的 agent 也会保存，见 [会话记录](../02-configuration.md#会话记录) |
| description、custom_config | 尚未实现 |

message 和 message_items 至少传入一项。

**示例**：

~~~js
const spec = {
  message: "对照截图检查页面布局",
  message_items: [{ type: "image", image_url: "data:image/png;base64,iVBORw0KGgo...", detail: "original" }],
  nickname: "layout_checker",
  ephemeral: true,
};
~~~

### CreateAgentSpec

**作用**：create_agent 的参数。

**类型定义**：

~~~ts
interface CreateAgentSpec {
  role: string;
  nickname?: string;
  description?: string;
  agent_config: AgentConfig;
}
~~~

| 字段 | 说明 |
| --- | --- |
| role | 必填，agent 的角色，只能使用小写字母、数字和 `_`，不能为 "root"，最长 64 字节。多个 agent 可以使用同一个 role |
| nickname | agent 的昵称，规则与 [SpawnSpec](#spawnspec).nickname 相同 |
| description | agent 的说明，只保存在返回的 Agent 对象上 |
| agent_config | 必填，见 [AgentConfig](02-agent-config.md) |

### InterruptResult

**作用**：interrupt 的结果。

**类型定义**：

~~~ts
interface InterruptResult {
  agent_id: string;
  interrupted: boolean;
  agent_status: AgentStatus;
}
~~~

| 字段 | 说明 |
| --- | --- |
| agent_id | agent 的 ID |
| interrupted | 是否打断了正在运行的一轮；agent 本来没有在运行时为 false |
| agent_status | 打断后的状态：打断了一轮时为 interrupted，否则为原来的状态 |

### RemoteAgent

**状态**：尚未实现，以下为设计描述。

**作用**：描述可通过远程协议调用的 agent。

**类型定义**：

~~~ts
interface RemoteAgent {
  agent_id: string;
  name: string;
  description: string;
  agent_card: unknown;
}
~~~

**示例**：

~~~js
const remote = agentExecutor.create_remote_agent("researcher", "远程检索 agent", "./researcher_agent_card.json");
text(remote.name);
~~~

### TokenUsage

**状态**：尚未实现，以下为设计描述。

**作用**：记录 agent 的 token 消耗。

**类型定义**：

~~~ts
interface TokenUsage {
  agent_id: string;
  input_tokens: number;
  cached_input_tokens: number;
  output_tokens: number;
  reasoning_tokens: number;
  total_tokens: number;
}
~~~

**示例**：

~~~js
const usage = await agentExecutor.get_token_usage(handle.agent_id);
text(usage.total_tokens);
~~~

### ContextUsage

**状态**：尚未实现，以下为设计描述。

**作用**：记录 agent 上下文窗口的使用情况。

**类型定义**：

~~~ts
interface ContextUsage {
  agent_id: string;
  used_tokens: number;
  window_tokens: number;
  remaining_tokens: number;
  used_ratio: number;
}
~~~

**示例**：

~~~js
const usage = await agentExecutor.get_context_usage(handle.agent_id);
text(usage.remaining_tokens);
~~~

## 方法

### create_agent({role, nickname?, description?, agent_config})

**作用**：按配置创建一个 agent。创建后状态为 pending_init，用 [Agent.run](01-agent.md#runmessage-message_items-timeout_ms) 开始任务。agent 的路径为 `/root/<role>_<n>`，n 从 1 开始依次编号；agent 可以使用的工具见 [agent 的工具](02-agent-config.md#agent-的工具)。

**输入类型**：spec: [CreateAgentSpec](#createagentspec)。

**返回值类型**：Promise<[Agent](01-agent.md)>。

**异常**：

- role 为空白：抛出 TypeError。
- Brainary 拒绝时抛出 [MultiAgentError](01-agent.md#multiagenterror)，agent 不会创建。例如：
  - role 不合法、nickname 为空白或过长：INVALID_ARGUMENT；
  - nickname 与打开的 agent 重名：AGENT_NICKNAME_CONFLICT；
  - 模型或推理强度不可用：UNKNOWN_MODEL、UNSUPPORTED_REASONING_EFFORT；
  - 新 agent 的层级超过 [max_depth](../02-configuration.md#agent-层级和数量)：AGENT_DEPTH_LIMIT；
  - 打开的 agent 数已达到 max_concurrent_threads_per_session：AGENT_LIMIT_REACHED。

**示例**：

~~~js
const reviewer = await agentExecutor.create_agent({
  role: "reviewer",
  description: "审查代码",
  agent_config: {
    model: { model_id: "gpt-5.6-luna", reasoning_effort: "high" },
    developer_instructions: "只报告可复现的问题",
  },
});
const result = await reviewer.run("检查 src/ 中的错误处理", undefined, 300_000);
~~~

### spawn_agent(spawn_spec)

**作用**：派出一个 agent 执行任务，立即返回句柄，不等待任务完成。
派出的 agent 的模型、推理强度、开发者指令、provider、沙箱、审批策略和工作目录都与当前程序相同，对话从空白开始。路径为 `/root/default_agent_<n>`。可以使用的工具与 create_agent 得到的 agent 相同，见 [agent 的工具](02-agent-config.md#agent-的工具)。

**输入类型**：spawn_spec: [SpawnSpec](#spawnspec)。

**返回值类型**：Promise<[Handle](#handle)>。

**异常**：

- message 和 message_items 都未传入，或 message 为空白字符串：抛出 TypeError。
- Brainary 拒绝时抛出 [MultiAgentError](01-agent.md#multiagenterror)，例如 nickname 重名（AGENT_NICKNAME_CONFLICT）、层级超过 max_depth（AGENT_DEPTH_LIMIT）、打开的 agent 数已达到 max_concurrent_threads_per_session（AGENT_LIMIT_REACHED）。

**示例**：

~~~js
const review = await agentExecutor.spawn_agent({
  message: "分析 src/ 目录",
  nickname: "reviewer",
});
~~~

### acquire_agent_result(agent_id, timeout_ms?)

**作用**：等待指定 agent 的当前一轮结束并返回结果。agent 的这一轮已经结束，或已关闭时立即返回。

**输入类型**：

- agent_id: string，必填。
- timeout_ms?: number，最长等待时间，单位毫秒。省略时使用 config.toml 中的 [默认等待时间](../02-configuration.md#等待时间)，默认 30000。有效范围默认为 10000–3600000，超出范围按边界值处理。

**返回值类型**：Promise<[AgentResult](01-agent.md#agentresult)>。不存在或已关闭的 agent 返回 agent_status 为 not_found 的结果。

**异常**：

- timeout_ms 不是正整数：抛出 RangeError。
- 超时：抛出 [AgentResultTimeoutError](01-agent.md#agentresulttimeouterror)。agent 继续运行，可以再次调用本方法。
- Brainary 拒绝本次调用：抛出 [MultiAgentError](01-agent.md#multiagenterror)。

用 create_agent 创建后还没有运行过的 agent（pending_init）不会结束，本方法会等到超时。

**示例**：

~~~js
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 120_000);
text({ status: result.agent_status, answer: result.result, error: result.error });
~~~

### get_status(agent_id)

**作用**：返回指定 agent 的当前状态，不等待。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>。不存在或已关闭的 agent 返回 "not_found"。

**示例**：

~~~js
const status = await agentExecutor.get_status(handle.agent_id);
if (status === "completed" || status === "errored") {
  const result = await agentExecutor.acquire_agent_result(handle.agent_id);
  text(result.result ?? result.error);
}
~~~

### append_message(agent_id, message?, message_items?, immediately?)

**作用**：向指定 agent 追加消息，不等待处理结果。[Agent.append_message](01-agent.md#append_messagemessage-message_items-immediately) 是作用于该 agent 自身的同一操作。

**输入类型**：

- agent_id: string，必填。
- message?: string，追加的文本消息。
- message_items?: readonly [MessageItem](04-message-item.md)[]，追加的消息项。同时传入 message 时，message 作为第一个文本项排在消息项之前。
- immediately?: boolean，默认为 false：
  - false：正在运行的一轮会读到这条消息；agent 空闲时，消息留到下一轮。
  - true：打断正在运行的一轮，立即开始新的一轮处理这条消息，用 acquire_agent_result 获取这一轮的结果。

message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](#handle)>，agent_status 为消息送达后 agent 的状态。

**异常**：

- message 和 message_items 都未传入，或 message 为空白字符串：抛出 TypeError。
- agent 不存在或已关闭：抛出 [MultiAgentError](01-agent.md#multiagenterror)，code 为 THREAD_NOT_FOUND。

**示例**：

~~~js
await agentExecutor.append_message(handle.agent_id, "补充一条风险", undefined, true);
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 60_000);
text(result.result);
~~~

### interrupt(agent_id)

**作用**：打断指定 agent 正在运行的一轮，等这一轮停止后返回。agent 没有在运行时不做任何操作。
打断后，之后用 acquire_agent_result 得到 agent_status 为 interrupted 的结果；用 append_message 默认方式留到下一轮的消息仍然保留。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[InterruptResult](#interruptresult)>。

**异常**：agent 不存在或已关闭时抛出 [MultiAgentError](01-agent.md#multiagenterror)，code 为 THREAD_NOT_FOUND。

**示例**：

~~~js
const stopped = await agentExecutor.interrupt(handle.agent_id);
text({ interrupted: stopped.interrupted, status: stopped.agent_status });
~~~

### close(agent_id)

**作用**：按 ID 关闭 agent，以及它派出的、仍处于打开状态的各层子 agent。spawn_agent 和 create_agent 得到的 agent 都可以关闭。
关闭后 get_status 返回 "not_found"，并释放 max_concurrent_threads_per_session 的名额。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>，关闭前的状态。

**异常**：agent 不存在或已关闭时抛出 [MultiAgentError](01-agent.md#multiagenterror)，code 为 THREAD_NOT_FOUND。

**示例**：

~~~js
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 300_000);
text(result.result);
text(await agentExecutor.close(handle.agent_id)); // "completed"
~~~

### resume(agent_id)

**状态**：尚未实现，以下为设计描述。

**作用**：按 ID 恢复已关闭的 agent，与 [Agent.resume()](01-agent.md#resume) 相同。恢复后的 agent 保留关闭前的对话记录，使用创建时的配置。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>，恢复后的状态。

**示例**：

~~~js
await agentExecutor.close(handle.agent_id);
text(await agentExecutor.resume(handle.agent_id));
~~~

### create_remote_agent(name, description, agent_card)

**状态**：尚未实现，以下为设计描述。

**作用**：注册一个由远程 agent 服务提供的 agent 描述。

**输入类型**：name: string；description: string；agent_card: unknown，远程 agent 的卡片描述。三个参数均必填。

**返回值类型**：[RemoteAgent](#remoteagent)。

**示例**：

~~~js
const analyst = agentExecutor.create_remote_agent("analyst", "调用远程分析服务", "./remote_agent_card.json");
~~~

### send_message(agent_id, message?, message_items?, output_schema?)

**状态**：尚未实现，以下为设计描述。

**作用**：向指定 agent 发送消息，可附带消息项和输出 schema。

**输入类型**：agent_id: string；message?: string；message_items?: [MessageItem](04-message-item.md)[]；output_schema?: JsonSchema。message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](#handle)>。

**示例**：

~~~js
const sent = await agentExecutor.send_message(handle.agent_id, "分析这个文件", undefined, { type: "object" });
~~~

### context_compact(agent_id, compact_template?)

**状态**：尚未实现，以下为设计描述。

**作用**：压缩指定 agent 的上下文，并按模板保留关键信息。

**输入类型**：agent_id: string；compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agentExecutor.context_compact(handle.agent_id, "保留结论、待办和文件路径");
~~~

### get_context_usage(agent_id)

**状态**：尚未实现，以下为设计描述。

**作用**：查询指定 agent 的上下文用量。

**输入类型**：agent_id: string。

**返回值类型**：Promise<[ContextUsage](#contextusage)>。

**示例**：

~~~js
text(await agentExecutor.get_context_usage(handle.agent_id));
~~~

### get_token_usage(agent_id)

**状态**：尚未实现，以下为设计描述。

**作用**：查询指定 agent 的 token 用量。

**输入类型**：agent_id: string。

**返回值类型**：Promise<[TokenUsage](#tokenusage)>。

**示例**：

~~~js
text(await agentExecutor.get_token_usage(handle.agent_id));
~~~
