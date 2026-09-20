# 使用 MCP

MCP 为 Agent 提供外部工具。Python SDK 不直接连接 MCP server；它启动 Brainary Codex app-server，运行时读取 MCP 配置、启动或连接 server，并把可用工具暴露给 Agent。

## 工作流程

1. 在 `CODEX_HOME/config.toml` 中声明 MCP server，或用 `CodexConfig.config_overrides` 传入临时覆盖。
2. 用 `CodexConfig(codex_bin=...)` 启动 Brainary 运行时。
3. 创建 Agent，并在 prompt 中要求它完成需要该工具的任务。
4. 使用 `run()` 等待结果，或用 `stream()` 观察 MCP 调用事件。

## 配置 stdio server

在 `CODEX_HOME/config.toml` 中添加：

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

stdio transport 以 `command` 为判据。可用字段包括 `args`、`env`、`env_vars` 和 `cwd`；不要同时配置 `url`、HTTP header 或 bearer token 字段。

`required = true` 表示 server 初始化失败时任务应失败，而不是静默跳过。对业务必需的 server 建议开启。

## 配置 Streamable HTTP server

```toml
[mcp_servers.search]
url = "https://mcp.example.com/mcp"
bearer_token_env_var = "SEARCH_MCP_TOKEN"
enabled = true
required = true
startup_timeout_sec = 10.0
tool_timeout_sec = 60.0

[mcp_servers.search.http_headers]
X-Client = "brainary"

[mcp_servers.search.env_http_headers]
X-Tenant-Token = "TENANT_TOKEN"
```

把 secret 放进 `bearer_token_env_var` 或 `env_http_headers` 指向的环境变量，不要把 token 直接写入 TOML 或 Python 源码。运行时明确拒绝 `bearer_token` 明文字段。

## 控制工具范围与审批

```toml
[mcp_servers.search]
url = "https://mcp.example.com/mcp"
enabled_tools = ["search", "fetch"]
disabled_tools = ["delete"]
default_tools_approval_mode = "prompt"

[mcp_servers.search.tools.search]
approval_mode = "auto"

[mcp_servers.search.tools.fetch]
approval_mode = "approve"
```

`enabled_tools` 是 allow-list，设置后只注册列出的工具；`disabled_tools` 在 allow-list 之后继续移除工具。工具级 `approval_mode` 会覆盖 server 默认值。

当前源码接受的审批值为 `auto`、`prompt`、`writes`、`approve`。

## 从 Python 启动

使用默认 `CODEX_HOME`：

```python
from openai_codex import Codex, CodexConfig, Sandbox

config = CodexConfig(codex_bin="/absolute/path/to/brainary-codex")

with Codex(config=config) as codex:
    agent = codex.thread_start(sandbox=Sandbox.read_only)
    result = agent.run("使用 search MCP 工具查找项目的发布记录并总结。")
    print(result.final_response)
```

为应用使用独立配置目录：

```python
config = CodexConfig(
    codex_bin="/absolute/path/to/brainary-codex",
    env={
        "CODEX_HOME": "/absolute/path/to/app-codex-home",
        "SEARCH_MCP_TOKEN": "...",
    },
)
```

`CODEX_HOME` 指向的目录必须已经存在，并在其中保存 `config.toml`。

只做一次临时配置时，可以通过启动参数覆盖：

```python
config = CodexConfig(
    codex_bin="/absolute/path/to/brainary-codex",
    config_overrides=(
        'mcp_servers.echo={command="python",args=["/absolute/path/to/echo_server.py"],required=true}',
    ),
)
```

复杂配置更适合写入 TOML；`config_overrides` 会逐项转换成 `codex --config <value>` 启动参数。

## 观察 MCP 调用

阻塞模式下，完成的 MCP 调用会包含在 `TurnResult.items` 中：

```python
result = agent.run("调用 search 工具查询 Brainary。")

for item in result.items:
    root = item.root if hasattr(item, "root") else item
    if root.type == "mcpToolCall":
        print(root.server, root.tool, root.status.value)
        print(root.result)
        print(root.error)
```

流式模式还会收到 `item/mcpToolCall/progress`：

```python
handle = agent.turn("调用 search 工具完成查询。")

for event in handle.stream():
    if event.method == "item/mcpToolCall/progress":
        print(event.payload.message)
    elif event.method == "turn/completed":
        print(event.payload.turn.status.value)
```

不要只等待某个 MCP progress 事件来判断 turn 完成；终止条件始终是匹配当前 turn ID 的 `turn/completed`。

## MCP 与 PoA 的区别

普通 Agent 使用的 MCP server 来自运行时配置。PoA 还可以在 `.poa` 的 `manifest.toml` 中声明随包启动的 MCP server；这类 server 属于该 PoA 包的能力，具体打包方式见[打包与分发](../../08-packaging.md)。
