# PoA API 简表

## 全局Primitive

使用globalThis注入，挂载在Javascript全局对象上，能够直接通过object_name（）调用。

| 工具名称 | 作用 |
| --- | --- |
| **`text`** | 输出文本或 JSON 数据。 |
| **`image`** | 输出图片。 |
| **`audio`** | 输出音频。 |
| **`generatedImage`** | 输出生成图片及其提示信息。 |
| `store` | 在当前 code-mode 会话中保存 JSON 数据。 |
| `load` | 读取当前 code-mode 会话中保存的数据。 |
| **`exit`** | 立即结束当前脚本。 |
| `setTimeout` | 安排一个函数在指定毫秒数后执行一次，返回定时器 ID。 |
| `clearTimeout` | 按定时器 ID 取消一个尚未执行的延迟函数。 |
| **`notify`** | 异步并立即将文本注入当前线程历史。 |
| **`yield_control`** | 将当前脚本通过 `text`、`image` 等积累的输出立即返回给模型。 |
| **`ask`** | PoA 脚本一次向用户提出 1–3 个问题并等待回答，按问题顺序返回选择项和自由文本；`interactive` 模式下等待用户回答，`unavailable` 模式下立即返回不可交互状态，`disabled` 模式下不提供此函数。 |
| `ALL_TOOLS` | 提供当前可用工具的名称和说明。 |
| **`POA_INPUT`** | 提供本次 PoA 的业务输入。 |
| **tools** | 工具集，根据配置文件，模型，扩展决定 |

## prelude

使用JavaScript提供的高层封装，大部分内部都调用的是tools的能力，原理是将一段固定的JavaScript粘贴到PoA程序前面

| 名称              | 更具体的作用                                                 |
| --- | --- |
| **`runBatch`**    | 按任务列表**创建**多个子代理并收集答复，按输入顺序返回 `[{handle, reply}]`；未取得答复时 `reply` 为 `null`。 |
| **`spawnAgent`**  | 向一个新子代理派发初始任务，返回 `{key, label, meta}` 句柄；`key` 是 v1 的代理 ID 或 v2 的完整任务名。 |
| **`spawnMany`**   | 按任务列表创建多个子代理，限制创建调用的并发数，按输入顺序返回句柄数组。 |
| **`collectAll`**  | 按句柄列表收集多个子代理的最终答复，返回以 `handle.key` 为键的 `Map`；超时、失败或没有答复正文时对应值为 `null`。 |
| **`sendAndWait`** | 向一个**已有子代理**发送后续任务，等待与上次答复文本不同的新答复；超时或未观察到文本变化时返回 `null`。 |
| **`closeAll`**    | 按handle列表尝试关闭多个 v1 子代理，忽略关闭异常；v2 后端下直接返回。 |
| `shellLines`      | 调用一次 `exec_command`，将本次返回的输出按行拆分、去除首尾空白和空行，可再按过滤函数筛选，返回字符串数组。 |
| `parseJsonReply`  | 从一段回复文本的第一个 `{` 到最后一个 `}` 提取内容并解析 JSON，返回 `{value, error}`。 |
| `mapLimit`        | 对输入数组的每个元素执行异步函数，同时最多运行 `limit` 个调用，按输入顺序返回结果数组。 |
| `SAFE_NAME`       | 一个正则表达式；用 `SAFE_NAME.test(name)` 判断非空名称是否只包含英文字母、数字、点、下划线和连字符。 |
| `AGENT_BACKEND`   | 根据 `ALL_TOOLS` 中的代理创建工具确定后端，输出 `"v1"`、`"v2"` 或 `null`；两者都存在时优先选 v1。 |
| `requireAgents`   | 返回已识别的代理后端（V1/V2）；未识别到后端时抛出异常，并列出当前工具名称。 |
| `hasTool`         | 判断一个工具名称是否在 prelude 初始化时的 `ALL_TOOLS` 中，返回布尔值。 |
| `callTool`        | 按名称调用一个工具，返回名称、状态、结果、异常信息和耗时；状态为 `"ok"`、`"absent"` 或 `"error"`。 |
| `shapeOf`         | 查看一个值有哪些字段、字段是什么类型 shapeOf({ name: "worker", count: 3 }) |
| `timed`           | 执行一个函数并记录耗时，返回 `{ok, value, error, ms}`；函数抛出的异常被捕获到 `error`。 |

## tools

通过 `tools.<名称>` 调用；可用项取决于配置、模型、环境和扩展。

