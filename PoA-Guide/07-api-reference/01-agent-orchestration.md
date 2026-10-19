# AgentOrchestration

[AgentOrchestration](01-agent-orchestration.md) 负责创建 [Agent](03agent.md)、派发任务、等待结果和批量收口。每个 Agent 通过自己的
[AgentConfig](02-agent-config.md) 定义模型、能力和运行时。

## 类型

### Handle

**作用**：表示一次已派发 Agent 任务，供等待、追加输入和关闭操作使用。

**输入类型**：

~~~ts
interface Handle {
  agent_id: string;
  agent_name?: string;
  meta?: unknown;
}
~~~

**返回值类型**：由 spawnAgent 或 spawnMany 返回 [Handle](01-agent-orchestration.md)。

**示例**：

~~~js
const handle = await orchestration.spawnAgent("检查 README", [], "reviewer");
text(handle.agent_id);
~~~

### RemoteAgent

**作用**：描述可通过远程协议调用的 [Agent](03agent.md)。

**输入类型**：由 create_remote_agent 创建。

**返回值类型**：

~~~ts
interface RemoteAgent {
  agent_id: string;
  name: string;
  description?: string;
  agent_card?: unknown;
}
~~~

**示例**：

~~~js
const remote = orchestration.create_remote_agent("researcher", "远程检索 Agent");
text(remote.name);
~~~

## 方法

### create_agent(name, description?, agent_config?)

**作用**：创建一个具有独立配置的本地 Agent。

**输入类型**：name: string；description?: string；agent_config?: [AgentConfig](02-agent-config.md)。

**返回值类型**：[Agent](03agent.md)。

**示例**：

~~~js
const reviewer = orchestration.create_agent("reviewer", "审查代码", {
  model: { model_id: "gpt-6" },
  system_prompt: "只报告可复现的问题",
});
~~~

### create_remote_agent(name, description, agent_config?)

**作用**：注册一个由远程 Agent 服务提供的 Agent 描述。

**输入类型**：name: string；description: string；agent_config?: AgentConfig。

**返回值类型**：[RemoteAgent](01-agent-orchestration.md)。

**示例**：

~~~js
const analyst = orchestration.create_remote_agent("analyst", "调用远程分析服务");
~~~

### spawnAgent(message, input_items?, agent_name, meta?)

**作用**：按 Agent 名称派发一次任务，并返回可用于后续管理的句柄。

**输入类型**：message: string；input_items?: IOInput[]；agent_name: string；meta?: unknown。

**返回值类型**：Promise<Handle>。

**示例**：

~~~js
const handle = await orchestration.spawnAgent(
  "分析 src/ 目录",
  [],
  "reviewer",
  { source: "nightly" },
);
~~~

### spawnMany(specs, concurrency)

**作用**：按任务规格批量派发 Agent，并限制同时创建的任务数。

**输入类型**：specs: SpawnSpec[]；concurrency: number。

**返回值类型**：Promise<Handle[]>，顺序与 specs 相同。

**示例**：

~~~js
const handles = await orchestration.spawnMany(
  files.map((file) => ({ message: "检查 " + file, agent_name: "reviewer" })),
  3,
);
~~~

### sendAndWait(handle, message?, input_items?)

**作用**：向已派发的 Agent 追加任务，并等待新的 Agent 结果。

**输入类型**：handle: Handle；message?: string；input_items?: IOInput[]。

**返回值类型**：Promise<[AgentResult](03agent.md)>。

**示例**：

~~~js
const followUp = await orchestration.sendAndWait(handle, "补充一条风险");
text(followUp.text);
~~~

### collectAll(handles, timeoutMs?)

**作用**：等待一组句柄对应的 Agent 完成，并按输入句柄收集结果。

**输入类型**：handles: Handle[]；timeoutMs?: number。

**返回值类型**：Promise<AgentResult[]>。

**示例**：

~~~js
const results = await orchestration.collectAll(handles, 120000);
text(results.map((result) => result.text));
~~~

### closeAll(handles)

**作用**：关闭一组句柄对应的 Agent，并释放运行资源。

**输入类型**：handles: Handle[]。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await orchestration.closeAll(handles);
~~~

### runBatch(specs, concurrency, timeoutMs?)

**作用**：组合批量派发、等待和结果收集。

**输入类型**：specs: SpawnSpec[]；concurrency: number；timeoutMs?: number。

**返回值类型**：Promise<AgentResult[]>。

**示例**：

~~~js
const results = await orchestration.runBatch(
  [{ message: "列出入口文件", agent_name: "reviewer" }],
  2,
  60000,
);
~~~
