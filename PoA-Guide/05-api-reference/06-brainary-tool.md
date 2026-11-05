# BrainaryTool

### BrainaryTool

**作用**：Brainary 工具的枚举，用作 [Capabilities](02-agent-config.md#capabilities).brainary_tool 的元素，为 create_agent 创建的 agent 授予工具。

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

| 成员 | 授予后 agent 可以 |
| --- | --- |
| exec_command、write_stdin | 运行命令，与运行中的命令交互 |
| wait_for_environment | 等待执行环境启动 |
| apply_patch | 新增、删除和修改文件 |
| view_image | 查看本地图片 |
| web_search | 搜索网页 |
| request_user_input | 向用户提问 |
| clock__sleep | 等待一段时间 |
| list_mcp_resources、list_mcp_resource_templates、read_mcp_resource | 列出和读取 MCP 资源 |
| spawn_agent、create_agent、send_input、wait_agent、resume_agent、close_agent、get_agent_status | 派出、创建、等待、查询、关闭和恢复 agent |

- 授予的每个工具都须是创建者自己拥有的，否则 create_agent 抛出错误。wait_for_environment、request_user_input、clock__sleep 和 web_search 只在 Brainary 启用对应功能时存在。
- MCP 工具不在本枚举中，用 Capabilities.mcp 授予。
- 各执行器的 [tools](05-executors.md#tools) 是本枚举的子集，可以与其他成员合并为一个列表。

**示例**：

~~~js
import { BrainaryTool, FileExecutor } from "@brainary/poa-sdk";

const brainary_tool = [
  BrainaryTool.exec_command,
  BrainaryTool.write_stdin,
  ...Object.values(FileExecutor.tools),
];
~~~
