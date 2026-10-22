# PoA SDK API 参考

本文按新的 PoA SDK 对象边界组织公共 API。每个文件只描述一个对象或一个紧密相关的类型组，
类型按“作用、类型定义、示例”说明；函数和方法按“作用、输入类型、返回值类型、示例”说明。

## 文件索引

### 对象 API

| 文件 | 内容 |
| --- | --- |
| [IO](07-api-reference/06-io.md) | text、image、audio、yield_control、notify，以及输入和交互文档入口 |
| [global](07-api-reference/11-global.md) | set_timeout、clear_timeout、timed、exit |
| [helper](07-api-reference/12-helper.md) | parse_json_reply、map_limit、shape_of |
| [SkillsManager](07-api-reference/13-skills-manager.md) | tools 能力枚举、skills_list、skills_read |
| [AgentController](07-api-reference/05-agent-controller.md) | compact、close、steer |
| [AgentMonitor](07-api-reference/04-agent-monitor.md) | AgentState、TokenUsage、ContextUsage 和监控方法 |
| [Agent](07-api-reference/03agent.md) | Agent 生命周期、运行方法和 AgentResult |
| [AgentOrchestration](07-api-reference/01-agent-orchestration.md) | Agent 创建、任务派发、批量等待、Handle 和 RemoteAgent |
| [Executor](07-api-reference/09-executors.md) | 执行器上下文、Hooks、PreExecutorHooks、PostExecutorHooks、文件、终端、远程 Agent、MCP 调用 |
| [AgentConfig](07-api-reference/02-agent-config.md) | ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec 和 AgentConfig |

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

### 输入、交互与工具配置

| 文件 | 内容 |
| --- | --- |
| [PoA Input](07-api-reference/07-poa-input.md) | input_schema.ts、JSON Schema、POA_INPUT 和校验规则 |
| [ask()](07-api-reference/08-ask.md) | 问题结构、答案结构和交互模式 |
| [MCP tool 配置](07-api-reference/10-mcp-tool-config.md) | manifest.toml、MCP server、工具发现和生命周期 |

## 类型索引

| 类型或对象 | 对应文档 |
| --- | --- |
| AgentOrchestration、RemoteAgent、Handle | [AgentOrchestration](07-api-reference/01-agent-orchestration.md) |
| ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec、AgentConfig | [Agent 配置类型](07-api-reference/02-agent-config.md) |
| Agent、AgentResult | [Agent](07-api-reference/03agent.md) |
| AgentMonitor、AgentState、TokenUsage、ContextUsage | [AgentMonitor](07-api-reference/04-agent-monitor.md) |
| AgentController | [AgentController](07-api-reference/05-agent-controller.md) |
| IO、POA_INPUT | [IO](07-api-reference/06-io.md)、[PoA Input](07-api-reference/07-poa-input.md) |
| Executor、ExecutorContext、PostExecutorContext、Hooks、PreExecutorHooks、PostExecutorHooks | [执行器上下文与 hook](07-api-reference/09-executors.md#上下文与-hook) |
| FileExecutor、TerminalExecutor、RemoteAgentExecutor、McpExecutor | [各类执行器](07-api-reference/09-executors.md#fileexecutor) |
| global、helper | [global](07-api-reference/11-global.md)、[helper](07-api-reference/12-helper.md) |
| tools（枚举）、SkillsManager | [工具枚举与 SkillsManager](07-api-reference/13-skills-manager.md) |
| memory | [memory](07-api-reference/14-working-memory/01-memory.md) |
| BGraph | [BGraph](07-api-reference/14-working-memory/02-bgraph.md) |
| BTree | [BTree](07-api-reference/14-working-memory/03-btree.md) |
| BList、BMap | [BList 与 BMap](07-api-reference/14-working-memory/04-blist-bmap.md) |
| BNode、BEdge、BatchBuilder、BatchGraph、BatchTree、BatchList、BatchMap、CommitResult、BPage、BListItem、BMapItem、BNeighborItem | [Working Memory 元素与返回类型](07-api-reference/14-working-memory/05-elements.md) |
| WmError、WmUsageError、WmRequestError、WmErrorCode | [Working Memory 异常](07-api-reference/14-working-memory/06-errors.md) |
| SnapshotPage | [memory.snapshot](07-api-reference/14-working-memory/07-snapshot.md) |

## 共享约定

- 代码示例使用 TypeScript 风格类型，实际 PoA 程序可以使用 JavaScript。
- 函数和方法统一使用 snake_case 命名；类型名、配置字段和协议字段保留各自定义的名称。
- ModelSpec 的 model_id、provider、reasoningEffort 均为必填字段；所有 model 配置示例必须包含这三个字段。
- Promise<T> 表示异步方法；需要等待 Agent 完成时必须 await。
- JsonSchema 表示 JSON Schema 对象。传入 output_schema 后，AgentResult.output 是校验后的对象；
  不传时 output 是最终文本，text 始终保留原始最终回复。
- 参数或字段后的 ? 表示可选，未标注 ? 的表示必填；示例必须传入所有必填参数和字段。
  数组项的具体结构以对应接口定义为准。
- 同时声明 message? 和 input_items? 的接口要求二选一：必须且只能传入其中一个。
  未选择的参数应省略，后面还有参数时使用 undefined 占位；空数组 [] 也算传入 input_items，不能与 message 同时传入。
- 能力是否可用由 Capabilities 和运行时配置共同决定；执行器上下文可用于审计和 hook。
