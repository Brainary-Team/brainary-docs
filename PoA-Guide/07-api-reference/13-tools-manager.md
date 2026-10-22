# Tools

### tools

**作用**：标识当前运行时可暴露的工具类别；具体工具参数和返回值由对应执行器或 provider 定义。

**类型定义**：

~~~ts
type Tool =
  | "exec_command"
  | "web_search"
  | "apply_patch"
  | "request_user_input"
  | "clock_sleep"
  | "multi_agent"
  | "memory_*";
~~~

工具枚举值可用于 [Capabilities](02-agent-config.md).brainary_tool 的声明和能力检查。

**示例**：

~~~js
const requested = ["exec_command", "multi_agent"];
const enabled = requested.filter((name) =>
  capabilities.brainary_tool?.includes(name),
);
~~~
