# Agent

Agent 是 [create_agent](03-agent-executor.md#create_agentname-description-agent_config) 返回的对象，代表一个可以多次运行的 agent。

一个 agent 可以开多个会话（thread），每个会话有自己的 thread_id 和对话记录，互不影响。run 不传 thread_id 时开一个新会话，传入时在该会话中继续；close、resume 和 status 都作用于指定的会话。

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
| agent_id | agent 的 ID，可以传给 [AgentExecutor](03-agent-executor.md) 中按 ID 操作的方法。关闭会话不影响 agent，之后仍可用 run 开新会话 |
| name、description、agent_config | 调用 create_agent 时传入的值 |

**示例**：

~~~js
text({ id: agent.agent_id, name: agent.name });
~~~

### run(message?, items?, output_schema?, timeout_ms?, thread_id?)

**作用**：向 agent 发送一轮任务，并等待这一轮结束。不传 thread_id 时开一个新会话，传入时在该会话中继续对话。

**输入类型**：

- message?: string，本轮的文本消息。
- items?: readonly [MessageItem](04-message-item.md)[]，本轮的消息项。同时传入 message 时，message 作为第一个文本项排在 items 之前。
- output_schema?: null，预留参数，省略或传 null。
- timeout_ms?: number，最长等待时间，单位毫秒，默认 30000，有效范围与 [acquire_agent_result](03-agent-executor.md#acquire_agent_resultthread_id-timeout_ms-turn_id) 相同。
- thread_id?: string，要继续的会话，取自之前的 [AgentResult](#agentresult).thread_id。省略时开一个新会话。

message 和 items 至少传入一项。传入 thread_id 时，该会话的状态为 pending_init、completed、errored 或 interrupted 才能运行。不传 thread_id 的多次调用各开一个会话，可以并发运行。

**返回值类型**：Promise<[AgentResult](#agentresult)>。

**异常**：

- message 和 items 都未传入，或 message 为空白字符串：抛出 TypeError。
- timeout_ms 不是正整数：抛出 RangeError。
- thread_id 不是该 agent 的会话：抛出 Error。
- thread_id 指定的会话状态为 running、shutdown 或 not_found：抛出 [AgentNotIdleError](#agentnotidleerror)，消息不会发送。已关闭的会话先调用 [resume(thread_id)](#resumethread_id)。
- 超时：抛出 [AgentResultTimeoutError](#agentresulttimeouterror)。会话继续运行，之后用错误的 thread_id 和 turn_id 调用 [acquire_agent_result](03-agent-executor.md#acquire_agent_resultthread_id-timeout_ms-turn_id) 获取本轮结果。
- 打开的会话数已达到 [max_concurrent_threads_per_session](../02-configuration.md#agent-层级和数量)：抛出 Error。

**示例**：

~~~js
const result = await agent.run("检查 README", undefined, undefined, 120_000);
text({ status: result.agent_status, answer: result.result });

// 在同一个会话中继续
const next = await agent.run(
  "结合截图说明问题",
  [{ type: "local_image", path: "/workspace/screenshot.png" }],
  undefined,
  undefined,
  result.thread_id,
);
text(next.result);

// 每次调用开一个新会话，并发运行
const reviews = await Promise.all(["src/auth.ts", "src/db.ts"].map((file) => agent.run(`审查 ${file}`)));
text(reviews.map((review) => ({ thread: review.thread_id, answer: review.result })));
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

### close(thread_id)

**作用**：关闭 agent 的一个会话。正在运行的一轮会被中断。agent 本身和它的其他会话不受影响，之后仍可用 run 开新会话。关闭后 status(thread_id) 返回 "not_found"，并释放 [max_concurrent_threads_per_session](../02-configuration.md#agent-层级和数量) 的名额。

**输入类型**：thread_id: string，要关闭的会话。

**返回值类型**：Promise<[CloseResult](#closeresult)>。

**异常**：thread_id 不是该 agent 的会话，或该会话已关闭时抛出 Error。

**示例**：

~~~js
const result = await agent.run("检查 README");
text(await agent.close(result.thread_id)); // { agent_id, thread_id, status: "closed" }
~~~

### resume(thread_id)

**作用**：恢复 agent 已关闭的一个会话。恢复后的会话保留关闭前的对话记录，使用创建时的配置。会话未关闭时不做任何操作。

**输入类型**：thread_id: string，要恢复的会话。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>，恢复后的状态。

**异常**：thread_id 不是该 agent 的会话，或打开的会话数已达到 max_concurrent_threads_per_session 时抛出 Error。

**示例**：

~~~js
const first = await agent.run("检查 README");
await agent.close(first.thread_id);
await agent.resume(first.thread_id);
text((await agent.run("继续上一轮的检查", undefined, undefined, undefined, first.thread_id)).result);
~~~

### status(thread_id)

**作用**：返回 agent 一个会话的当前状态，不等待。

**输入类型**：thread_id: string，要查询的会话。

**返回值类型**：Promise<[AgentStatus](03-agent-executor.md#agentstatus)>。已关闭的会话返回 "not_found"。

**示例**：

~~~js
if ((await agent.status(result.thread_id)) === "completed") {
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
  thread_id: string;
  turn_id: string;
  agent_status: AgentStatus;
  result: string | null;
  error: string | null;
  schema_result: null;
}
~~~

| 字段 | 说明 |
| --- | --- |
| agent_id | agent 的 ID |
| thread_id | 本轮所在会话的 ID，传给 run 在该会话中继续，或传给 close、resume 和 status |
| turn_id | 本轮的 ID，可以传给 acquire_agent_result 指定要等待的一轮 |
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

## CloseResult

**作用**：close 的返回值。

**类型定义**：

~~~ts
interface CloseResult {
  agent_id: string;
  thread_id: string;
  status: "closed";
}
~~~

| 字段 | 说明 |
| --- | --- |
| agent_id | agent 的 ID |
| thread_id | 被关闭的会话的 ID |
| status | 固定为 "closed" |

## 错误

### AgentNotIdleError

**作用**：run 发现 thread_id 指定的会话正在运行（running）或已关闭（shutdown、not_found）时抛出。

**类型定义**：

~~~ts
class AgentNotIdleError extends Error {
  readonly agent_id: string;
  readonly thread_id: string;
  readonly agent_status: AgentStatus;  // 抛出时会话的状态
}
~~~

### AgentResultTimeoutError

**作用**：run 或 acquire_agent_result 超时时抛出。会话继续运行，可以用 thread_id 和 turn_id 再次调用 acquire_agent_result 获取这一轮的结果。

**类型定义**：

~~~ts
class AgentResultTimeoutError extends Error {
  readonly agent_id: string;
  readonly thread_id: string;  // 本轮所在的会话
  readonly turn_id: string;    // 超时的这一轮
}
~~~

**示例**：

~~~js
import { AgentNotIdleError, AgentResultTimeoutError } from "@brainary/poa-sdk";

try {
  text((await agent.run("继续检查")).result);
} catch (error) {
  if (error instanceof AgentNotIdleError) text({ busy: error.agent_status });
  else if (error instanceof AgentResultTimeoutError) text(`会话 ${error.thread_id} 仍在运行`);
  else throw error;
}
~~~
