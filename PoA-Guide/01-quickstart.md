# 快速开始

本篇安装 Brainary 和 PoA SDK，写一个派出 agent 的程序，打包后运行。

## 前置条件

- Node.js 22 或更高版本。
- 一个模型服务的模型 ID、服务地址和 API Key。

## 1. 安装 Brainary

从 [Brainary Releases](https://github.com/Brainary-Team/brainary-codex/releases) 下载 `brainary-poa-<平台>.tar.gz`，解压到一个目录并加入 `PATH`：

```bash
mkdir -p ~/brainary
tar -xzf brainary-poa-<平台>.tar.gz -C ~/brainary
export PATH="$HOME/brainary:$PATH"
brainary --version
```

压缩包里的 `brainary` 和 `codex-code-mode-host` 要放在同一目录。

## 2. 配置模型

把下面的内容写入 `~/.brainary/config.toml`，替换 `model`、`base_url` 和 `wire_api`：

```toml
model = "替换为模型ID"
model_provider = "vendor"
suppress_unstable_features_warning = true

[model_providers.vendor]
name = "我的模型服务"
base_url = "https://api.example.com/v1"
env_key = "VENDOR_API_KEY"
wire_api = "responses"
requires_openai_auth = false
```

在运行 Brainary 的终端设置 API Key：

```bash
export VENDOR_API_KEY="替换为你的 API Key"
```

各字段的含义，以及配置多个 provider、默认沙箱等，见 [配置文件](02-configuration.md)。

## 3. 创建项目

从 [PoA SDK Releases](https://github.com/Brainary-Team/brainary-poa-sdk/releases) 下载 `brainary-poa-sdk-<版本>.tgz`，然后创建项目并安装 SDK：

```bash
mkdir my-poa && cd my-poa
npm init -y
npm install /path/to/brainary-poa-sdk-<版本>.tgz
```

## 4. 编写程序

新建 `src/main.ts`：

```ts
import { AgentExecutor, text } from "@brainary/poa-sdk";

const agents = new AgentExecutor();

// 派出一个 agent，最多等待 5 分钟
const handle = await agents.spawn_agent({
  message: "用一句话说明当前目录里有哪些文件。",
});
const reply = await agents.acquire_agent_result(handle.agent_id, 300_000);

text(reply.result);
```

- `spawn_agent` 派出一个 agent，立即返回它的句柄，它的配置来源于配置文件中，默认模型的配置。
- `acquire_agent_result` 等待 agent 完成，返回它的最终回复。
- `text` 把内容输出给运行方。

## 5. 填写 manifest

在项目根目录新建 `manifest.toml`，声明包的名称、版本和入口文件：

```toml
[poa]
namespace = "demo"
name = "hello_agent"
version = "0.1.0"
description = "派出一个 agent，介绍当前目录"
runtime = "codex-v8"
poa_api_version = 1
entry = "src/main.ts"
```

现在项目目录如下：

```text
my-poa/
├── src/main.ts      程序入口
├── manifest.toml    包信息
├── package.json
└── node_modules/
```

全部字段见 [manifest.toml](02-configuration.md#manifesttoml)。

## 6. 打包

在项目根目录执行：

```bash
brainary poa pack
```

打包时，`entry` 和它引用的模块（包括 SDK）合成一个 JavaScript 文件，和 `manifest.toml` 一起写入 `hello_agent.poa`。用 `-o <文件>` 可以指定输出路径。

## 7. 运行

```bash
brainary poa hello_agent.poa --skip-git-repo-check
```

在 Git 仓库外运行时需要加 `--skip-git-repo-check`。运行结束后，终端输出 agent 的回答：

```text
Script completed
Wall time 132.3 seconds
Output:

当前目录包含 manifest.toml、package.json、package-lock.json 和 src 目录。
```

## 下一步

- [配置文件](02-configuration.md)：配置多个 provider、默认沙箱和 agent 层级。
- [输入与输出](03-input-output.md)：接收业务输入，在 TUI 或 Python SDK 中运行。
- [示例](04-examples.md)：完整的多 agent 程序。
