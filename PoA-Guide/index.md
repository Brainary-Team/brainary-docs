# PoA 开发文档

PoA（Program of Agent）用一段 TypeScript 程序编排多个 AI agent：程序负责拆分任务、并发派发和汇总结果，agent 负责读懂内容、做出判断。

程序基于 `@brainary/poa-sdk` 编写，用 `brainary poa pack` 打包成 `.poa` 包后由 Brainary 运行。下面的程序为当前目录的每个子目录派出一个 agent，最多 3 个同时运行，最后把结果汇成一张表：

```ts
import { AgentExecutor, TerminalExecutor, map_limit, parse_json_reply, text } from "@brainary/poa-sdk";

const terminal = new TerminalExecutor();
const agents = new AgentExecutor();

const folders = await terminal.shell_lines("ls -d */");
const rows = await map_limit(folders, 3, async (folder) => {
  const handle = await agents.spawn_agent({
    message: `阅读目录 ${folder}，只回复一行 JSON：{"purpose": "这个目录的用途"}`,
  });
  const reply = await agents.acquire_agent_result(handle.agent_id, 300_000);
  await agents.close(handle.agent_id);
  return { folder, summary: parse_json_reply(reply.result).value };
});

text(rows);
```

## 文档导航

| 文档 | 内容 |
| --- | --- |
| [快速开始](01-quickstart.md) | 安装 Brainary 和 SDK，编写、打包并运行第一个程序 |
| [配置文件](02-configuration.md) | `config.toml`：多个 provider、默认沙箱、agent 层级和数量；`manifest.toml`：包信息和随包携带的 MCP server |
| [输入与输出](03-input-output.md) | 接收业务输入、输出结果、向用户提问，以及 CLI、TUI 和 Python SDK 三种运行方式 |
| [示例](04-examples.md) | 完整程序：并行扫描、先分后总、交叉评审、随包携带 MCP 工具 |
| [API 参考](05-api-reference.md) | 全部对象、方法和类型 |
