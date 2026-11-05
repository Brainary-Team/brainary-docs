# 快速开始

本篇从 PoA 模板创建项目，写一个派出 agent 的程序，打包后运行。

## 前置条件

- 已安装 Brainary，`brainary` 命令可以执行。
- Node.js 22 或更高版本，pnpm 12。
- Git。
- 一个模型服务的模型 ID、服务地址和 API Key。

## 1. 配置模型

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
```

在运行 Brainary 的终端设置 API Key：

```bash
export VENDOR_API_KEY="替换为你的 API Key"
```

各字段的含义，以及配置多个 provider、默认沙箱等，见 [配置文件](02-configuration.md)。

## 2. 创建项目

PoA 项目从 [PoA 模板](https://github.com/Brainary-Team/brainary-poa-template) 创建。模板已依赖 PoA SDK（`@brainary/poa-sdk`），并提供构建和打包命令：

```bash
git clone https://github.com/Brainary-Team/brainary-poa-template.git my-poa
cd my-poa
pnpm install
```

在其他项目中使用 SDK 时，从 npm 安装：

```bash
npm install @brainary/poa-sdk
```

## 3. 编写程序

把 `src/main.ts` 替换为：

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

## 4. 填写 manifest

项目根目录的 `manifest.toml` 声明包的信息。按自己的项目修改 `namespace`、`name`、`version` 和 `description`：

```toml
[poa]
namespace = "demo"
name = "hello_agent"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
description = "派出一个 agent，介绍当前目录"
```

`runtime`、`poa_api_version` 和 `entry` 保持模板的默认值，打包时会校验这三项。`entry` 是包内的入口文件，固定为 `main.js`，由 `pnpm package` 从 `src/main.ts` 构建。

项目中与 PoA 相关的文件如下：

```text
my-poa/
├── src/main.ts        程序入口
├── input_schema.ts    业务输入，见 输入与输出
├── mcp/               随包携带的 MCP server，见 配置文件
├── manifest.toml      包信息
└── package.json
```

全部字段见 [manifest.toml](02-configuration.md#manifesttoml)。

## 5. 打包

在项目根目录执行：

```bash
pnpm package
```

打包时，`src/main.ts` 和它引用的模块（包括 SDK）构建成一个 JavaScript 文件，和 `manifest.toml`、由 `input_schema.ts` 编译出的输入 schema 以及 `mcp/` 目录一起写入 `dist/my-poa.poa`。

## 6. 运行

```bash
brainary poa dist/my-poa.poa
```

在 Git 仓库外运行时需要加 `--skip-git-repo-check`。运行结束后，终端输出 agent 的回答：

```text
Script completed
Wall time 8.4 seconds
Output:

当前目录包含 README、配置文件、TypeScript 源码、构建脚本、测试文件、类型声明、锁文件及 `assets/poa-tui-demo.png` 图片。
```

## 下一步

- [配置文件](02-configuration.md)：配置多个 provider、默认沙箱和 agent 层级。
- [输入与输出](03-input-output.md)：接收业务输入，在 TUI 或 Python SDK 中运行。
- [示例](04-examples.md)：完整的多 agent 程序。
- [使用模板开发 PoA](10-template-workflow.md)：模板的目录结构、检查和测试命令，以及打包流程。
