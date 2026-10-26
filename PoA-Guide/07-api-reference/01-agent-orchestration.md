# AgentOrchestration

[AgentOrchestration](01-agent-orchestration.md) 负责创建 [Agent](03-agent.md)、派发任务、等待结果和批量收口。每个 Agent 通过自己的
[AgentConfig](02-agent-config.md) 定义模型、能力和运行时。

## 类型

### Handle

**作用**：表示一次已派发 Agent 任务，供等待、追加输入和关闭操作使用。

**类型定义**：

~~~ts
interface Handle {
  agent_id: string;
  agent_name: string;
  meta?: unknown;
}
~~~

**示例**：

~~~js
const handle = await orchestration.spawn_agent("检查 README", undefined, "reviewer");
text(handle.agent_id);
~~~

### RemoteAgent

**作用**：描述可通过远程协议调用的 [Agent](03-agent.md)。

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
const remote = orchestration.create_remote_agent(
  "researcher",
  "远程检索 Agent",
  "./researcher_agent_card.json",
);
text(remote.name);
~~~

## 方法

### create_agent(name, description?, agent_config)

**作用**：创建一个具有独立配置的本地 Agent。

**输入类型**：name: string；description?: string；agent_config: [AgentConfig](02-agent-config.md)。

name 和 agent_config 必填；

**返回值类型**：[Agent](03-agent.md)。

**示例**：

~~~js
const reviewer = orchestration.create_agent("reviewer", "审查代码", {
  model: {
    model_id: "gpt-6-sol",
    provider: "openAI",
    reasoningEffort: "high",
  },
  system_prompt: "只报告可复现的问题",
});
~~~

### create_remote_agent(name, description, agent_card)

**作用**：注册一个由远程 Agent 服务提供的 Agent 描述。

**输入类型**：name: string；description: string；agent_card: unknown，远程 Agent 的卡片描述。三个参数均必填。

**返回值类型**：[RemoteAgent](01-agent-orchestration.md)。

**示例**：

~~~js
const analyst = orchestration.create_remote_agent("analyst", "调用远程分析服务", "./remote_agent_card.json");
~~~

### spawn_agent(message?, input_items?, agent_name, meta?)

**作用**：按 Agent 名称派发一次任务，并返回可用于后续管理的句柄。派发Agent的能力来源于～/brainary/config.toml文件。

**输入类型**：message?: string；input_items?: IOInput[]；agent_name: string；meta?: unknown。

agent_name 必填；省略前面的可选参数时，使用 undefined 保留参数位置。

**参数约束**：message 和 input_items 必须且只能传入一个；传入 message 时，input_items 使用 undefined 占位。

**返回值类型**：Promise<Handle>。

**示例**：

~~~js
const handle = await orchestration.spawn_agent(
  "分析 src/ 目录",
  undefined,
  "reviewer",
  { source: "nightly" },
);
~~~

使用已准备好的 input_items 数组时，省略 message：

~~~js
const handle = await orchestration.spawn_agent(undefined, input_items, "reviewer");
~~~

### spawn_many(specs, concurrency)

**作用**：按任务规格批量派发 Agent，并限制同时创建的任务数。

**输入类型**：specs: SpawnSpec[]；concurrency: number。

**返回值类型**：Promise<Handle[]>，顺序与 specs 相同。

**示例**：

~~~js
const handles = await orchestration.spawn_many(
  files.map((file) => ({ message: "检查 " + file, agent_name: "reviewer" })),
  3,
);
~~~

### send_wait(handle, message?, input_items?)

**作用**：向已派发的 Agent 追加任务，并等待新的 Agent 结果。

**输入类型**：handle: Handle；message?: string；input_items?: IOInput[]。

**参数约束**：message 和 input_items 必须且只能传入一个；仅传入 input_items 时，message 使用 undefined 占位。

**返回值类型**：Promise<[AgentResult](03-agent.md)>。

**示例**：

~~~js
const followUp = await orchestration.send_wait(handle, "补充一条风险");
text(followUp.text);
~~~

### collect_all(handles, timeout_ms?)

**作用**：等待一组句柄对应的 Agent 完成，并按输入句柄收集结果。

**输入类型**：handles: Handle[]；timeout_ms?: number。

**返回值类型**：Promise<AgentResult[]>。

**示例**：

~~~js
const results = await orchestration.collect_all(handles, 120000);
text(results.map((result) => result.text));
~~~

### close_all(handles)

**作用**：关闭一组句柄对应的 Agent，并释放运行资源。

**输入类型**：handles: Handle[]。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await orchestration.close_all(handles);
~~~

### run_batch(specs, concurrency, timeout_ms?)

**作用**：组合批量派发、等待和结果收集。

**输入类型**：specs: SpawnSpec[]；concurrency: number；timeout_ms?: number。

**返回值类型**：Promise<AgentResult[]>。

**示例**：

~~~js
const results = await orchestration.run_batch(
  [{ message: "列出入口文件", agent_name: "reviewer" }],
  2,
  60000,
);
~~~
