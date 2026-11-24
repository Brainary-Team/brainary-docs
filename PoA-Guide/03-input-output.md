# 输入与输出

本篇说明程序如何接收业务输入、输出结果，以及在运行中向用户提问。

## 运行方式

同一个 `.poa` 包有三种运行方式，它们在输入和提问上有所不同：

| 方式 | 用法 | 业务输入 | `ask()` |
| --- | --- | --- | --- |
| CLI | `brainary poa [--input JSON\|--input-file FILE] <.poa 文件>` | 通过命令行参数传入 | 返回 `unavailable` |
| TUI | 在 `brainary` 界面中输入 `/poa <.poa 文件> [--input JSON\|--input-file FILE]` | 通过斜杠命令传入 | 显示问题，等待用户回答 |
| Python SDK | `Poa(thread, package).run(input=...)` | 传入 | 不提供，用业务输入代替提问 |

TUI 中用 `/poa bg <.poa 文件>` 在后台会话中运行，此时 `ask()` 同样返回 `unavailable`，详见 [Brainary TUI 使用说明](../CLITUI-Command/Brainary-TUI使用说明.md#运行-poa-包)。

CLI 和 TUI 都支持两种传入方式：直接给出 JSON，或从文件读取 JSON。`--input` 与 `--input-file` 只能二选一；输入会在程序启动前按包内的 `input_schema` 校验，校验失败时程序不会运行。不传输入时，`POA_INPUT` 为 `{}`。

```bash
brainary poa --input '{"root":"src","concurrency":5}' ./workflow.poa
brainary poa --input-file ./input.json ./workflow.poa
```

在 TUI 中使用：

```text
/poa ./workflow.poa --input {"root":"src","concurrency":5}
/poa ./workflow.poa --input-file ./input.json
```

## 接收输入

在项目根目录新建 `input_schema.ts`，用名为 `InputParams` 的类型声明输入结构，注释中的标签会成为校验约束：

```ts
export interface InputParams {
  /**
   * 要扫描的目录
   * @minLength 1
   */
  root: string;
  /**
   * @asType integer
   * @minimum 1
   * @maximum 10
   */
  concurrency?: number;
}
```

`pnpm package` 会把它编译成 JSON Schema 写入包中。`input_schema.ts` 只能包含类型声明：interface、type、可选属性、数组、字面量、联合类型、交叉类型、对其他声明的引用和 JSDoc 约束标签；不支持 import、泛型声明和 interface extends。对象不接受未声明的字段。支持的类型和标签见 [输入 schema 写法](#输入-schema-写法)。

程序从 `POA_INPUT` 读取输入：

```ts
import { POA_INPUT, text } from "@brainary/poa-sdk";
import type { InputParams } from "../input_schema";

const { root, concurrency = 3 } = POA_INPUT as unknown as InputParams;
text({ root, concurrency });
```

调用方用 Python SDK 传入输入：

```python
from brainary import Brainary, BrainaryConfig, Poa, PoaPackage

package = PoaPackage.load("dist/my-poa.poa")

with Brainary(config=BrainaryConfig(brainary_bin="/path/to/brainary")) as brainary:
    thread = brainary.thread_start(
        cwd="/path/to/workspace",
        config={"features": {"code_mode": {"enabled": True}}},
    )
    result = Poa(thread, package).run(input={"root": "src", "concurrency": 5})
    print(result)
```

- 输入在程序启动前按 schema 校验，`POA_INPUT` 拿到的一定是合法的值。
- 校验失败时程序不会运行，调用方收到 `invalid_input` 错误，`errors` 中列出每个不合法字段的路径（`fieldPath`）和违反的约束（`schemaPath`）。
- 调用方不传输入时，`POA_INPUT` 为 `{}`。没有 `input_schema.ts` 的包只接受空输入。

Python SDK 的安装和完整用法见 [使用 Python SDK 运行 PoA](../App-SDK/Python-SDK/run-poa.md)。

## 输入 schema 写法

`InputParams` 是输入的根，必须是对象类型。JSDoc 写在 interface、type 或成员的上方：正文成为 `description`，标签成为约束。

### 类型对应关系

| JSON Schema | TypeScript 写法 |
| --- | --- |
| `$schema` | `InputParams` 上的 `@schema`；不写时为 `https://json-schema.org/draft/2020-12/schema` |
| `type` | `string`、`number`、`boolean`、`null`、对象类型、数组类型；`integer` 用 `number` 加 `@asType integer` |
| `properties` | interface 或对象类型 `{ ... }` 的成员，成员名可以是标识符或带引号的字符串 |
| `required` | 不带 `?` 的成员 |
| `additionalProperties: false` | 默认值：对象不接受未声明的字段；也可以显式写 `@additionalProperties false` |
| `additionalProperties: true` | `@additionalProperties true` |
| `additionalProperties: <schema>` | `@additionalProperties {"type":"string","minLength":1}`、`@additionalProperties NonEmptyString`，或索引签名 `[key: string]: T` |
| `const` | 单个字面量类型，如 `"image"`、`3`、`true`，同时生成对应的 `type` |
| `enum` | 字面量联合，如 `"fast" \| "accurate"`；重复的值只保留一个，各值类型相同时同时生成 `type` |
| `items` | `T[]` 或 `Array<T>` |
| `anyOf` | 含非字面量分支的联合类型，如 `string \| ImageInput` |
| `oneOf` | 带 `@oneOf` 的联合类型 |
| `allOf` | 交叉类型 `A & B`。非对象类型生成 `allOf`；对象类型合并为一个对象：成员和必填字段取并集，同名成员的约束同时生效，任一侧不接受额外字段时结果也不接受。各部分类型必须相同，如 `string & number` 打包时报错 |
| `$defs` | 从 `InputParams` 可达的命名 interface 和 type |
| `$ref` | 对命名 interface 或 type 的引用，生成 `#/$defs/<类型名>` |
| `title` | `@title` |
| `description` | JSDoc 正文，即第一个标签之前的文字 |
| `format`、`pattern`、`minLength` 等约束 | 同名 JSDoc 标签，见下表 |

### JSDoc 标签

只支持下表中的标签。其他标签（如 `@default`、`@example`）以及在同一段注释中重复出现的标签会导致打包失败。

| 标签 | 作用 |
| --- | --- |
| `@schema <URI>` | 指定根 schema 的 `$schema`，只在 `InputParams` 上生效 |
| `@title <文本>` | 生成 `title` |
| `@format <格式>` | 字符串须符合指定格式，如 `uri`、`email`、`date-time`；校验输入时会检查格式 |
| `@contentEncoding <编码>` | 标注字符串内容的编码，如 `base64`，生成 `contentEncoding`；只作标注，不校验或解码内容 |
| `@pattern <正则>` | 字符串须匹配该正则，表达式原样写入 `pattern` |
| `@minLength` / `@maxLength <n>` | 字符串的最小 / 最大长度，`n` 为非负整数 |
| `@minItems` / `@maxItems <n>` | 数组的最少 / 最多元素个数，`n` 为非负整数 |
| `@minimum` / `@maximum <x>` | 数值下限 / 上限，可以等于边界，`x` 为 JSON 数字 |
| `@exclusiveMinimum` / `@exclusiveMaximum <x>` | 数值严格下限 / 上限，不能等于边界，`x` 为 JSON 数字 |
| `@multipleOf <x>` | 数值须为 `x` 的整数倍，`x` 为大于 0 的 JSON 数字 |
| `@uniqueItems [true \| false]` | 数组元素不能重复，不写值等同于 `true`；有重复元素时校验失败，不会自动去重 |
| `@asType integer` | 把 `number` 生成的 `type` 改为 `integer`，要求输入为整数；只能标注 `number` 类型的成员或 type |
| `@oneOf` | 要求恰好匹配联合类型的一个分支；不带值，只能标注联合类型的成员或 type。非字面量联合的 `anyOf` 改为 `oneOf`，字面量联合的 `enum` 改为多个 `const` 分支 |
| `@additionalProperties <值>` | 控制对象是否接受未声明的字段，只能标注对象类型。值为 `true`、`false`、以 `{` 开头的内联 JSON Schema 或已声明的类型名，后两种要求额外字段的值符合该 schema |

## 输出结果

`text()` 输出文本或 JSON，可以多次调用，内容按顺序追加：

```ts
import { image, notify, text } from "@brainary/poa-sdk";

notify({ phase: "scanning", done: 3, total: 10 });  // 立即发送一条进度通知

text("## 扫描结果");
text({ folders: 10, failed: 0 });                  // 对象输出为 JSON
image("data:image/png;base64,iVBORw0KGgo...");     // 输出图片，取值为 data URI
```

| 函数 | 作用 |
| --- | --- |
| `text(value)` | 追加文本、数字、布尔值或可序列化的 JSON |
| `image(value)` | 追加图片：data URI、MCP 图片内容或 `FileExecutor.view_image()` 的结果 |
| `audio(value)` | 追加音频：data URI 或 MCP 音频内容 |
| `notify(value)` | 立即发送一条进度或状态通知 |
| `yield_control()` | 把已经追加的输出立即交给调用方，程序继续运行 |
| `exit()` | 立即结束本次运行 |

CLI 在运行结束后把输出打印到终端，Python SDK 的 `run()` 返回这些输出。完整说明见 [IO](05-api-reference/07-io.md)。

## 向用户提问

`ask()` 向用户提出 1–3 个问题，每个问题可以给 2–3 个选项，也可以让用户自由填写：

```ts
import { ask, text } from "@brainary/poa-sdk";

const result = await ask({
  questions: [
    {
      question: "部署到哪个环境？",
      options: [
        { label: "staging", description: "预发布环境" },
        { label: "production", description: "生产环境" },
      ],
    },
    { question: "需要补充说明吗？" },
  ],
});

if (result.status === "answered") {
  const [environment, note] = result.answers;
  text({ environment: environment.selected, note: note.text });
} else {
  text({ environment: "staging" });  // 无法提问时使用默认值
}
```

在 TUI 前台运行时，用户可以回答问题，派出的 agent 也可以向用户提问。CLI 和 TUI 后台运行时，`ask()` 立即返回 `{ status: "unavailable" }`，程序应准备默认值。问题和答案的完整结构见 [ask](05-api-reference/07-io.md#askrequest)。
