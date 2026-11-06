# 使用模板开发 PoA

本篇使用 `brainary-poa-template` 独立模板开发 PoA。克隆项目后编写代码，用模板自带的脚本生成 `.poa` 包，再交给 Brainary 运行。开发和打包不需要安装 Brainary，运行时才需要。

## 1. 获取模板

先安装 Git、Node.js 22 或更新版本，以及 pnpm 12（模板指定 `12.14.0`）。推荐使用 Node.js 24.x。
如果开启了依赖版本的严格检查，请使用 Node.js 22.12+（22.x）、24.x，或 26 及以上版本。

```bash
git clone https://github.com/Brainary-Team/brainary-poa-template.git my-poa
cd my-poa
pnpm install
```

`my-poa` 是项目文件夹的名字，可以自己改。后面的命令都在这个文件夹里执行。

## 2. 编写自己的 PoA

主要改下面几个地方：

| 文件 | 要做什么 |
| --- | --- |
| `src/main.ts` | 编写 PoA 的功能 |
| `input_schema.ts` 或 `input_schema.json` | 定义允许传入哪些参数 |
| `manifest.toml` | 填写项目名称、版本和说明 |
| `tests/` | 为自己的功能编写测试 |

### 程序入口

`src/main.ts` 里的 `run()` 是默认示例的业务逻辑。示例收到 `message` 后返回这条消息，不传时返回默认问候语。

文件末尾已经写好了读取 `POA_INPUT`、调用 `run()` 和用 `text()` 输出结果的代码。修改业务时保留这部分调用；如果改成异步函数，要等待执行完成再输出。

主程序运行在 Brainary 的 V8 环境中，可用能力以 PoA API 说明为准，不能直接使用 `node:fs`、`process` 等 Node.js API。

如果使用 JavaScript，把 `src/main.ts` 换成 `src/main.js`，并去掉类型声明。两份入口同时存在时，构建工具优先使用 TS 文件。测试仍可以用 TypeScript 编写。

### 输入参数

默认在根目录的 `input_schema.ts` 中定义参数，类型名必须是 `InputParams`。例如：

```ts
/** @additionalProperties false */
export interface InputParams {
  message?: string;
}
```

这里表示 `message` 是可选的字符串，不接受其他字段。`{}` 和 `{"message":"你好"}` 都是合法输入。

打包时，这个文件会自动转换成 JSON Schema。它只用来写类型定义，目前不支持 `import`、函数、自定义泛型和接口继承。需要限制字符串长度、数值范围等时，可以添加 `@minLength`、`@minimum` 等注解。

**也支持直接写 JSON。** 删除模板里的 `input_schema.ts`，改用根目录的 `input_schema.json`：

```json
{
  "type": "object",
  "properties": {
    "message": { "type": "string" }
  },
  "additionalProperties": false
}
```

两个文件同时存在时，优先使用 TS，忽略 JSON。TS 写错了会报错，不会自动改用 JSON。
修改输入参数后，记得一起修改 `src/main.ts` 中的类型、业务逻辑和相关测试。

### 项目名称和版本

在 `manifest.toml` 中修改 `namespace`、`name`、`version` 和 `description`。下面三项保持默认值：

```toml
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
```

提供输入定义后，打包脚本会在包内的 manifest 中自动加上 `input_schema = "input_schema.json"`，不用手写。
如果项目不需要输入定义，可以删除 TS 和 JSON 输入文件；manifest 中手写过的 `input_schema` 也要删除。

## 3. 检查并打包

依次执行，每一步成功后再继续：

```bash
pnpm lint
pnpm typecheck
pnpm test
pnpm package
```

前三条分别检查代码格式和常见问题、检查类型、运行测试。`pnpm package` 会先构建，再生成 `.poa` 包，不会自动执行前面的检查和测试。

打包时，模板会把源码构建成 `dist/main.js`，读取并校验输入定义，再将所需文件打进 `dist/my-poa.poa`。成功时会显示：

```text
Created dist/my-poa.poa
```

默认示例的包里有三个文件：

