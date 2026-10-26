# Agent

[Agent](03-agent.md) 表示一个可重复运行、可暂停和可恢复的独立会话。

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
const agent = orchestration.create_agent("reviewer", "代码审查", {
  model: {
    model_id: "gpt-6-sol",
    provider: "openAI",
    reasoningEffort: "high",
  },
});
~~~

### run(message?, input_items?, output_schema?, timeout?)

**作用**：执行一次完整 Agent 任务并等待最终结果。

**输入类型**：message?: string；input_items?: IOInput[]；output_schema?: JsonSchema；timeout?: number，等待返回的超时时间，单位为毫秒（ms）。

**参数约束**：message 和 input_items 必须且只能传入一个；传入 output_schema 或 timeout 时，省略的前置可选参数使用 undefined 占位。

**返回值类型**：Promise<[AgentResult](03-agent.md)>。

**示例**：

~~~js
const result = await agent.run("检查 README", undefined, {
  type: "object",
  properties: { issues: { type: "array" } },
}, timeout);
text(result.output);
~~~

### wait_run(message?, input_items?, output_schema?)

**作用**：提交任务并等待 Agent 运行完成。

**输入类型**：message?: string；input_items?: IOInput[]；output_schema?: JsonSchema。

**参数约束**：message 和 input_items 必须且只能传入一个；仅传入 input_items 时，message 使用 undefined 占位。

**返回值类型**：Promise<[AgentResult](03-agent.md)>。

**示例**：

~~~js
const result = await agent.wait_run("汇总检查结果");
text(result.text);
~~~

### send_input(message?, input_items?, interrupt?, timeout?)

**作用**：向 Agent 发送后续输入；interrupt=true 时请求打断当前运行。

**输入类型**：message?: string；input_items?: IOInput[]；interrupt?: boolean；timeout?: number，等待返回的超时时间，单位为毫秒（ms）。

**参数约束**：message 和 input_items 必须且只能传入一个；传入 interrupt 或 timeout 时，省略的前置可选参数使用 undefined 占位。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.send_input("优先检查认证逻辑", undefined, true, timeout);
~~~

### compact(compact_template?)

**作用**：压缩 Agent 上下文，减少后续运行的上下文占用。

**输入类型**：compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.compact("只保留结论和待办");
~~~

### close()

**作用**：关闭 Agent 和仍在运行的资源。

**输入类型**：无。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.close();
~~~

### resume()

**作用**：恢复已关闭的 Agent，使其可以继续接收任务。

**输入类型**：无。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.resume();
~~~

### state()、token_usage()、context_usage()

**作用**：读取 Agent 自身的状态、token 用量和上下文用量。

**输入类型**：无。

**返回值类型**：分别为 [AgentState](04-agent-monitor.md)、[TokenUsage](04-agent-monitor.md)、[ContextUsage](04-agent-monitor.md)。

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
  output: unknown;
  text: string;
  agent_state: AgentState;
}
~~~

output 在传入 output_schema 时是校验后的对象，否则是最终文本；text 始终保留原始最终回复。详见 [AgentResult](03-agent.md)。

agent_state 表示本次运行返回时的 Agent 状态，类型为 [AgentState](04-agent-monitor.md#agentstate)。

**示例**：

~~~js
const result = await agent.run("返回状态", undefined, { type: "object" });
text({ structured: result.output, raw: result.text, state: result.agent_state });
~~~
