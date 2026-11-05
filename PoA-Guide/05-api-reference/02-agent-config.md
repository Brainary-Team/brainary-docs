# AgentConfig、Capabilities、ModelSpec、RuntimeSpec、AgentGroup、ResourceShareSpace 与 SecureConfig

本页是 [create_agent](03-agent-executor.md#create_agentname-description-agent_config) 的 agent_config 参数用到的类型。配置不满足本页的约束时，create_agent 抛出错误，agent 不会创建。

以下示例假定已从根入口导入枚举：

~~~js
import { ApprovalPolicy, BrainaryTool, Sandbox } from "@brainary/poa-sdk";
~~~

### ModelSpec

**作用**：指定 agent 使用的模型。

**类型定义**：

~~~ts
interface ModelSpec {
  model_id: string;
  provider: string;
  reasoning_effort?: string;
}
~~~

| 字段 | 说明 |
| --- | --- |
| model_id | 必填，模型 ID，不能为空 |
| provider | 必填，provider 的 id：config.toml 中 `[model_providers.<id>]` 的 `<id>`，或内置 provider 的 id（如 openai） |
| reasoning_effort | 推理强度，如 "low"、"medium"、"high"，须为该模型支持的取值；省略时使用模型的默认值 |

provider 的配置方法见 [配置 provider](../02-configuration.md#配置-provider)。

**示例**：

~~~js
const model = { model_id: "gpt-5.6-luna", provider: "vendor", reasoning_effort: "high" };
~~~

### Capabilities

**作用**：列出 agent 可以使用的工具。用 create_agent 创建的 agent 只拥有这里列出的工具；省略 capabilities 时 agent 没有任何工具。

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

| 字段 | 说明 |
| --- | --- |
| brainary_tool | [BrainaryTool](06-brainary-tool.md) 成员列表。每一项都须是创建者（调用 create_agent 的程序或 agent）自己拥有的工具 |
| mcp | MCP 授权列表。`"<server>/*"` 授予该 server 的全部工具，`"<server>/<tool>"` 授予单个工具。server 须来自 config.toml 的 `[mcp_servers]` 或当前 PoA 包的 `[[capabilities.mcp]]`，工具须是创建者能使用的工具 |
| poa、skills | 尚未实现 |

**示例**：

~~~js
const capabilities = {
  brainary_tool: [BrainaryTool.exec_command, BrainaryTool.write_stdin],
  mcp: ["echo/*"],
};
~~~

### Sandbox

**作用**：agent 执行命令和修改文件的权限，从严到宽依次为 read_only、workspace_write、danger_full_access。

**类型定义**：

~~~ts
enum Sandbox {
  read_only = "read_only",                    // 只能读取文件
  workspace_write = "workspace_write",        // 可以修改工作目录内的文件
  danger_full_access = "danger_full_access",  // 不做限制
}
~~~

TypeScript 中须使用枚举成员，如 `Sandbox.read_only`。

**示例**：

~~~js
const sandbox = Sandbox.workspace_write;
~~~

### ApprovalPolicy

**作用**：agent 的命令需要超出沙箱的权限时的处理方式。

**类型定义**：

~~~ts
enum ApprovalPolicy {
  auto_review = "auto_review",  // 由自动审查决定是否放行
  deny_all = "deny_all",        // 一律拒绝
  // 尚未实现：manual_review = "manual_review"
}
~~~

auto_review 要求创建者自身会请求审批。`brainary poa` 运行的程序默认不请求审批，此时 create_agent 只接受 deny_all。TypeScript 中须使用枚举成员，如 `ApprovalPolicy.deny_all`。

**示例**：

~~~js
const approval_policy = ApprovalPolicy.deny_all;
~~~

### RuntimeSpec

**作用**：指定 agent 的工作目录、沙箱和审批策略。

**类型定义**：

~~~ts
interface RuntimeSpec {
  workspace?: string;
  sandbox?: Sandbox;
  approval_policy?: ApprovalPolicy;
}
~~~

| 字段 | 说明 |
| --- | --- |
| workspace | agent 的工作目录，须为已存在的目录；相对路径相对于创建者的工作目录。sandbox 为 workspace_write 时，须位于创建者可写的目录内 |
| sandbox | 不能比创建者的沙箱宽 |
| approval_policy | 见 [ApprovalPolicy](#approvalpolicy) |

省略的字段使用创建者的值。

**示例**：

~~~js
const runtime = {
  workspace: "./packages/core",
  sandbox: Sandbox.read_only,
  approval_policy: ApprovalPolicy.deny_all,
};
~~~

### AgentGroup

**状态**：尚未实现，以下为设计描述。

**作用**：把本地 agent 与远程 agent 组合为一组，供 subagents 和 peer_agents 配置使用。

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

**作用**：指定 agent 的资源共享空间、上下文分叉设置和资源上限。

**类型定义**：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `wm_share_space?` | `string[]` | 资源共享空间列表 |
| `fork_context` | `boolean` | 是否分叉上下文 |
| `token_limit` | `number` | token 上限 |
| `runing_time_limit` | `number` | 运行时长上限 |

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
| `allow_unstrusted_tools?` | `Set<string>` | 允许使用的不可信工具集合 |
| `block_on_violation?` | `boolean` | 是否在违规时阻塞 |
| `apprroval_on_violation?` | `boolean` | 是否在违规时请求审批 |
| `pending_approval_ttl?` | `timedelta` | 待审批请求的有效期 |

**示例**：

~~~js
const secureConfig = { block_on_violation: true };
~~~

### AgentConfig

**作用**：agent 的完整配置。

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

| 字段 | 说明 |
| --- | --- |
| model | 必填，见 [ModelSpec](#modelspec) |
| system_prompt | 追加在 agent 基础指令之后的系统提示。创建者自己的 system_prompt 不会传给它 |
| capabilities | 见 [Capabilities](#capabilities)；省略时 agent 没有工具 |
| runtime | 见 [RuntimeSpec](#runtimespec)；省略时与创建者相同 |
| subagents、peer_agents、handof、resource_share_space、secure_config、max_agent_depth | 尚未实现，分别用于配置子 agent 组、同级 agent 组、handoff、资源共享空间、安全配置和最大 agent 深度 |

**示例**：

~~~js
const config = {
  model: { model_id: "gpt-5.6-luna", provider: "vendor", reasoning_effort: "high" },
  system_prompt: "只输出可执行的审查结论",
  capabilities: { brainary_tool: [BrainaryTool.exec_command] },
  runtime: { sandbox: Sandbox.read_only, approval_policy: ApprovalPolicy.deny_all },
};
~~~
