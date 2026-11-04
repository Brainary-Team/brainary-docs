# 示例

四个完整程序：并行处理、先分后总、交叉评审，以及随包携带 MCP 工具。前三个程序都可以直接作为 `src/main.ts`，按 [快速开始](01-quickstart.md) 打包运行。

## 并行扫描目录

为当前目录的每个子目录派出一个 agent，最多 4 个同时运行，最后汇成一张表。单个目录失败不影响其他目录。

```ts
import { AgentExecutor, TerminalExecutor, map_limit, notify, parse_json_reply, text } from "@brainary/poa-sdk";

type Summary = { purpose: string; entry: string };

const terminal = new TerminalExecutor();
const agents = new AgentExecutor();

// 1. 程序列出要处理的目录
const folders = await terminal.shell_lines("ls -d */ | sed 's#/$##'");

// 2. 每个目录派一个 agent，取得结果后关闭
let done = 0;
const rows = await map_limit(folders, 4, async (folder) => {
  const handle = await agents.spawn_agent({
    message: [
      `分析目录 ./${folder}，只读这个目录里的文件。`,
      "先用 ls 查看结构，再打开入口文件和一个主要源文件。",
      '只回复一行 JSON：{"purpose": "一句话说明用途", "entry": "入口文件路径"}',
    ].join("\n"),
  });
  const reply = await agents.acquire_agent_result(handle.agent_id, 600_000);
  await agents.close(handle.agent_id);
  notify({ done: ++done, total: folders.length });

  const parsed = parse_json_reply<Summary>(reply.result);
  if (parsed.error !== null) return { folder, purpose: `失败：${reply.error ?? parsed.error}`, entry: "" };
  return { folder, ...parsed.value };
});

// 3. 程序汇总成表格
text([
  "| 目录 | 用途 | 入口 |",
  "| --- | --- | --- |",
  ...rows.map((row) => `| ${row.folder} | ${row.purpose} | ${row.entry} |`),
].join("\n"));
```

- 列目录、汇总表格由程序完成，只有"这个目录是做什么的"交给 agent。
- 任务文字写清范围、步骤和回复格式；需要程序处理的回复规定为 JSON，用 `parse_json_reply` 解析。
- 解析失败时保留错误信息，表格照常输出。

## 先分后总

找出 `docs/` 下反复出现、仍未解决的问题。第一轮每个 agent 读 5 个文档并提取问题；第二轮只把提取结果交给一个汇总 agent，由它找出跨文档重复的问题。

```ts
import { AgentExecutor, TerminalExecutor, map_limit, parse_json_reply, text } from "@brainary/poa-sdk";

type Issues = { issues: { title: string; file: string }[] };
type Report = { repeated: { title: string; files: string[] }[] };

const terminal = new TerminalExecutor();
const agents = new AgentExecutor();

async function run_agent(message: string): Promise<string | null> {
  const handle = await agents.spawn_agent({ message });
  const reply = await agents.acquire_agent_result(handle.agent_id, 600_000);
  await agents.close(handle.agent_id);
  return reply.result;
}

// 1. 每 5 个文档分成一批
const files = await terminal.shell_lines("find docs -name '*.md' | sort");
const batches: string[][] = [];
for (let i = 0; i < files.length; i += 5) batches.push(files.slice(i, i + 5));

// 2. 每批交给一个 agent 提取问题
const extracted = await map_limit(batches, 3, async (batch) => {
  const reply = await run_agent([
    `阅读这几个文档：${batch.join("、")}`,
    "列出其中提到但还没有解决的问题。",
    '只回复一行 JSON：{"issues": [{"title": "问题", "file": "所在文件"}]}',
  ].join("\n"));
  return parse_json_reply<Issues>(reply).value?.issues ?? [];
});

// 3. 汇总 agent 只根据提取结果回答
const reply = await run_agent([
  "下面是从多个文档中提取的未解决问题。不要读取任何文件，只根据这些数据回答。",
  JSON.stringify(extracted.flat()),
  "找出在 2 个以上文档中出现的问题，合并说法相近的条目。",
  '只回复一行 JSON：{"repeated": [{"title": "问题", "files": ["文件"]}]}',
].join("\n"));

text(parse_json_reply<Report>(reply).value ?? { raw: reply });
```

