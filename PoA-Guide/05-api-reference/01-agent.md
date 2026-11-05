# Agent

Agent 是 [create_agent](03-agent-executor.md#create_agentname-description-agent_config) 返回的对象，代表一个可以多次运行的 agent。

以下示例中的 agent 均按如下方式创建：

~~~js
import { AgentExecutor, text } from "@brainary/poa-sdk";

const agentExecutor = new AgentExecutor();
const agent = await agentExecutor.create_agent({
  name: "reviewer",
  description: "代码审查",
  agent_config: {
    model: { model_id: "gpt-5.6-luna", provider: "vendor", reasoning_effort: "high" },
  },
});
~~~

### Agent 对象

**作用**：create_agent 返回的 agent。Agent 对象只能用 create_agent 创建。

**类型定义**：

~~~ts
class Agent {
  readonly agent_id: string;
  readonly name: string;
  readonly description: string | undefined;
  readonly agent_config: AgentConfig;
}
~~~

| 属性 | 说明 |
| --- | --- |
| agent_id | agent 的 ID，可以传给 [AgentExecutor](03-agent-executor.md) 中按 ID 操作的方法 |
| name、description、agent_config | 调用 create_agent 时传入的值 |

**示例**：

~~~js
text({ id: agent.agent_id, name: agent.name });
~~~

### run(message?, items?, output_schema?, timeout_ms?)

**作用**：向 agent 发送一轮任务，并等待这一轮结束。

**输入类型**：

- message?: string，本轮的文本消息。
- items?: readonly [MessageItem](04-message-item.md)[]，本轮的消息项。同时传入 message 时，message 作为第一个文本项排在 items 之前。
- output_schema?: null，预留参数，省略或传 null。
- timeout_ms?: number，最长等待时间，单位毫秒，默认 30000，有效范围与 [acquire_agent_result](03-agent-executor.md#acquire_agent_resultagent_id-timeout_ms) 相同。

message 和 items 至少传入一项。agent 的状态为 pending_init、completed、errored 或 interrupted 时才能运行。

**返回值类型**：Promise<[AgentResult](#agentresult)>。

**异常**：

- message 和 items 都未传入，或 message 为空白字符串：抛出 TypeError。
- timeout_ms 不是正整数：抛出 RangeError。
- agent 的状态为 running、shutdown 或 not_found：抛出 [AgentNotIdleError](#agentnotidleerror)，消息不会发送。已关闭的 agent 先调用 [resume()](#resume)。
- 超时：抛出 [AgentResultTimeoutError](#agentresulttimeouterror)。agent 继续运行，之后用 `agentExecutor.acquire_agent_result(agent.agent_id)` 获取本轮结果。

**示例**：

~~~js
const result = await agent.run("检查 README", undefined, undefined, 120_000);
text({ status: result.agent_status, answer: result.result });

const next = await agent.run("结合截图说明问题", [{ type: "local_image", path: "/workspace/screenshot.png" }]);
text(next.result);
~~~

### append_message(message?, message_items?, Immediately?)

**状态**：尚未实现，以下为设计描述。

**作用**：向 agent 追加消息，并返回任务句柄；Immediately 指定是否立即处理。
与 [AgentExecutor.append_message](03-agent-executor.md#append_messageagent_id-message-message_items-immediately) 相同，作用于当前 agent。

**输入类型**：message?: string；message_items?: [MessageItem](04-message-item.md)[]；Immediately?: boolean，默认为 false。message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](03-agent-executor.md#handle)>。

**示例**：

~~~js
const handle = await agent.append_message("优先检查认证逻辑", undefined, true);
const result = await agentExecutor.acquire_agent_result(handle.agent_id, 60_000);
text(result.result);
~~~

### context_compact(compact_template?)

**状态**：尚未实现，以下为设计描述。

**作用**：压缩 agent 的上下文，减少后续运行的上下文占用。

**输入类型**：compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agent.context_compact("只保留结论和待办");
~~~

### close()

**作用**：关闭 agent，以及它派出的、仍处于打开状态的各层子 agent。关闭后 status() 返回 "not_found"，并释放 [max_concurrent_threads_per_session](../02-configuration.md#agent-层级和数量) 的名额。

**输入类型**：无。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>，关闭前的状态。

**异常**：agent 已关闭时抛出 Error。

**示例**：

~~~js
text(await agent.close());
~~~

### resume()

**作用**：恢复已关闭的 agent。恢复后的 agent 保留关闭前的对话记录，使用创建时的配置。agent 未关闭时不做任何操作。

**输入类型**：无。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>，恢复后的状态。

**异常**：打开的 agent 数已达到 max_concurrent_threads_per_session 时抛出 Error。

**示例**：

~~~js
await agent.close();
await agent.resume();
text((await agent.run("继续上一轮的检查")).result);
~~~

### status()

**作用**：返回 agent 的当前状态，不等待。

**输入类型**：无。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>。已关闭的 agent 返回 "not_found"。

**示例**：

~~~js
if ((await agent.status()) === "completed") {
  text("上一轮已完成");
}
~~~

### token_usage()、context_usage()

**状态**：尚未实现，以下为设计描述。

**作用**：读取 agent 的 token 用量和上下文用量。

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

**作用**：一轮运行的结果，由 run 和 acquire_agent_result 返回。

**类型定义**：

~~~ts
interface AgentResult {
  agent_id: string;
  agent_status: AgentStatus;
  result: string | null;
  error: string | null;
  schema_result: null;
}
~~~

| 字段 | 说明 |
| --- | --- |
| agent_id | agent 的 ID |
| agent_status | 本轮结束时的状态：completed、errored、shutdown 或 not_found |
| result | agent_status 为 completed 时为 agent 的最终回复；agent 没有回复或为其他状态时为 null |
| error | agent_status 为 errored 时为错误信息；其他状态为 null |
| schema_result | 预留字段，始终为 null。需要结构化结果时，让 agent 回复 JSON，再用 [parse_json_reply](09-helper.md#parse_json_replyraw) 解析 result |

**示例**：

~~~js
import { parse_json_reply } from "@brainary/poa-sdk";

const result = await agent.run('只回复 JSON：{"issues": string[]}');
if (result.agent_status === "completed") {
  const parsed = parse_json_reply(result.result);
  text(parsed.error === null ? parsed.value : result.result);
} else {
  text({ status: result.agent_status, error: result.error });
}
~~~

## 错误

### AgentNotIdleError

**作用**：run 发现 agent 正在运行（running）或已关闭（shutdown、not_found）时抛出。

**类型定义**：

~~~ts
class AgentNotIdleError extends Error {
  readonly agent_id: string;
  readonly agent_status: AgentStatus;  // 抛出时 agent 的状态
}
~~~

### AgentResultTimeoutError

**作用**：run 或 acquire_agent_result 超时时抛出。agent 继续运行，可以再次调用 acquire_agent_result 获取结果。

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
