# AgentExecutor

[AgentExecutor](03-agent-executor.md) 是 kind 为 "agent" 的 [Executor](05-executors.md#executor)，负责 Agent 的创建、派发、结果获取、
状态查询、关闭和恢复。create_agent 按 [AgentConfig](02-agent-config.md) 创建可多次运行的 [Agent](01-agent.md)；
spawn_agent 派出子 Agent，再用 acquire_agent_result 获取结果。

以下示例假定已导入 SDK，并用 agentExecutor 指代这个执行器，handle 是 spawn_agent 返回的句柄：

~~~js
import { AgentExecutor, ApprovalPolicy, BrainaryTool, Sandbox, text } from "@brainary/poa-sdk";

const agentExecutor = new AgentExecutor();
const handle = await agentExecutor.spawn_agent({ message: "检查 README", agent_name: "reviewer" });
~~~

## 类型

### AgentStatus

**作用**：表示 Agent 当前生命周期状态。

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

首轮开始前为 pending_init，运行中为 running；completed 和 errored 表示上一轮已完成或出错，interrupted 表示被中断，
这三种状态下 Agent 都可以继续接收输入；shutdown 和 not_found 表示 Agent 已关闭或不存在。
completed 的最终回复和 errored 的错误信息由 acquire_agent_result 返回。"idle" 尚未实现。

**示例**：

~~~js
if ((await agentExecutor.get_status(handle.agent_id)) === "completed") {
  text("finished");
}
~~~

### Handle

**作用**：表示一次已派发的 Agent 任务，包含 Agent 身份、宿主创建响应和初始状态。

**类型定义**：

~~~ts
interface Handle {
  agent_id: string;
  agent_name: string;
  meta?: Record<string, unknown>;
  agent_status: AgentStatus;
}
~~~

agent_name 为派发时指定的名称，省略时由宿主分配。meta 为宿主返回的派发信息。agent_status 为初始状态 pending_init，
后续状态用 [get_status](#get_statusagent_id) 或 [acquire_agent_result](#acquire_agent_resultagent_id-timeout_ms) 获取。

**示例**：

~~~js
text({ id: handle.agent_id, name: handle.agent_name, status: handle.agent_status });
~~~

### SpawnSpec

**作用**：描述一次派发的消息和显示名。

**类型定义**：

~~~ts
type SpawnSpec = {
  agent_name?: string;
} & (
  | { message: string; message_items?: readonly MessageItem[] }
  | { message?: string; message_items: readonly MessageItem[] }
);
~~~

message 和 message_items 至少传入一项；同时传入时 message 排在消息项之前。message_items 的元素类型为 [MessageItem](04-message-item.md)。

设计中 SpawnSpec 还包括 description?: string 和 custom_config?: [AgentConfig](02-agent-config.md)，当前 SDK 尚未实现。

**示例**：

~~~js
const spec = {
  message: "分析 src/ 目录",
  message_items: [{ type: "mention", name: "README", path: "/workspace/README.md" }],
  agent_name: "reviewer",
};
~~~

### RemoteAgent

**状态**：尚未实现，以下为设计描述。

**作用**：描述可通过远程协议调用的 Agent。

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
const remote = agentExecutor.create_remote_agent("researcher", "远程检索 Agent", "./researcher_agent_card.json");
text(remote.name);
~~~

### TokenUsage

**状态**：尚未实现，以下为设计描述。

**作用**：记录 Agent 的 token 消耗。

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

**作用**：记录 Agent 上下文窗口的使用情况。

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

**作用**：按配置创建一个 Agent，随后用 [Agent.run](01-agent.md#runmessage-items-output_schema-timeout_ms) 让它开始任务。

**输入类型**：CreateAgentSpec 命名参数对象，包含 name: string；description?: string；agent_config: [AgentConfig](02-agent-config.md)。

name 和 agent_config 必填。配置校验失败时抛出异常，规则见 [AgentConfig](02-agent-config.md)。

**返回值类型**：Promise<[Agent](01-agent.md)>。

**示例**：

~~~js
const reviewer = await agentExecutor.create_agent({
  name: "reviewer",
  description: "审查代码",
  agent_config: {
    model: {
      model_id: "gpt-5.6-luna",
      provider: "openai",
      reasoning_effort: "high",
    },
    system_prompt: "只报告可复现的问题",
    capabilities: { brainary_tool: [BrainaryTool.exec_command, BrainaryTool.write_stdin] },
    runtime: { sandbox: Sandbox.read_only, approval_policy: ApprovalPolicy.deny_all },
  },
});
~~~

### spawn_agent(spawn_spec)

**作用**：派出一个子 Agent，并返回用于获取结果的句柄。

**输入类型**：spawn_spec: [SpawnSpec](#spawnspec)，必填。

**返回值类型**：Promise<[Handle](#handle)>。

**示例**：

~~~js
const review = await agentExecutor.spawn_agent({
  message: "分析 src/ 目录",
  agent_name: "reviewer",
});
~~~

### acquire_agent_result(agent_id, timeout_ms?)

**作用**：等待指定 Agent 进入终态并返回结果。

**输入类型**：agent_id: string，必填；timeout_ms?: number，等待结果的超时时间，单位为毫秒（ms），默认 30000。

timeout_ms 的有效范围为 10000–3600000。Agent 结束时返回结果；超时抛出 [AgentResultTimeoutError](01-agent.md#agentresulttimeouterror)，
Agent 继续运行，可以再次调用获取结果。

**返回值类型**：Promise<[AgentResult](01-agent.md#agentresult)>。

**示例**：

~~~js
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 120000);
text({ status: result.agent_status, answer: result.result, error: result.error });
~~~

### get_status(agent_id)

**作用**：立即返回指定 Agent 的当前状态。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>。

状态为 completed 或 errored 时，可用 [acquire_agent_result](#acquire_agent_resultagent_id-timeout_ms) 取回最终回复或错误信息。

**示例**：

~~~js
const status = await agentExecutor.get_status(handle.agent_id);
if (status === "completed" || status === "errored") {
  const result = await agentExecutor.acquire_agent_result(handle.agent_id);
  text(result.result ?? result.error);
}
~~~

### close(agent_id)

**作用**：关闭指定 Agent 及其子 Agent，并返回关闭前的状态。关闭后可用 resume 恢复。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>。

**示例**：

~~~js
const previous = await agentExecutor.close(handle.agent_id);
text(previous);
~~~

### resume(agent_id)

**作用**：恢复已关闭的 Agent，并返回恢复后的状态。Agent 按创建时的配置恢复。

**输入类型**：agent_id: string，必填。

**返回值类型**：Promise<[AgentStatus](#agentstatus)>。

与 [Agent.resume()](01-agent.md#resume) 相同，按 ID 调用。

**示例**：

~~~js
await agentExecutor.close(handle.agent_id);
text(await agentExecutor.resume(handle.agent_id));
~~~

### create_remote_agent(name, description, agent_card)

**状态**：尚未实现，以下为设计描述。

**作用**：注册一个由远程 Agent 服务提供的 Agent 描述。

**输入类型**：name: string；description: string；agent_card: unknown，远程 Agent 的卡片描述。三个参数均必填。

**返回值类型**：[RemoteAgent](#remoteagent)。

**示例**：

~~~js
const analyst = agentExecutor.create_remote_agent("analyst", "调用远程分析服务", "./remote_agent_card.json");
~~~

### send_message(agent_id, message?, message_items?, output_schema?)

**状态**：尚未实现，以下为设计描述。

**作用**：向指定 Agent 发送消息，可附带消息项和输出 schema。

**输入类型**：agent_id: string；message?: string；message_items?: [MessageItem](04-message-item.md)[]；output_schema?: JsonSchema。

**参数约束**：message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](#handle)>。

**示例**：

~~~js
const sent = await agentExecutor.send_message(handle.agent_id, "分析这个文件", undefined, { type: "object" });
~~~

### context_compact(agent_id, compact_template?)

**状态**：尚未实现，以下为设计描述。

**作用**：压缩目标 Agent 的上下文，并按模板保留关键信息。

**输入类型**：agent_id: string；compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agentExecutor.context_compact(handle.agent_id, "保留结论、待办和文件路径");
~~~

### append_message(agent_id, message?, message_items?, immediately?)

**状态**：尚未实现，以下为设计描述。

**作用**：向指定 Agent 追加消息，并返回任务句柄；immediately 用于指定是否立即处理。
[Agent.append_message](01-agent.md#append_messagemessage-message_items-immediately) 是作用于该 Agent 自身的同一操作。

**输入类型**：agent_id: string；message?: string；message_items?: [MessageItem](04-message-item.md)[]；immediately?: boolean，默认为 false。

**参数约束**：message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](#handle)>。

**示例**：

~~~js
const added = await agentExecutor.append_message(handle.agent_id, "补充一条风险", undefined, true);
const result = await agentExecutor.acquire_agent_result(added.agent_id, 60000);
text(result.result);
~~~

### get_context_usage(agent_id)

**状态**：尚未实现，以下为设计描述。

**作用**：查询指定 Agent 的上下文用量。

**输入类型**：agent_id: string。

**返回值类型**：Promise<[ContextUsage](#contextusage)>。

**示例**：

~~~js
text(await agentExecutor.get_context_usage(handle.agent_id));
~~~

### get_token_usage(agent_id)

**状态**：尚未实现，以下为设计描述。

**作用**：查询指定 Agent 的 token 用量。

**输入类型**：agent_id: string。

**返回值类型**：Promise<[TokenUsage](#tokenusage)>。

**示例**：

~~~js
text(await agentExecutor.get_token_usage(handle.agent_id));
~~~

## tools

**作用**：列出 AgentExecutor 封装的宿主工具。tools 是静态属性，值为 [BrainaryTool](07-brainary-tool.md) 成员。

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

send_input 和 wait_agent 由 Agent.run 和 acquire_agent_result 使用。

**示例**：

~~~js
text(AgentExecutor.tools.close_agent); // "multi_agent_v1__close_agent"
~~~
