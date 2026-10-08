# PoA API 简表

## helper

使用globalThis注入，挂载在Javascript全局对象上的tools及对象

| 工具名称 | 作用 |
| --- | --- |
| **`text`** | 输出文本或 JSON 数据。 |
| `image` | 输出图片。 |
| `audio` | 输出音频。 |
| `generatedImage` | 输出生成图片及其提示信息。 |
| `store` | 在当前 code-mode 会话中保存 JSON 数据。 |
| `load` | 读取当前 code-mode 会话中保存的数据。 |
| `exit` | 立即结束当前脚本。 |
| `setTimeout` | 安排延迟执行的回调。 |
| `clearTimeout` | 取消已安排的延迟回调。 |
| `notify` | 异步将文本注入当前线程历史。 |
| `yield_control` | 让出当前积累的输出，脚本继续执行。 |
| `ALL_TOOLS` | 提供当前可用工具的名称和说明。 |
| **`POA_INPUT`** | 提供本次 PoA 的业务输入。 |

## prelude

 JavaScript 封装的代码，系统会把这段代码自动粘贴到PoA程序前，并且大部分内部都调用的是tools的能力。作用是提供更方便的使用方式。

| 工具名称 | 作用 |
| --- | --- |
| `runBatch` | 批量派发 agent，并收集答复。 |
| `spawnAgent` | 派发一个 agent，返回其句柄。 |
| `spawnMany` | 批量派发 agent，返回句柄列表。 |
| `collectAll` | 等待一组 agent 的终态答复，超时或无答复时返回空值。 |
| `sendAndWait` | 向已有 agent 追问，并尝试收集新的答复。 |
| `closeAll` | 尽力关闭一组 v1 agent；对 v2 无操作。 |
| `shellLines` | 执行命令、续接输出、检查退出码，并按行返回文本。 |
| `parseJsonReply` | 从回复文本中提取并解析 JSON 对象。 |
| `mapLimit` | 限制异步函数的调用并发数，保留结果顺序。 |
| `SAFE_NAME` | 检查名称是否只包含字母、数字、点、下划线和连字符。 |
| `AGENT_BACKEND` | 标识满足 prelude 要求的 agent 后端。 |
| `requireAgents` | 检查 prelude 所需的 agent 工具组是否可用。 |
| `hasTool` | 检查指定工具是否存在。 |
| `callTool` | 调用工具，统一返回调用结果或异常信息。 |
| `shapeOf` | 描述返回值的数据形状。 |
| `timed` | 执行函数，记录耗时并捕获异常。 |

## tools

通过 `tools.<名称>` 调用；可用项取决于配置、模型、环境和扩展。

