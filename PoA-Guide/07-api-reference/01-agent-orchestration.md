# AgentOrchestration

[AgentOrchestration](01-agent-orchestration.md) 负责创建 [Agent](03-agent.md)、派发任务和获取结果。每个 Agent 通过自己的
[AgentConfig](02-agent-config.md) 定义模型、能力和运行时。

## 类型

### Handle

**作用**：表示一次已派发的 Agent 任务，包含 Agent 身份和派发时返回的状态。

**类型定义**：

~~~ts
interface Handle {
  agent_id: string;
  agent_name: string;
  meta?: string;
  agent_state: AgentState;
}
~~~

agent_state 的取值见 [AgentState](04-agent-executor.md#agentstate)。后续状态可通过 AgentExecutor.get_state 查询。
meta 原样保留派发时传入的 SpawnSpec.meta。

**示例**：

~~~js
const handle = await orchestration.spawn_agent({
  message: "检查 README",
  agent_name: "reviewer",
  meta: "readme-review",
});
text({ id: handle.agent_id, state: handle.agent_state, meta: handle.meta });
~~~

### RemoteAgent

**作用**：描述可通过远程协议调用的 Agent。

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
const remote = orchestration.create_remote_agent({
  name: "researcher",
  description: "远程检索 Agent",
  agent_card: "./researcher_agent_card.json",
});
text(remote.name);
~~~

### SpawnSpec

**作用**：描述一次 Agent 任务的消息、目标 Agent、自定义配置和附加元数据。

**类型定义**：

~~~ts
interface SpawnSpec {
  message?: string;
  message_items?: MessageItem[];
  agent_name: string;
  description?: string;
  custom_config?: AgentConfig;
  meta?: string;
}
~~~

message_items 的元素类型为 [MessageItem](05-message-item.md)。message 和 message_items 至少传入一项。
agent_name 为必填字符串；custom_config 是可选配置，meta 是可选的附加字符串。

**示例**：

~~~js
const spec = {
  message: "分析 src/ 目录",
  agent_name: "reviewer",
  description: "检查入口与依赖关系",
  meta: "src-review",
};
~~~

## 方法

### create_agent({name, description?, agent_config})

**作用**：创建一个具有独立配置的本地 Agent。

**输入类型**：命名参数对象，包含 name: string；description?: string；agent_config: [AgentConfig](02-agent-config.md)。

name 和 agent_config 必填。

**返回值类型**：[Agent](03-agent.md)。

**示例**：

~~~js
const reviewer = orchestration.create_agent({
  name: "reviewer",
  description: "审查代码",
  agent_config: {
    model: {
      model_id: "gpt-6-sol",
      provider: "openAI",
      reasoningEffort: "high",
    },
    system_prompt: "只报告可复现的问题",
    max_agent_depth: 3,
  },
});
~~~

### create_remote_agent({name, description, agent_card})

**作用**：注册一个由远程 Agent 服务提供的 Agent 描述。

**输入类型**：命名参数对象，包含 name: string；description: string；agent_card: unknown，远程 Agent 的卡片描述。三个参数均必填。

**返回值类型**：[RemoteAgent](#remoteagent)。

**示例**：

~~~js
const analyst = orchestration.create_remote_agent({
  name: "analyst",
  description: "调用远程分析服务",
  agent_card: "./remote_agent_card.json",
});
~~~

### spawn_agent(spawn_spec)

**作用**：按任务规格派发 Agent，并返回可用于后续获取结果的句柄。

**输入类型**：spawn_spec: [SpawnSpec](#spawnspec)，必填。

**返回值类型**：Promise<[Handle](#handle)>。

**示例**：

~~~js
const handle = await orchestration.spawn_agent({
  message: "分析 src/ 目录",
  agent_name: "reviewer",
});
~~~

### acquire_agent_result({agent_id, timeoutMs?})

**作用**：按 Agent ID 等待并获取一次任务结果。

**输入类型**：命名参数对象，包含 agent_id: string；timeoutMs?: number，等待结果的超时时间，单位为毫秒（ms）。

**返回值类型**：Promise<[AgentResult](03-agent.md#agentresult)>。

**示例**：

~~~js
const result = await orchestration.acquire_agent_result({ agent_id: handle.agent_id, timeoutMs: 120000 });
text(result.result);
~~~
