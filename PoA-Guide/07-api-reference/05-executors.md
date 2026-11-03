# Executor

[Executor](05-executors.md) 及其子类提供 PoA 访问文件、终端、Agent 和 MCP 的统一入口。按设计，每次调用还会生成
[ExecutorContext](#executorcontext) 和 [PostExecutorContext](#postexecutorcontext) 交给 hook 处理，这部分尚未实现。

以下示例假定已导入并创建各执行器：

~~~js
import { FileExecutor, McpExecutor, ShellCommandError, TerminalExecutor } from "@brainary/poa-sdk/executor";
import { image, text } from "@brainary/poa-sdk/io";

const fileExecutor = new FileExecutor();
const terminalExecutor = new TerminalExecutor();
const mcpExecutor = new McpExecutor();
~~~

## 上下文与 hook

### ExecutorContext

**状态**：尚未实现，以下为设计描述。

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

**状态**：尚未实现，以下为设计描述。

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

**状态**：尚未实现，以下为设计描述。

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
const audited = new TerminalExecutor(hooks);
~~~

### PreExecutorHooks

**状态**：尚未实现，以下为设计描述。

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

**状态**：尚未实现，以下为设计描述。

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

**作用**：FileExecutor、TerminalExecutor、AgentExecutor 和 McpExecutor 的公共基类。

**类型定义**：

~~~ts
type ExecutorKind = "file" | "terminal" | "remote_agent" | "mcp" | "agent";

abstract class Executor {
  readonly kind: ExecutorKind;
}
~~~

各执行器用 `new` 创建，并以静态属性 tools 列出所封装的工具，见 [执行器的 tools 入口](#执行器的-tools-入口)。
"remote_agent" 对应的执行器尚未实现。

**示例**：

~~~js
text([fileExecutor.kind, terminalExecutor.kind, mcpExecutor.kind]); // ["file", "terminal", "mcp"]
~~~

## FileExecutor

文件执行器，kind 为 "file"，提供以下文件操作方法。

### apply_patch(patch)

**作用**：按 apply_patch 格式对工作区应用文件补丁。

**输入类型**：patch: string，必填。

**返回值类型**：Promise<void>；补丁无法应用时抛出异常。

**示例**：

~~~js
await fileExecutor.apply_patch("*** Begin Patch\n*** Add File: note.md\n+done\n*** End Patch");
~~~

### view_image(path, detail?, environment_id?)

**作用**：读取执行环境中的本地图片。

**输入类型**：path: string，必填；detail?: "high" | "original"；environment_id?: string。

detail 默认为 high；传入 original 时保留原始分辨率。

**返回值类型**：Promise<ViewImageResult>。

~~~ts
interface ViewImageResult {
  image_url: string;
  detail: "high" | "original";
}
~~~

结果可直接传给 [image](08-io.md) 输出图片。

**示例**：

~~~js
const picture = await fileExecutor.view_image("./out.png", "high");
image(picture);
~~~

## TerminalExecutor

终端执行器，kind 为 "terminal"，提供以下命令和会话管理方法。

### exec_command(cmd, options?)

**作用**：在指定环境运行 shell 命令或脚本。

**输入类型**：cmd: string，必填；options?: ExecCommandOptions。

~~~ts
interface ExecCommandOptions {
  workdir?: string;
  tty?: boolean;
  yield_time_ms?: number;
  max_output_tokens?: number;
  shell?: string;
  login?: boolean;
  environment_id?: string;
  sandbox_permissions?: "use_default" | "with_additional_permissions" | "require_escalated";
  justification?: string;
  prefix_rule?: string[];
  additional_permissions?: object;
}
~~~

max_output_tokens 省略时返回完整输出。

**返回值类型**：Promise<ExecOutput>。

~~~ts
interface ExecOutput {
  chunk_id?: string;
  wall_time_seconds: number;
  exit_code?: number;
  session_id?: number;
  original_token_count?: number;
  output: string;
}
~~~

命令结束时返回 exit_code；在 yield_time_ms 内未结束时返回 session_id，可用 write_stdin 继续交互。

**示例**：

~~~js
const result = await terminalExecutor.exec_command("pwd", { workdir: "." });
text(result.output);
~~~

### write_stdin(session_id, chars?, yield_time_ms?, max_output_tokens?)

**作用**：向已有终端会话写入标准输入，或轮询新增输出。

**输入类型**：session_id: number，必填；chars?: string，传入空字符串时只轮询、不写入；yield_time_ms?: number；max_output_tokens?: number。

**返回值类型**：Promise<ExecOutput>。

**示例**：

~~~js
const started = await terminalExecutor.exec_command("read answer && echo got $answer", {
  tty: true,
  yield_time_ms: 500,
});
if (started.session_id !== undefined) {
  const result = await terminalExecutor.write_stdin(started.session_id, "yes\n", 1000);
  text(result.output);
}
~~~

### shell_lines(command, options?)

**作用**：运行命令直到结束，并按行返回输出。

**输入类型**：command: string，必填；options?: ExecCommandOptions，与 exec_command 相同。

**返回值类型**：Promise<string[]>。

退出码不为 0 时抛出 [ShellCommandError](#shellcommanderror)。

**示例**：

~~~js
const names = await terminalExecutor.shell_lines("find . -maxdepth 1 -type f");
text(names);
~~~

### wait_for_environment(environment_id)

**作用**：等待处于 starting 状态的指定执行环境就绪；环境已经就绪时直接返回。

**输入类型**：environment_id: string，必填。

**返回值类型**：Promise<EnvironmentReady>。环境不存在或启动失败时抛出异常。

~~~ts
interface EnvironmentReady {
  environment_id: string;
  status: "ready";
}
~~~

**示例**：

~~~js
const result = await terminalExecutor.wait_for_environment("env-123");
text(result.status);
~~~

### ShellCommandError

**作用**：shell_lines 运行的命令以非 0 退出码结束时抛出。

**类型定义**：

~~~ts
class ShellCommandError extends Error {
  readonly command: string;
  readonly exit_code: number | null;
  readonly output: string;
}
~~~

**示例**：

~~~js
try {
  text(await terminalExecutor.shell_lines("git status --short"));
} catch (error) {
  if (error instanceof ShellCommandError) text({ exit_code: error.exit_code, output: error.output });
  else throw error;
}
~~~

## AgentExecutor

Agent 执行器的 kind 为 "agent"，负责 Agent 的创建、派发、结果获取、状态查询、关闭和恢复，见 [AgentExecutor](03-agent-executor.md)。

## McpExecutor

MCP 执行器，kind 为 "mcp"，提供以下资源查询、读取和工具调用方法。

### list_mcp_resources(server?, cursor?)

**作用**：列出 MCP server 提供的资源；省略 server 时列出全部 server 的资源。

**输入类型**：server?: string；cursor?: string，上一页返回的 nextCursor。

**返回值类型**：Promise<McpResourcePage>。

~~~ts
interface McpResourcePage {
  server?: string;
  resources: McpResource[];
  nextCursor?: string;
}

interface McpResource {
  server: string;
  uri: string;
  name: string;
  title?: string;
  description?: string;
  mimeType?: string;
  size?: number;
  annotations?: unknown;
}
~~~

**示例**：

~~~js
const page = await mcpExecutor.list_mcp_resources("docs");
text(page.resources.map((resource) => resource.uri));
~~~

### list_mcp_resource_templates(server?, cursor?)

**作用**：列出参数化 MCP 资源模板；省略 server 时列出全部 server 的模板。

**输入类型**：server?: string；cursor?: string，上一页返回的 nextCursor。

**返回值类型**：Promise<McpResourceTemplatePage>。

~~~ts
interface McpResourceTemplatePage {
  server?: string;
  resourceTemplates: McpResourceTemplate[];
  nextCursor?: string;
}

interface McpResourceTemplate {
  server: string;
  uriTemplate: string;
  name: string;
  title?: string;
  description?: string;
  mimeType?: string;
  annotations?: unknown;
}
~~~

**示例**：

~~~js
const page = await mcpExecutor.list_mcp_resource_templates("docs");
text(page.resourceTemplates);
~~~

### read_mcp_resource(server, uri)

**作用**：读取指定 MCP 资源。

**输入类型**：server: string；uri: string。两个参数均必填。

**返回值类型**：Promise<McpResourceRead>。

~~~ts
interface McpResourceRead {
  server: string;
  uri: string;
  contents: McpResourceContents[];
}

type McpResourceContents =
  | { uri: string; mimeType?: string; text: string }
  | { uri: string; mimeType?: string; blob: string };
~~~

**示例**：

~~~js
const resource = await mcpExecutor.read_mcp_resource("docs", "file:///README.md");
text(resource.contents);
~~~

### call_mcp_tool(server, tool, args?)

**作用**：按 server 名和工具名调用 MCP 工具。

**输入类型**：server: string，必填；tool: string，必填；args?: object，工具参数，须为 JSON 对象，省略时为 `{}`，
结构由该工具的 input schema 决定。TypeScript 中可用泛型参数声明 structuredContent 的类型。

**返回值类型**：Promise<[McpToolResult](06-mcp-tool-config.md#calltoolresult)>。

工具调用失败时返回 `isError: true` 的结果，不抛出异常。

**示例**：

~~~js
const result = await mcpExecutor.call_mcp_tool("echo", "echo", { message: "hello" });
if (result.isError) throw new Error("echo failed");
text(result.structuredContent ?? result.content);
~~~

MCP server 的 manifest 配置和生命周期见 [MCP tool 配置](06-mcp-tool-config.md)。

## 执行器的 tools 入口

**作用**：列出执行器封装的宿主工具。每个执行器类以静态属性 tools 提供自己的子集，值为 [BrainaryTool](07-brainary-tool.md#brainarytool) 成员，
可与其他成员合并为 [Capabilities](02-agent-config.md#capabilities).brainary_tool 的授予列表。

**类型定义**：静态只读对象，键为成员名，值为同名的 BrainaryTool 成员。

| 执行器 | tools |
| --- | --- |
| FileExecutor | apply_patch、view_image |
| TerminalExecutor | exec_command、write_stdin、wait_for_environment |
| McpExecutor | list_mcp_resources、list_mcp_resource_templates、read_mcp_resource |
| AgentExecutor | spawn_agent、create_agent、send_input、wait_agent、resume_agent、close_agent、get_agent_status |

call_mcp_tool 调用的工具由 MCP server 提供，不在 tools 中。

**示例**：

~~~js
text(FileExecutor.tools.view_image);

const brainary_tool = [...Object.values(TerminalExecutor.tools), FileExecutor.tools.apply_patch];
~~~
