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

宿主启动声明的 stdio server，调用 MCP `tools/list`，然后把工具放入当前 PoA 的 `tools`
对象。默认名称通常是：

```text
mcp__<server_name>__<tool_name>
```

例如上面的 server 和工具都叫 `echo` 时通常为 `mcp__echo__echo`。名称清洗、长度限制或
哈希可能改变最终名称，因此生产代码应从 `ALL_TOOLS` 发现，而不是硬编码：

```js
const candidates = ALL_TOOLS.filter((item) => item.name.endsWith("__echo"));
if (candidates.length !== 1) throw new Error("echo tool must have one matching name");
const echo = candidates[0];

const result = await tools[echo.name]({
  message: "你好",
  times: 2,
});
text(result.structuredContent ?? result.content);
```

`ALL_TOOLS` 是全局工具列表，类型为 `Array<{ name: string; description: string }>`。
`tools[tool_name]` 是工具调用函数，其输入和返回 schema 取决于 MCP server 的声明。
后缀匹配仅适用于工具名未被截断或追加哈希的场景；无法匹配时，应检查
`ALL_TOOLS` 中的实际名称与说明，并使用该名称调用。

## `CallToolResult`

MCP 工具通常返回如下结构：

```ts
type CallToolResult<TStructured = Record<string, unknown>> = {
  content: Array<unknown>;
  isError?: boolean;
  structuredContent?: TStructured;
  [key: string]: unknown;
};
```

调用示例：

```js
const result = await tools[echo.name]({ message: "hello", times: 2 });
if (result.isError) {
  throw new Error("MCP echo failed");
}
text(result.structuredContent ?? result.content);
```

输入参数结构由具体工具的 input schema 决定；`content` 是 MCP 内容块数组，成功工具可以
同时提供 `structuredContent`；`isError: true` 表示工具执行失败。未知扩展字段应保留，
不要假设所有 server 都只返回上述三个字段。

## 生命周期、失败和边界

- server 只在运行该 `.poa` 包的 thread 中注册；包里的 server 不会安装成全局 server。
- server 的工作目录是包解压目录，因此 `shell = "node mcp/echo.mjs"` 可以解析包内路径。
- 当前实现只支持 `stdio`，server 是宿主启动的直接子进程。
- 声明的 server 属于必需能力；启动失败或在约 30 秒内未完成初始化时，本次 PoA 运行失败，
  不会悄悄删去该 server 的工具。
- 一个 server 可以声明多个工具；工具的参数、返回值和并行能力以运行时 `ALL_TOOLS` 与
  MCP `tools/list` 的 schema 为准。
