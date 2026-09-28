# Brainary CLI 使用说明

本文说明当前 Brainary 项目的公开 CLI 用法，包括全局参数、子命令、自动化调用和会话管理。实际可用参数以本机 `brainary --help` 和子命令的 `--help` 为准。

适用版本：`feat/brainary-rebrand`，核对提交 `644e1aab6`（2026-09-28）。交互界面用法见 [Brainary TUI 使用说明](Brainary-TUI使用说明.md)。

![Brainary CLI 帮助界面](Brainary-使用说明图片/CLI帮助.png)

图 1：`brainary help` 的公开命令列表，包含 `poa` 入口。

## 快速开始

```bash
# 查看帮助和版本
brainary --help
brainary --version

# 指定项目目录并启动 Brainary
brainary -C /path/to/project

# 非交互执行一次任务
brainary exec "运行测试并总结失败原因"

# 运行一个已有的 PoA 包
brainary poa /path/to/package.poa
```

## 命令写法

命令格式中的符号含义如下：

| 写法 | 含义 |
| --- | --- |
| `<value>` | 必填参数，例如 `<SESSION>` |
| `[value]` | 可选参数，例如 `[PROMPT]` |
| `...` | 可以提供多个值 |
| `-x` / `--xxx` | 短参数与长参数，两种写法作用相同 |
| `brainary <subcommand> --help` | 查看该子命令的完整参数 |

不带子命令运行 `brainary` 时，会进入 TUI。带 `exec` 等子命令时，会执行对应的 CLI 功能。

## 全局参数

以下参数用于启动 TUI，也有一部分可以用于 `resume`、`fork` 等子命令。

| 参数 | 类型或取值 | 说明 |
| --- | --- | --- |
| `[PROMPT]` | 字符串 | 启动会话时直接提交的任务描述 |
| `-C, --cd` | 目录路径 | 将指定目录作为 Brainary 的工作目录 |
| `--add-dir` | 目录路径，可重复 | 额外允许写入的目录 |
| `-m, --model` | 模型名称 | 指定本次会话使用的模型 |
| `-i, --image` | 文件路径，可重复 | 将一个或多个图片附加到初始任务 |
| `-c, --config` | `key=value`，可重复 | 临时覆盖配置项；嵌套项使用点路径，值按 TOML 解析 |
| `--strict-config` | boolean flag | 配置中出现当前版本无法识别的字段时直接报错 |
| `-s, --sandbox` | `read-only`、`workspace-write`、`danger-full-access` | 设置模型生成命令的沙箱级别 |
| `-a, --ask-for-approval` | `untrusted`、`on-request`、`never` | 设置命令执行的审批策略 |
| `--approve-for-me` | boolean flag | 使用 `workspace-write` 沙箱，并把审批请求交给自动审查；适合希望减少人工确认但仍保留沙箱的任务 |
| `--search` | boolean flag | 启用实时网页搜索能力 |
| `--oss` | boolean flag | 使用开源模型提供方 |
| `--local-provider` | `lmstudio`、`ollama` | 指定本地模型提供方，通常与 `--oss` 一起使用 |
| `--remote` | `ws://`、`wss://`、`unix://` 地址 | 连接远程 app-server |
| `--remote-auth-token-env` | 环境变量名 | 从指定环境变量读取远程连接的 Bearer Token |
| `--no-alt-screen` | boolean flag | 使用内联模式运行 TUI，保留终端滚动历史 |
| `-h, --help` | boolean flag | 显示帮助 |
| `-V, --version` | boolean flag | 显示版本 |

## 命令总览

### 默认显示的命令

| 命令 | 说明 |
| --- | --- |
| `brainary` | 启动交互式 TUI |
| `brainary exec` / `brainary e` | 非交互执行任务，适合脚本和 CI |
| `brainary poa <PATH>` | 非交互运行 PoA 包目录或 `.poa` 文件 |
| `brainary resume` | 恢复已有会话 |
| `brainary fork` | 从已有会话创建分支会话 |
| `brainary archive` | 归档指定会话 |
| `brainary unarchive` | 恢复已归档会话 |
| `brainary delete` | 永久删除指定会话 |
| `brainary completion` | 生成 Shell 自动补全脚本 |
| `brainary doctor` | 检查安装、配置、认证、运行环境、Git、终端、MCP 和会话状态 |
| `brainary sandbox` | 在 Brainary 提供的沙箱中执行命令 |
| `brainary help [COMMAND]` | 查看顶层或指定命令的帮助 |

### Brainary 项目扩展能力

PoA（Program of Agent）把程序逻辑、`manifest.toml`、资源及可选的 MCP 服务声明封装成一个可执行包。当前提供三种入口：

