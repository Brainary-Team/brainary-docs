# PoA SDK API 参考

本文按新的 PoA SDK 对象边界组织公共 API。每个文件只描述一个对象或一个紧密相关的类型组，
类型按“作用、类型定义、示例”说明；函数和方法按“作用、输入类型、返回值类型、示例”说明。
SDK 方法通过单个对象传入命名参数，可选字段直接省略。IO 输出函数和 JavaScript 标准定时器保持各自的调用形式。

## 文件索引

### 对象 API

| 文件 | 内容 |
| --- | --- |
| [IO](07-api-reference/06-io.md) | POA_INPUT、text、image、audio、yield_control、notify、ask |
| [global](07-api-reference/11-global.md) | setTimeout、clearTimeout、timed、exit |
| [helper](07-api-reference/12-helper.md) | parseJsonReply、mapLimit、shapeOf |
| [Tools](07-api-reference/13-tools-manager.md) | tools 能力枚举 |
| [AgentExecutor](07-api-reference/04-agent-executor.md) | send_message、context_compact、close、addition_message、状态与用量查询，以及 tools 入口 |
| [Agent](07-api-reference/03-agent.md) | run、append_message、context_compact、close、resume、状态与用量读取，以及 AgentResult |
| [AgentOrchestration](07-api-reference/01-agent-orchestration.md) | Agent 创建、SpawnSpec 派发、acquire_agent_result、Handle 和 RemoteAgent |
| [Executor](07-api-reference/09-executors.md) | 执行器上下文、Hooks、PreExecutorHooks、PostExecutorHooks、文件、终端、Agent、MCP 和 tools 入口 |
| [AgentConfig](07-api-reference/02-agent-config.md) | ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec、AgentGroup、ResourceShareSpace、SecureConfig 和 AgentConfig |

### Working Memory

这一组共七篇，另有一个入口页 [Working Memory](07-api-reference/14-working-memory.md) 汇总它们的共同约定。

| 文件 | 内容 |
| --- | --- |
| [memory](07-api-reference/14-working-memory/01-memory.md) | 集合访问器、batch、close_run_memory，以及 Working Memory 的作用域与版本规则 |
| [BGraph](07-api-reference/14-working-memory/02-bgraph.md) | 图的节点与边读写、一跳邻接和查询 |
| [BTree](07-api-reference/14-working-memory/03-btree.md) | 树的读写、约束模型和层级查询 |
| [BList 与 BMap](07-api-reference/14-working-memory/04-blist-bmap.md) | 有序项追加与键值读写 |
| [Working Memory 元素与返回类型](07-api-reference/14-working-memory/05-elements.md) | BNode、BEdge、BatchBuilder 与批次专用句柄、CommitResult、BPage 和三个项类型 |
| [Working Memory 异常](07-api-reference/14-working-memory/06-errors.md) | WmError、WmUsageError、WmRequestError 和错误码 |
| [memory.snapshot](07-api-reference/14-working-memory/07-snapshot.md) | 快照创建、列举、只读查询和 SnapshotPage |

### Episodic Memory（设计预览）

[Episodic Memory](07-api-reference/15-episodic-memory.md) 介绍情景记录与 Working Memory 快照的关系，以及七个 operator 的设计职责。目前没有可调用的 PoA API。

### 输入、交互与工具配置

| 文件 | 内容 |
| --- | --- |
| [MessageItem](07-api-reference/05-message-item.md) | ImageDetail，以及文本、图片、本地图片、音频、本地音频、skill 和 mention 消息项 |
| [PoA Input](07-api-reference/07-poa-input.md) | input_schema.ts、JSON Schema、POA_INPUT 和校验规则 |
| [ask()](07-api-reference/08-ask.md) | 问题结构、答案结构和交互模式 |
| [MCP tool 配置](07-api-reference/10-mcp-tool-config.md) | manifest.toml、MCP server、工具发现和生命周期 |

## 类型索引

| 类型或对象 | 对应文档 |
| --- | --- |
| AgentOrchestration、RemoteAgent、Handle、SpawnSpec | [AgentOrchestration](07-api-reference/01-agent-orchestration.md) |
| ImageDetail、MessageItem | [Agent 消息项类型](07-api-reference/05-message-item.md) |
| ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec、AgentConfig | [Agent 配置类型](07-api-reference/02-agent-config.md) |
| AgentGroup | [Agent 分组](07-api-reference/02-agent-config.md#agentgroup) |
| ResourceShareSpace | [资源共享空间与预算配置](07-api-reference/02-agent-config.md#resourcesharespace) |
| SecureConfig | [安全配置](07-api-reference/02-agent-config.md#secureconfig) |
| Handof | [AgentConfig](07-api-reference/02-agent-config.md#agentconfig) |
| Agent | [Agent](07-api-reference/03-agent.md) |
| AgentResult | [运行结果（JSON 对象 schema_result、原始文本 result 和 agent_state）](07-api-reference/03-agent.md#agentresult) |
| AgentState | [Agent 状态枚举](07-api-reference/04-agent-executor.md#agentstate) |
| AgentExecutor、TokenUsage、ContextUsage | [AgentExecutor](07-api-reference/04-agent-executor.md) |
| IO、POA_INPUT | [IO](07-api-reference/06-io.md)、[PoA Input](07-api-reference/07-poa-input.md) |
| InputParams | [PoA 输入类型](07-api-reference/07-poa-input.md#input_schemats) |
| PoaInputValidationFailure | [输入校验失败结构](07-api-reference/07-poa-input.md#校验失败结构) |
| AskQuestionOption、AskQuestion、AskRequest | [ask 请求类型](07-api-reference/08-ask.md#函数签名) |
| AskAnswer、AskResult | [ask 返回类型](07-api-reference/08-ask.md#返回结构) |
| Executor、ExecutorContext、PostExecutorContext、Hooks、PreExecutorHooks、PostExecutorHooks | [执行器上下文与 hook](07-api-reference/09-executors.md#上下文与-hook) |
| FileExecutor、TerminalExecutor、McpExecutor | [各类执行器](07-api-reference/09-executors.md#fileexecutor) |
| RemoteAgentExecutor | [Executor](07-api-reference/09-executors.md#executor) |
| CallToolResult | [MCP 工具返回类型](07-api-reference/10-mcp-tool-config.md#calltoolresult) |
| global、helper | [global](07-api-reference/11-global.md)、[helper](07-api-reference/12-helper.md) |
| Tool（tools 能力枚举） | [Tools](07-api-reference/13-tools-manager.md#tools) |
| memory | [memory](07-api-reference/14-working-memory/01-memory.md) |
| BGraph | [BGraph](07-api-reference/14-working-memory/02-bgraph.md) |
| BTree | [BTree](07-api-reference/14-working-memory/03-btree.md) |
| BList、BMap | [BList 与 BMap](07-api-reference/14-working-memory/04-blist-bmap.md) |
| BNode、BEdge、BatchBuilder、BatchGraph、BatchTree、BatchList、BatchMap、CommitResult、BPage、BListItem、BMapItem、BNeighborItem | [Working Memory 元素与返回类型](07-api-reference/14-working-memory/05-elements.md) |
| WmError、WmUsageError、WmRequestError、WmErrorCode | [Working Memory 异常](07-api-reference/14-working-memory/06-errors.md) |
| SnapshotPage | [memory.snapshot](07-api-reference/14-working-memory/07-snapshot.md) |
