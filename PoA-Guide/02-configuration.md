# 配置文件

PoA 用到两个配置文件：

| 文件 | 作用 |
| --- | --- |
| `~/.brainary/config.toml` | Brainary 的运行配置：模型、沙箱、agent 层级和数量，对本机运行的所有 PoA 生效 |
| `manifest.toml` | 包的配置：名称、入口、随包携带的 MCP server，随包分发 |

## config.toml

`config.toml` 位于 `~/.brainary/`；设置了 `BRAINARY_HOME` 环境变量时，位于该变量指定的目录。下面是一份完整示例，各部分的说明见后文：

```toml
model = "gpt-5.6-luna"
model_provider = "vendor"
sandbox_mode = "workspace-write"
suppress_unstable_features_warning = true

[features]
code_mode = true

[model_providers.vendor]
name = "主模型服务"
base_url = "https://api.example.com/v1"
env_key = "VENDOR_API_KEY"
wire_api = "responses"
requires_openai_auth = false

[model_providers.deepseek]
name = "DeepSeek"
base_url = "https://api.deepseek.com/anthropic/v1"
env_key = "DEEPSEEK_API_KEY"
wire_api = "anthropic"
requires_openai_auth = false

[agents]
max_depth = 2
max_concurrent_threads_per_session = 8
```

