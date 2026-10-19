# Executor

[Executor](09-executors.md) 及其相关类型提供 PoA 访问文件、终端、远程 Agent 和 MCP 的统一边界。每次调用都可以
通过 [ExecutorContext](09-executors.md) 和 [PostExecutorContext](09-executors.md) 交给审计、计时和 hook 处理。

## 上下文与 hook

### ExecutorContext

**作用**：描述一次执行调用的调用方、方法、输入和环境。

**输入类型**：

~~~ts
interface ExecutorContext {
  caller: "poa" | "agent";
  agent_id?: string;
  agent_name?: string;
  executor: "file" | "terminal" | "remote_agent" | "mcp";
  method: string;
  call_id: string;
  input: object;
  environment_id?: string;
  started_at: number;
}
~~~

**返回值类型**：不是函数返回值；作为 hook 的输入。

**示例**：

~~~js
const context = {
  caller: "poa",
  executor: "terminal",
  method: "exec_command",
  call_id: "call-1",
  input: { cmd: "pwd" },
  started_at: Date.now(),
};
~~~

### PostExecutorContext

**作用**：补充执行完成后的状态、输出、错误和耗时。

**输入类型**：ExecutorContext 加上 outcome、output、error 和 duration_ms。

~~~ts
interface PostExecutorContext extends ExecutorContext {
  outcome: "completed" | "failed" | "timed_out" |
    "cancelled" | "denied" | "stubbed";
  output?: unknown;
  error?: string;
  duration_ms: number;
}
~~~

**返回值类型**：不是函数返回值；作为 post hook 的输入。

**示例**：

~~~js
const post = {
  ...context,
  outcome: "completed",
  output: { exit_code: 0 },
  duration_ms: 12,
};
~~~

### Hooks

**作用**：在执行器调用前后插入审计、限流或结果处理逻辑。

**输入类型**：

~~~ts
interface Hooks {
  pre_executor_hooks?: Array<(context: ExecutorContext) => void | Promise<void>>;
  post_executor_hooks?: Array<(context: PostExecutorContext) => void | Promise<void>>;
}
~~~

**返回值类型**：[Hooks](09-executors.md) 配置对象。

**示例**：

~~~js
const hooks = {
  pre_executor_hooks: [(ctx) => text("calling " + ctx.method)],
};
~~~

### Executor

**作用**：聚合各类执行器和可选的 hooks。

**输入类型**：

~~~ts
interface Executor {
  fileExecutor: FileExecutor;
  terminalExecutor: TerminalExecutor;
  remoteAgentExecutor: RemoteAgentExecutor;
  mcpExecutor: McpExecutor;
  hooks?: Hooks;
}
~~~

**返回值类型**：[Executor](09-executors.md) 实例。

**示例**：

~~~js
const result = await executor.terminalExecutor.exec_command({ cmd: "pwd" });
text(result.output);
~~~

## FileExecutor

### apply_patch(patch)

**作用**：对工作区应用文件补丁。

**输入类型**：patch: string。

**返回值类型**：Promise<unknown>，宿主补丁结果。

**示例**：

~~~js
await executor.fileExecutor.apply_patch(
  "*** Begin Patch\\n*** Update File: note.md\\n+done\\n*** End Patch",
);
~~~

### view_image(path, detail?, environment_id?)

**作用**：读取执行环境中的本地图片。

**输入类型**：path: string；detail?: "high" | "original"；environment_id?: string。

**返回值类型**：Promise<{ detail: "high" | "original"; image_url: string }>。

**示例**：

~~~js
const picture = await executor.fileExecutor.view_image("./out.png", "high");
image(picture);
~~~

## TerminalExecutor

### exec_command(input)

**作用**：在指定环境运行 shell 命令或脚本。

**输入类型**：ExecCommandInput，至少包含 cmd: string，可选工作目录、shell、TTY、等待时间、
输出上限和环境权限字段。

