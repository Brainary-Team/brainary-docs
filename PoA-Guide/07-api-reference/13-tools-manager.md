# Tools

### tools

**作用**：全局工具枚举，汇总各执行器的工具。每个枚举成员的值都是字符串，用于标识对应工具；具体调用由执行器方法完成。

**类型定义**：

~~~ts
type Tool =
  | "exec_command"
  | "write_stdin"
  | "shelllines"
  | "wait_for_environment"
  | "apply_patch"
  | "view_image"
  | "send_message"
  | "context_compact"
  | "close"
  | "addition_message"
  | "get_context_usage"
  | "get_token_usage"
  | "get_state"
  | "web_search"
  | "request_user_input"
  | "clock_sleep"
  | "multi_agent";

declare const tools: Readonly<Record<string, Tool>>;
~~~

各执行器的 tools 使用同样的字符串枚举形式，包含该执行器提供的工具；全局 tools 汇总这些枚举成员。
工具枚举值可用于 [Capabilities](02-agent-config.md).brainary_tool 的声明和能力检查。

**示例**：

~~~js
text(tools.exec_command);
text(executor.fileExecutor.tools.view_image);

const requested = [tools.exec_command, tools.multi_agent];
const enabled = requested.filter((name) =>
  capabilities.brainary_tool?.includes(name),
);
~~~