| 工具名称 | 作用 |
| --- | --- |
| `exec_command` | 在选定执行环境的工作目录中运行一段 shell 命令或脚本，返回进程输出；进程已结束时返回退出码，仍在运行时返回一个可供续接的 `session_id`。 |
| `write_stdin` | 按 `session_id` 向一个由 `exec_command` 创建的运行中进程会话写入标准输入，并读取新增输出；输入为空时只读取输出。 |
| `shell_command` | 在指定工作目录中运行一段一次性 shell 命令或脚本，返回进程输出和退出码。 |
| `apply_patch` | 用一份补丁新增、修改、删除或移动一个或多个文件。 |
| `view_image` | 按文件路径读取执行环境中的一张本地图片，返回图片的 Data URL 和图像细节级别，供模型查看。 |
| `update_plan` | 提交步骤列表，更新当前任务的一份计划及每个步骤的状态；同时至多一个步骤处于进行中状态。 |
| `create_goal` | 为当前会话创建并持久化一个工作目标，记录要完成的事项及可选的 token 预算；每个会话只保留一个当前目标，已有未完成目标时拒绝创建。 |
| `get_goal` | 读取当前会话的一个工作目标，返回目标内容、状态、token 预算与用量、剩余 token 预算及已用时间；尚未创建目标时返回空目标。 |
| `update_goal` | 将当前会话已有的一个工作目标标记为 `complete`（完成）或 `blocked`（阻塞），返回更新后的目标。 |
| `get_context_remaining` | 返回当前会话的当前上下文窗口剩余的 token 数；无法获取时返回空值。 |
| `request_permissions` | 向用户申请一个执行环境的网络访问权限或一个或多个文件系统路径的读写权限，等待授权并返回实际获批的权限范围。 |
| `clock__curr_time` | 返回当前 UTC 日期和时间，格式为 `YYYY-MM-DD HH:MM:SS UTC`。 |
| `memories__list` | 分页列出 Codex 记忆存储中一个目录的直接子文件和子目录；省略路径时列出记忆存储根目录。 |
| `memories__read` | 按记忆存储内的相对路径读取一个记忆文件，返回文件文本；可指定起始行和最多读取的行数。 |
| `memories__search` | 用一个或多个查询字符串搜索记忆文件，返回匹配的文件路径、行号和文本；可限定文件或目录范围及匹配方式。 |
| `memories__add_ad_hoc_note` | 在用户明确要求记住、忘记或更新信息后，将一条 Markdown 笔记保存为记忆存储中的一个新文件，供后续记忆整理使用。 |
| **`multi_agent_v1__spawn_agent`** | 创建一个 v1 子代理并派发初始任务，返回子代理的 `agent_id` 及可用的昵称。 |
| **`multi_agent_v1__wait_agent`** | 按代理 ID 等待一个或多个 v1 子代理，任一代理到达终态时返回终态状态及可用的最终答复；超时时返回空状态。 |
| **`multi_agent_v1__send_input`** | 按代理 ID 向一个已有 v1 子代理发送文本或结构化输入；可排队处理，也可打断子代理当前任务后立即处理，返回输入提交 ID。 |
| **`multi_agent_v1__resume_agent`** | 按代理 ID 恢复一个已关闭的 v1 子代理，使子代理可再次接收输入和等待调用。 |
| **`multi_agent_v1__close_agent`** | 按代理 ID 关闭一个 v1 子代理及所有仍打开的后代代理，返回被指定子代理关闭前的状态。 |
| `collaboration__spawn_agent` | 按任务名创建一个 v2 子代理并派发初始任务，返回子代理在任务树中的完整任务名及可用的昵称。 |
| `collaboration__wait_agent` | 等待当前代理信箱收到任一代理的消息或终态通知；新用户输入或超时也会结束等待，返回活动摘要而非消息正文。 |
| `collaboration__send_message` | 按相对或完整任务名向一个已有 v2 代理发送消息，不启动接收代理的新一轮任务。 |
| `collaboration__followup_task` | 按代理 ID 或任务名向一个已有、非根节点的 v2 子代理派发后续任务；子代理空闲时启动新一轮，运行中时递送任务消息。 |
| `collaboration__interrupt_agent` | 按代理 ID 或任务名打断一个 v2 代理当前轮次，返回代理被打断前的状态；代理仍可接收消息和后续任务。 |
| `collaboration__list_agents` | 列出当前根会话任务树中的存活代理及各自的任务名和状态；完成状态可包含最终答复，也可按任务路径前缀筛选代理。 |
| `list_mcp_resources` | 列出一个指定 MCP server 或所有当前可访问 MCP server 提供的资源条目，包括文件、数据库结构等资源的名称和 URI；指定 server 时可用游标翻页。 |
| `list_mcp_resource_templates` | 列出一个指定 MCP server 或所有当前可访问 MCP server 提供的参数化资源模板，包括 URI 模板；指定 server 时可用游标翻页。 |
| `read_mcp_resource` | 按 server 名和资源 URI 读取一个 MCP 资源，返回 server 提供的文本或二进制内容。 |
| **`mcp__<server>__<tool>`** | 按实际 server 名和工具名调用一个已注册的 MCP 工具，传入工具参数并返回执行结果；名称为占位示例，具体操作实体由该工具定义。 |
| **`web__run`** | 通过网页扩展执行一个或多个联网查询、网页打开或页面内查找等操作，返回对应的搜索结果或网页内容。 |
| `skills__list` | 分页列出所选来源（orchestrator 或 executor）提供的技能，返回技能名称、说明及读取资源所需的来源、包名和主资源标识。 |
| `skills__read` | 按来源、包名和资源标识读取一个技能包内的主说明或引用资源，每次返回一页文本及后续页游标。 |
| `image_gen__imagegen` | 按文字描述生成一张新图片，或基于最多五张本地或会话参考图片生成一张编辑后的图片，返回图片 Data URL 及可用的保存位置提示。 |
| `wait_for_environment` | 按 `environment_id` 等待一个标记为 `starting` 的执行环境完成启动，成功时返回环境 ID 和 `ready` 状态，启动失败时返回错误。 |
| `list_available_plugins_to_install` | 列出当前已知的可安装插件或连接器，返回候选项的 ID、名称、类型和说明，供后续安装请求选择。 |
| `request_plugin_install` | 按插件或连接器 ID 向用户发起一个安装请求，等待用户响应并检查安装是否完成，返回安装结果。 |

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