```text
my-poa.poa
├── manifest.toml
├── main.js
└── input_schema.json
```

生成的 JSON 和补充后的 manifest 只写进包内，项目里的源文件不会被改写。
输出文件名固定为 `my-poa.poa`，不会随 manifest 中的名称或版本自动变化。

在 macOS 或 Linux 上，可以检查包内文件和转换结果：

```bash
unzip -l dist/my-poa.poa
unzip -p dist/my-poa.poa input_schema.json
unzip -p dist/my-poa.poa manifest.toml
```

如果打包报错，先处理错误再重试；`dist/` 中可能还留着上次生成的旧包。

## 4. 用 Brainary 运行

准备好 Brainary 后，在项目目录执行：

```bash
brainary poa dist/my-poa.poa --json --skip-git-repo-check --ephemeral
```

如果终端找不到 `brainary`，将它替换为二进制文件的完整路径。这条命令使用默认空输入，默认示例应返回：

```json
{"ok":true,"message":"Hello from Brainary PoA."}
```

`--json` 会同时输出运行事件，业务结果在 `custom_tool_call_output` 中。确认命令没有报错，并且返回了预期内容。
自己的 PoA 如果要调用模型或工具，还需要配置对应服务。

也可以运行 `brainary` 进入 TUI，再在输入框中输入：

```text
/poa /absolute/path/to/my-poa/dist/my-poa.poa
```

将路径换成项目的实际绝对路径。修改代码后，重新检查、测试和打包，再运行新包。
提交代码时保留源码和配置，不要把 `dist/`、`node_modules/` 提交进仓库；交付运行包时提供生成的 `.poa` 文件。

## 5. 项目结构和内部流程

### 5.1 目录结构

```text
my-poa/
├── src/
│   └── main.ts                   # 程序入口
├── agents/                       # Agent 代码
├── prompts/                      # 提示词
├── memory/                       # 记忆和状态处理代码的预留目录
├── tools/                        # 工具调用代码
├── mcp/                          # 可选的 stdio MCP 服务文件
├── manifest.toml                 # 项目说明和运行配置
├── input_schema.ts               # 输入参数定义，也可以改用 JSON
├── tests/
│   ├── agent.test.ts              # 默认示例的业务测试
│   ├── runtime.test.ts            # 入口调用和包内文件测试
│   ├── input-schema.test.ts       # 输入定义转换测试
│   ├── package.test.ts            # 打包流程测试
│   └── fixtures/                  # 测试样例
├── types/
│   └── runtime.d.ts               # 运行时能力的类型声明
├── scripts/
│   ├── compile-input-schema.mjs   # TS 输入定义转 JSON Schema
│   ├── validate-input-schema.mjs  # 校验 JSON Schema
│   └── package.mjs                # 组织文件并生成 .poa
├── assets/                       # README 截图等文档图片
├── dist/                         # 生成的 main.js 和 my-poa.poa
├── package.json                  # 项目命令和依赖
├── pnpm-lock.yaml                # 锁定依赖版本
├── tsconfig.json                 # TypeScript 检查配置
├── biome.json                    # 代码检查和格式化配置
├── rolldown.config.mjs            # 构建配置
├── vitest.config.ts              # 测试配置
├── README.md
└── .gitignore
```

`agents/`、`prompts/`、`memory/`、`tools/` 是给业务代码预留的位置，可以按需要添加文件。例如，`memory/` 目录本身不提供记忆功能，需要项目自己实现。

### 5.2 各个工具负责什么

Node.js 负责运行开发和打包工具，pnpm 负责安装依赖、执行 `package.json` 中的命令。项目使用 ES module，`package.json` 中设置了 `"type": "module"`。

| 工具 | 职责 | 对应配置或脚本 |
| --- | --- | --- |
| TypeScript | 检查代码类型；提供解析 TS 输入定义的能力 | `tsconfig.json`、`compile-input-schema.mjs` |
| Biome | 检查代码和统一格式 | `biome.json` |
| Vitest | 执行 `tests/` 中的测试 | `vitest.config.ts` |
| Rolldown | 把业务源码构建成 JavaScript | `rolldown.config.mjs` |
| smol-toml | 读取和生成 manifest | `package.mjs` |
| Ajv、ajv-formats | 校验输入规则是否有效 | `validate-input-schema.mjs` |
| fflate | 将整理好的文件压缩成 `.poa` | `package.mjs` |

