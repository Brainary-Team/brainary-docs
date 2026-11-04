# PoA MCP Tool 扩展 SDK

本页说明如何把 MCP server 随 `.poa` 包分发，并让宿主在运行该包的 thread 中发现和调用
这些工具。实现对应 `codex-rs/poa`、`codex-rs/ext/poa` 和 MCP tool 注册链路。

## `manifest.toml` 声明

包根目录必须包含 `[poa]` 元数据；每个包内 MCP server 使用一个
`[[capabilities.mcp]]` 表项：

```toml
[poa]
namespace = "example"
name = "echo_demo"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"

[[capabilities.mcp]]
name = "echo"
type = "stdio"
shell = "node mcp/echo.mjs"
description = "回显消息"
```

字段结构：

| 字段 | 类型与约束 | 作用 |
| --- | --- | --- |
| `name` | 非空字符串，只允许字母、数字、`_`、`-` | server 名，参与工具注册名生成 |
| `type` | 当前只能是 `"stdio"` | 指定 MCP transport |
| `shell` | 非空命令字符串 | 由宿主拆为直接子进程的 argv，不通过 shell 执行 |
| `description` | 可选字符串 | server 说明 |

manifest 解析成功时得到一个已校验的 server 声明；字段缺失、transport 不支持、
命令语法无效或路径不合法时，包在启动前被拒绝。

## MCP server 最小实现

server 通过 MCP `tools/list` 声明一个或多个工具。每个工具提供名称、输入 schema 和可选
输出 schema；以下为伪代码，具体 SDK 由包作者选择：

```js
// mcp/echo.mjs
server.tool(
  "echo",
  { message: "string", times: "number?" },
  async ({ message, times = 1 }) => ({
    structuredContent: {
      echoed: Array(times).fill(message).join(" "),
      count: times,
    },
    content: [{ type: "text", text: message }],
  }),
);
```

工具调用输入结构由 server 在 `tools/list` 返回的 input schema 决定，例如：

```json
{ "message": "你好", "times": 2 }
```

## 工具发现与动态名称

宿主启动声明的 stdio server，调用 MCP `tools/list`，把每个工具提供给当前 PoA，并加入全局 `ALL_TOOLS`。
工具在宿主中的名称通常是：

```text
mcp__<server_name>__<tool_name>
```

例如上面的 server 和工具都叫 `echo` 时为 `mcp__echo__echo`。SDK 的
[McpExecutor.call_mcp_tool](05-executors.md#call_mcp_toolserver-tool-args) 直接使用 server 名和工具名调用，无需拼接工具名：

```js
import { McpExecutor } from "@brainary/poa-sdk/executor";
import { text } from "@brainary/poa-sdk/io";

const result = await new McpExecutor().call_mcp_tool("echo", "echo", {
  message: "你好",
  times: 2,
});
text(result.structuredContent ?? result.content);
```

`ALL_TOOLS` 是全局工具列表，类型为 `Array<{ name: string; description: string }>`，可用于查看当前可用的工具及其说明。
工具的输入和返回 schema 取决于 MCP server 的声明。

## `CallToolResult`

MCP 工具的返回结构在宿主声明中写作 `CallToolResult`，SDK 中对应类型为 `McpToolResult`：

```ts
interface McpToolResult<S = Record<string, unknown>> {
  content: McpContentBlock[];
  isError?: boolean;
  structuredContent?: S;
}

type McpContentBlock =
  | { type: "text"; text: string; annotations?: unknown; _meta?: Record<string, unknown> }
  | { type: "image"; data: string; mimeType: string; annotations?: unknown; _meta?: Record<string, unknown> }
  | { type: "audio"; data: string; mimeType: string; annotations?: unknown; _meta?: Record<string, unknown> }
  | {
    type: "resource_link";
    uri: string;
    name: string;
    title?: string;
    description?: string;
    mimeType?: string;
    size?: number;
    icons?: unknown;
    annotations?: unknown;
    _meta?: Record<string, unknown>;
  }
  | { type: "resource"; resource: McpResourceContents; annotations?: unknown; _meta?: Record<string, unknown> };
```

`McpResourceContents` 见 [read_mcp_resource](05-executors.md#read_mcp_resourceserver-uri)。

调用示例：

```js
const result = await new McpExecutor().call_mcp_tool("echo", "echo", { message: "hello", times: 2 });
if (result.isError) {
  throw new Error("MCP echo failed");
}
text(result.structuredContent ?? result.content);
```

`content` 是 MCP 内容块数组，image 块可直接交给 [image](08-io.md) 输出；成功的工具可以同时提供 `structuredContent`。
工具调用失败时 `isError` 为 `true`，不抛出异常。

## 生命周期、失败和边界

- server 只在运行该 `.poa` 包的 thread 中注册；包里的 server 不会安装成全局 server。
- server 的工作目录是包解压目录，因此 `shell = "node mcp/echo.mjs"` 可以解析包内路径。
- 当前实现只支持 `stdio`，server 是宿主启动的直接子进程。
- 声明的 server 属于必需能力；启动失败或在约 30 秒内未完成初始化时，本次 PoA 运行失败，
  不会悄悄删去该 server 的工具。
- 未声明 `readOnlyHint: true` 的工具在调用前需要审批；审批被拒绝时返回 `isError: true` 的结果。
- 一个 server 可以声明多个工具；工具的参数、返回值和并行能力以运行时 `ALL_TOOLS` 与
  MCP `tools/list` 的 schema 为准。
