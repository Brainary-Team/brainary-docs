# PoA SDK API 参考

`@brainary/poa-sdk` 是 ESM 包。全部 API 都可以从根入口导入，部分 API 也可以从子入口导入：

| 入口 | 导出 |
| --- | --- |
| `@brainary/poa-sdk` | 全部 API |
| `@brainary/poa-sdk/io` | POA_INPUT、text、image、audio、yield_control、notify、ask 及其类型 |
| `@brainary/poa-sdk/global` | setTimeout、clearTimeout、timed、exit |
| `@brainary/poa-sdk/helper` | parse_json_reply、map_limit、shape_of |
| `@brainary/poa-sdk/executor` | AgentExecutor、FileExecutor、TerminalExecutor、McpExecutor、BrainaryTool、ShellCommandError 及其类型 |

Agent、AgentResult、AgentStatus、MessageItem、AgentConfig 及其字段类型、Sandbox、ApprovalPolicy、AgentNotIdleError 和 AgentResultTimeoutError 只从根入口导出。

调用约定：

- 方法按位置传参。跳过中间的可选参数时传 `undefined`，例如 `agent.run("检查 README", undefined, undefined, 120_000)`。
- create_agent 和 spawn_agent 接收一个对象参数。
- 标注“尚未实现”的类型、字段和方法当前不可用。

## 文件索引

| 文件 | 内容 |
| --- | --- |
| [Agent](05-api-reference/01-agent.md) | run、close、resume、status，AgentResult 和错误类型；append_message、context_compact 与用量读取尚未实现 |
| [AgentConfig](05-api-reference/02-agent-config.md) | ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec 和 AgentConfig；AgentGroup、ResourceShareSpace 与 SecureConfig 尚未实现 |
| [AgentExecutor](05-api-reference/03-agent-executor.md) | create_agent、spawn_agent、acquire_agent_result、get_status、close、resume 和 tools，以及 AgentStatus、Handle 和 SpawnSpec；create_remote_agent、send_message、context_compact、append_message 与用量查询尚未实现 |
| [MessageItem](05-api-reference/04-message-item.md) | ImageDetail，以及文本、图片、本地图片、音频、本地音频、skill 和 mention 消息项 |
| [Executor](05-api-reference/05-executors.md) | FileExecutor、TerminalExecutor、McpExecutor、ShellCommandError、McpToolResult 和 tools；执行器上下文与 Hooks 尚未实现 |
| [BrainaryTool](05-api-reference/06-brainary-tool.md) | 宿主工具枚举 |
| [IO](05-api-reference/07-io.md) | POA_INPUT、text、image、audio、yield_control、notify、ask |
| [global](05-api-reference/08-global.md) | setTimeout、clearTimeout、timed、exit |
| [helper](05-api-reference/09-helper.md) | parse_json_reply、map_limit、shape_of |

### Working Memory

使用步骤见 [quickstart](05-api-reference/10-working-memory/00-quickstart.md)。入口页 [Working Memory](05-api-reference/10-working-memory.md) 汇总以下七篇的共同约定。

| 文件 | 内容 |
| --- | --- |
| [memory](05-api-reference/10-working-memory/01-memory.md) | 集合访问器、batch、close_run_memory，以及 Working Memory 的作用域与版本规则 |
| [BGraph](05-api-reference/10-working-memory/02-bgraph.md) | 图的节点与边读写、一跳邻接和查询 |
| [BTree](05-api-reference/10-working-memory/03-btree.md) | 树的读写、约束模型和层级查询 |
| [BList 与 BMap](05-api-reference/10-working-memory/04-blist-bmap.md) | 有序项追加与键值读写 |
| [Working Memory 元素与返回类型](05-api-reference/10-working-memory/05-elements.md) | BNode、BEdge、BatchBuilder 与批次专用句柄、CommitResult、BPage 和三个项类型 |
| [Working Memory 异常](05-api-reference/10-working-memory/06-errors.md) | WmError、WmUsageError、WmRequestError 和错误码 |
| [memory.snapshot](05-api-reference/10-working-memory/07-snapshot.md) | 快照创建、列举、只读查询和 SnapshotPage |

### Episodic Memory（设计预览）

