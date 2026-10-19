# AgentMonitor

[AgentMonitor](04-agent-monitor.md) 提供 [Agent](03agent.md) 的状态、token 用量和上下文用量快照。

### AgentState

**作用**：表示 Agent 当前生命周期状态。

**输入类型**：枚举值 "PendingInit" | "Running" | "Interrupted" | "Completed" |
"Errored" | "Shutdown" | "NotFound"。

**返回值类型**：状态值。

**示例**：

~~~js
if (agentMonitor.get_state(agent.agent_id) === "Completed") {
  text("finished");
}
~~~

### TokenUsage

**作用**：记录 Agent 的 token 消耗。

**输入类型**：

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

**返回值类型**：[TokenUsage](04-agent-monitor.md) 快照。

**示例**：

~~~js
const usage = agentMonitor.get_token_usage(agent.agent_id);
text(usage.total_tokens);
~~~

### ContextUsage

**作用**：记录 Agent 上下文窗口的使用情况。

**输入类型**：

~~~ts
interface ContextUsage {
  agent_id: string;
  used_tokens: number;
  window_tokens: number;
  remaining_tokens: number;
  used_ratio: number;
}
~~~

**返回值类型**：[ContextUsage](04-agent-monitor.md) 快照。

**示例**：

~~~js
const usage = agentMonitor.get_context_usage(agent.agent_id);
text(usage.remaining_tokens);
~~~

### get_context_usage(agent_id)

**作用**：查询指定 Agent 的上下文用量。

**输入类型**：agent_id: string。

**返回值类型**：[ContextUsage](04-agent-monitor.md)。

**示例**：

~~~js
text(agentMonitor.get_context_usage(agent.agent_id));
~~~

### get_token_usage(agent_id)

**作用**：查询指定 Agent 的 token 用量。

**输入类型**：agent_id: string。

**返回值类型**：[TokenUsage](04-agent-monitor.md)。

**示例**：

~~~js
text(agentMonitor.get_token_usage(agent.agent_id));
~~~

### get_state(agent_id)

**作用**：查询指定 Agent 的生命周期状态。

**输入类型**：agent_id: string。

**返回值类型**：[AgentState](04-agent-monitor.md)。

**示例**：

~~~js
text(agentMonitor.get_state(agent.agent_id));
~~~
