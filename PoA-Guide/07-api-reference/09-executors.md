# Executor

[Executor](09-executors.md) 及其相关类型提供 PoA 访问文件、终端、Agent 和 MCP 的统一边界。每次调用都可以
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
    "canceled" | "denied" | "stubbed";
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

接收 [ExecutorContext](#executorcontext) 的回调函数。

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

接收 [PostExecutorContext](#postexecutorcontext) 的回调函数。

**示例**：

~~~js
const post_executor_hook = (context) => {
  text({ outcome: context.outcome, duration_ms: context.duration_ms });
};
const hooks = { post_executor_hooks: [post_executor_hook] };
~~~

### Executor

**作用**：聚合各类执行器、工具入口、hooks 和执行上下文。

**类型定义**：

~~~ts
interface Executor {
  fileExecutor: FileExecutor;
  terminalExecutor: TerminalExecutor;
  RemoteAgentExecutor: AgentExecutor;
  McpExecutor: McpExecutor;
  hooks?: Hooks;
  tools: Readonly<Record<string, Tool>>;
}
~~~

**示例**：

~~~js
const result = await executor.terminalExecutor.exec_command({ cmd: "pwd" });
text(result.output);
~~~

## FileExecutor

文件执行器，通过 executor.fileExecutor 访问，提供以下文件操作方法。

### apply_patch({patch})

**作用**：对工作区应用文件补丁。

**输入类型**：命名参数对象，包含 patch: string。

**返回值类型**：Promise<unknown>，宿主补丁结果。

**示例**：

~~~js
await executor.fileExecutor.apply_patch({
  patch: "*** Begin Patch\n*** Add File: note.md\n+done\n*** End Patch",
});
~~~

### view_image({path, detail?, environment_id?})

**作用**：读取执行环境中的本地图片。

**输入类型**：命名参数对象，包含 path: string；detail?: "high" | "original"；environment_id?: string。

detail 默认使用 high；传入 original 时保留原始分辨率。参数与底层 view_image tool 一致。

**返回值类型**：Promise<{ detail: "high" | "original"; image_url: string }>。

**示例**：

~~~js
const picture = await executor.fileExecutor.view_image({ path: "./out.png", detail: "high" });
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

### write_stdin({session_id, chars, yield_time_ms, max_output_tokens})

**作用**：向已有终端会话写入标准输入，或轮询新增输出。

**输入类型**：命名参数对象，包含 session_id: number；chars: string；yield_time_ms: number；max_output_tokens: number。四个参数均必填；轮询时 chars 传入空字符串。

**返回值类型**：Promise<ExecCommandResult>。

**示例**：

~~~js
const result = await executor.terminalExecutor.write_stdin({
  session_id,
  chars: "yes\n",
  yield_time_ms: 1000,
  max_output_tokens: 2000,
});
text(result.output);
~~~

### shelllines({command})

**作用**：执行命令并把输出按行切分、去除空行和首尾空白。

**输入类型**：命名参数对象，包含 command: string。

**返回值类型**：Promise<string[]>。

**示例**：

~~~js
const names = await executor.terminalExecutor.shelllines({ command: "find . -maxdepth 1 -type f" });
text(names);
~~~

### wait_for_environment({environment_id})

**作用**：等待处于 starting 状态的指定执行环境就绪；环境已经就绪时直接返回。

**输入类型**：命名参数对象，包含 environment_id: string，必填。参数与底层 wait_for_environment tool 一致。

**返回值类型**：Promise<{ environment_id: string; status: "ready" }>。环境不存在或启动失败时返回错误。

**示例**：

~~~js
const result = await executor.terminalExecutor.wait_for_environment({ environment_id: "env-123" });
text(result.status);
~~~

## AgentExecutor

Agent 执行器的消息发送、追加消息、上下文压缩、关闭和状态查询，见 [AgentExecutor](04-agent-executor.md)。

## McpExecutor

MCP 执行器，通过 executor.McpExecutor 访问，提供以下资源查询、读取和工具调用方法。

### list_mcp_resources({server?, cursor?})

**作用**：列出 MCP server 提供的资源。

**输入类型**：命名参数对象，包含 server?: string；cursor?: string。

**返回值类型**：Promise<{ resources?: unknown[]; nextCursor?: string | null }>。

**示例**：

~~~js
const page = await executor.McpExecutor.list_mcp_resources({ server: "docs" });
text(page.resources);
~~~

### list_mcp_resource_templates({server?, cursor?})

**作用**：列出参数化 MCP 资源模板。

**输入类型**：命名参数对象，包含 server?: string；cursor?: string。

**返回值类型**：Promise<{ resourceTemplates?: unknown[]; nextCursor?: string | null }>。

**示例**：

~~~js
const page = await executor.McpExecutor.list_mcp_resource_templates({ server: "docs" });
text(page.resourceTemplates);
~~~

### read_mcp_resource({server, uri})

**作用**：读取指定 MCP 资源。

**输入类型**：命名参数对象，包含 server: string；uri: string。

**返回值类型**：Promise<ReadResourceResult>。

**示例**：

~~~js
const resource = await executor.McpExecutor.read_mcp_resource({
  server: "docs",
  uri: "file:///README.md",
});
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
const result = await executor.McpExecutor["mcp__echo__echo"]({ message: "hello" });
if (result.isError) throw new Error("echo failed");
text(result.structuredContent ?? result.content);
~~~

MCP server 的 manifest 配置、生命周期和失败边界见
[MCP tool 配置](10-mcp-tool-config.md)。声明的 server 启动失败时，PoA 初始化失败；server
不会静默降级为不可用工具。

## 执行器的 tools 入口

**作用**：枚举执行器提供的工具。Executor.tools 汇总其各执行器的工具，FileExecutor、TerminalExecutor、AgentExecutor 和 McpExecutor 的 tools 分别枚举各自提供的工具。每个枚举成员的值都是字符串。

**类型定义**：Readonly<Record<string, [Tool](13-tools-manager.md#tools)>>。

**示例**：

~~~js
text(executor.fileExecutor.tools.view_image);
~~~