TypeScript 检查和 Rolldown 构建是两件事：`pnpm typecheck` 检查类型，`pnpm build` 生成 JS。构建成功不代表类型检查通过。

### 5.3 一次打包内部做了什么

`pnpm package` 实际执行：

```text
pnpm build && node scripts/package.mjs
```

1. **构建业务代码。** Rolldown 读取 `src/main.ts`，没有 TS 入口时读取 `src/main.js`，将静态导入的模块合并，输出 ES module 格式的 `dist/main.js`。
2. **检查项目配置。** `package.mjs` 读取 manifest，检查名称、版本等必填项，以及固定的运行时、API 版本和入口。
3. **准备输入规则。** 有 `input_schema.ts` 时，调用 `compile-input-schema.mjs` 解析类型声明，生成 JSON Schema；只有 JSON 时直接读取。两种文件都没有时，跳过输入规则处理。
4. **校验输入规则。** `validate-input-schema.mjs` 使用 Ajv 检查 Schema 的类型、引用和限制条件写法，要求根节点为对象。这里检查的是规则本身，不是某一次传入的业务数据。
5. **整理包内文件。** 在系统临时目录中放入 `main.js`、manifest、可选的输入 Schema 和 MCP 文件。有输入规则时，在包内 manifest 中补上对应路径。
6. **生成运行包。** 使用 fflate 生成 ZIP 格式的 `dist/my-poa.poa`，最后清理临时目录。临时目录不会留在项目根目录。

整个过程都在本地 Node.js 中完成。Brainary 在运行阶段读取包内的 manifest，并按 `entry` 加载入口程序。

### 5.4 哪些文件会进入包

| 项目中的内容 | 打包时怎么处理 |
| --- | --- |
| 入口及其静态导入的业务模块 | 合并进 `main.js` |
| 根目录的 TS 或 JSON 输入定义 | 按 TS 优先的规则读取，校验后以 `input_schema.json` 放入包 |
| `manifest.toml` | 复制进包，有输入定义时自动补上 `input_schema` |
| `mcp/` 中的普通文件 | 按原目录结构复制，跳过点号开头的文件和目录，拒绝符号链接 |
| 独立的提示词文本和其他资源文件 | 不会自动复制，需要额外设计导入或打包方式 |
| 测试、配置、文档、`node_modules/` | 不直接收集进包 |

提示词可以先写成 TS 字符串常量，再由入口或 Agent 模块导入。当前打包脚本只收集一个 `main.js`；如果动态导入产生了额外 JS 文件，需要相应调整文件收集逻辑。

MCP 文件会原样复制，模板不会自动编译其中的 TS 或安装服务依赖。需要 MCP 时，在 manifest 中声明 stdio 启动命令，并准备好运行依赖；使用 Node.js 编写的服务需要目标环境装有 Node.js。

`types/runtime.d.ts` 供编辑器和类型检查使用，不会给运行中的 PoA 增加能力。`POA_INPUT`、`text()` 等实际能力由 Brainary 提供，具体用法以目标版本的 PoA API 为准。

### 5.5 测试覆盖哪些部分

| 测试文件 | 检查内容 |
| --- | --- |
| `agent.test.ts` | 默认示例的消息返回和默认值 |
| `runtime.test.ts` | 模拟 `POA_INPUT`、`text()`，检查入口调用、manifest 和包内文件 |
| `input-schema.test.ts` | TS 类型和注解的转换结果，以及不支持的写法能否正确报错 |
| `package.test.ts` | TS 优先、手写 JSON、无输入规则、MCP 文件、重复打包和失败处理 |

默认测试不会启动真实 Brainary。业务或输入定义改动后，要同步调整测试，最后按第 4 节用 Brainary 验证实际运行结果。
