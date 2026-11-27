# AgentConfig 与 ModelSpec

本页是 [create_agent](03-agent-executor.md#create_agentname-description-agent_config) 的 agent_config 参数用到的类型。Brainary 拒绝配置时，create_agent 抛出 [MultiAgentError](01-agent.md#multiagenterror)，agent 不会创建。

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

### AgentConfig

**作用**：agent 的完整配置。

**类型定义**：

~~~ts
interface AgentConfig {
  model: ModelSpec;
  developer_instructions?: string;
  ephemeral?: boolean;
}
~~~

| 字段 | 说明 |
| --- | --- |
| model | 必填，见 [ModelSpec](#modelspec) |
| developer_instructions | agent 的开发者指令，代替当前程序的开发者指令；省略时与当前程序相同 |
| ephemeral | 为 true 时不保存这个 agent 的会话记录；默认为 false，即使本次运行不保存会话记录，创建的 agent 也会保存，见 [会话记录](../02-configuration.md#会话记录) |

**示例**：

~~~js
const config = {
  model: { model_id: "gpt-5.6-luna", reasoning_effort: "high" },
  developer_instructions: "只输出可执行的审查结论",
};
~~~
