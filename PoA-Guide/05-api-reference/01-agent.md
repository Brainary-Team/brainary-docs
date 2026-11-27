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
    model: { model_id: "gpt-5.6-luna", reasoning_effort: "high" },
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

### run(message?, message_items?, timeout_ms?)

**作用**：向 agent 发送一轮任务，并等待这一轮结束。

**输入类型**：

- message?: string，本轮的文本消息。
- message_items?: readonly [MessageItem](04-message-item.md)[]，本轮的消息项。同时传入 message 时，message 作为第一个文本项排在消息项之前。
- timeout_ms?: number，最长等待时间，单位毫秒，与 [acquire_agent_result](03-agent-executor.md#acquire_agent_resultagent_id-timeout_ms) 相同。

message 和 message_items 至少传入一项。agent 的状态为 pending_init、completed、errored 或 interrupted 时才能运行。

**返回值类型**：Promise<[AgentResult](#agentresult)>。

**异常**：

- message 和 message_items 都未传入，或 message 为空白字符串：抛出 TypeError。
- timeout_ms 不是正整数：抛出 RangeError。
- agent 的状态为 running、shutdown 或 not_found：抛出 [AgentNotIdleError](#agentnotidleerror)，消息不会发送。
- 超时：抛出 [AgentResultTimeoutError](#agentresulttimeouterror)。agent 继续运行，之后用 `agentExecutor.acquire_agent_result(agent.agent_id)` 获取本轮结果。
- Brainary 拒绝本次调用：抛出 [MultiAgentError](#multiagenterror)。

**示例**：

~~~js
const result = await agent.run("检查 README", undefined, 120_000);
text({ status: result.agent_status, answer: result.result });

const next = await agent.run("结合截图说明问题", [{ type: "image", image_url: "data:image/png;base64,iVBORw0KGgo..." }]);
text(next.result);
~~~

### append_message(message?, message_items?, immediately?)

**作用**：向 agent 追加消息，不等待处理结果。与 [AgentExecutor.append_message](03-agent-executor.md#append_messageagent_id-message-message_items-immediately) 相同，作用于当前 agent。

**输入类型**：

- message?: string，追加的文本消息。
- message_items?: readonly [MessageItem](04-message-item.md)[]，追加的消息项。同时传入 message 时，message 作为第一个文本项排在消息项之前。
- immediately?: boolean，默认为 false：
  - false：正在运行的一轮会读到这条消息；agent 空闲时，消息留到下一轮。
  - true：打断正在运行的一轮，立即开始新的一轮处理这条消息，用 acquire_agent_result 获取这一轮的结果。

message 和 message_items 至少传入一项。

**返回值类型**：Promise<[Handle](03-agent-executor.md#handle)>，agent_name 为 agent 的 name。

**异常**：

- message 和 message_items 都未传入，或 message 为空白字符串：抛出 TypeError。
- agent 已关闭等 Brainary 拒绝本次调用的情况：抛出 [MultiAgentError](#multiagenterror)。

**示例**：

~~~js
await agent.append_message("优先检查认证逻辑", undefined, true);
const result = await agentExecutor.acquire_agent_result(agent.agent_id, 60_000);
text(result.result);
~~~

### interrupt()

**作用**：打断 agent 正在运行的一轮，等这一轮停止后返回。与 [AgentExecutor.interrupt](03-agent-executor.md#interruptagent_id) 相同，作用于当前 agent。

**输入类型**：无。

**返回值类型**：Promise<[InterruptResult](03-agent-executor.md#interruptresult)>。

**异常**：agent 已关闭等 Brainary 拒绝本次调用的情况，抛出 [MultiAgentError](#multiagenterror)。

**示例**：

~~~js
const stopped = await agent.interrupt();
text({ interrupted: stopped.interrupted, status: stopped.agent_status });
~~~

### close()

**作用**：关闭 agent，以及它派出的、仍处于打开状态的各层子 agent。关闭后 status() 返回 "not_found"，并释放 [max_concurrent_threads_per_session](../02-configuration.md#agent-层级和数量) 的名额。

**输入类型**：无。

**返回值类型**：Promise<[CloseResult](03-agent-executor.md#closeresult)>。

**异常**：agent 已关闭时抛出 [MultiAgentError](#multiagenterror)，code 为 THREAD_NOT_FOUND。

**示例**：

~~~js
const closed = await agent.close();
text(closed.status); // "shutdown"
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

## AgentResult

**作用**：一轮运行的结果，由 run 和 acquire_agent_result 返回。

**类型定义**：

~~~ts
interface AgentResult {
  agent_id: string;
  result: string | null;
  error: string | null;
  truncated: boolean;
  agent_status: AgentStatus;
}
~~~

| 字段 | 说明 |
| --- | --- |
| agent_id | agent 的 ID |
| result | agent_status 为 completed 时为 agent 的最终回复；agent 没有回复或为其他状态时为 null |
| error | agent_status 为 errored 时为错误信息；其他状态为 null |
| truncated | result 或 error 是否被截断。超过约 900 token 的文本只保留开头和结尾，中间删去的部分换成 `…N tokens truncated…` 这样的标记 |
| agent_status | 本轮结束时的状态：completed、errored、interrupted、shutdown 或 not_found |

需要结构化结果时，让 agent 回复 JSON，再用 [parse_json_reply](09-helper.md#parse_json_replyraw) 解析 result。

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

### MultiAgentError

**作用**：Brainary 拒绝 Agent 或 AgentExecutor 的一次调用时抛出，例如参数不合法、agent 已关闭、agent 数量达到上限。按 code 判断原因，details 是该原因附带的信息。

**类型定义**：

~~~ts
class MultiAgentError extends Error {
  readonly code: MultiAgentErrorCode;
  readonly details: MultiAgentErrorDetailsMap[MultiAgentErrorCode];  // 判断 code 后收窄为对应的类型
}
~~~

| code | details | 原因 |
| --- | --- | --- |
| INVALID_ARGUMENT | [InvalidArgumentDetails](#invalidargumentdetails) | 参数不合法 |
| UNKNOWN_MODEL | {} | model_id 不是可用的模型 |
| UNSUPPORTED_REASONING_EFFORT | { supported: string[] } | 模型不支持该推理强度，supported 为支持的取值 |
| AGENT_LIMIT_REACHED | { max_threads: number } | 打开的 agent 数已达到 [max_concurrent_threads_per_session](../02-configuration.md#agent-层级和数量) |
| AGENT_DEPTH_LIMIT | { max_depth: number } | 新 agent 的层级超过 [max_depth](../02-configuration.md#agent-层级和数量) |
| AGENT_PATH_CONFLICT | { path: string } | 新 agent 的路径已被打开的 agent 占用 |
| AGENT_NICKNAME_CONFLICT | { nickname: string } | 打开的 agent 中已有同名 nickname |
| AGENT_NICKNAMES_EXHAUSTED | {} | 未传入 nickname，且可分配的昵称已用完 |
| SELF_TARGET | {} | 操作的是调用方自己 |
| ROOT_TARGET | {} | 操作的是运行程序的根线程 |
| THREAD_NOT_FOUND | {} | agent 不存在或已关闭 |
| THREAD_UNAVAILABLE | {} | agent 正在关闭 |
| PERSISTENCE_FAILED | {} | 关闭前保存会话记录失败，agent 仍处于打开状态 |
| UNAVAILABLE | {} | Brainary 暂时无法处理，通常是会话正在结束 |
| BUSY | {} | 程序正在等待提问的回答，暂时不能发起调用 |
| CANCELLED | {} | 调用被取消，通常是程序正在结束 |
| INTERNAL_ERROR | {} | Brainary 内部错误 |

**示例**：

~~~js
import { MultiAgentError } from "@brainary/poa-sdk";

try {
  await agentExecutor.spawn_agent({ message: "检查 README", nickname: "Ada" });
} catch (error) {
  if (error instanceof MultiAgentError && error.code === "AGENT_NICKNAME_CONFLICT") text(`${error.details.nickname} 已被使用`);
  else throw error;
}
~~~

### InvalidArgumentDetails

**作用**：code 为 INVALID_ARGUMENT 时的 details，field 指出不合法的参数，reason 说明原因。

**类型定义**：

~~~ts
type InvalidArgumentDetails = { field: string } & (
  | { reason: "malformed" | "missing" | "empty" | "unsupported_module" | "unknown_module" | "invalid_name" | "invalid_path" | "not_supported" }
  | { reason: "too_long"; max_bytes: number }
  | { reason: "too_many_attachments"; max_attachments: number }
  | { reason: "too_many_modules"; max_modules: number }
  | { reason: "out_of_range"; min: number; max: number }
);
~~~

| reason | 含义 |
| --- | --- |
| malformed | 类型或格式不对 |
| missing | 缺少必填参数 |
| empty | 为空或只有空白 |
| invalid_name | 不是合法的名称，例如 create_agent 的 name 含 `[a-z0-9_]` 以外的字符或为 "root"，此时 field 为 role |
| invalid_path | 不是合法的 agent 路径 |
| not_supported | 不支持的取值，例如 Brainary 不接受的消息项类型 |
| too_long | 超过 max_bytes 字节，例如 create_agent 的 name、spawn_agent 的 nickname 超过 64 字节 |
| too_many_attachments | 一条消息中的图片和音频超过 max_attachments 个 |
| out_of_range | 数值超出 [min, max] |
| unsupported_module、unknown_module、too_many_modules | 内联 PoA 包的模块不合法，PoA SDK 当前不会用到 |
