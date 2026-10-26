# AgentExecutor

[AgentExecutor](04-agent-executor.md) 提供 Agent 消息发送、上下文压缩、关闭和状态查询。
状态与用量类型同时供 [Agent](03-agent.md) 的实例方法使用。

以下示例用 agentExecutor 指代这个执行器。

### AgentState

**作用**：表示 Agent 当前生命周期状态。

**类型定义**：

~~~ts
type AgentState =
  | "PendingInit"
  | "Running"
  | "Interrupted"
  | "Completed"
  | "Errored"
  | "Shutdown"
  | "NotFound";
~~~

**示例**：

~~~js
if (agentExecutor.get_state({ agent_id: agent.agent_id }) === "Completed") {
  text("finished");
}
~~~

### TokenUsage

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
const usage = agentExecutor.get_token_usage({ agent_id: agent.agent_id });
text(usage.total_tokens);
~~~

### ContextUsage

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
const usage = agentExecutor.get_context_usage({ agent_id: agent.agent_id });
text(usage.remaining_tokens);
~~~

### send_message({agent_id, message?, message_items?, output_schema?})

**作用**：向指定 Agent 发送消息，可附带消息项和输出 schema。

**输入类型**：命名参数对象，包含 agent_id: string；message?: string；message_items?: [MessageItem](05-message-item.md)[]；output_schema?: JsonSchema。

**参数约束**：message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](01-agent-orchestration.md#handle)>。

**示例**：

~~~js
const handle = await agentExecutor.send_message({
  agent_id: agent.agent_id,
  message: "分析这个文件",
  output_schema: { type: "object" },
});
~~~

### context_compact({agent_id, compact_template?})

**作用**：压缩目标 Agent 的上下文，并按模板保留关键信息。

**输入类型**：命名参数对象，包含 agent_id: string；compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agentExecutor.context_compact({ agent_id: agent.agent_id, compact_template: "保留结论、待办和文件路径" });
~~~

### close({agent_id})

**作用**：关闭指定 Agent 并释放运行资源，返回 Agent 状态。

**输入类型**：命名参数对象，包含 agent_id: string。

**返回值类型**：Promise<[AgentState](#agentstate)>。

**示例**：

~~~js
await agentExecutor.close({ agent_id: agent.agent_id });
~~~

### addition_message({agent_id, message?, message_items?, immediately})

**作用**：追加 Agent 消息，并返回任务句柄；immediately 用于指定是否立即处理。

**输入类型**：命名参数对象，包含 agent_id: string，必填；message?: string；message_items?: [MessageItem](05-message-item.md)[]；immediately: boolean，必填，默认取值为 false。

**参数约束**：message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](01-agent-orchestration.md#handle)>。

**示例**：

~~~js
const handle = await agentExecutor.addition_message({
  agent_id: agent.agent_id,
  message: "补充一条风险",
  immediately: true,
});
const result = await orchestration.acquire_agent_result({ agent_id: handle.agent_id, timeoutMs: 60000 });
text(result.result);
~~~

### get_context_usage({agent_id})

**作用**：查询指定 Agent 的上下文用量。

**输入类型**：命名参数对象，包含 agent_id: string。

**返回值类型**：[ContextUsage](04-agent-executor.md)。

**示例**：

~~~js
text(agentExecutor.get_context_usage({ agent_id: agent.agent_id }));
~~~

### get_token_usage({agent_id})

**作用**：查询指定 Agent 的 token 用量。

**输入类型**：命名参数对象，包含 agent_id: string。

**返回值类型**：[TokenUsage](04-agent-executor.md)。

**示例**：

~~~js
text(agentExecutor.get_token_usage({ agent_id: agent.agent_id }));
~~~

### get_state({agent_id})

**作用**：查询指定 Agent 的生命周期状态。

**输入类型**：命名参数对象，包含 agent_id: string。

**返回值类型**：[AgentState](04-agent-executor.md)。

**示例**：

~~~js
text(agentExecutor.get_state({ agent_id: agent.agent_id }));
~~~

### tools

**作用**：枚举 AgentExecutor 提供的工具，每个枚举成员的值都是字符串。

**类型定义**：Readonly<Record<string, [Tool](13-tools-manager.md#tools)>>。

**示例**：

~~~js
text(agentExecutor.tools.send_message);
~~~
