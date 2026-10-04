# 使用 MCP

MCP 为 Agent 提供外部工具。Python SDK 不直接连接 MCP server；它启动 Brainary Codex app-server，运行时读取 MCP 配置、启动或连接 server，并把可用工具暴露给 Agent。

## 配置 stdio server

通过 `thread_start(config=...)` 的 `mcp_servers` 字典配置 server。下面的脚本应是
已实现 MCP stdio 协议的 server，运行时会启动它并发现工具：

```python
from brainary import Brainary, BrainaryConfig, Sandbox

config = BrainaryConfig(brainary_bin="/absolute/path/to/brainary-codex")

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(
        sandbox=Sandbox.read_only,
        config={
            "mcp_servers": {
                "files": {
                    "command": "python",
                    "args": ["/absolute/path/to/files_server.py"],
                    "cwd": "/absolute/path/to/server-workdir",
                    "env": {"LOG_LEVEL": "info"},
                    "enabled": True,
                    "required": True,
                    "startup_timeout_sec": 10.0,
                    "tool_timeout_sec": 60.0,
                    "supports_parallel_tool_calls": False,
                }
            }
        },
    )
    result = agent.run("使用 files MCP 工具列出项目文件并总结。")
    print(result.final_response)
```

`files` 是 server ID。配置键使用 snake_case；`command` 决定使用 stdio transport。
`env` 设置 server 子进程环境，`env_vars` 可传递运行时环境中的变量；不要同时配置
`url`、HTTP header 或 bearer token 字段。

`enabled` 默认是 `True`；`required=True` 表示初始化失败时应报告错误，不能忽略该
server。`startup_timeout_sec` 控制初始化与首次工具发现的超时，`tool_timeout_sec`
控制工具调用超时，单位都是秒。仅当 server 的所有工具都可以安全并发时，才将
`supports_parallel_tool_calls` 设为 `True`。

## 配置 Streamable HTTP server

使用 `url` 配置 HTTP transport。认证变量通过运行时进程环境提供：

```python
import os

from brainary import Brainary, BrainaryConfig

config = BrainaryConfig(
    brainary_bin="/absolute/path/to/brainary-codex",
    env={
        "SEARCH_MCP_TOKEN": os.environ["SEARCH_MCP_TOKEN"],
        "TENANT_TOKEN": os.environ["TENANT_TOKEN"],
    },
)

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(
        config={
            "mcp_servers": {
                "search": {
                    "url": "https://mcp.example.com/mcp",
                    "bearer_token_env_var": "SEARCH_MCP_TOKEN",
                    "http_headers": {"X-Client": "brainary"},
                    "env_http_headers": {"X-Tenant-Token": "TENANT_TOKEN"},
                    "required": True,
                    "startup_timeout_sec": 10.0,
                    "tool_timeout_sec": 60.0,
                }
            }
        },
    )
    result = agent.run("使用 search MCP 工具查找项目的发布记录并总结。")
    print(result.final_response)
```

`bearer_token_env_var` 的值是环境变量名，运行时发送 `Authorization: Bearer <token>`。
`env_http_headers` 将 header 名映射到环境变量名，`http_headers` 放固定的非敏感值。
运行时拒绝 `bearer_token` 明文字段。HTTP server 不接受 stdio 的 `args`、`env`、
`env_vars` 和 `cwd` 字段。`BrainaryConfig.env` 会合并父进程环境，已导出上述变量时
可以省略 `env`。

## 控制工具范围与审批

在 server 字典中添加工具过滤和审批配置，并在已打开的 `Brainary` 客户端中创建 thread：

```python
search_server = {
    "url": "https://mcp.example.com/mcp",
    "bearer_token_env_var": "SEARCH_MCP_TOKEN",
    "enabled_tools": ["search", "fetch"],
    "disabled_tools": ["delete"],
    "default_tools_approval_mode": "prompt",
    "tools": {
        "search": {"approval_mode": "auto"},
        "fetch": {"approval_mode": "approve"},
    },
}

agent = brainary.thread_start(
    config={"mcp_servers": {"search": search_server}},
)
```

`enabled_tools` 设置后只注册列出的工具；`disabled_tools` 在此基础上继续移除工具。
列表和 `tools` 字典使用 server 返回的原始工具名。工具级 `approval_mode` 覆盖
server 默认值，当前接受 `auto`、`prompt`、`writes`、`approve`；最终审批还受
Agent 的审批策略约束。

这些配置覆盖作用于创建的 thread，不会写入配置文件。多个 Agent 可复用同一配置
字典；跨进程恢复或分叉 thread 时，应再次传入所需配置。配置文件的用法见
[配置文件使用说明](configuration.md)。

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

普通 Agent 使用的 MCP server 来自运行时配置。PoA 还可以在 `.poa` 的 `manifest.toml` 中声明随包启动的 MCP server；这类 server 属于该 PoA 包的能力，具体打包方式见[打包与分发](../../PoA-Guide/08-packaging.md)。
