# MCP 资源工具

MCP server 是否存在由宿主配置决定。资源工具返回值遵循 MCP 协议，实际结构可能随 server
变化；对 `unknown` 返回值先用 `JSON.stringify` 探测。

## `list_mcp_resources`

- 输入：`{ server?: string, cursor?: string }`。
- 返回：MCP 资源页（通常包含 `resources` 和 `nextCursor`）。

```js
const page = await tools.list_mcp_resources({ server: "filesystem" });
text(page);
```

## `list_mcp_resource_templates`

- 输入：`{ server?: string, cursor?: string }`。
- 返回：MCP 资源模板页（通常包含 `resourceTemplates` 和 `nextCursor`）。

```js
const templates = await tools.list_mcp_resource_templates({});
text(templates);
```

## `read_mcp_resource`

- 输入：`{ server: string, uri: string }`。
- 返回：MCP `ReadResourceResult`，可能包含文本或 base64 blob 内容。

```js
const resource = await tools.read_mcp_resource({ server: "filesystem", uri: "file:///tmp/a.txt" });
text(resource);
```

## MCP tool 通用调用

宿主会为每个 MCP server 的每个工具注册一个动态工具。常见名称形状是
`mcp__<server>__<tool>`，但命名空间清洗、截断和哈希可能改变最终名称；不要硬编码名称，
应从 `ALL_TOOLS` 找到实际项。输入 schema 和输出 schema 来自 MCP `tools/list`。

```js
const candidate = ALL_TOOLS.find((item) => item.name.endsWith("__echo"));
if (!candidate) throw new Error("echo tool is unavailable");
const result = await tools[candidate.name]({ message: "hello" });
text(result);
```

```ts
type CallToolResult<TStructured = Record<string, unknown>> = {
  content: Array<unknown>;
  isError?: boolean;
  structuredContent?: TStructured;
  [key: string]: unknown;
};
```

包自带 MCP server 的 SDK 式声明见 [PoA MCP Tool 扩展 SDK](08-poa-mcp-extension-sdk.md)。
