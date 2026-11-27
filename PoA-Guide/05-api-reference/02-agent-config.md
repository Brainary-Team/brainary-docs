# AgentConfig、ModelSpec、AgentGroup、ResourceShareSpace 与 SecureConfig

本页是 [create_agent](03-agent-executor.md#create_agentrole-nickname-description-agent_config) 的 agent_config 参数用到的类型。Brainary 拒绝配置时，create_agent 抛出 [MultiAgentError](01-agent.md#multiagenterror)，agent 不会创建。

agent_config 只设置模型和开发者指令。agent 的 provider、沙箱、审批策略和工作目录与当前程序相同，见 [配置文件](../02-configuration.md)；agent 可以使用的工具见 [agent 的工具](#agent-的工具)。

### agent 的工具

agent 不能单独分配工具。spawn_agent 和 create_agent 得到的 agent 按程序的配置拿到同一组工具：

| 工具 | 作用 |
| --- | --- |
| exec、wait | 在代码模式中运行 JavaScript，组合调用其他工具 |
| exec_command、write_stdin | 运行命令，与运行中的命令交互 |
| apply_patch | 新增、删除和修改文件 |
| view_image | 查看本地图片 |
| update_plan | 记录任务计划 |
| request_user_input | 向用户提问 |
| get_goal、create_goal、update_goal | 读取、设置和更新任务目标 |
| web_search | 搜索网页 |
| config.toml 中 `[mcp_servers]` 声明的 MCP server 的工具，以及 list_mcp_resources、list_mcp_resource_templates、read_mcp_resource | 调用 MCP 工具，列出和读取 MCP 资源；没有声明 MCP server 时没有这些工具 |

- agent 在 exec 中可以再创建和管理 agent，能否创建由 [max_depth](../02-configuration.md#agent-层级和数量) 决定：为默认的 1 时，agent 不能再创建 agent。
- 随包携带的 MCP server（manifest.toml 的 `[[capabilities.mcp]]`）只供程序自己用 [McpExecutor](05-executors.md#mcpexecutor) 调用，agent 拿不到。
- 执行命令和修改文件受程序的 [沙箱](../02-configuration.md#沙箱) 限制。

### ModelSpec

**作用**：指定 agent 使用的模型。

**类型定义**：

~~~ts
interface ModelSpec {
  model_id: string;
  reasoning_effort?: string;
}
~~~

| 字段 | 说明 |
| --- | --- |
| model_id | 必填，模型 ID，须是当前 provider 提供的模型，否则抛出 MultiAgentError（UNKNOWN_MODEL） |
| reasoning_effort | 推理强度，如 "low"、"medium"、"high"，须为该模型支持的取值，否则抛出 MultiAgentError（UNSUPPORTED_REASONING_EFFORT）；省略时使用模型的默认值 |

当前 provider 是 config.toml 中 `model_provider` 指定的 provider，见 [配置 provider](../02-configuration.md#配置-provider)。

**示例**：

~~~js
const model = { model_id: "gpt-5.6-luna", reasoning_effort: "high" };
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
  developer_instructions?: string;
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
| developer_instructions | agent 的开发者指令，代替当前程序的开发者指令；省略时与当前程序相同 |
| subagents、peer_agents、handof、resource_share_space、secure_config、max_agent_depth | 尚未实现，分别用于配置子 agent 组、同级 agent 组、handoff、资源共享空间、安全配置和最大 agent 深度 |

**示例**：

~~~js
const config = {
  model: { model_id: "gpt-5.6-luna", reasoning_effort: "high" },
  developer_instructions: "只输出可执行的审查结论",
};
~~~
