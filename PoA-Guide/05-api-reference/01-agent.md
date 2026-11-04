# Agent

[Agent](01-agent.md) 是由 [create_agent](03-agent-executor.md#create_agentname-description-agent_config) 创建的独立会话，可多次运行、关闭和恢复。

以下示例中的 agent 均按如下方式创建：

~~~js
import { AgentExecutor, text } from "@brainary/poa-sdk";

const agentExecutor = new AgentExecutor();
const agent = await agentExecutor.create_agent({
  name: "reviewer",
  description: "代码审查",
  agent_config: {
    model: { model_id: "gpt-5.6-luna", provider: "openai", reasoning_effort: "high" },
  },
});
~~~

### Agent 对象

**作用**：保存 Agent 的身份、描述和配置。

**类型定义**：

~~~ts
class Agent {
  constructor(
    executor: AgentExecutor,
    agent_id: string,
    name: string,
    description: string | undefined,
    agent_config: AgentConfig,
  );
  readonly agent_id: string;
  readonly name: string;
  readonly description: string | undefined;
  readonly agent_config: AgentConfig;
}
~~~

PoA 对象不跨 cell 保留；在后续 cell 中可用 agent_id 重新构造 Agent。

**示例**：

~~~js
import { Agent } from "@brainary/poa-sdk";

// agent_id 与 agent_config 来自先前 cell 中 create_agent 的结果
const restored = new Agent(agentExecutor, agent_id, "reviewer", "代码审查", agent_config);
text(await restored.status());
~~~

### run(message?, items?, output_schema?, timeout_ms?)

**作用**：让 Agent 开始一轮任务，并等待该轮的最终结果。

**输入类型**：message?: string；items?: readonly [MessageItem](04-message-item.md)[]；output_schema?: null，预留参数；timeout_ms?: number，等待返回的超时时间，单位为毫秒（ms），默认 30000。

**参数约束**：message 和 items 至少传入一项；同时传入时 message 排在 items 之前。

**返回值类型**：Promise<[AgentResult](#agentresult)>。

Agent 正在运行或已关闭时抛出 [AgentNotIdleError](#agentnotidleerror)，已关闭的 Agent 需先调用 resume()。
超时抛出 [AgentResultTimeoutError](#agentresulttimeouterror)，Agent 继续运行，可用 AgentExecutor.acquire_agent_result 再次获取结果。

**示例**：

~~~js
const result = await agent.run("检查 README", undefined, undefined, 120000);
text({ status: result.agent_status, answer: result.result });

const next = await agent.run("结合截图说明问题", [{ type: "local_image", path: "/workspace/screenshot.png" }]);
text(next.result);
~~~

### append_message(message?, message_items?, Immediately?)

**状态**：尚未实现，以下为设计描述。

**作用**：向 Agent 追加消息，并返回任务句柄；Immediately 用于指定是否立即处理。
与 [AgentExecutor.append_message](03-agent-executor.md#append_messageagent_id-message-message_items-immediately) 相同，作用于当前 Agent。

**输入类型**：message?: string；message_items?: [MessageItem](04-message-item.md)[]；Immediately?: boolean，默认为 false。

**参数约束**：message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](03-agent-executor.md#handle)>。

**示例**：

~~~js
const handle = await agent.append_message("优先检查认证逻辑", undefined, true);
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 60000);
text(result.result);
~~~

### context_compact(compact_template?)

**状态**：尚未实现，以下为设计描述。

**作用**：压缩 Agent 上下文，减少后续运行的上下文占用。

**输入类型**：compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.context_compact("只保留结论和待办");
~~~

### close()

**作用**：关闭 Agent 及其子 Agent，并返回关闭前的状态。关闭后可用 resume() 恢复。

**输入类型**：无。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>。

**示例**：

~~~js
text(await agent.close());
~~~

### resume()

**作用**：恢复已关闭的 Agent，并返回恢复后的状态。Agent 按创建时的配置恢复。

**输入类型**：无。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>。

**示例**：

~~~js
await agent.close();
text(await agent.resume());
~~~

### status()

**作用**：立即返回 Agent 当前状态。

**输入类型**：无。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>。

**示例**：

~~~js
if ((await agent.status()) === "completed") {
  text("上一轮已完成");
}
~~~

### token_usage()、context_usage()

**状态**：尚未实现，以下为设计描述。

**作用**：读取 Agent 自身的 token 用量和上下文用量。

**输入类型**：无。

**返回值类型**：分别为 Promise<[TokenUsage](03-agent-executor.md#tokenusage)>、Promise<[ContextUsage](03-agent-executor.md#contextusage)>。

**示例**：

~~~js
text({
  tokens: (await agent.token_usage()).total_tokens,
  remaining: (await agent.context_usage()).remaining_tokens,
});
~~~

## AgentResult

**作用**：表示一次 Agent 运行的最终结果。

**类型定义**：

~~~ts
interface AgentResult {
  agent_id: string;
  schema_result: null;
  result: string | null;
  error: string | null;
  agent_status: AgentStatus;
}
~~~

agent_status 是本次运行结束时的状态。completed 时 result 为 Agent 的最终回复；errored 时 error 为错误信息。
schema_result 为预留字段，用于返回按 output_schema 校验后的对象，当前为 null；需要 JSON 时可用 [parse_json_reply](12-helper.md) 解析 result。

**示例**：

~~~js
import { parse_json_reply } from "@brainary/poa-sdk";

const result = await agent.run('以 JSON 返回 {"issues": string[]}');
if (result.agent_status === "completed") {
  const parsed = parse_json_reply(result.result);
  text(parsed.error === null ? parsed.value : result.result);
} else {
  text({ status: result.agent_status, error: result.error });
}
~~~

## 错误

两个错误类都从根入口 `@brainary/poa-sdk` 导出。

### AgentNotIdleError

**作用**：调用 run 时 Agent 正在运行或已关闭，抛出此错误。

**类型定义**：

~~~ts
class AgentNotIdleError extends Error {
  readonly agent_id: string;
  readonly agent_status: AgentStatus;
}
~~~

### AgentResultTimeoutError

**作用**：run 或 acquire_agent_result 等待超时时抛出；Agent 继续运行，可以稍后再次获取结果。

**类型定义**：

~~~ts
class AgentResultTimeoutError extends Error {
  readonly agent_id: string;
}
~~~

**示例**：

~~~js
import { AgentNotIdleError, AgentResultTimeoutError } from "@brainary/poa-sdk";

try {
  text((await agent.run("继续检查")).result);
} catch (error) {
  if (error instanceof AgentNotIdleError) text({ busy: error.agent_status });
  else if (error instanceof AgentResultTimeoutError) text(`${error.agent_id} 仍在运行`);
  else throw error;
}
~~~
