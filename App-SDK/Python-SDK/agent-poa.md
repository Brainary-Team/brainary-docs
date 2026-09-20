# 让 Agent 使用 PoA

Python SDK 可以在创建 Agent 时，通过 `dev_poas` 把 `.poa` 包注册为模型可调用的工具。
模型会根据工具描述和用户任务决定是否调用它。

需要由 Python 应用直接运行 PoA，而不是交给模型决定时，请阅读
[使用 Python SDK 运行 PoA](run-poa.md)。

## 前置条件

- Python 3.10 或更高版本。
- 支持 PoA 和 code mode 的 Brainary Codex 二进制；`codex` 与
  `codex-code-mode-host` 必须位于同一目录。
- 使用 `responses` 协议的模型 provider。当前 `anthropic` 协议不会向模型发送 PoA
  使用的 namespace tools。
- 一个已打包的 `.poa` 文件。SDK 不接受未打包目录；打包方式见
  [打包与分发](../../PoA-Guide/08-packaging.md)。

## 注册并使用 PoA

```python
from openai_codex import Codex, CodexConfig, PoaPackage

package = PoaPackage.load("/srv/poas/review.poa")

with Codex(config=CodexConfig(codex_bin="/opt/brainary/codex")) as codex:
    agent = codex.thread_start(
        cwd="/srv/workspaces/project",
        dev_poas=[package],
        config={"features": {"code_mode": {"enabled": True}}},
    )
    result = agent.run("使用 review 工具检查当前改动。")
    print(result.final_response)
```

也可以直接传入 `.poa` 路径：

```python
agent = codex.thread_start(
    dev_poas=["/srv/poas/review.poa"],
    config={"features": {"code_mode": {"enabled": True}}},
)
```

如果包中的 `[poa].name` 是 `review`，模型看到的工具名就是 `poa.review`；
`[poa].description` 会成为工具说明。PoA 工具当前不接收业务参数，模型调用时传入空对象
`{}`。

可以在 `dev_poas` 中注册多个包。不同包使用相同 `name` 时，运行时会自动生成包含
namespace 和 version 的限定名；为保持工具名稳定，建议让同一 Agent 使用的包名称互不
重复。

## 注意事项

- `PoaPackage.load()` 会检查文件是否为 zip、根目录是否包含 `manifest.toml`，以及包是否
  超过 64 MiB；其余 manifest 和入口规则由运行时校验。
- 包内声明的 stdio MCP server 会在 PoA 被调用时启动；能力声明方式见
  [打包与分发](../../PoA-Guide/08-packaging.md)。
- 是否调用 PoA 由模型决定。必须执行的步骤应改用 `thread.run_poa(...)`。
- `dev_poas` 只存在于 `thread_start()`；`thread_resume()` 和 `thread_fork()` 不接受该参数。
- `.poa` 是可执行程序，只加载可信来源的包，并使用合适的 `sandbox` 和
  `approval_mode`。
