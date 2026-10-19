# PoA SDK API 参考

本文按新的 PoA SDK 对象边界组织公共 API。每个文件只描述一个对象或一个紧密相关的类型组，
每个方法均按“作用、输入类型、返回值类型、示例”说明。

## 文件索引

| 文件 | 内容 |
| --- | --- |
| [IO](07-api-reference/06-io.md) | text、image、audio、yield_control、notify，以及输入和交互文档入口 |
| [global](07-api-reference/11-global.md) | setTimeout、clearTimeout、timed、exit |
| [helper](07-api-reference/12-helper.md) | parseJsonReply、mapLimit、shapeOf |
| [SkillsManager](07-api-reference/13-skills-manager.md) | tools 能力枚举、skills_list、skills_read |
| [AgentController](07-api-reference/05-agent-controller.md) | compact、close、steer |
| [AgentMonitor](07-api-reference/04-agent-monitor.md) | AgentState、TokenUsage、ContextUsage 和监控方法 |
| [Agent](07-api-reference/03agent.md) | Agent 生命周期、运行方法和 AgentResult |
| [AgentOrchestration](07-api-reference/01-agent-orchestration.md) | Agent 创建、任务派发、批量等待、Handle 和 RemoteAgent |
| [Executor](07-api-reference/09-executors.md) | 执行器上下文、Hooks、文件、终端、远程 Agent、MCP 调用 |
| [AgentConfig](07-api-reference/02-agent-config.md) | ModelSpec、Capabilities、Sandbox、ApprovalPolicy、RuntimeSpec 和 AgentConfig |
| [PoA Input](07-api-reference/07-poa-input.md) | 从 Git 恢复的 input_schema.ts、JSON Schema、POA_INPUT 和校验规则 |
| [ask()](07-api-reference/08-ask.md) | 从 Git 恢复的问题结构、答案结构和交互模式 |
| [MCP tool 配置](07-api-reference/10-mcp-tool-config.md) | 从 Git 恢复的 manifest.toml、MCP server、工具发现和生命周期 |

## 共享约定

- 代码示例使用 TypeScript 风格类型，实际 PoA 程序可以使用 JavaScript。
- Promise<T> 表示异步方法；需要等待 Agent 完成时必须 await。
- JsonSchema 表示 JSON Schema 对象。传入 output_schema 后，AgentResult.output 是校验后的对象；
  不传时 output 是最终文本，text 始终保留原始最终回复。
- 图中的 ? 表示可选字段；数组项的具体结构以对应接口定义为准。
- 能力是否可用由 Capabilities 和运行时配置共同决定；执行器上下文可用于审计和 hook。
