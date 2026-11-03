# AgentConfig、Capabilities、ModelSpec、RuntimeSpec、AgentGroup、ResourceShareSpace 与 SecureConfig

本页描述 [Agent](01-agent.md) 的配置类型。配置在 [create_agent](03-agent-executor.md#create_agentname-description-agent_config) 时传入，
校验失败时抛出异常。字段名使用 snake_case。

以下示例假定已从根入口导入枚举：

~~~js
import { ApprovalPolicy, BrainaryTool, Sandbox } from "@brainary/poa-sdk";
~~~

### ModelSpec

**作用**：指定 Agent 使用的模型和推理强度。

**类型定义**：

~~~ts
interface ModelSpec {
  model_id: string;
  provider: string;
  reasoning_effort?: string;
}
~~~

model_id 和 provider 必填。provider 为内置 provider 或配置文件 `[model_providers.<id>]` 中的 id。
reasoning_effort 如 "low"、"medium"、"high"，可用取值取决于模型；省略时使用模型的默认值。

**示例**：

~~~js
const model = {
  model_id: "gpt-5.6-luna",
  provider: "openai",
  reasoning_effort: "high",
};
~~~

### Capabilities

**作用**：声明 Agent 可以使用的工具。Agent 创建后默认没有任何工具，只持有这里列出的能力。

**类型定义**：

~~~ts
interface Capabilities {
  brainary_tool?: readonly BrainaryTool[];
  mcp?: readonly McpGrant[];
  // 以下字段尚未实现
  poa?: string[];
  skills?: string[];
}

type McpGrant = `${string}/${string}`;
~~~

授予的每一项都须为创建者自己拥有。brainary_tool 取 [BrainaryTool](07-brainary-tool.md#brainarytool) 成员；
mcp 中 `<server>/*` 授予该 server 的全部工具，`<server>/<tool>` 授予单个工具，server 来自配置文件 `[mcp_servers]` 或当前 PoA 包。
poa 和 skills 尚未实现。

**示例**：

~~~js
const capabilities = {
  brainary_tool: [BrainaryTool.exec_command, BrainaryTool.write_stdin, BrainaryTool.spawn_agent],
  mcp: ["github/*", "echo/echo"],
};
~~~

### Sandbox

**作用**：限定 Agent 的文件系统访问级别，按从严到宽排列。

**类型定义**：

~~~ts
enum Sandbox {
  read_only = "read_only",
  workspace_write = "workspace_write",
  danger_full_access = "danger_full_access",
}
~~~

TypeScript 中使用枚举成员，如 Sandbox.read_only。

**示例**：

~~~js
const sandbox = Sandbox.workspace_write;
~~~

### ApprovalPolicy

**作用**：决定 Agent 请求提权时的处理策略。

**类型定义**：

~~~ts
enum ApprovalPolicy {
  auto_review = "auto_review",
  deny_all = "deny_all",
  // 尚未实现：manual_review = "manual_review"
}
~~~

auto_review 由自动审查处理提权请求；deny_all 拒绝提权请求。manual_review 尚未实现。

**示例**：

~~~js
const approval_policy = ApprovalPolicy.deny_all;
~~~

### RuntimeSpec

**作用**：指定 Agent 的工作目录、沙箱和审批策略。

**类型定义**：

~~~ts
interface RuntimeSpec {
  workspace?: string;
  sandbox?: Sandbox;
  approval_policy?: ApprovalPolicy;
}
~~~

省略的字段沿用创建者的值，设置的值不能比创建者更宽松。workspace 须为已存在的目录。

**示例**：

~~~js
const runtime = {
  workspace: "/workspace/project",
  sandbox: Sandbox.read_only,
  approval_policy: ApprovalPolicy.deny_all,
};
~~~

### AgentGroup

**状态**：尚未实现，以下为设计描述。

**作用**：把本地 Agent 与远程 Agent 组合为一组，供 subagents 和 peer_agents 配置使用。

**类型定义**：

~~~ts
interface AgentGroup {
  agents: Agent[];
  remote_agents: RemoteAgent[];
}
~~~

Agent 与 RemoteAgent 分别见 [Agent](01-agent.md) 和 [RemoteAgent](03-agent-executor.md#remoteagent)。

**示例**：

~~~js
const group = {
  agents: [reviewer],
  remote_agents: [analyst],
};
~~~

### ResourceShareSpace

**状态**：尚未实现，以下为设计描述。

**作用**：指定 Agent 的资源共享空间、上下文分叉设置和资源上限。

**类型定义**：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `wm_share_space?` | `string[]` | 可选的资源共享空间列表。 |
| `fork_context` | `boolean` | 是否分叉上下文。 |
| `token_limit` | `number` | token 上限。 |
| `runing_time_limit` | `number` | 运行时长上限。 |

**示例**：

~~~js
const resourceShareSpace = {
  wm_share_space: ["shared-space"],
  fork_context: false,
  token_limit: 200000,
  runing_time_limit: 600000,
};
~~~

### SecureConfig

**状态**：尚未实现，以下为设计描述。

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

**作用**：组合 Agent 的模型、系统提示、能力和运行时。

**类型定义**：

~~~ts
interface AgentConfig {
  model: ModelSpec;
  system_prompt?: string;
  capabilities?: Capabilities;
  runtime?: RuntimeSpec;
  // 以下字段尚未实现
  subagents?: AgentGroup;
  peer_agents?: AgentGroup;
  handof?: Handof[];
  resource_share_space?: ResourceShareSpace;
  secure_config: SecureConfig;
  max_agent_depth: number;
}
~~~

model 必填。system_prompt 追加在 Agent 的基础指令之后。capabilities 省略时 Agent 没有工具。
尚未实现的字段分别用于配置子 Agent 组、同级 Agent 组、handoff、资源共享空间、安全配置和最大 Agent 深度。

**示例**：

~~~js
const config = {
  model: {
    model_id: "gpt-5.6-luna",
    provider: "openai",
    reasoning_effort: "high",
  },
  system_prompt: "只输出可执行的审查结论",
  capabilities: { brainary_tool: [BrainaryTool.exec_command] },
  runtime: { sandbox: Sandbox.read_only, approval_policy: ApprovalPolicy.deny_all },
};
~~~
