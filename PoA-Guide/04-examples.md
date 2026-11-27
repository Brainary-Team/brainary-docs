# 示例

两个完整程序：代码审查展示业务输入和 `create_agent` 的用法，随包携带 MCP 工具展示如何把 MCP server 和程序打进同一个包。两个项目都从 [PoA 模板](01-quickstart.md#2-创建项目) 创建，用 `pnpm package` 打包。运行前在 `config.toml` 中 [开启多 agent](02-configuration.md#开启多-agent)。

## 代码审查

调用方传入要审查的文件和审查重点。程序用 `create_agent` 创建一个审查 agent，逐个文件审查，再让它在同一段对话里挑出最需要先修的问题。

```text
code-review/
├── input_schema.ts    业务输入
├── src/main.ts        程序入口
├── manifest.toml
└── package.json
```

`input_schema.ts` 声明调用方要传入的内容：

```ts
export interface InputParams {
  /**
   * 要审查的文件路径
   * @minItems 1
   */
  files: string[];
  /** 审查重点，例如“错误处理” */
  focus?: string;
}
```

`manifest.toml`：

```toml
[poa]
namespace = "demo"
name = "code_review"
version = "0.1.0"
description = "审查指定文件，并指出最需要先修的问题"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
```

`src/main.ts`：

```ts
import { AgentExecutor, POA_INPUT, text } from "@brainary/poa-sdk";
import type { InputParams } from "../input_schema";

// 1. 读取输入，没传 focus 时使用默认值
const { files, focus = "错误处理" } = POA_INPUT as unknown as InputParams;

// 2. 创建审查 agent：指定模型和开发者指令
const agents = new AgentExecutor();
const reviewer = await agents.create_agent({
  role: "reviewer",
  description: "审查代码",
  agent_config: {
    model: { model_id: "gpt-5.6-luna" },
    developer_instructions: "你是代码审查员，只报告能在代码中找到依据的问题。",
  },
});

// 3. 逐个文件审查，每次审查都在同一段对话中进行
for (const file of files) {
  const review = await reviewer.run(`审查 ${file}，重点关注${focus}。用不超过三条要点列出问题。`, undefined, 600_000);
  text(`## ${file}\n${review.result}`);
}

// 4. 让它综合前面的审查结果
const top = await reviewer.run("综合前面所有文件的审查结果，只指出最需要先修的一个问题，并说明原因。", undefined, 600_000);
text(`## 优先修复\n${top.result}`);

await reviewer.close();
```

- `create_agent` 创建的 agent 保留对话历史，第 4 步不必重复各文件的审查内容。
- agent 使用与程序相同的沙箱。按默认的 `read-only` 沙箱运行时，审查过程不会修改文件，见 [沙箱](02-configuration.md#沙箱)。
- `model.model_id` 须是 `config.toml` 中 `model_provider` 提供的模型，见 [配置 provider](02-configuration.md#配置-provider)。

打包后，通过 Python SDK 传入输入运行：

```python
from brainary import Brainary, BrainaryConfig, Poa, PoaPackage

package = PoaPackage.load("dist/my-poa.poa")

with Brainary(config=BrainaryConfig(brainary_bin="/path/to/brainary")) as brainary:
    thread = brainary.thread_start(
        cwd="/path/to/repo",
        config={"features": {"code_mode": {"enabled": True}, "multi_agent_v3": True}},
    )
    result = Poa(thread, package).run(input={"files": ["src/auth.ts", "src/db.ts"], "focus": "错误处理"})
    print(result)
```

## 随包携带 MCP 工具

把一个 MCP server 和程序打进同一个包：程序调用它的工具，再把结果交给 agent 处理。

```text
echo-poa/
├── src/main.ts
├── mcp/echo.mjs
├── manifest.toml
└── package.json
```

`manifest.toml` 声明 server，字段说明见 [随包携带 MCP server](02-configuration.md#随包携带-mcp-server)：

```toml
[poa]
namespace = "demo"
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

`mcp/echo.mjs` 是一个不依赖第三方库的 stdio MCP server，提供一个 `echo` 工具：

```js
const TOOLS = [{
  name: "echo",
  description: "回显传入的文本",
  inputSchema: { type: "object", properties: { text: { type: "string" } }, required: ["text"] },
  annotations: { readOnlyHint: true, destructiveHint: false, openWorldHint: false },
}];

const send = (message) => process.stdout.write(JSON.stringify(message) + "\n");

const handle = (request) => {
  if (request.id === undefined) return;  // 通知不需要回复
  if (request.method === "initialize") {
    return send({ jsonrpc: "2.0", id: request.id, result: {
      protocolVersion: "2024-11-05",
      capabilities: { tools: {} },
      serverInfo: { name: "echo", version: "0.1.0" },
    } });
  }
  if (request.method === "tools/list") {
    return send({ jsonrpc: "2.0", id: request.id, result: { tools: TOOLS } });
  }
  if (request.method === "tools/call") {
    const text = String(request.params?.arguments?.text ?? "");
    return send({ jsonrpc: "2.0", id: request.id, result: { content: [{ type: "text", text }] } });
  }
  send({ jsonrpc: "2.0", id: request.id, error: { code: -32601, message: "unknown method" } });
};

let buffer = "";
process.stdin.on("data", (chunk) => {
  buffer += chunk;
  let index;
  while ((index = buffer.indexOf("\n")) >= 0) {
    const line = buffer.slice(0, index).trim();
    buffer = buffer.slice(index + 1);
    if (line) handle(JSON.parse(line));
  }
});
```

`src/main.ts` 先调用工具，再让 agent 处理工具的返回：

```ts
import { AgentExecutor, McpExecutor, text } from "@brainary/poa-sdk";

// 1. 调用随包携带的 server：server 名、工具名、参数
const result = await new McpExecutor().call_mcp_tool("echo", "echo", { text: "你好" });
if (result.isError) throw new Error("echo 调用失败");
text(result.content);

// 2. 把工具的返回交给 agent
const agents = new AgentExecutor();
const handle = await agents.spawn_agent({ message: `把这段工具返回的文本翻译成英文，只回复译文：${JSON.stringify(result.content)}` });
const reply = await agents.acquire_agent_result(handle.agent_id, 300_000);
text(reply.result);
await agents.close(handle.agent_id);
```

- 打包时 `mcp/` 目录随包写入，运行时按 `shell` 启动 server；`shell` 中的命令要在运行环境里可用，上例需要 Node.js。
- 只读工具在 `annotations` 中标注 `readOnlyHint: true`、`destructiveHint: false`，调用前不需要审批。
- 随包携带的 server 由程序通过 [McpExecutor](05-api-reference/05-executors.md#mcpexecutor) 调用，工具的返回结构见 [McpToolResult](05-api-reference/05-executors.md#mcptoolresult)。
