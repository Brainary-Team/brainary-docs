# PoA 项目开发流程：从模板创建到打包运行

## 1. 整体流程

获取模板并安装依赖 → 编写代码 → 检查与测试 → 构建打包，生成 `.poa` → 使用 Brainary CLI 或 TUI 运行并验证结果。

`git clone` 获取初始项目；`pnpm build` 编译已有源码；`pnpm package` 构建后生成运行包。
当前模板通过 clone 使用，尚未提供 `pnpm create poa` 初始化命令。

## 2. 准备环境并获取模板

开发环境需要 Git、Node.js 22 或更新版本、pnpm 12。模板指定的 pnpm 版本为 `12.14.0`。
先检查本地环境：

```bash
git --version
node --version
pnpm --version
```

缺少工具时，按 Node.js 和 pnpm 的安装说明准备环境。

在希望存放项目的父目录执行：

```bash
git clone https://github.com/Brainary-Team/brainary-poa-template.git my-poa
cd my-poa
pnpm install
```

`my-poa` 是本地目录名，可以自行替换。后续 pnpm 命令都在这个项目根目录中执行。
`pnpm install` 根据 `package.json` 和 `pnpm-lock.yaml` 安装开发及打包依赖，生成本地
`node_modules/`，不会运行 PoA。

构建和打包不需要 Brainary 二进制。到运行阶段，再按快速开始
准备 Brainary 及配套运行程序，保持发布包要求的目录布局。以下运行命令假设 `brainary`
已经加入 PATH；未加入时，将命令中的 `brainary` 替换为可执行文件的完整路径。

## 3. 目录和文件职责

```text
my-poa/
├── manifest.toml             # PoA 身份、版本、Runtime 和包内入口
├── input_schema.json         # 开发者手写的输入 JSON Schema
├── src/
│   └── main.ts               # 默认业务入口
├── agents/                   # Agent 实现的存放位置
├── prompts/                  # 提示词资源的存放位置
├── memory/                   # 记忆和状态适配的预留位置
├── tools/                    # 工具调用适配代码的存放位置
├── mcp/                      # 可选：随包分发的 stdio MCP 服务文件
├── tests/
│   ├── agent.test.ts         # 默认示例的业务逻辑测试
│   └── runtime.test.ts       # Runtime 入口和打包结果测试
├── types/
│   └── runtime.d.ts          # 开发期使用的 Runtime 全局能力类型声明
├── scripts/
│   └── package.mjs           # .poa 打包脚本
├── dist/                     # 构建和打包后生成，不提交源码仓库
├── package.json              # 项目清单、依赖和命令
├── pnpm-lock.yaml            # 依赖锁文件
├── tsconfig.json             # TypeScript 检查配置
├── biome.json                # Biome 检查和格式化配置
├── rolldown.config.mjs       # Rolldown 构建配置
├── vitest.config.ts          # Vitest 测试配置
├── README.md
└── .gitignore
```

## 4. 编写程序和输入规则

### 4.1 修改 src/main.ts

模板默认示例接受一个可选的 `message`，返回传入消息或默认问候语：

```ts
export interface PoaInput {
  message?: string;
}

export interface PoaOutput {
  ok: true;
  message: string;
}

export function run(input: PoaInput = {}): PoaOutput {
  return {
    ok: true,
    message: input.message ?? "Hello from Brainary PoA.",
  };
}
```

入口末尾还会读取 Runtime 的输入并输出结果：

```ts
type RuntimeGlobals = typeof globalThis & {
  POA_INPUT?: PoaInput;
  text?: (value: unknown) => void;
};

const runtime = globalThis as RuntimeGlobals;

if (typeof runtime.text === "function") {
  runtime.text(run(runtime.POA_INPUT));
}
```

`run()` 承担业务逻辑；`POA_INPUT` 由宿主提供；`text()` 把结果交给宿主展示。
仅导出函数不会自动执行，需要保留入口调用。业务变为异步时，也要修改入口，等待业务
完成再输出。

可将复杂逻辑拆到其他目录，由入口静态 `import`。主程序运行在 V8 环境，不能因开发时
使用 Node.js，就直接调用 `node:fs`、`process` 等 Node API。可用能力及 SDK 对象以
编写指南和API 参考为准；模板的`runtime.d.ts` 只提供部分类型声明，不能替代目标版本的 API 契约。