- 汇总 agent 被要求"不读取任何文件"，它的结论只能来自第一轮的结果，程序决定了每个 agent 能看到什么。
- 跨文档比对是单个提取 agent 做不到的，第二轮才产生新的信息。

## 交叉评审

一个 agent 回答问题，三个 agent 从不同角度独立检查这个回答，程序按多数票裁决。

```ts
import { AgentExecutor, parse_json_reply, text } from "@brainary/poa-sdk";

type Verdict = { refuted: boolean; reason: string };

const agents = new AgentExecutor();
const QUESTION = "这个仓库的用户登录流程在哪里实现？给出文件路径和关键函数名。";

async function run_agent(message: string): Promise<string | null> {
  const handle = await agents.spawn_agent({ message });
  const reply = await agents.acquire_agent_result(handle.agent_id, 600_000);
  await agents.close(handle.agent_id);
  return reply.result;
}

// 1. 一个 agent 给出回答
const answer = await run_agent(QUESTION);

// 2. 三个 agent 从不同角度尝试推翻它，彼此看不到对方的结论
const LENSES = [
  "回答中提到的文件和函数真的存在吗？逐一检查。",
  "代码的实际行为和回答描述的一致吗？",
  "回答是否遗漏了其他实现登录的位置？",
];
const verdicts = await Promise.all(LENSES.map(async (lens) => {
  const reply = await run_agent([
    `问题：${QUESTION}`,
    `回答：${answer}`,
    lens,
    "证据不足时，判定为被推翻。",
    '只回复一行 JSON：{"refuted": true 或 false, "reason": "一句话理由"}',
  ].join("\n"));
  return parse_json_reply<Verdict>(reply).value;
}));

// 3. 程序裁决：无法解析的评审按推翻计，多数推翻即不通过
const refuted = verdicts.filter((verdict) => verdict?.refuted !== false).length;
text({ answer, passed: refuted < 2, verdicts });
```

- 三个评审角度各不相同，能发现不同类型的错误。
- 模型倾向于同意，所以默认判定为"被推翻"。
- 裁决写在程序里，结果可复现；评审之间相互隔离，也只有程序能保证。

## 随包携带 MCP 工具

把一个 MCP server 和程序打进同一个包：程序直接调用它的工具，也授权给 agent 使用。

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
entry = "src/main.ts"

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

`src/main.ts` 先直接调用工具，再创建一个只能使用该 server 工具的 agent：

```ts
import { AgentExecutor, McpExecutor, text } from "@brainary/poa-sdk";

// 1. 程序直接调用：server 名、工具名、参数
const result = await new McpExecutor().call_mcp_tool("echo", "echo", { text: "hello" });
if (result.isError) throw new Error("echo 调用失败");
text(result.content);

// 2. 授权 agent 使用 echo server 的全部工具
const agent = await new AgentExecutor().create_agent({
  name: "echoer",
  agent_config: {
    model: { model_id: "gpt-5.6-luna", provider: "vendor" },
    capabilities: { mcp: ["echo/*"] },
  },
});
const reply = await agent.run("用 echo 工具回显“你好”，再告诉我工具返回了什么。", undefined, undefined, 300_000);
text(reply.result);
await agent.close();
```

- 打包时 `mcp/` 目录随包写入，运行时按 `shell` 启动 server；`shell` 中的命令要在运行环境里可用，上例需要 Node.js。
- 只读工具在 `annotations` 中标注 `readOnlyHint: true`、`destructiveHint: false`，调用前不需要审批。
- `capabilities.mcp` 中的 `<server>/*` 授予该 server 的全部工具，`<server>/<tool>` 授予单个工具，见 [Capabilities](05-api-reference/02-agent-config.md#capabilities)。
