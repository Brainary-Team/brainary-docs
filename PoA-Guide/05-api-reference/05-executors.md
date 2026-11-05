# Executor

执行器把文件、终端、agent 和 MCP 操作封装成方法，用 `new` 创建，构造时不传参数：

| 执行器 | kind | 文档 |
| --- | --- | --- |
| FileExecutor | "file" | [FileExecutor](#fileexecutor) |
| TerminalExecutor | "terminal" | [TerminalExecutor](#terminalexecutor) |
| McpExecutor | "mcp" | [McpExecutor](#mcpexecutor) |
| AgentExecutor | "agent" | [AgentExecutor](03-agent-executor.md) |

Executor 是它们共同的基类，不从 SDK 导出。每个执行器都有只读属性 kind 和静态属性 [tools](#tools)。kind 为 "remote_agent" 的执行器尚未实现。

以下示例假定已导入并创建各执行器：

~~~js
import { FileExecutor, McpExecutor, ShellCommandError, TerminalExecutor } from "@brainary/poa-sdk/executor";
import { image, text } from "@brainary/poa-sdk/io";

const fileExecutor = new FileExecutor();
const terminalExecutor = new TerminalExecutor();
const mcpExecutor = new McpExecutor();
~~~

## FileExecutor

### apply_patch(patch)

**作用**：按 apply_patch 格式新增、删除、修改或移动文件。

**输入类型**：patch: string，必填。格式如下，路径相对于当前程序的工作目录：

~~~text
*** Begin Patch
*** Add File: <路径>
+<新文件的每一行>
*** Delete File: <路径>
*** Update File: <路径>
*** Move to: <新路径>
@@ <定位用的上下文行，可省略>
 <不变的行>
-<删除的行>
+<新增的行>
*** End Patch
~~~

一个补丁可以包含多个 Add File、Delete File 和 Update File 段；Move to 可选，只能紧跟在 Update File 之后。

**返回值类型**：Promise<void>。

**异常**：补丁格式错误或无法应用（如要修改的行不存在）时抛出 Error。

**示例**：

~~~js
await fileExecutor.apply_patch(`*** Begin Patch
*** Add File: notes/todo.md
+- 补充测试
*** Update File: README.md
@@ ## 安装
-npm install
+npm install --save-dev
*** End Patch`);
~~~

### view_image(path, detail?, environment_id?)

**作用**：读取本地图片文件，返回 data URL。

**输入类型**：

- path: string，必填，图片文件的路径。
- detail?: "high" | "original"。默认 "high"，返回缩放后的图片；"original" 在模型支持时保留原始分辨率。
- environment_id?: string，执行环境 ID，省略时使用主环境。

**返回值类型**：Promise<ViewImageResult>，可以直接传给 [image](07-io.md#imagevalue-detail) 输出。

~~~ts
interface ViewImageResult {
  image_url: string;                // 图片的 data URL
  detail: "high" | "original";      // 实际使用的细节级别
}
~~~

**示例**：

~~~js
const picture = await fileExecutor.view_image("./out.png");
image(picture);
~~~

## TerminalExecutor

### exec_command(cmd, options?)

**作用**：运行一条 shell 命令。命令在 yield_time_ms 内结束时返回 exit_code 和全部输出；未结束时返回 session_id，之后用 [write_stdin](#write_stdinsession_id-chars-yield_time_ms-max_output_tokens) 读取后续输出或写入输入。

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

| 字段 | 说明 |
| --- | --- |
| workdir | 工作目录，默认为当前程序的工作目录 |
| tty | true 时为命令分配伪终端，需要交互输入的命令设为 true；默认 false |
| yield_time_ms | 等待命令结束的时间，单位毫秒，默认 10000，范围 250–30000，超出范围按边界值处理 |
| max_output_tokens | 返回输出的 token 上限；省略时返回全部输出 |
| shell | 运行命令的 shell，默认为用户的默认 shell |
| login | 是否以 login shell 运行 |
| environment_id | 执行环境 ID，省略时使用主环境 |
| sandbox_permissions | 本条命令的权限：use_default（默认）按当前沙箱运行；require_escalated 申请在沙箱外运行；with_additional_permissions 申请 additional_permissions 中的额外权限。申请须经审批策略批准，未批准时命令不运行 |
| justification | 申请 require_escalated 时向审批方说明的理由 |
| prefix_rule | 申请 require_escalated 时一并申请的命令前缀，如 `["git", "pull"]`，批准后以该前缀开头的命令不再审批 |
| additional_permissions | 申请 with_additional_permissions 时需要的文件系统或网络权限 |

**返回值类型**：Promise<ExecOutput>。

~~~ts
interface ExecOutput {
  output: string;                 // 本次返回的输出
  exit_code?: number;             // 命令已结束时为退出码
  session_id?: number;            // 命令仍在运行时为会话 ID
  wall_time_seconds: number;      // 本次调用的耗时，单位秒
  original_token_count?: number;  // 截断前输出的 token 数
  chunk_id?: string;              // 输出块 ID
}
~~~

**示例**：

~~~js
const result = await terminalExecutor.exec_command("pwd", { workdir: "." });
text(result.output);
~~~

### write_stdin(session_id, chars?, yield_time_ms?, max_output_tokens?)

**作用**：向 exec_command 返回的会话写入输入，并返回此后新增的输出。chars 为空时只读取输出。

**输入类型**：

- session_id: number，必填，exec_command 返回的 session_id。
- chars?: string，写入的内容，默认为空字符串。
- yield_time_ms?: number，等待输出的时间，单位毫秒，默认 250。chars 非空时范围 250–30000；chars 为空时范围 5000–300000。超出范围按边界值处理，命令提前结束时立即返回。
- max_output_tokens?: number，返回输出的 token 上限，省略时返回全部输出。

**返回值类型**：Promise<ExecOutput>。命令结束后 exit_code 有值，session_id 不再可用。

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

**作用**：运行命令直到结束，按行返回全部输出。行尾的 `\n` 或 `\r\n` 被去掉，输出末尾的空行不计入结果。

**输入类型**：command: string，必填；options?: ExecCommandOptions，同 [exec_command](#exec_commandcmd-options)。

**返回值类型**：Promise<string[]>。

**异常**：退出码不为 0 时抛出 [ShellCommandError](#shellcommanderror)。

**示例**：

~~~js
const names = await terminalExecutor.shell_lines("find . -maxdepth 1 -type f");
text(names);
~~~

### wait_for_environment(environment_id)

**作用**：等待状态为 starting 的执行环境启动完成。环境已就绪时立即返回。

**输入类型**：environment_id: string，必填。

**返回值类型**：Promise<EnvironmentReady>。

~~~ts
interface EnvironmentReady {
  environment_id: string;
  status: "ready";
}
~~~

**异常**：环境既不是 ready 也不是 starting，或启动失败时抛出 Error。

**示例**：

~~~js
const result = await terminalExecutor.wait_for_environment("env-123");
text(result.status);
~~~

### ShellCommandError

**作用**：shell_lines 运行的命令退出码不为 0 时抛出。

**类型定义**：

~~~ts
class ShellCommandError extends Error {
  readonly command: string;          // 运行的命令
  readonly exit_code: number | null; // 退出码
  readonly output: string;           // 命令的全部输出
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

## McpExecutor

McpExecutor 访问当前程序可用的 MCP server，包括 config.toml 的 `[mcp_servers]` 和 manifest.toml 的 [`[[capabilities.mcp]]`](../02-configuration.md#随包携带-mcp-server) 中声明的 server。

### list_mcp_resources(server?, cursor?)

**作用**：列出 MCP server 提供的资源，省略 server 时列出所有 server 的资源。

**输入类型**：server?: string，server 名；cursor?: string，上一页返回的 nextCursor。

**返回值类型**：Promise<McpResourcePage>。

~~~ts
interface McpResourcePage {
  server?: string;       // 传入 server 时为该 server 名
  resources: McpResource[];
  nextCursor?: string;   // 还有下一页时出现
}

interface McpResource {
  server: string;        // 资源所属的 server
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

**作用**：列出 MCP server 提供的资源模板，省略 server 时列出所有 server 的模板。

**输入类型**：server?: string，server 名；cursor?: string，上一页返回的 nextCursor。

**返回值类型**：Promise<McpResourceTemplatePage>。

~~~ts
interface McpResourceTemplatePage {
  server?: string;       // 传入 server 时为该 server 名
  resourceTemplates: McpResourceTemplate[];
  nextCursor?: string;   // 还有下一页时出现
}

interface McpResourceTemplate {
  server: string;        // 模板所属的 server
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

**作用**：读取一个 MCP 资源。

**输入类型**：server: string；uri: string。两个参数均必填。

**返回值类型**：Promise<McpResourceRead>。

~~~ts
interface McpResourceRead {
  server: string;
  uri: string;
  contents: McpResourceContents[];
}

type McpResourceContents =
  | { uri: string; mimeType?: string; text: string }   // 文本资源
  | { uri: string; mimeType?: string; blob: string };  // 二进制资源，blob 为 base64
~~~

**示例**：

~~~js
const resource = await mcpExecutor.read_mcp_resource("docs", "file:///README.md");
text(resource.contents);
~~~

### call_mcp_tool(server, tool, args?)

**作用**：调用 MCP server 的一个工具。

**输入类型**：

- server: string，必填，server 名：config.toml 中 `[mcp_servers.<name>]` 的 `<name>`，或 manifest.toml 中 `[[capabilities.mcp]]` 的 name。
- tool: string，必填，工具名，即 server 在 tools/list 中返回的 name。
- args?: object，工具参数，结构由该工具的 inputSchema 决定，默认 `{}`。

TypeScript 中可以用泛型参数声明 structuredContent 的类型：`call_mcp_tool<{ count: number }>(...)`。

**返回值类型**：Promise<[McpToolResult](#mcptoolresult)>。

工具执行出错、server 出错或调用未获审批时，返回 isError 为 true 的结果，不抛出异常。未在 annotations 中标注 `readOnlyHint: true` 的工具，调用前需要审批。

**异常**：server 或工具不存在时抛出 ReferenceError。

**示例**：

~~~js
const result = await mcpExecutor.call_mcp_tool("echo", "echo", { text: "hello" });
if (result.isError) throw new Error("echo 调用失败");
text(result.structuredContent ?? result.content);
~~~

### McpToolResult

**作用**：call_mcp_tool 的返回值，即 MCP 的 CallToolResult。

**类型定义**：

~~~ts
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
~~~

| 字段 | 说明 |
| --- | --- |
| content | 工具返回的内容块。image 块可以直接传给 [image](07-io.md#imagevalue-detail)，audio 块可以直接传给 [audio](07-io.md#audiovalue) |
| isError | 为 true 时表示调用失败，content 中是错误信息 |
| structuredContent | 工具返回的结构化结果；工具没有返回时不存在 |

McpResourceContents 见 [read_mcp_resource](#read_mcp_resourceserver-uri)。

**示例**：

~~~js
const result = await mcpExecutor.call_mcp_tool("render", "chart", { data: [1, 2, 3] });
for (const block of result.content) {
  if (block.type === "image") image(block);
  else if (block.type === "text") text(block.text);
}
~~~

## tools

**作用**：每个执行器类的静态只读属性，列出该执行器使用的 [BrainaryTool](06-brainary-tool.md) 成员，可以直接放入 [Capabilities](02-agent-config.md#capabilities).brainary_tool。

**类型定义**：静态只读对象，键为成员名，值为同名的 BrainaryTool 成员。

| 执行器 | tools |
| --- | --- |
| FileExecutor | apply_patch、view_image |
| TerminalExecutor | exec_command、write_stdin、wait_for_environment |
| McpExecutor | list_mcp_resources、list_mcp_resource_templates、read_mcp_resource |
| AgentExecutor | spawn_agent、create_agent、send_input、wait_agent、resume_agent、close_agent、get_agent_status |

MCP 工具不在 tools 中，用 Capabilities.mcp 授予。wait_for_environment 只在 Brainary 启用执行环境功能时存在，未启用时不能授予。

**示例**：

~~~js
const brainary_tool = [TerminalExecutor.tools.exec_command, TerminalExecutor.tools.write_stdin, FileExecutor.tools.apply_patch];
~~~

## 上下文与 hook

以下类型尚未实现。

### ExecutorContext

**状态**：尚未实现，以下为设计描述。

**作用**：描述一次执行调用的调用方、方法、输入和环境，作为 pre hook 的输入。

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

**作用**：在 ExecutorContext 的基础上补充执行结果、错误和耗时，作为 post hook 的输入。

**类型定义**：

~~~ts
interface PostExecutorContext extends ExecutorContext {
  outcome: "completed" | "failed" | "timed_out" |
    "canceled" | "denied" | "stubbed";
  output?: unknown;
  error?: string;
  duration_ms: number;
}
~~~

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

**作用**：在执行器调用前后插入审计、限流或结果处理逻辑，作为执行器构造函数的参数。

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

**作用**：执行器调用前运行的单个 hook，接收 [ExecutorContext](#executorcontext)。

**类型定义**：`(context: ExecutorContext) => unknown`。

**示例**：

~~~js
const pre_executor_hook = (context) => {
  text({ call_id: context.call_id, method: context.method });
};
const hooks = { pre_executor_hooks: [pre_executor_hook] };
~~~

### PostExecutorHooks

**状态**：尚未实现，以下为设计描述。

**作用**：执行器调用完成后运行的单个 hook，接收 [PostExecutorContext](#postexecutorcontext)。

**类型定义**：`(context: PostExecutorContext) => unknown`。

**示例**：

~~~js
const post_executor_hook = (context) => {
  text({ outcome: context.outcome, duration_ms: context.duration_ms });
};
const hooks = { post_executor_hooks: [post_executor_hook] };
~~~
