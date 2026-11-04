# BrainaryTool

### BrainaryTool

**作用**：宿主工具枚举，包括各执行器封装的工具和可授予 Agent 的工具，可直接作为
[Capabilities](02-agent-config.md#capabilities).brainary_tool 的授予项。

**类型定义**：

~~~ts
enum BrainaryTool {
  exec_command = "exec_command",
  write_stdin = "write_stdin",
  wait_for_environment = "wait_for_environment",
  apply_patch = "apply_patch",
  view_image = "view_image",
  web_search = "web_search",
  request_user_input = "request_user_input",
  clock__sleep = "clock__sleep",
  list_mcp_resources = "list_mcp_resources",
  list_mcp_resource_templates = "list_mcp_resource_templates",
  read_mcp_resource = "read_mcp_resource",
  spawn_agent = "multi_agent_v1__spawn_agent",
  create_agent = "multi_agent_v1__create_agent",
  send_input = "multi_agent_v1__send_input",
  wait_agent = "multi_agent_v1__wait_agent",
  resume_agent = "multi_agent_v1__resume_agent",
  close_agent = "multi_agent_v1__close_agent",
  get_agent_status = "multi_agent_v1__get_agent_status",
}
~~~

| 类别 | 成员 |
| --- | --- |
| 终端 | exec_command、write_stdin、wait_for_environment |
| 文件 | apply_patch、view_image |
| 交互与工具 | web_search、request_user_input、clock__sleep |
| MCP 资源 | list_mcp_resources、list_mcp_resource_templates、read_mcp_resource |
| 多 Agent | spawn_agent、create_agent、send_input、wait_agent、resume_agent、close_agent、get_agent_status |

wait_for_environment、request_user_input、clock__sleep 和 web_search 需宿主开启后可用。MCP 工具通过 Capabilities.mcp 授予。
各执行器的 tools 是该枚举的子集，见 [执行器的 tools 入口](05-executors.md#执行器的-tools-入口)，可与其他成员合并为一个授予列表。

**示例**：

~~~js
import { BrainaryTool, TerminalExecutor, text } from "@brainary/poa-sdk";

text(BrainaryTool.close_agent); // "multi_agent_v1__close_agent"

const brainary_tool = [
  ...Object.values(TerminalExecutor.tools),
  BrainaryTool.web_search,
  BrainaryTool.spawn_agent,
];
~~~