| 场景 | 入口 |
| --- | --- |
| 从 Shell 或脚本运行 | `brainary poa <PATH>` |
| 沿用已有的 exec 调用 | `brainary exec --poa <PATH>` |
| 在交互界面中运行 | `/poa <PATH>` |

PoA 中的控制程序直接执行；如果程序调用 Agent，相关调用仍然会产生模型请求。

## 命令详情

### 启动 Brainary

```bash
brainary [全局参数] [PROMPT]
```

示例：

```bash
# 在当前项目启动
brainary

# 指定目录和模型
brainary -C /path/to/project -m <model-name>

# 附带图片说明问题
brainary -i screenshot.png "分析这个界面问题"

# 启用网页搜索
brainary --search "调研这个依赖的最新版本"
```

### 非交互执行：`exec`

```bash
brainary exec [OPTIONS] [PROMPT]
brainary exec resume [OPTIONS]
```

如果不在参数中提供任务，或将任务写成 `-`，Brainary 会从标准输入读取内容。若同时提供任务参数和管道输入，管道内容会作为 `<stdin>` 块追加。

```bash
# 一次性执行
brainary exec "检查格式并报告问题"

# 从标准输入传入任务
printf '%s\n' "总结当前修改" | brainary exec -

# 输出 JSONL 事件，供程序处理
brainary exec --json "运行测试"

# 将最后一条模型消息写入文件
brainary exec -o result.txt "生成发布说明"

# 不保存本次会话文件
brainary exec --ephemeral "解释这个函数"
```

常用的 `exec` 专用参数：

| 参数 | 类型或取值 | 说明 |
| --- | --- | --- |
| `--skip-git-repo-check` | boolean flag | 允许在非 Git 仓库目录中运行 |
| `--ephemeral` | boolean flag | 不在磁盘上保存本次会话 |
| `--ignore-user-config` | boolean flag | 不读取用户配置；认证仍使用 `CODEX_HOME` |
| `--ignore-rules` | boolean flag | 不读取用户或项目的 execpolicy `.rules` 文件 |
| `--output-schema` | 文件路径 | 用 JSON Schema 约束最终回复结构 |
| `--json` | boolean flag | 以 JSONL 输出执行事件 |
| `-o, --output-last-message` | 文件路径 | 将最后一条模型消息写入文件 |
| `--color` | `always`、`never`、`auto` | 设置标准输出的颜色模式 |
| `--poa` | 目录路径或 `.poa` 文件 | 运行 PoA 包；这是当前 Brainary 项目的扩展能力 |

### 运行 PoA：`poa`

```bash
brainary poa [OPTIONS] <PATH>
```

`<PATH>` 可以是 PoA 包目录或已打包的 `.poa` 文件。CLI 先加载、校验并解包，再交给本地执行后端运行；后端负责启动清单声明的 MCP 服务并执行程序。

```bash
# 运行包目录
brainary poa /path/to/package

# 运行已经打包好的文件
brainary poa ./workflow.poa

# 以 JSONL 输出事件，供脚本处理
brainary poa --json ./workflow.poa

# 旧写法仍可用，使用同一套执行逻辑
brainary exec --poa ./workflow.poa
```

`poa` 支持 `--json`、`--ephemeral`、`--skip-git-repo-check`、`--strict-config`，以及帮助中列出的模型、配置和沙箱参数。完整列表用 `brainary poa --help` 查看，不要直接套用全部 `exec` 参数。

使用条件和限制：

- 包目录根部必须包含有效的 `manifest.toml`；`.poa` 文件是相同目录结构的打包形式。
- 本机需要可用的 code-mode 运行环境；如果报缺少 `codex-code-mode-host`，需要补齐这个配套程序。当前不支持通过 `--remote` 执行 PoA。
- `brainary poa` 不接受普通任务描述或图片参数；使用 `exec --poa` 时，也不能与普通 `[PROMPT]`、`exec resume` 或 `exec review` 组合。
- CLI 中的相对包路径按启动命令时的 Shell 目录解析；`-C` 指定的是任务工作目录。两者不同时，建议使用 absolute path，避免加载错包。
- 路径包含空格时，按 Shell 的规则加引号，例如 `brainary poa "/path/to/my package"`。TUI 的 `/poa` 路径规则不同。
- 一次 PoA 调用执行一个程序单元，完成后退出 CLI；程序内部仍可调用 Agent。

预期结果：Brainary 打印 PoA 的执行输出；包路径、清单或包格式无效时，在启动执行前报错。

### 会话管理