| 工具名称 | 作用 |
| --- | --- |
| `exec_command` | 执行命令，返回输出或可继续交互的进程会话。 |
| `write_stdin` | 向进程会话写入输入，或继续读取输出。 |
| `shell_command` | 在回落执行路径中运行一次性 shell 命令。 |
| `apply_patch` | 通过补丁修改文件。 |
| `view_image` | 读取本地图片。 |
| `update_plan` | 更新任务计划。 |
| `create_goal` | 创建目标。 |
| `get_goal` | 查询目标状态、预算和用量。 |
| `update_goal` | 将目标标记为完成或阻塞。 |
| `get_context_remaining` | 查询剩余上下文 token 数。 |
| `request_permissions` | 申请额外的文件系统或网络权限。 |
| `clock__curr_time` | 查询当前 UTC 时间。 |
| `memories__list` | 列出记忆文件和目录。 |
| `memories__read` | 读取记忆文件。 |
| `memories__search` | 搜索记忆内容。 |
| `memories__add_ad_hoc_note` | 追加一条临时记忆笔记。 |
| `multi_agent_v1__spawn_agent` | 派发 v1 子代理。 |
| `multi_agent_v1__wait_agent` | 等待指定 v1 子代理到达终态。 |
| `multi_agent_v1__send_input` | 向 v1 子代理发送输入，可选择打断当前工作。 |
| `multi_agent_v1__resume_agent` | 恢复已关闭的 v1 子代理。 |
| `multi_agent_v1__close_agent` | 关闭 v1 子代理及其后代。 |
| `collaboration__spawn_agent` | 按任务名派发 v2 子代理。 |
| `collaboration__wait_agent` | 等待信箱活动、新用户输入或超时，返回状态摘要。 |
| `collaboration__send_message` | 向代理发送消息，不主动启动新一轮。 |
| `collaboration__followup_task` | 下发后续任务，目标空闲时启动新一轮。 |
| `collaboration__interrupt_agent` | 打断代理当前轮次。 |
| `collaboration__list_agents` | 列出代理及其状态和终态答复。 |
| `list_mcp_resources` | 列出 MCP server 提供的资源。 |
| `list_mcp_resource_templates` | 列出 MCP 资源模板。 |
| `read_mcp_resource` | 读取指定 MCP 资源。 |
| `mcp__<server>__<tool>` | 调用 MCP server 提供的工具；此处名称为示意。 |
| `web__run` | 使用扩展进行联网搜索。 |
| `skills__list` | 列出 orchestrator 提供的技能。 |
| `skills__read` | 读取 orchestrator 提供的技能。 |
| `image_gen__imagegen` | 使用扩展生成图片。 |
| `wait_for_environment` | 等待执行环境就绪。 |
| `list_available_plugins_to_install` | 列出可安装的插件或连接器。 |
| `request_plugin_install` | 请求安装插件或连接器。 |

## 在 PoA 中扩展 MCP tool

将 MCP server 文件放入 PoA 包，通过 `manifest.toml` 声明启动方式。宿主启动 server，通过 MCP 的 `tools/list` 发现它提供的工具，再将工具注册到当前 PoA 的 `tools` 中。一个 server 可以提供多个工具。

包目录：

```text
echo-demo/
├── manifest.toml
├── main.js
├── input_schema.ts
└── mcp/
    └── echo.mjs
```

`manifest.toml`：

```toml
[poa]
namespace = "example"
name = "echo_demo"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
# 打包时自动添加 input_schema = "input_schema.json"

[[capabilities.mcp]]
name = "echo"
type = "stdio"
shell = "node mcp/echo.mjs"
```

## MCP tool name 的拼接规则

普通包自带 MCP 工具在默认配置下通常按下面的规则命名：

```text
mcp__<server_name>__<tool_name>
```

| 名称部分 | 来源 |
| --- | --- |
| `mcp` | 宿主默认添加的 MCP 前缀。 |
| `server_name` | `[[capabilities.mcp]]` 中的 `name`。 |
| `tool_name` | MCP server 在 `tools/list` 中声明的工具 `name`。 |
| `__` | namespace 与工具名的连接符。 |

例如，上面的 server 名为 `echo`，工具名也为 `echo`，所以通常是 `mcp__echo__echo`。如果 server 提供名为 `say` 的工具，则通常是 `mcp__echo__say`。

## POA Input

`POA_INPUT` 是调用方传给 PoA 的业务输入，供 `main.js` 直接读取。

在包根目录创建 `input_schema.ts`，使用 TypeScript 声明输入类型和约束：

```typescript
export interface InputParams {
  /** @minLength 1 */
  message: string;
}
```

使用 `codex-poa-pack echo-demo echo-demo.poa` 打包目录时，会将该声明转换为包内的 `input_schema.json`，并在包内 manifest 的 `[poa]` 中自动设置 `input_schema = "input_schema.json"`。源目录中的 manifest 不会被修改。

转换后的 JSON Schema：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "message": { "type": "string", "minLength": 1 }
  },
  "required": ["message"],
  "additionalProperties": false
}
```

调用 PoA 时传入的业务 JSON 示例：

```json
{
  "message": "你好"
}
```

`main.js` 读取输入并调用前面的 MCP echo 工具：

```javascript
const result = await tools.mcp__echo__echo({ message: POA_INPUT.message });
```