[Episodic Memory](05-api-reference/11-episodic-memory.md) 介绍情景记录与 Working Memory 快照的关系，以及七个 operator 的设计职责。目前没有可调用的 PoA API。

## 类型索引

| 类型或对象 | 对应文档 |
| --- | --- |
| Agent | [Agent](05-api-reference/01-agent.md) |
| AgentResult | [AgentResult](05-api-reference/01-agent.md#agentresult) |
| AgentNotIdleError、AgentResultTimeoutError | [Agent 错误类型](05-api-reference/01-agent.md#错误) |
| ModelSpec、Capabilities、McpGrant、Sandbox、ApprovalPolicy、RuntimeSpec、AgentConfig | [AgentConfig](05-api-reference/02-agent-config.md) |
| AgentGroup | [AgentGroup](05-api-reference/02-agent-config.md#agentgroup) |
| ResourceShareSpace | [ResourceShareSpace](05-api-reference/02-agent-config.md#resourcesharespace) |
| SecureConfig | [SecureConfig](05-api-reference/02-agent-config.md#secureconfig) |
| Handof | [AgentConfig](05-api-reference/02-agent-config.md#agentconfig) |
| AgentExecutor、CreateAgentSpec、TokenUsage、ContextUsage | [AgentExecutor](05-api-reference/03-agent-executor.md) |
| AgentStatus | [AgentStatus](05-api-reference/03-agent-executor.md#agentstatus) |
| Handle、SpawnSpec、RemoteAgent | [Handle](05-api-reference/03-agent-executor.md#handle)、[SpawnSpec](05-api-reference/03-agent-executor.md#spawnspec)、[RemoteAgent](05-api-reference/03-agent-executor.md#remoteagent) |
| ImageDetail、MessageItem | [MessageItem](05-api-reference/04-message-item.md) |
| Executor | [Executor](05-api-reference/05-executors.md#executor) |
| ExecutorContext、PostExecutorContext、Hooks、PreExecutorHooks、PostExecutorHooks | [执行器上下文与 hook](05-api-reference/05-executors.md#上下文与-hook) |
| FileExecutor、ViewImageResult | [FileExecutor](05-api-reference/05-executors.md#fileexecutor) |
| TerminalExecutor、ExecCommandOptions、ExecOutput、EnvironmentReady、ShellCommandError | [TerminalExecutor](05-api-reference/05-executors.md#terminalexecutor) |
| McpExecutor、McpResourcePage、McpResourceTemplatePage、McpResourceRead | [McpExecutor](05-api-reference/05-executors.md#mcpexecutor) |
| McpToolResult、McpContentBlock | [McpToolResult](05-api-reference/05-executors.md#mcptoolresult) |
| BrainaryTool | [BrainaryTool](05-api-reference/06-brainary-tool.md) |
| POA_INPUT、JsonValue | [POA_INPUT](05-api-reference/07-io.md#poa_input) |
| ImageInput、ImageContent、AudioInput、AudioContent | [IO](05-api-reference/07-io.md) |
| AskQuestionOption、AskQuestion、AskRequest | [AskRequest 类型](05-api-reference/07-io.md#askrequest-类型) |
| AskAnswer、AskResult | [AskResult 类型](05-api-reference/07-io.md#askresult-类型) |
| TimeoutId、TimedResult | [global](05-api-reference/08-global.md) |
| JsonReply | [helper](05-api-reference/09-helper.md) |
| memory | [memory](05-api-reference/10-working-memory/01-memory.md) |
| BGraph | [BGraph](05-api-reference/10-working-memory/02-bgraph.md) |
| BTree | [BTree](05-api-reference/10-working-memory/03-btree.md) |
| BList、BMap | [BList 与 BMap](05-api-reference/10-working-memory/04-blist-bmap.md) |
| BNode、BEdge、BatchBuilder、BatchGraph、BatchTree、BatchList、BatchMap、CommitResult、BPage、BListItem、BMapItem、BNeighborItem | [Working Memory 元素与返回类型](05-api-reference/10-working-memory/05-elements.md) |
| WmError、WmUsageError、WmRequestError、WmErrorCode | [Working Memory 异常](05-api-reference/10-working-memory/06-errors.md) |
| SnapshotPage | [memory.snapshot](05-api-reference/10-working-memory/07-snapshot.md) |
