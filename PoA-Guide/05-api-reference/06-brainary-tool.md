# BrainaryTool

### BrainaryTool

**作用**：执行器调用的 Brainary 工具的枚举。各执行器的 [tools](05-executors.md#tools) 是本枚举的子集。

**类型定义**：

~~~ts
enum BrainaryTool {
  exec_command = "exec_command",
  write_stdin = "write_stdin",
  wait_for_environment = "wait_for_environment",
  apply_patch = "apply_patch",
  view_image = "view_image",
  list_mcp_resources = "list_mcp_resources",
  list_mcp_resource_templates = "list_mcp_resource_templates",
  read_mcp_resource = "read_mcp_resource",
}
~~~

| 成员 | 使用它的执行器 | 作用 |
| --- | --- | --- |
| exec_command、write_stdin | TerminalExecutor | 运行命令，与运行中的命令交互 |
| wait_for_environment | TerminalExecutor | 等待执行环境启动 |
| apply_patch | FileExecutor | 新增、删除和修改文件 |
| view_image | FileExecutor | 查看本地图片 |
| list_mcp_resources、list_mcp_resource_templates、read_mcp_resource | McpExecutor | 列出和读取 MCP 资源 |

值是工具在 Brainary 中的名称。wait_for_environment 只在 Brainary 启用执行环境功能时存在。

**示例**：

~~~js
import { BrainaryTool, TerminalExecutor, text } from "@brainary/poa-sdk";

text(TerminalExecutor.tools.exec_command === BrainaryTool.exec_command); // true
~~~
