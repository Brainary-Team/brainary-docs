# AgentConfig、Capabilities、ModelSpec、RuntimeSpec、AgentGroup、ResourceShareSpace 与 SecureConfig

本页描述 [Agent](03-agent.md) 的静态配置类型。配置在 Agent 创建时确定，执行时由运行时转换为对应的模型、
能力和环境设置。

### ModelSpec

**作用**：指定 Agent 使用的模型和推理参数。

**类型定义**：

~~~ts
interface ModelSpec {
  model_id: string;
  provider: string;
  reasoningEffort: string;
}
~~~

model_id、provider 和 reasoningEffort 均为必填字段。

**示例**：

~~~js
const model = {
  model_id: "gpt-6-sol",
  provider: "openAI",
  reasoningEffort: "high",
};
~~~

### Capabilities

**作用**：声明 Agent 可以访问的 Brainary tool、MCP server、PoA 和 skills 能力。

**类型定义**：

~~~ts
interface Capabilities {
  brainary_tool?: string[];
  mcp?: string[];
  poa?: string[];
  skills?: string[];
}
~~~

**示例**：

~~~js
const capabilities = {
  brainary_tool: ["exec_command", "multi_agent"],
  mcp: ["github_server/*"],
  poa: ["review-package"],
  skills: ["code-review"],
};
~~~

### Sandbox

**作用**：限定 Agent 的文件系统访问级别。

**类型定义**：

~~~ts
type Sandbox = "read_only" | "workspace_write" | "danger_full_access";
~~~

**示例**：

~~~js
const sandbox = "workspace_write";
~~~

### ApprovalPolicy

**作用**：决定执行需要审批时的处理策略。

**类型定义**：

~~~ts
type ApprovalPolicy = "auto_review" | "deny_all" | "manual_review";
~~~

**示例**：

~~~js
const approvalPolicy = "auto_review";
~~~

### RuntimeSpec

**作用**：指定 Agent 的工作目录、沙箱和审批策略。

**类型定义**：

~~~ts
interface RuntimeSpec {
  workspace?: string;
  sandbox?: Sandbox;
  approvalPolicy?: ApprovalPolicy;
}
~~~

**示例**：

~~~js
const runtime = {
  workspace: "/workspace/project",
  sandbox: "workspace_write",
  approvalPolicy: "auto_review",
};
~~~

### AgentGroup

**作用**：把本地 Agent 与远程 Agent 组合为一组，供 subagents 和 peer_agents 配置使用。

**类型定义**：

~~~ts
interface AgentGroup {
  agents: Agent[];
  remote_agents: RemoteAgent[];
}
~~~

Agent 与 RemoteAgent 分别见 [Agent](03-agent.md) 和 [RemoteAgent](01-agent-orchestration.md#remoteagent)。

**示例**：

~~~js
const group = {
  agents: [reviewer],
  remote_agents: [analyst],
};
~~~

### ResourceShareSpace

**作用**：指定 Agent 的资源共享空间、上下文分叉设置和资源预算。

**类型定义**：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `wm_share_space?` | `string[]` | 可选的资源共享空间列表。 |
| `fork_context` | `boolean` | 是否分叉上下文。 |
| `token_budget` | `number` | token 预算。 |
| `runtime_budget_ms` | `number` | 运行时长预算，单位为毫秒。 |

**示例**：

~~~js
const resourceShareSpace = {
  wm_share_space: ["shared-space"],
  fork_context: false,
  token_budget,
  runtime_budget_ms,
};
~~~

### SecureConfig

**作用**：配置不可信工具的使用权限、违规处理和待审批请求的有效期。

**类型定义**：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `allow_unstrusted_tools?` | `Set<string>` | 允许使用的不可信工具集合。 |
| `block_on_violation?` | `boolean` | 是否在违规时阻塞。 |
| `apprroval_on_violation?` | `boolean` | 是否在违规时请求审批。 |
| `pending_approval_ttl?` | `timedelta` | 待审批请求的有效期。 |

**示例**：

~~~js
const secureConfig = { block_on_violation: true };
~~~

### AgentConfig

**作用**：组合 Agent 的模型、系统提示、能力、运行时、子 Agent 组和同级 Agent 组。

**类型定义**：

~~~ts
interface AgentConfig {
  model: ModelSpec;
  system_prompt?: string;
  capabilities?: Capabilities;
  runtime?: RuntimeSpec;
  subagents?: AgentGroup;
  peer_agents?: AgentGroup;
  handof?: Handof[];
  resource_share_space?: ResourceShareSpace;
  max_agent_depth: number;
  secure_config?: SecureConfig;
}
~~~

**示例**：

~~~js
const config = {
  model: {
    model_id: "gpt-6-sol",
    provider: "openAI",
    reasoningEffort: "high",
  },
  system_prompt: "只输出可执行的审查结论",
  capabilities: { brainary_tool: ["exec_command"] },
  runtime: { sandbox: "read_only" },
  subagents: { agents: [reviewer], remote_agents: [] },
  peer_agents: { agents: [], remote_agents: [analyst] },
  resource_share_space: resourceShareSpace,
  max_agent_depth: 3,
  secure_config: { block_on_violation: true },
};
~~~
