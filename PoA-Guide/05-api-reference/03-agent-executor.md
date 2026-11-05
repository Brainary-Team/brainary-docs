# AgentExecutor

AgentExecutor 创建、派发、等待、查询、关闭和恢复 agent，kind 为 "agent"。

- [create_agent](#create_agentname-description-agent_config)：按 [AgentConfig](02-agent-config.md) 创建 agent，返回可以多次运行的 [Agent](01-agent.md)。
- [spawn_agent](#spawn_agentspawn_spec)：沿用当前程序的配置派出 agent 并立即开始任务，再用 [acquire_agent_result](#acquire_agent_resultagent_id-timeout_ms) 获取结果。

以下示例中的 handle 是 spawn_agent 返回的句柄：

~~~js
import { AgentExecutor, ApprovalPolicy, BrainaryTool, Sandbox, text } from "@brainary/poa-sdk";

const agentExecutor = new AgentExecutor();
const handle = await agentExecutor.spawn_agent({ message: "检查 README", agent_name: "reviewer" });
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
// 尚未实现："idle"
~~~

| 状态 | 含义 |
| --- | --- |
| pending_init | 已创建，尚未开始第一轮 |
| running | 正在运行一轮任务 |
| interrupted | 当前一轮被中断 |
| completed | 上一轮已完成 |
| errored | 上一轮出错 |
| shutdown | 已停止 |
| not_found | 不存在或已关闭 |

处于 pending_init、interrupted、completed、errored 时，可以用 [Agent.run](01-agent.md#runmessage-items-output_schema-timeout_ms) 开始新一轮。
completed 的最终回复和 errored 的错误信息用 [acquire_agent_result](#acquire_agent_resultagent_id-timeout_ms) 获取。"idle" 尚未实现。

**示例**：

~~~js
if ((await agentExecutor.get_status(handle.agent_id)) === "completed") {
  text("finished");
}
~~~

### Handle

**作用**：spawn_agent 返回的句柄。

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
| agent_id | agent 的 ID，传给 acquire_agent_result、get_status、close 和 resume |
| agent_name | 派发时传入的 agent_name；未传入时为 Brainary 分配的昵称，没有昵称时为 agent_id |
| meta | Brainary 返回的派发信息，包含 agent_id 和 nickname |
| agent_status | 固定为 "pending_init"；之后的状态用 [get_status](#get_statusagent_id) 获取 |

**示例**：

~~~js
text({ id: handle.agent_id, name: handle.agent_name, status: handle.agent_status });
~~~

### SpawnSpec

**作用**：spawn_agent 的参数。

**类型定义**：

~~~ts
type SpawnSpec = {
  agent_name?: string;
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
| agent_name | 写入 Handle.agent_name 的名称，只保存在 Handle 中 |
| description、custom_config | 尚未实现 |

message 和 message_items 至少传入一项。

**示例**：

~~~js
const spec = {
  message: "对照截图检查页面布局",
  message_items: [{ type: "local_image", path: "/workspace/screenshot.png" }],
  agent_name: "layout-checker",
};
~~~

### CreateAgentSpec

**作用**：create_agent 的参数。

**类型定义**：

~~~ts
interface CreateAgentSpec {
  name: string;
  description?: string;
  agent_config: AgentConfig;
}
~~~

| 字段 | 说明 |
| --- | --- |
| name | 必填，agent 的名称，不能为空白 |
| description | agent 的说明，只保存在返回的 Agent 对象上 |
| agent_config | 必填，见 [AgentConfig](02-agent-config.md) |

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

### create_agent({name, description?, agent_config})

**作用**：按配置创建一个 agent。创建后状态为 pending_init，用 [Agent.run](01-agent.md#runmessage-items-output_schema-timeout_ms) 开始任务。

**输入类型**：spec: [CreateAgentSpec](#createagentspec)。

**返回值类型**：Promise<[Agent](01-agent.md)>。

**异常**：以下情况抛出 Error，agent 不会创建：

- name 为空白，或 agent_config 不满足 [AgentConfig](02-agent-config.md) 中的约束。
- 新 agent 的层级超过 [max_depth](../02-configuration.md#agent-层级和数量)。
- 打开的 agent 数已达到 max_concurrent_threads_per_session。

**示例**：

~~~js
const reviewer = await agentExecutor.create_agent({
  name: "reviewer",
  description: "审查代码",
  agent_config: {
    model: { model_id: "gpt-5.6-luna", provider: "vendor", reasoning_effort: "high" },
    system_prompt: "只报告可复现的问题",
    capabilities: { brainary_tool: [BrainaryTool.exec_command, BrainaryTool.write_stdin] },
    runtime: { sandbox: Sandbox.read_only, approval_policy: ApprovalPolicy.deny_all },
  },
});
const result = await reviewer.run("检查 src/ 中的错误处理", undefined, undefined, 300_000);
~~~

### spawn_agent(spawn_spec)

**作用**：派出一个 agent 执行任务，立即返回句柄，不等待任务完成。
派出的 agent 的 provider、沙箱、审批策略和工作目录与当前程序相同；模型为 [default_subagent_model](../02-configuration.md#agent-层级和数量)，未设置时与当前程序相同。

**输入类型**：spawn_spec: [SpawnSpec](#spawnspec)。

**返回值类型**：Promise<[Handle](#handle)>。

**异常**：

- message 和 message_items 都未传入，或 message 为空白字符串：抛出 TypeError。
- 新 agent 的层级超过 max_depth，或打开的 agent 数已达到 max_concurrent_threads_per_session：抛出 Error。

**示例**：

~~~js
const review = await agentExecutor.spawn_agent({
  message: "分析 src/ 目录",
  agent_name: "reviewer",
});
~~~

### acquire_agent_result(agent_id, timeout_ms?)

**作用**：等待指定 agent 的当前一轮结束并返回结果。agent 的状态已是 completed、errored、shutdown 或 not_found 时立即返回。

**输入类型**：

- agent_id: string，必填。
- timeout_ms?: number，最长等待时间，单位毫秒，默认 30000。有效范围为 10000–3600000：小于 10000 按 10000 处理，大于 3600000 按 3600000 处理。

**返回值类型**：Promise<[AgentResult](01-agent.md#agentresult)>。

**异常**：

- timeout_ms 不是正整数：抛出 RangeError。
- 超时：抛出 [AgentResultTimeoutError](01-agent.md#agentresulttimeouterror)。agent 继续运行，可以再次调用本方法。
- agent_id 格式无效：抛出 Error。

**示例**：

~~~js
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 120_000);
text({ status: result.agent_status, answer: result.result, error: result.error });
~~~

### get_status(agent_id)

**作用**：返回指定 agent 的当前状态，不等待。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>。不存在或已关闭的 agent 返回 "not_found"。

**异常**：agent_id 格式无效时抛出 Error。

**示例**：

~~~js
const status = await agentExecutor.get_status(handle.agent_id);
if (status === "completed" || status === "errored") {
  const result = await agentExecutor.acquire_agent_result(handle.agent_id);
  text(result.result ?? result.error);
}
~~~

### close(agent_id)

**作用**：按 ID 关闭 agent，以及它派出的、仍处于打开状态的各层子 agent。spawn_agent 和 create_agent 得到的 agent 都可以关闭。
关闭后 get_status 返回 "not_found"，并释放 max_concurrent_threads_per_session 的名额。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>，关闭前的状态。

**异常**：agent_id 格式无效，或 agent 已关闭时抛出 Error。

**示例**：

~~~js
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 300_000);
text(result.result);
text(await agentExecutor.close(handle.agent_id)); // "completed"
~~~

### resume(agent_id)

**作用**：按 ID 恢复已关闭的 agent，与 [Agent.resume()](01-agent.md#resume) 相同。恢复后的 agent 保留关闭前的对话记录，使用创建时的配置。agent 未关闭时不做任何操作。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>，恢复后的状态。

**异常**：agent_id 格式无效，或打开的 agent 数已达到 max_concurrent_threads_per_session 时抛出 Error。

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

### append_message(agent_id, message?, message_items?, immediately?)

**状态**：尚未实现，以下为设计描述。

**作用**：向指定 agent 追加消息，并返回任务句柄；immediately 指定是否立即处理。
[Agent.append_message](01-agent.md#append_messagemessage-message_items-immediately) 是作用于该 agent 自身的同一操作。

**输入类型**：agent_id: string；message?: string；message_items?: [MessageItem](04-message-item.md)[]；immediately?: boolean，默认为 false。message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](#handle)>。

**示例**：

~~~js
const added = await agentExecutor.append_message(handle.agent_id, "补充一条风险", undefined, true);
const result = await agentExecutor.acquire_agent_result(added.agent_id, 60_000);
text(result.result);
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

## tools

**作用**：AgentExecutor 使用的 [BrainaryTool](06-brainary-tool.md) 成员，静态只读属性，可以直接放入 [Capabilities](02-agent-config.md#capabilities).brainary_tool。

**类型定义**：

~~~ts
class AgentExecutor {
  static readonly tools: {
    readonly spawn_agent: BrainaryTool.spawn_agent;
    readonly create_agent: BrainaryTool.create_agent;
    readonly send_input: BrainaryTool.send_input;
    readonly wait_agent: BrainaryTool.wait_agent;
    readonly resume_agent: BrainaryTool.resume_agent;
    readonly close_agent: BrainaryTool.close_agent;
    readonly get_agent_status: BrainaryTool.get_agent_status;
  };
}
~~~

**示例**：

~~~js
const brainary_tool = Object.values(AgentExecutor.tools);
~~~