### 4.2 手写 input_schema.json

模板的输入规则核心如下：

```json
{
  "type": "object",
  "properties": {
    "message": { "type": "string" }
  },
  "additionalProperties": false
}
```

允许 `{}` 或 `{"message":"你好"}`。没有 `required`，所以 `message` 可以省略；有这个
字段时必须是字符串。修改输入时，需要同步修改源码中的类型、Schema 和相关测试。
TypeScript 类型自身不会在运行时校验外部输入。

模板当前使用**手写 JSON Schema**，不会从 TypeScript 输入类型自动生成 Schema。
PoA input 编写说明中的 `input_schema.ts` 自动编译属于另一条打包路径，
不能直接认为 `pnpm package` 已支持它。需要通过 SDK 或 app-server 传入业务输入时，
也可参考该篇；本篇 CLI 和 TUI 示例使用默认空输入。

### 4.3 维护 manifest.toml

```toml
[poa]
namespace = "brainary"
name = "my_poa"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
input_schema = "input_schema.json"
description = "A minimal Brainary PoA created from the template."
```

- `namespace`、`name`、`version`、`description`：由开发者维护业务身份、版本和描述。
- `runtime`、`poa_api_version`：当前模板固定并校验为 `codex-v8`、`1`。
- `entry`：当前模板固定为包根目录的 `main.js`，不要填写源码路径。
- `input_schema`：当前指向手写的 `input_schema.json`。不需要 Schema 时，应同时删除文件
  和该字段。

`package.json` 的版本和 manifest 的版本不会自动同步。更改 manifest 的名称或版本，
也不会改变输出文件名；当前打包脚本固定生成 `dist/my-poa.poa`。

## 5. 检查与测试

```bash
pnpm lint
pnpm typecheck
pnpm test
```

| 命令 | 工具 | 作用 |
| --- | --- | --- |
| `pnpm lint` | Biome | 检查代码格式和部分常见问题，不修改文件 |
| `pnpm format` | Biome | 统一格式，会修改文件；执行后审阅改动 |
| `pnpm typecheck` | TypeScript | 检查类型，不生成 JavaScript |
| `pnpm test` | Vitest | 执行测试断言，验证业务和适配行为 |

当前默认示例有 2 个测试文件、5 个测试：业务测试验证消息返回和默认值；Runtime 测试
模拟 `POA_INPUT`、`text()`，并检查 manifest 和包内文件。打包测试内部会执行
`pnpm package`，因此测试时也可能生成 `dist/`。

这些本地测试没有启动真实 Brainary。新增业务、MCP 或更改输入规则后，应调整相应测试，
最后再验证目标 Runtime 中的真实行为。

## 6. 构建和打包

单独构建时运行 `pnpm build`；要生成运行包，直接执行：

```bash
pnpm package
```

它在 `package.json` 中对应：

```text
pnpm build && node scripts/package.mjs
```

### 6.1 构建生成 dist/main.js

Rolldown 从 `src/main.ts` 出发，处理 TypeScript 并合并入口引用的模块，输出 ESM 格式的
`dist/main.js`。它是生成文件，应修改源码后重新构建。构建和类型检查是独立命令；构建
成功不代表 `pnpm typecheck` 已通过。

当前打包脚本只收集一个 `main.js` 入口。若新增动态导入并产生分块，需同步设计构建和
文件收集方式，避免遗漏运行所需文件。

### 6.2 打包脚本收集文件

`scripts/package.mjs` 执行以下步骤：

1. 使用 `smol-toml` 解析 manifest，检查必填字符串、Runtime、API 版本和固定入口。
2. 确认 `dist/main.js` 存在，输入 Schema 与 manifest 引用一致。
3. 检查输入 Schema 是合法 JSON，并声明 `"type": "object"`。
4. 创建系统临时目录，整理 manifest、入口、可选 Schema 和 `mcp/` 文件。
5. 使用 `fflate` 生成 ZIP 格式的 `dist/my-poa.poa`，最后清理临时目录。

当前 Node 脚本没有完整校验所有 JSON Schema 规则，生成的包仍需由目标 Runtime 加载
验证。`pnpm package` 也不会自动执行 `lint`、`typecheck` 或 `test`。