| 配置 | 说明 |
| --- | --- |
| `model` | 默认模型 ID |
| `model_provider` | 默认 provider 的 id，即 `[model_providers.<id>]` 中的 `<id>` |
| `sandbox_mode` | 默认沙箱，见 [沙箱](#沙箱) |
| `suppress_unstable_features_warning` | 不显示实验功能提示 |
| `[features] code_mode` | 运行 PoA 必须设为 `true` |

### 配置 provider

每个 `[model_providers.<id>]` 分组声明一个模型服务：

| 字段 | 说明 |
| --- | --- |
| `name` | 显示名称 |
| `base_url` | API 根地址，不含末尾的 `/responses` 或 `/messages` |
| `env_key` | 保存 API Key 的环境变量名 |
| `wire_api` | 接口类型：Responses 接口填 `"responses"`，Anthropic Messages 接口填 `"anthropic"` |
| `requires_openai_auth` | 固定为 `false` |

需要多个模型服务时，为每个服务写一个分组，并分别设置各自的 API Key：

```bash
export VENDOR_API_KEY="替换为主模型服务的 API Key"
export DEEPSEEK_API_KEY="替换为 DeepSeek 的 API Key"
```

`spawn_agent` 派出的 agent 使用默认 provider 和默认模型。程序用 `create_agent` 创建 agent 时，可以在 `model.provider` 中指定任一 provider 的 id：

```ts
import { AgentExecutor, text } from "@brainary/poa-sdk";

const agents = new AgentExecutor();
const writer = await agents.create_agent({
  name: "writer",
  agent_config: { model: { model_id: "deepseek-v4-pro", provider: "deepseek" } },
});
const editor = await agents.create_agent({
  name: "editor",
  agent_config: { model: { model_id: "gpt-5.6-luna", provider: "vendor" } },
});

const draft = await writer.run("写一段 100 字以内的介绍：PoA 用程序编排多个 AI agent。", undefined, undefined, 300_000);
const final = await editor.run(`把下面这段介绍改得更简洁：\n${draft.result}`, undefined, undefined, 300_000);
text(final.result);
```

`create_agent` 的其他配置见 [AgentConfig](05-api-reference/02-agent-config.md)。

### 沙箱

`sandbox_mode` 决定 agent 执行命令和修改文件的权限：

| 取值 | 说明 |
| --- | --- |
| `read-only` | 默认值。只能读取文件 |
| `workspace-write` | 可以修改工作目录内的文件；agent 需要写文件时使用 |
| `danger-full-access` | 不做限制，只在隔离环境中使用 |

`workspace-write` 默认不能访问网络。agent 需要联网时，例如安装依赖，加上：

```toml
[sandbox_workspace_write]
network_access = true
```

用 `create_agent` 创建的 agent 可以在 `runtime.sandbox` 中单独设置沙箱，见 [RuntimeSpec](05-api-reference/02-agent-config.md#runtimespec)。

### agent 层级和数量

```toml
[agents]
max_depth = 2
max_concurrent_threads_per_session = 8
default_subagent_model = "gpt-5.6-terra"
```

| 配置 | 默认值 | 说明 |
| --- | --- | --- |
| `max_depth` | `1` | agent 的最大层级。程序派出的 agent 是第 1 层：为 `1` 时它们不能再派出 agent，为 `2` 时每个 agent 可以再派出一层 |
| `max_concurrent_threads_per_session` | `6` | 同时打开的 agent 数量上限。agent 用 [close](05-api-reference/03-agent-executor.md#closeagent_id) 关闭后释放名额 |
| `default_subagent_model` | 默认模型 | `spawn_agent` 派出的 agent 使用的模型 |

程序派出的 agent 较多时，取得结果后及时关闭，或调高 `max_concurrent_threads_per_session`。

### 命令行参数

运行 `brainary poa` 时，命令行参数优先于 `config.toml`：

```bash
brainary poa my-poa.poa -s workspace-write -c agents.max_depth=2
```

| 参数 | 说明 |
| --- | --- |
| `-m <模型>` | 覆盖 `model` |
| `-s <模式>` | 覆盖 `sandbox_mode` |
| `-c <键>=<值>` | 覆盖任一配置项，值按 TOML 解析，例如 `-c agents.max_depth=2` |
| `-C <目录>` | agent 的工作目录，默认为执行命令的目录 |
| `--skip-git-repo-check` | 允许在 Git 仓库外运行 |
| `--json` | 以 JSONL 输出运行事件 |
| `--ephemeral` | 不保存会话记录 |

完整参数用 `brainary poa --help` 查看。

## manifest.toml

`manifest.toml` 放在项目根目录，`brainary poa pack` 按它打包：

```toml
[poa]
namespace = "demo"
name = "repo_scan"
version = "0.1.0"
description = "扫描仓库并汇总每个目录的用途"
runtime = "codex-v8"
poa_api_version = 1
entry = "src/main.ts"

[[capabilities.mcp]]
name = "echo"
type = "stdio"
shell = "node mcp/echo.mjs"
description = "回显消息"
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `namespace`、`name` | 是 | 包的命名空间和名称 |
| `version` | 是 | 包的版本 |
| `description` | 否 | 包的说明；PoA 作为工具提供给模型时，它就是工具说明 |
| `runtime` | 是 | 固定为 `"codex-v8"` |
| `poa_api_version` | 是 | 固定为 `1` |
| `entry` | 是 | 程序入口的源文件，相对于项目根目录 |
| `[[capabilities.mcp]]` | 否 | 随包携带的 MCP server，可以写多个 |

程序接收业务输入时，在项目根目录放一个 `input_schema.ts`，打包时会自动编译并写入包中，不用在 manifest 中声明，见 [输入与输出](03-input-output.md#接收输入)。

### 随包携带 MCP server

把 MCP server 放进项目的 `mcp/` 目录，并在 manifest 中用 `[[capabilities.mcp]]` 声明，它就会随包分发，运行时自动启动：

| 字段 | 说明 |
| --- | --- |
| `name` | server 名，只能使用字母、数字、`_` 和 `-`；调用工具时用这个名字 |
| `type` | 固定为 `"stdio"` |
| `shell` | 启动命令，在包的根目录执行 |
| `description` | 可选，server 的说明 |

完整的 server 实现和调用方式见 [示例](04-examples.md#随包携带-mcp-工具)，返回结构见 [MCP tool 配置](05-api-reference/06-mcp-tool-config.md)。
