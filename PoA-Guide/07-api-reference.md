# PoA SDK API 参考

本文按新的 PoA SDK 对象边界组织公共 API。每个文件只描述一个对象或一个紧密相关的类型组，
类型按“作用、类型定义、示例”说明；函数和方法按“作用、输入类型、返回值类型、示例”说明。
SDK 以 ESM 包 `@brainary/poa-sdk` 提供：根入口导出全部公开 API，`/io`、`/global`、`/helper`、`/executor`
四个子入口分别导出对应模块。SDK 方法按位置传参，可选参数直接省略；create_agent 和 spawn_agent 接收单个参数对象。
设计中尚未实现的类型、字段和方法仍保留在对应页面，并标注“尚未实现”。

PoA 运行环境没有模块加载能力，入口须与 SDK 一起打包为单个 ESM 文件后运行：

```sh
npx esbuild main.ts --bundle --format=esm --platform=neutral --target=es2022 --outfile=main.js
```

## 文件索引

### 对象 API

| 文件 | 内容 |
| --- | --- |
| [Agent](07-api-reference/01-agent.md) | run、close、resume、status、AgentResult 和错误类型；append_message、context_compact 与用量读取尚未实现 |
| [AgentConfig](07-api-reference/02-agent-config.md) | ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec 和 AgentConfig；AgentGroup、ResourceShareSpace 与 SecureConfig 尚未实现 |
| [AgentExecutor](07-api-reference/03-agent-executor.md) | create_agent、spawn_agent、acquire_agent_result、get_status、close、resume 和 tools，以及 AgentStatus、Handle 和 SpawnSpec；create_remote_agent、send_message、context_compact、append_message 与用量查询尚未实现 |
| [Executor](07-api-reference/05-executors.md) | Executor 基类、FileExecutor、TerminalExecutor、McpExecutor、ShellCommandError 和 tools 入口；执行器上下文与 Hooks 尚未实现 |
| [BrainaryTool](07-api-reference/07-brainary-tool.md) | 宿主工具枚举 |
| [IO](07-api-reference/08-io.md) | POA_INPUT、text、image、audio、yield_control、notify、ask |
| [global](07-api-reference/11-global.md) | setTimeout、clearTimeout、timed、exit |
| [helper](07-api-reference/12-helper.md) | parse_json_reply、map_limit、shape_of |

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
| [MessageItem](07-api-reference/04-message-item.md) | ImageDetail，以及文本、图片、本地图片、音频、本地音频、skill 和 mention 消息项 |
| [MCP tool 配置](07-api-reference/06-mcp-tool-config.md) | manifest.toml、MCP server、工具名推导、CallToolResult 和生命周期 |
| [PoA Input](07-api-reference/09-poa-input.md) | input_schema.ts、JSON Schema、POA_INPUT 和校验规则 |
| [ask()](07-api-reference/10-ask.md) | 问题结构、答案结构和交互模式 |

## 类型索引

| 类型或对象 | 对应文档 |
| --- | --- |
| Agent | [Agent](07-api-reference/01-agent.md) |
| AgentResult | [运行结果（最终回复 result、错误 error、agent_status 和保留的 schema_result）](07-api-reference/01-agent.md#agentresult) |
| AgentNotIdleError、AgentResultTimeoutError | [Agent 错误类型](07-api-reference/01-agent.md#错误) |
| ModelSpec、Capabilities、McpGrant、Sandbox、ApprovalPolicy、RuntimeSpec、AgentConfig | [Agent 配置类型](07-api-reference/02-agent-config.md) |
| AgentGroup | [Agent 分组](07-api-reference/02-agent-config.md#agentgroup) |
| ResourceShareSpace | [资源共享空间与预算配置](07-api-reference/02-agent-config.md#resourcesharespace) |
| SecureConfig | [安全配置](07-api-reference/02-agent-config.md#secureconfig) |
| Handof | [AgentConfig](07-api-reference/02-agent-config.md#agentconfig) |
| AgentExecutor、CreateAgentSpec、TokenUsage、ContextUsage | [AgentExecutor](07-api-reference/03-agent-executor.md) |
| AgentStatus | [Agent 状态枚举](07-api-reference/03-agent-executor.md#agentstatus) |
| Handle、SpawnSpec、RemoteAgent | [派发句柄与派发规格](07-api-reference/03-agent-executor.md#handle) |
| ImageDetail、MessageItem | [Agent 消息项类型](07-api-reference/04-message-item.md) |
| Executor、ExecutorKind | [Executor 基类](07-api-reference/05-executors.md#executor) |
| ExecutorContext、PostExecutorContext、Hooks、PreExecutorHooks、PostExecutorHooks | [执行器上下文与 hook](07-api-reference/05-executors.md#上下文与-hook) |
| FileExecutor、ViewImageResult | [FileExecutor](07-api-reference/05-executors.md#fileexecutor) |
| TerminalExecutor、ExecCommandOptions、ExecOutput、EnvironmentReady、ShellCommandError | [TerminalExecutor](07-api-reference/05-executors.md#terminalexecutor) |
| McpExecutor、McpResourcePage、McpResourceTemplatePage、McpResourceRead | [McpExecutor](07-api-reference/05-executors.md#mcpexecutor) |
| CallToolResult、McpToolResult、McpContentBlock | [MCP 工具返回类型](07-api-reference/06-mcp-tool-config.md#calltoolresult) |
| BrainaryTool | [BrainaryTool](07-api-reference/07-brainary-tool.md) |
| IO、POA_INPUT | [IO](07-api-reference/08-io.md)、[PoA Input](07-api-reference/09-poa-input.md) |
| ImageInput、ImageContent、AudioInput、AudioContent | [IO 媒体输入类型](07-api-reference/08-io.md) |
| InputParams | [PoA 输入类型](07-api-reference/09-poa-input.md#input_schemats) |
| PoaInputValidationFailure | [输入校验失败结构](07-api-reference/09-poa-input.md#校验失败结构) |
| AskQuestionOption、AskQuestion、AskRequest | [ask 请求类型](07-api-reference/10-ask.md#函数签名) |
| AskAnswer、AskResult | [ask 返回类型](07-api-reference/10-ask.md#返回结构) |
| global、TimeoutId、TimedResult | [global](07-api-reference/11-global.md) |
| helper、JsonReply、JsonValue | [helper](07-api-reference/12-helper.md) |
| memory | [memory](07-api-reference/14-working-memory/01-memory.md) |
| BGraph | [BGraph](07-api-reference/14-working-memory/02-bgraph.md) |
| BTree | [BTree](07-api-reference/14-working-memory/03-btree.md) |
| BList、BMap | [BList 与 BMap](07-api-reference/14-working-memory/04-blist-bmap.md) |
| BNode、BEdge、BatchBuilder、BatchGraph、BatchTree、BatchList、BatchMap、CommitResult、BPage、BListItem、BMapItem、BNeighborItem | [Working Memory 元素与返回类型](07-api-reference/14-working-memory/05-elements.md) |
| WmError、WmUsageError、WmRequestError、WmErrorCode | [Working Memory 异常](07-api-reference/14-working-memory/06-errors.md) |
| SnapshotPage | [memory.snapshot](07-api-reference/14-working-memory/07-snapshot.md) |
