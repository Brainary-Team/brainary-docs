# 使用 Python SDK 运行 PoA

Brainary Python SDK 把 `.poa` 包和业务输入提交到开启了 code mode 的线程，等待 PoA
（Program of Agent）程序执行完成。

本文介绍由 Python 应用直接运行 PoA。需要让模型决定何时调用时，请阅读
[让 Agent 使用 PoA](agent-poa.md)。输入定义、TypeScript 小示例和完整字段列表见
[PoA input 编写说明](../../PoA-Guide/09-input.md)。

## 前置条件

- Python 3.10 或更高版本。
- 支持 PoA input 和 `thread/codeMode/exec` 的 Brainary Codex 二进制；`codex` 与
  `codex-code-mode-host` 必须放在同一目录。
- 已按[快速开始](../../PoA-Guide/02-quickstart.md)配置模型、认证信息与
  `CODEX_HOME/config.toml`。
- 一个已打包的 `.poa` 文件。SDK 不接受包目录，也不编译 `input_schema.ts`；请按
  [PoA input 编写说明](../../PoA-Guide/09-input.md)准备包含 JSON Schema 的包。

## 安装 SDK

分发包名为 `brainary-sdk`，Python 导入名为 `brainary`。包发布到所使用的包索引后安装：

```bash
python -m pip install brainary-sdk
```

包尚未发布时，从项目的 [GitHub Releases](https://github.com/Brainary-Team/brainary-codex/releases)
取得可用的 SDK 和运行时 wheel。运行时必须支持本文使用的 PoA input 接口。

## 运行带 input 的程序

下面运行 [PoA input 示例](../../PoA-Guide/09-input.md#1-示例城市计划)中的 `city_plan.poa`。
包内 schema 定义必填的 `city` 和可选的整数 `days`，程序通过 `POA_INPUT` 读取它们。

新建 `run_poa.py`：

```python
from pathlib import Path

from brainary import Brainary, BrainaryConfig, Poa, PoaPackage

# codex 和 codex-code-mode-host 必须保存在同一目录。
BRAINARY_BIN = Path("/path/to/codex-poa/codex")
POA_FILE = Path("/path/to/city_plan.poa")
WORKSPACE = Path("/path/to/workspace")

package = PoaPackage.load(POA_FILE)
poa_input = {"city": "Singapore", "days": 3}

with Brainary(config=BrainaryConfig(brainary_bin=str(BRAINARY_BIN))) as brainary:
    thread = brainary.thread_start(
        cwd=str(WORKSPACE),
        config={"features": {"code_mode": {"enabled": True}}},
    )
    poa = Poa(thread, package)
    result = poa.run(input=poa_input)
    print("thread_id:", thread.id)
    print("output:", result)
```

执行：

```bash
python run_poa.py
```

`Poa` 对象把包与线程绑定在一起。`run(input=...)` 返回 `dict[str, object]`，即
`thread/codeMode/exec` 响应中 `output` 字段的值，是 PoA 通过 `text()`、`image()` 或
`audio()` 产生的 cell 输出 envelope；它不是 `TurnResult`，也没有 `final_response`。

只需临时运行一次时，可以在同一个已开启 code mode 的线程上使用简写：

```python
result = thread.run_poa(
    "/path/to/city_plan.poa",
    input={"city": "Singapore", "days": 3},
)
```

## 输入与校验

`input` 是包含 JSON 可序列化值的 Python `dict`，直接传入对象即可，无需调用
`json.dumps()`。它与普通 Agent 的文本或图片 turn 输入不同。

- 包内 `[poa].input_schema` 指向 JSON Schema；运行时在启动程序和包内 MCP server
  之前校验输入。字段缺失、类型错误或不满足约束时，请求失败，程序不会执行。
- 省略 `input` 或传入 `None` 时按 `{}` 校验，不会自动补齐必填字段。
- `days` 可以省略；示例中的默认值 `1` 由 JavaScript 解构设置，不由 schema 注入。
- 没有声明 input schema 的包只能省略 input 或传入空对象 `{}`。

## 注意事项

- `.poa` 必须是 zip 格式，且根目录包含 `manifest.toml`；大小不能超过 64 MiB。
- 必须在创建线程时启用 code mode。
- 默认绑定的运行时版本可能尚未支持这些接口；通过
  `BrainaryConfig(brainary_bin=...)` 明确指定兼容的 Brainary Codex 二进制。
- 启动 PoA 本身不会触发模型采样，程序调用子 agent 等工具时可能会调用模型。
- `Poa(thread, package).run(input=poa_input)` 与
  `thread.run_poa(package, input=poa_input)` 等价。
- 异步应用使用 `AsyncBrainary` 和 `AsyncPoa`，对应调用为
  `await poa.run(input=poa_input)` 或 `await thread.run_poa(package, input=poa_input)`。
