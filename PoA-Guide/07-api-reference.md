# PoA API 简表

## 拆分后的 API 文档

原简表中的接口已经按功能拆分为以下文档。每个文档都说明工具作用、输入、返回结构和最小示例。

| 文档 | 内容 |
| --- | --- |
| [全局 Primitive](07-api-reference/01-global-primitives.md) | 输出、状态、定时器、`ALL_TOOLS`、`POA_INPUT`、`tools` |
| [Prelude 编排（v1）](07-api-reference/02-prelude.md) | v1 agent 编排和通用 JavaScript helper |
| [执行与文件](07-api-reference/03-exec-files.md) | `exec_command`、`write_stdin`、`shell_command`、`apply_patch`、`view_image` |
| [计划、上下文与记忆](07-api-reference/04-plan-context-memory.md) | 计划、目标、权限、时间和 memory 工具 |
| [子代理 v1](07-api-reference/05-agent-v1.md) | `multi_agent_v1__*` 原生接口 |
| [MCP 资源](07-api-reference/06-mcp-resources.md) | 资源列表、模板、读取和动态 MCP tool 调用 |
| [POA INPUT SDK](07-api-reference/07-poa-input-sdk.md) | `input_schema.ts`、JSON Schema、`POA_INPUT`、输入校验和调用方输入 |
| [`ask()` 交互](07-api-reference/08-ask.md) |
| [PoA MCP Tool 扩展 SDK](07-api-reference/09-poa-mcp-extension-sdk.md) | `manifest.toml`、stdio server、工具发现、`CallToolResult` 和生命周期 |
| [Provider 扩展工具](07-api-reference/10-provider-tools.md) | web、skills、图片生成、环境和插件工具 |
