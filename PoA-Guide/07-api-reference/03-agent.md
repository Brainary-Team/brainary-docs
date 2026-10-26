# Agent

[Agent](03-agent.md) 表示一个可重复运行、可关闭和可恢复的独立会话。

### Agent 对象

**作用**：保存 Agent 身份、描述和只读配置。

**类型定义**：

~~~ts
interface Agent {
  readonly agent_id?: string;
  readonly name?: string;
  readonly description?: string;
  readonly agent_config?: AgentConfig;
}
~~~

**示例**：

~~~js
const agent = orchestration.create_agent({
  name: "reviewer",
  description: "代码审查",
  agent_config: {
    model: {
      model_id: "gpt-6-sol",
      provider: "openAI",
      reasoningEffort: "high",
    },
    max_agent_depth: 3,
    secure_config: { block_on_violation: true },
  },
});
~~~

### run({message?, items?, output_schema?, timeout_ms?})

**作用**：执行一次完整 Agent 任务并等待最终结果。

**输入类型**：命名参数对象，包含 message?: string；items?: [MessageItem](05-message-item.md)[]；output_schema?: JsonSchema；timeout_ms?: number，等待返回的超时时间，单位为毫秒（ms）。

**参数约束**：message 和 items 至少传入一项。

**返回值类型**：Promise<[AgentResult](#agentresult)>。

**示例**：

~~~js
const result = await agent.run({
  message: "检查 README",
  output_schema: {
    type: "object",
    properties: { issues: { type: "array" } },
  },
  timeout_ms: 60000,
});
text(result.schema_result);
~~~

### append_message({message?, message_items?, Immediately})

**作用**：向 Agent 追加消息，并返回任务句柄；Immediately 用于指定是否立即处理。

**输入类型**：命名参数对象，包含 message?: string；message_items?: [MessageItem](05-message-item.md)[]；Immediately: boolean，必填，默认取值为 false。

**参数约束**：message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](01-agent-orchestration.md#handle)>。

**示例**：

~~~js
const handle = await agent.append_message({ message: "优先检查认证逻辑", Immediately: true });
const result = await orchestration.acquire_agent_result({ agent_id: handle.agent_id, timeoutMs: 60000 });
text(result.result);
~~~

### context_compact({compact_template?})

**作用**：压缩 Agent 上下文，减少后续运行的上下文占用。

**输入类型**：命名参数对象，包含 compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.context_compact({ compact_template: "只保留结论和待办" });
~~~

### close()

**作用**：关闭 Agent 和仍在运行的资源，并返回 Agent 状态。

**输入类型**：无。

**返回值类型**：Promise<[AgentState](04-agent-executor.md#agentstate)>。

**示例**：

~~~js
text(await agent.close());
~~~

### resume()

**作用**：恢复已关闭的 Agent，使其可以继续接收任务，并返回 Agent 状态。

**输入类型**：无。

**返回值类型**：Promise<[AgentState](04-agent-executor.md#agentstate)>。

**示例**：

~~~js
text(await agent.resume());
~~~

### state()、token_usage()、context_usage()

**作用**：读取 Agent 自身的状态、token 用量和上下文用量。

**输入类型**：无。

**返回值类型**：分别为 [AgentState](04-agent-executor.md#agentstate)、[TokenUsage](04-agent-executor.md#tokenusage)、[ContextUsage](04-agent-executor.md#contextusage)。

**示例**：

~~~js
text({
  state: agent.state(),
  tokens: agent.token_usage().total_tokens,
  remaining: agent.context_usage().remaining_tokens,
});
~~~

## AgentResult

**作用**：表示一次 Agent 运行的最终结果。

**类型定义**：

~~~ts
interface AgentResult {
  agent_id: string;
  schema_result: Record<string, unknown>;
  result: string;
  agent_state: AgentState;
}
~~~

schema_result 是将 Agent 输出文本转换得到的 JSON 对象；传入 output_schema 时，转换后的对象按 schema 校验。
result 始终保留 Agent 的原始最终回复文本。

agent_state 表示本次运行返回时的 Agent 状态，类型为 [AgentState](04-agent-executor.md#agentstate)。

**示例**：

~~~js
const result = await agent.run({ message: "返回状态", output_schema: { type: "object" } });
text({ structured: result.schema_result, raw: result.result, state: result.agent_state });
~~~
