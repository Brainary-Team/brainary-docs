# 输入与输出

本篇说明程序如何接收业务输入、输出结果，以及在运行中向用户提问。

## 运行方式

同一个 `.poa` 包有三种运行方式，它们在输入和提问上有所不同：

| 方式 | 用法 | 业务输入 | `ask()` |
| --- | --- | --- | --- |
| CLI | `brainary poa <.poa 文件>` | 不传入 | 返回 `unavailable` |
| TUI | 在 `brainary` 界面中输入 `/poa <.poa 文件>` | 不传入 | 显示问题，等待用户回答 |
| Python SDK | `Poa(thread, package).run(input=...)` | 传入 | 不提供，用业务输入代替提问 |

TUI 中用 `/poa bg <.poa 文件>` 在后台会话中运行，此时 `ask()` 同样返回 `unavailable`，详见 [Brainary TUI 使用说明](../CLITUI-Command/Brainary-TUI使用说明.md#运行-poa-包)。

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

`pnpm package` 会把它编译成 JSON Schema 写入包中。`input_schema.ts` 只能包含类型声明：interface、type、可选属性、数组、字面量、联合类型、交叉类型、对其他声明的引用和 JSDoc 约束标签；不支持 import、泛型声明和 interface extends。对象不接受未声明的字段。

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
