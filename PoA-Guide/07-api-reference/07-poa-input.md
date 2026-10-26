# POA INPUT SDK

本页说明 `.poa` 包如何声明业务输入、如何把 TypeScript 类型编译为 JSON Schema，以及
程序如何在运行时读取经过校验的 `POA_INPUT`。

## 包结构

```text
echo-demo/
├── manifest.toml
├── input_schema.ts
└── main.js
```

`input_schema.ts` 是打包时的源文件；运行时实际读取包内的 `input_schema.json`。
`manifest.toml` 最小示例：

```toml
[poa]
namespace = "example"
name = "echo_demo"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
```

## `input_schema.ts`

在包根目录声明名为 `InputParams` 的 interface 或 type。根类型必须解析为 JSON Schema
对象。必填属性直接声明，可选属性使用 `?`；普通 TypeScript 类型和注释约束会转换为
JSON Schema 约束。

```ts
export interface InputParams {
  /** @minLength 1 */
  message: string;
  /**
   * @asType integer
   * @minimum 1
   * @maximum 10
   */
  repeat?: number;
}
```

类型定义（简化）：

```ts
type InputParams = {
  message: string; // 必填
  repeat?: number; // 可选
};
```

打包器成功编译该文件时生成 `input_schema.json`，失败时报告 schema 编译错误。
没有声明 `InputParams`、根类型不是
对象或语法无效时，打包应失败。

## 打包生成 JSON Schema

```bash
codex-poa-pack ./echo-demo ./echo_demo.poa
```

打包器会编译 `input_schema.ts`，在归档中写入 `input_schema.json`，并把归档内
`[poa].input_schema` 设置为该文件名。源目录的 `manifest.toml` 不会被改写。默认使用
JSON Schema Draft 2020-12。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "message": { "type": "string", "minLength": 1 },
    "repeat": { "type": "integer", "minimum": 1, "maximum": 10 }
  },
  "required": ["message"],
  "additionalProperties": false
}
```

如果手写 JSON Schema，则在 `manifest.toml` 中将 `[poa].input_schema` 设为包内相对路径，
且文件必须是对象 schema、以 `.json` 结尾并位于包目录内。

## `POA_INPUT`

`POA_INPUT` 是调用方传入的、已经解析并通过 schema 校验的 JSON 值；程序不需要再次
`JSON.parse`。有 schema 的包在启动 JavaScript 前校验输入；没有 schema 的包只接受空对象，
此时 `POA_INPUT` 的值是空对象 `{}`。

全局常量类型：

```ts
declare const POA_INPUT: InputParams;
```

程序直接读取该常量获取业务 JSON：

```js
const { message, repeat = 1 } = POA_INPUT;
text({ message, repeat });
```

调用方传入：

```json
{ "message": "你好", "repeat": 2 }
```

直接 SDK 调用的抽象形状：

```python
result = thread.run_poa(
    "echo_demo.poa",
    input={"message": "你好", "repeat": 2},
)
```

## 校验失败结构

校验发生在 PoA 初始化和 JavaScript 执行之前。失败时，直接 SDK 调用得到异常；模型工具
调用得到结构化失败结果。错误的公开形状为：

```ts
type PoaInputValidationFailure = {
  status: "invalid_input";
  errors: Array<{
    fieldPath: string;
    schemaPath: string;
    fieldSchema: unknown;
  }>;
};
```

示例：缺少必填的 `message` 时，结果类似：

```json
{
  "status": "invalid_input",
  "errors": [
    {
      "fieldPath": "/message",
      "schemaPath": "/required",
      "fieldSchema": { "type": "string", "minLength": 1 }
    }
  ]
}
```

`fieldPath` 和 `schemaPath` 使用 JSON Pointer；`fieldSchema` 是约束该字段的完整 schema
片段。错误数组会按字段路径和 schema 路径排序，便于稳定展示。

## SDK 使用边界

- `input_schema.ts` 只在打包阶段执行编译，不会在 V8 中作为 JavaScript 运行。
- `POA_INPUT` 是一次调用的业务输入。
- schema 必须描述对象，根级数组、字符串或数字输入不受此包格式支持。
- 输入通过校验后才会启动 PoA 和它声明的 MCP server；校验失败不会执行 `main.js`。