**返回值类型**：Promise<ExecCommandResult>，包含 output、可选 exit_code、session_id 等字段。

**示例**：

~~~js
const result = await executor.terminalExecutor.exec_command({ cmd: "pwd", workdir: "." });
text(result.output);
~~~

### write_stdin(input)

**作用**：向已有终端会话写入标准输入，或轮询新增输出。

**输入类型**：{ session_id: number; chars?: string; yield_time_ms?: number; max_output_tokens?: number }。

**返回值类型**：Promise<ExecCommandResult>。

**示例**：

~~~js
const result = await executor.terminalExecutor.write_stdin({
  session_id,
  chars: "yes\\n",
});
text(result.output);
~~~

### shellines(command)

**作用**：执行命令并把输出按行切分、去除空行和首尾空白。

**输入类型**：command: string。

**返回值类型**：Promise<string[]>。

**示例**：

~~~js
const names = await executor.terminalExecutor.shellines("find . -maxdepth 1 -type f");
text(names);
~~~

### wait_environment(environment_id)

**作用**：等待指定执行环境从 starting 进入可用状态。

**输入类型**：environment_id: string。

**返回值类型**：Promise<{ environment_id: string; status: "ready" }>。

**示例**：

~~~js
await executor.terminalExecutor.wait_environment("env-123");
~~~

## RemoteAgentExecutor

### send_input(agent_id, message?, input_items?, output_schema?)

**作用**：向远程 Agent 发送输入，并请求一次结构化或文本结果。

**输入类型**：agent_id: string；message?: string；input_items?: IOInput[]；output_schema?: JsonSchema。

**返回值类型**：Promise<[AgentResult](03agent.md)>。

**示例**：

~~~js
const result = await executor.remoteAgentExecutor.send_input(
  remote.agent_id,
  "分析这个文件",
  [],
  { type: "object" },
);
text(result.output);
~~~

## McpExecutor

### list_mcp_resources(server?, cursor?)

**作用**：列出 MCP server 提供的资源。

**输入类型**：server?: string；cursor?: string。

**返回值类型**：Promise<{ resources?: unknown[]; nextCursor?: string | null }>。

**示例**：

~~~js
const page = await executor.mcpExecutor.list_mcp_resources("docs");
text(page.resources);
~~~

### list_mcp_resource_templates(server?, cursor?)

**作用**：列出参数化 MCP 资源模板。

**输入类型**：server?: string；cursor?: string。

**返回值类型**：Promise<{ resourceTemplates?: unknown[]; nextCursor?: string | null }>。

**示例**：

~~~js
const page = await executor.mcpExecutor.list_mcp_resource_templates("docs");
text(page.resourceTemplates);
~~~

### read_mcp_resource(server, uri)

**作用**：读取指定 MCP 资源。

**输入类型**：server: string；uri: string。

**返回值类型**：Promise<ReadResourceResult>。

**示例**：

~~~js
const resource = await executor.mcpExecutor.read_mcp_resource(
  "docs",
  "file:///README.md",
);
text(resource);
~~~

### mcp_<server>_<tool>

**作用**：调用 manifest 中声明的 MCP tool。一个 server 可以声明多个 tool。

**输入类型**：由具体 MCP tool 的 input schema 决定。

**返回值类型**：Promise<CallToolResult>。

~~~ts
interface CallToolResult {
  content: unknown[];
  isError?: boolean;
  structuredContent?: unknown;
  [key: string]: unknown;
}
~~~

**示例**：

~~~js
const result = await tools["mcp__echo__echo"]({ message: "hello" });
if (result.isError) throw new Error("echo failed");
text(result.structuredContent ?? result.content);
~~~

MCP server 的 manifest 配置、生命周期和失败边界见
[MCP tool 配置](10-mcp-tool-config.md)。声明的 server 启动失败时，PoA 初始化失败；server
不会静默降级为不可用工具。