| 参数或子命令 | 类型或取值 | 说明 |
| --- | --- | --- |
| `brainary resume` | 命令 | 打开会话选择器 |
| `brainary resume --last` | boolean flag | 直接恢复当前目录中最近一次会话 |
| `brainary resume <SESSION>` | 会话 UUID 或名称 | 恢复指定会话；UUID 优先于名称匹配 |
| `brainary resume --all` | boolean flag | 选择最近会话时不按当前目录过滤 |
| `brainary fork` | 命令 | 打开会话选择器并创建分支会话 |
| `brainary fork --last` | boolean flag | 从当前目录中最近一次会话创建分支会话 |
| `brainary archive <SESSION>` | 会话 UUID 或名称 | 归档会话但保留记录 |
| `brainary unarchive <SESSION>` | 会话 UUID 或名称 | 恢复已归档会话 |
| `brainary delete <SESSION>` | 会话 UUID 或名称 | 永久删除会话；名称仍需要交互确认 |
| `brainary delete <UUID> --force` | 会话 UUID | 跳过确认并永久删除指定会话 |

> `brainary delete` 是不可恢复操作。`--force` 会跳过确认，而且此时会话参数必须是 UUID。

### 环境诊断：`doctor`

```bash
brainary doctor
brainary doctor --summary
brainary doctor --json
```

| 参数 | 类型或取值 | 说明 |
| --- | --- | --- |
| `--summary` | boolean flag | 只显示分组检查结果和最终统计 |
| `--json` | boolean flag | 输出已脱敏的机器可读报告 |
| `--all` | boolean flag | 展开详细输出中的长列表 |
| `--no-color` | boolean flag | 关闭 ANSI 颜色 |
| `--ascii` | boolean flag | 使用 ASCII 状态标记和分隔符 |

`doctor` 用于安装异常、配置加载失败、认证失效、终端显示问题或服务连接问题。报告会检查安装来源与可执行文件、配置、已有认证状态、运行时、Git、终端、状态目录、MCP、沙箱、网络、app-server 和会话索引等项目。

```bash
# 日常排障：查看完整的人类可读报告
brainary doctor

# 开会或提 issue 前：生成已脱敏的机器可读报告
brainary doctor --json > brainary-doctor.json
```

预期结果：每项检查显示正常、警告或失败，并给出汇总；`--json` 适合交给排障工具处理，但分享前仍应人工检查内容。

### Shell 自动补全

```bash
brainary completion <SHELL>
```

支持 `bash`、`elvish`、`fish`、`powershell` 和 `zsh`。例如，为 zsh 生成补全脚本：

```bash
brainary completion zsh > _brainary
```

具体安装位置取决于本机 Shell 的补全目录设置。

### 临时功能开关

只对本次运行启用或禁用功能：

```bash
brainary --enable <FEATURE>
brainary --disable <FEATURE>
```

## 常见操作流程

### 在自动化中运行

```bash
brainary exec --json -C /path/to/project \
  "运行项目测试，只报告失败项和可能原因" > events.jsonl
```

自动化环境中仍建议保留沙箱。只有外部环境已经完成可靠隔离时，才考虑绕过审批和沙箱。

### 恢复或整理会话

```bash
# 恢复最近一次会话
brainary resume --last

# 查看所有目录中的会话
brainary resume --all

# 归档不再活跃的会话
brainary archive <会话名称或UUID>

# 恢复归档会话
brainary unarchive <会话名称或UUID>
```

## 参数组合与安全提示

- 日常本地开发优先使用 `--sandbox workspace-write --ask-for-approval on-request`，让 Brainary 可以修改工作区，但在需要额外权限时暂停确认。
- 需要访问工作区以外的目录时，优先使用 `--add-dir <PATH>` 明确增加目录，不要直接切换到 `danger-full-access`。
- `--dangerously-bypass-approvals-and-sandbox` 会同时关闭审批和沙箱，只能用于外部已经提供可靠隔离的环境。
- `--dangerously-bypass-hook-trust` 会跳过 Hook 的持久化信任检查，只适用于已经独立验证 Hook 来源的自动化环境。
- `-a never` 表示执行过程中不再请求人工确认；它不等同于关闭沙箱，实际访问范围仍由 `--sandbox` 决定。
- CI 中可组合 `--json` 与 `--output-last-message <FILE>`，分别保存过程事件和最终自然语言结果。

## 配置与兼容路径

当前版本仍沿用上游兼容配置路径和环境变量：

| 标识 | 用途 |
| --- | --- |
| `~/.codex/config.toml` | 默认用户配置文件 |
| `CODEX_HOME` | 覆盖配置、认证及会话数据目录 |
| `$CODEX_HOME/<name>.config.toml` | `--profile <name>` 对应的叠加配置文件 |

这些名称属于存储和配置兼容标识。保留它们可以继续读取已有配置与会话，不代表用户启动命令仍是 `codex`；当前 CLI 入口是 `brainary`。

当前版本不提供顶层 `brainary login`、`brainary logout` 或 `brainary update`。如果后续接入新的账户或更新体系，应以对应版本的 `brainary --help` 为准。

## 获取帮助

```bash
brainary --help
brainary poa --help
brainary exec --help
brainary doctor --help
```
