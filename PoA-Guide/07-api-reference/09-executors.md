# Executor

[Executor](09-executors.md) 及其相关类型提供 PoA 访问文件、终端、远程 Agent 和 MCP 的统一边界。每次调用都可以
通过 [ExecutorContext](09-executors.md) 和 [PostExecutorContext](09-executors.md) 交给审计、计时和 hook 处理。

## 上下文与 hook

### ExecutorContext

**作用**：描述一次执行调用的调用方、方法、输入和环境。

**类型定义**：

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

该上下文作为 pre hook 的输入。

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

**类型定义**：继承 ExecutorContext，并补充必填的 outcome、duration_ms 和可选的 output、error。

~~~ts
interface PostExecutorContext extends ExecutorContext {
  outcome: "completed" | "failed" | "timed_out" |
    "cancelled" | "denied" | "stubbed";
  output?: unknown;
  error?: string;
  duration_ms: number;
}
~~~

该上下文作为 post hook 的输入。

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

**类型定义**：

~~~ts
interface Hooks {
  pre_executor_hooks?: PreExecutorHooks[];
  post_executor_hooks?: PostExecutorHooks[];
}
~~~

**示例**：

~~~js
const hooks = {
  pre_executor_hooks: [(ctx) => text("calling " + ctx.method)],
};
~~~

### PreExecutorHooks

**作用**：表示执行器调用前运行的单个 hook，接收 ExecutorContext。

**类型定义**：

~~~ts
type PreExecutorHooks = (context: ExecutorContext) => void | Promise<void>;
~~~

**示例**：

~~~js
const pre_executor_hook = (context) => {
  text({ call_id: context.call_id, method: context.method });
};
const hooks = { pre_executor_hooks: [pre_executor_hook] };
~~~

### PostExecutorHooks

**作用**：表示执行器调用完成后运行的单个 hook，接收 PostExecutorContext。

**类型定义**：

~~~ts
type PostExecutorHooks = (context: PostExecutorContext) => void | Promise<void>;
~~~

**示例**：

~~~js
const post_executor_hook = (context) => {
  text({ outcome: context.outcome, duration_ms: context.duration_ms });
};
const hooks = { post_executor_hooks: [post_executor_hook] };
~~~

### Executor

**作用**：聚合各类执行器和可选的 hooks。

**类型定义**：

~~~ts
interface Executor {
  fileExecutor: FileExecutor;
  terminalExecutor: TerminalExecutor;
  remoteAgentExecutor: RemoteAgentExecutor;
  mcpExecutor: McpExecutor;
  hooks?: Hooks;
}
~~~

**示例**：

~~~js
const result = await executor.terminalExecutor.exec_command({ cmd: "pwd" });
text(result.output);
~~~

## FileExecutor

文件执行器，通过 executor.fileExecutor 访问，提供以下文件操作方法。

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

终端执行器，通过 executor.terminalExecutor 访问，提供以下命令和会话管理方法。

### exec_command(input)

**作用**：在指定环境运行 shell 命令或脚本。

**输入类型**：input: ExecCommandInput，至少包含 cmd: string，可选工作目录、shell、TTY、等待时间、
输出上限和环境权限字段。

**返回值类型**：Promise<ExecCommandResult>，包含 output、可选 exit_code、session_id 等字段。

**示例**：

~~~js
const result = await executor.terminalExecutor.exec_command({ cmd: "pwd", workdir: "." });
text(result.output);
~~~

### write_stdin(session_id, chars, yield_time_ms, max_output_tokens)

**作用**：向已有终端会话写入标准输入，或轮询新增输出。

**输入类型**：session_id: number；chars: string；yield_time_ms: number；max_output_tokens: number。四个参数均必填；轮询时 chars 传入空字符串。

**返回值类型**：Promise<ExecCommandResult>。

**示例**：

~~~js
const result = await executor.terminalExecutor.write_stdin(
  session_id,
  "yes\n",
  1000,
  2000,
);
text(result.output);
~~~

### shelllines(command)

**作用**：执行命令并把输出按行切分、去除空行和首尾空白。

**输入类型**：command: string。

**返回值类型**：Promise<string[]>。

**示例**：

~~~js
const names = await executor.terminalExecutor.shelllines("find . -maxdepth 1 -type f");
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

远程 Agent 执行器，通过 executor.remoteAgentExecutor 访问，提供以下远程输入方法。

### send_input(agent_id, message?, input_items?, output_schema?, timeout?)

**作用**：向远程 Agent 发送输入，并请求一次结构化或文本结果。

**输入类型**：agent_id: string；message?: string；input_items?: IOInput[]；output_schema?: JsonSchema；timeout?: number，等待返回的超时时间，单位为毫秒（ms）。

**参数约束**：message 和 input_items 必须且只能传入一个；传入 output_schema 或 timeout 时，省略的前置可选参数使用 undefined 占位。

**返回值类型**：Promise<[AgentResult](03-agent.md)>。

**示例**：

~~~js
const result = await executor.remoteAgentExecutor.send_input(
  remote.agent_id,
  "分析这个文件",
  undefined,
  { type: "object" },
  timeout,
);
text(result.output);
~~~

## McpExecutor

MCP 执行器，通过 executor.mcpExecutor 访问，提供以下资源查询、读取和工具调用方法。

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
