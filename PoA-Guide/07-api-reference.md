# PoA SDK API 参考

本文按新的 PoA SDK 对象边界组织公共 API。每个文件只描述一个对象或一个紧密相关的类型组，
类型按“作用、类型定义、示例”说明；函数和方法按“作用、输入类型、返回值类型、示例”说明。

## 文件索引

| 文件 | 内容 |
| --- | --- |
| [IO](07-api-reference/06-io.md) | text、image、audio、yield_control、notify，以及输入和交互文档入口 |
| [global](07-api-reference/11-global.md) | set_timeout、clear_timeout、timed、exit |
| [helper](07-api-reference/12-helper.md) | parse_json_reply、map_limit、shape_of |
| [Tools](07-api-reference/13-tools-manager.md) | tools 能力枚举 |
| [AgentController](07-api-reference/05-agent-controller.md) | compact、close、steer |
| [AgentMonitor](07-api-reference/04-agent-monitor.md) | AgentState、TokenUsage、ContextUsage 和监控方法 |
| [Agent](07-api-reference/03agent.md) | Agent 生命周期、运行方法 |
| [AgentOrchestration](07-api-reference/01-agent-orchestration.md) | Agent 创建、任务派发、批量等待、Handle 和 RemoteAgent |
| [Executor](07-api-reference/09-executors.md) | 执行器上下文、Hooks、PreExecutorHooks、PostExecutorHooks、文件、终端、远程 Agent、MCP 调用 |
| [AgentConfig](07-api-reference/02-agent-config.md) | ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec、ResourceShareSpace、SecureConfig 和 AgentConfig |
| [PoA Input](07-api-reference/07-poa-input.md) | input_schema.ts、JSON Schema、POA_INPUT 和校验规则 |
| [ask()](07-api-reference/08-ask.md) | 问题结构、答案结构和交互模式 |
| [MCP tool 配置](07-api-reference/10-mcp-tool-config.md) | manifest.toml、MCP server、工具发现和生命周期 |

## 类型索引

| 类型或对象 | 对应文档 |
| --- | --- |
| AgentOrchestration、RemoteAgent、Handle | [AgentOrchestration](07-api-reference/01-agent-orchestration.md) |
| ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec、AgentConfig | [Agent 配置类型](07-api-reference/02-agent-config.md) |
| ResourceShareSpace | [资源共享空间配置](07-api-reference/02-agent-config.md#resourcesharespace) |
| SecureConfig | [安全配置](07-api-reference/02-agent-config.md#secureconfig) |
| Agent | [Agent](07-api-reference/03agent.md) |
| AgentResult | [运行结果（含 agent_state）](07-api-reference/03agent.md#agentresult) |
| AgentState | [Agent 状态枚举](07-api-reference/04-agent-monitor.md#agentstate) |
| AgentMonitor、TokenUsage、ContextUsage | [AgentMonitor](07-api-reference/04-agent-monitor.md) |
| AgentController | [AgentController](07-api-reference/05-agent-controller.md) |
| IO、POA_INPUT | [IO](07-api-reference/06-io.md)、[PoA Input](07-api-reference/07-poa-input.md) |
| InputParams | [PoA 输入类型](07-api-reference/07-poa-input.md#input_schemats) |
| PoaInputValidationFailure | [输入校验失败结构](07-api-reference/07-poa-input.md#校验失败结构) |
| AskQuestionOption、AskQuestion、AskRequest | [ask 请求类型](07-api-reference/08-ask.md#函数签名) |
| AskAnswer、AskResult | [ask 返回类型](07-api-reference/08-ask.md#返回结构) |
| Executor、ExecutorContext、PostExecutorContext、Hooks、PreExecutorHooks、PostExecutorHooks | [执行器上下文与 hook](07-api-reference/09-executors.md#上下文与-hook) |
| FileExecutor、TerminalExecutor、RemoteAgentExecutor、McpExecutor | [各类执行器](07-api-reference/09-executors.md#fileexecutor) |
| CallToolResult | [MCP 工具返回类型](07-api-reference/10-mcp-tool-config.md#calltoolresult) |
| global、helper | [global](07-api-reference/11-global.md)、[helper](07-api-reference/12-helper.md) |
| Tool（tools 能力枚举） | [Tools](07-api-reference/13-tools-manager.md#tools) |
