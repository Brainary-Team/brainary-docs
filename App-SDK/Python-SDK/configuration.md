# 配置文件使用说明

本页集中说明 Python SDK 如何加载配置文件，以及如何通过文件管理 model provider、
MCP 和 Skills。代码配置示例分别见[创建与运行 Agent](agents.md)、[使用 MCP](mcp.md)
和[使用 Skills](skills.md)。

## 配置位置与加载

运行时默认读取 `~/.codex/config.toml`。设置 `CODEX_HOME` 后，用户配置改为
`$CODEX_HOME/config.toml`；该环境变量指向的目录必须已经存在。
`CODEX_HOME` 同时决定会话等运行时数据的存放位置。

先在 `/srv/my-app/codex-home/config.toml` 中保存下面需要的配置，再启动客户端：

```python
from brainary import Brainary, BrainaryConfig

config = BrainaryConfig(
    brainary_bin="/opt/brainary/codex",
    env={"CODEX_HOME": "/srv/my-app/codex-home"},
)

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(cwd="/srv/workspaces/repo-a")
    result = agent.run("检查当前改动。")
    print(result.final_response)
```

`BrainaryConfig.env` 在父进程环境基础上更新变量，部署环境中已导出的认证变量
会被运行时继承。项目级 `.codex/config.toml` 是否加载受项目可信状态影响；本文的
持久配置示例统一放在用户级配置文件中。

`BrainaryConfig.config_overrides` 会转换为 `codex --config <value>` 启动参数，
`thread_start(config=...)` 则提供 thread 级配置覆盖。两者都不修改配置文件；在同一
配置键上，thread 覆盖优先于启动覆盖，启动覆盖优先于普通文件配置。管理员要求仍可
约束最终配置。修改文件后，重新启动客户端并创建新 thread，避免沿用旧会话的设置。

## Model provider

在用户配置中注册 provider，并设置默认模型：

```toml
model = "<default-responses-model-id>"
model_provider = "corp-responses"

[model_providers.corp-responses]
name = "Corporate Responses gateway"
base_url = "https://llm.example.com/v1"
env_key = "CORP_LLM_API_KEY"
wire_api = "responses"
request_max_retries = 4
stream_max_retries = 5
stream_idle_timeout_ms = 300000

[model_providers.corp-responses.env_http_headers]
X-Gateway-Key = "CORP_GATEWAY_KEY"

[model_providers.anthropic-direct]
name = "Anthropic Messages"
base_url = "https://api.anthropic.com/v1"
env_key = "ANTHROPIC_API_KEY"
wire_api = "anthropic"
```

`model` 是模型 ID；`model_provider` 是 `model_providers` 中的键。
`base_url` 是 API 根地址，`wire_api` 决定追加 `/responses` 或 `/messages`。
当前只支持 `responses` 和 `anthropic`，未填写时默认为 `responses`。

`env_key` 和 `env_http_headers` 的值都是环境变量名，分别用于 API key 和自定义
认证 header；应在启动 Python 前提供这些变量。固定的非敏感 header 使用
`http_headers`。自定义 provider 应使用新的 ID；内置 ID 的覆盖边界见
[模型配置说明](agents.md#在-python-中注册并应用-provider)。

`thread_start()` 未指定模型和 provider 时使用文件默认值，也可在上面示例的 `with`
块内选择已注册项：

```python
responses_agent = brainary.thread_start(
    model="<responses-model-id>",
    model_provider="corp-responses",
)
anthropic_agent = brainary.thread_start(
    model="<anthropic-model-id>",
    model_provider="anthropic-direct",
)
```

## MCP

stdio server 的配置：

```toml
[mcp_servers.files]
command = "python"
args = ["/absolute/path/to/files_server.py"]
cwd = "/absolute/path/to/server-workdir"
enabled = true
required = true
startup_timeout_sec = 10.0
tool_timeout_sec = 60.0
supports_parallel_tool_calls = false

[mcp_servers.files.env]
LOG_LEVEL = "info"
```

Streamable HTTP server 的配置，包括工具过滤和审批规则：

```toml
[mcp_servers.search]
url = "https://mcp.example.com/mcp"
bearer_token_env_var = "SEARCH_MCP_TOKEN"
enabled = true
required = true
startup_timeout_sec = 10.0
tool_timeout_sec = 60.0
enabled_tools = ["search", "fetch"]
disabled_tools = ["delete"]
default_tools_approval_mode = "prompt"

[mcp_servers.search.http_headers]
X-Client = "brainary"

[mcp_servers.search.env_http_headers]
X-Tenant-Token = "TENANT_TOKEN"

[mcp_servers.search.tools.search]
approval_mode = "auto"

[mcp_servers.search.tools.fetch]
approval_mode = "approve"
```

`command` 和 `url` 分别选择 stdio 和 HTTP transport，不能混用各自的专属字段。
认证字段使用环境变量名，运行时拒绝 `bearer_token` 明文字段。
配置字段的含义、工具过滤和调用事件见[使用 MCP](mcp.md)。

加载文件后，普通 `thread_start()` 即可获得这些 server，无须再次传入
`config={"mcp_servers": ...}`。实际调用由 Agent 根据任务和可用工具决定。

## Skills 管理

Skill 内容保存在 `SKILL.md` 和配套资源中。配置文件控制已发现 Skill 的启用状态，
不会通过 `skills.config.path` 注册新的发现目录。

| 发现位置 | 作用范围 |
| --- | --- |
| 项目根到当前 `cwd` 路径上各级 `.agents/skills` | 对应目录及其子目录中的 Agent；通过 `cwd` 和项目根识别规则确定范围 |
| 项目配置层的 `.codex/skills` | 加载该项目配置层的 Agent |
| `$HOME/.agents/skills` | 当前操作系统用户的 Skills |
| `$CODEX_HOME/skills` | 共用该配置目录的 Agent；源码保留此位置用于兼容旧用法 |
| `$CODEX_HOME/skills/.system` | 运行时内置 Skills，可通过 `skills.bundled.enabled` 控制 |
| 系统配置目录中的 `skills`（Unix 通常为 `/etc/codex/skills`） | 管理员部署的 Skills |
| 已启用插件的 Skill 目录 | 随插件加载的 Skills |

保持自动 Skills 目录说明，同时关闭内置 Skills：

```toml
[skills]
include_instructions = true

[skills.bundled]
enabled = false
```

这两个开关默认均为 `true`。`include_instructions` 只控制自动目录说明，
`bundled.enabled` 只控制内置 Skills，均不会禁用项目或用户目录中的单项 Skill。

按 `SKILL.md` 的绝对路径禁用本地 Skill：

```toml
[[skills.config]]
path = "/absolute/path/to/workspace/.agents/skills/diagnose-failure/SKILL.md"
enabled = false
```

也可以按名称禁用所有已发现的同名 Skill：

```toml
[[skills.config]]
name = "diagnose-failure"
enabled = false
```

每项规则必须且只能指定 `path` 或 `name`，并提供 `enabled`。本地 Skill 推荐按路径
选择，以免影响其他作用域中的同名 Skill。重新启用时将 `enabled` 改为 `true`，或在
没有其他禁用规则时删除该项。

当前运行时仅从用户配置层和 session 覆盖层读取 `skills.config` 启用规则，因此应把
这些持久规则写在 `$CODEX_HOME/config.toml` 中。项目级 Skill 内容仍可放在
`.agents/skills` 中。用户级规则作用于共用此配置目录的 Agent，thread 级覆盖的示例见
[使用 Skills](skills.md#用代码配置-skill-池)。