成功时输出：

```text
Created dist/my-poa.poa
```

最小示例包内有三个文件：

```text
my-poa.poa
├── manifest.toml
├── main.js
└── input_schema.json
```

macOS 或 Linux 上可使用 `unzip -l dist/my-poa.poa` 查看内容。

### 6.3 模块、资源和 MCP 如何进入包

| 内容 | 当前处理方式 |
| --- | --- |
| 被入口静态引用的业务模块 | 通过 Rolldown 合并到 `main.js` |
| `prompts/` 中的独立文本、其他资源目录 | 不自动复制；需明确的导入或资源打包方案 |
| `mcp/` 的普通文件 | 原样复制，跳过点号开头的文件和目录 |
| 测试、开发配置、文档、`node_modules/` | 不直接收集进包 |

提示词可先使用 TypeScript 字符串常量模块，在入口或 Agent 模块中导入。目录存在本身
不会让资源进入包。

当前收集过程按固定入口和上述规则执行，不会扫描任意已有项目并自动推断所有入口、资源
或 manifest 字段。

MCP 是可选能力；需要时在 manifest 中声明 stdio 服务，并准备目标环境能够启动的服务
和依赖。模板不会自动编译 `mcp/` 中的 TypeScript，也不会自动安装其运行依赖。Node.js
MCP 需要运行环境中的 Node.js。
新增随包文件后，还要调整默认打包测试中对包内文件清单的断言。

## 7. 使用 Brainary 运行

在同一项目根目录执行：

```bash
brainary poa dist/my-poa.poa --json --skip-git-repo-check --ephemeral
```

`--json` 输出 JSONL 事件；`--skip-git-repo-check` 允许在 Git 仓库外运行；`--ephemeral`
不持久化会话文件。默认空输入的业务结果为：

```json
{"ok":true,"message":"Hello from Brainary PoA."}
```

JSONL 还包含运行事件，业务结果可能嵌在 `custom_tool_call_output` 的输出字段中。
应同时检查命令退出状态和业务结果。这个最小示例没有调用模型，复杂的 Agent 业务还需
按快速开始配置模型和工具。

也可以先在系统终端执行 `brainary` 进入 TUI，再在 TUI 输入框中运行：

```text
/poa /absolute/path/to/my-poa/dist/my-poa.poa
```

把 `/absolute/path/to/my-poa` 替换为项目实际的绝对路径。CLI 的 `brainary poa` 在系统
终端中执行；`/poa` 在 TUI 中输入，pnpm 命令仍在系统终端中执行。

## 8. JavaScript 入口

使用 JavaScript 时，把 `src/main.ts` 替换为 `src/main.js`，移除 TypeScript 专用语法。
构建配置在存在 `main.ts` 时优先选它，不存在时选 `main.js`。不要同时保留两份入口并
误以为会自动识别本次修改的是哪一份。

当前 `tsconfig.json` 开启 `allowJs`，但 `checkJs` 为 `false`，未启用对 JavaScript
业务文件的类型检查。测试发现规则当前只包含 `tests/**/*.test.ts`，可以继续用
TypeScript 写测试；若改写成 JavaScript 测试，需要同步调整 Vitest 的发现规则。

每次业务修改后，依次执行以下命令，处理当前步骤的错误后再继续：

```bash
pnpm lint
pnpm typecheck
pnpm test
pnpm package
brainary poa dist/my-poa.poa --json --skip-git-repo-check --ephemeral
```

提交源码、测试、manifest、Schema、配置、脚本和锁文件，保持 `dist/`、`node_modules/`
不进入源码仓库。需要交付运行包时，交付新生成的 `.poa`。

| 常见现象 | 检查方向 |
| --- | --- |
| 找不到 pnpm、Node 或开发工具 | 检查环境，并在当前项目运行 `pnpm install` |
| 找不到项目命令或 `package.json` | 确认终端当前位于 clone 得到的项目根目录 |
| 打包校验失败 | 按错误信息检查 manifest、输入 Schema 和构建入口 |
| 运行结果没有变化 | 修改后重新打包，并确认运行的是同一份项目的产物 |
| 新增资源没有进入包 | 检查是否被入口导入，或是否已加入打包收集规则 |
