# PoA input 编写说明

## 1. 示例：城市计划

这个小程序接收城市和天数，输出一条计划标题。先在包根目录创建三个文件：

```text
city-plan/
├── manifest.toml
├── input_schema.ts
└── main.js
```

`input_schema.ts` 定义输入结构，根类型必须命名为 `InputParams`：

```ts
/** 城市计划的输入。 */
export interface InputParams {
  /**
   * 城市名称。
   * @minLength 1
   */
  city: string;

  /**
   * 计划天数，省略时由程序使用 1 天。
   * @asType integer
   * @minimum 1
   * @maximum 7
   */
  days?: number;
}
```

准备好包内文件后，在 `city-plan` 目录的上一级打包：

```bash
codex-poa-pack ./city-plan ./city_plan.poa
```

第一个参数是源目录，第二个是输出文件。包根目录的 `input_schema.ts` 必须定义
`InputParams`；打包时自动生成 `input_schema.json` 并设置包内 manifest 的
`input_schema`，不修改源目录文件。

`manifest.toml` 声明程序和输入 schema：

```toml
[poa]
namespace = "examples"
name = "city_plan"
version = "0.1.0"
runtime = "codex-v8"
poa_api_version = 1
entry = "main.js"
description = "根据城市和天数生成计划标题"
input_schema = "input_schema.json" #如果poa目录中包含了规范的input_schema.ts文件可以不指定, 使用codex-poa-pack poa目录地址 poa包保存地址，打包poa时会自动解析并赋值
```

`main.js` 通过全局变量 `POA_INPUT` 读取业务输入：

```js
// @exec: {"max_output_tokens": 1000}
const { city, days = 1 } = POA_INPUT;
text(JSON.stringify({ city, days, title: `${city}：${days} 天计划` }));
```

调用方传入的 JSON 对象是：

```json
{"city": "Singapore", "days": 3}
```

程序输出的文本是：

```json
{"city":"Singapore","days":3,"title":"Singapore：3 天计划"}
```

`city` 必填，`days` 可选。传入 `days: 0`、非整数天数或缺少 `city`，都会在程序
执行前被 schema 拒绝；省略 `days` 时，`main.js` 中的默认值才会生效。

## 2. 从 TypeScript 到 JSON Schema

`input_schema.ts` 是声明输入结构的文件，不是程序入口，也不在 V8 中执行。
运行时打包目录时会解析 TypeScript 的类型和 JSDoc，生成 JSON Schema，再校验 schema
是否合法。默认使用 JSON Schema Draft 2020-12。

- 自动编译只识别包根目录下名为 `input_schema.ts` 的文件。
- 生成的 `input_schema.json` 写入归档，并将归档内的 `[poa].input_schema` 指向它。
  若已有同名 JSON 文件或其他 schema 声明，归档中会使用新生成的 schema。
- 这一过程只改变打包结果，不会改写源目录中的 JSON 或 manifest。
- 根类型必须是 `InputParams`，并且最终解析为对象 schema。
- `$defs` 只包含从 `InputParams` 可达的命名类型；未使用的声明不会加入 `$defs`。
- 普通属性是必填字段，带 `?` 的属性是可选字段；对象默认拒绝未声明字段
  （`additionalProperties: false`）。

例如，`codex exec --poa ./city-plan` 的目录打包路径会执行上述编译。不过当前 CLI
不提供业务 input 参数，它以空对象 `{}` 运行；本例含必填字段，应通过 Python SDK
或 app-server 传入输入。

### 2.1 为 Python SDK 准备包

SDK 读取已打包的 `.poa`，不会编译 TypeScript；普通 `zip` 命令也不会编译它。
若手工打包本例，请在 `city-plan/input_schema.json` 中保存以下等价 schema，保留
manifest 中的 `input_schema = "input_schema.json"`：

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "description": "城市计划的输入。",
  "properties": {
    "city": {
      "type": "string",
      "description": "城市名称。",
      "minLength": 1
    },
    "days": {
      "type": "integer",
      "description": "计划天数，省略时由程序使用 1 天。",
      "minimum": 1,
      "maximum": 7
    }
  },
  "required": ["city"],
  "additionalProperties": false
}
```

从包目录的上一级运行，生成可供 SDK 加载的 `city_plan.poa`：

```bash
python - <<'PY'
from pathlib import Path
from zipfile import ZIP_DEFLATED, ZipFile

root = Path("city-plan")
with ZipFile("city_plan.poa", "w", compression=ZIP_DEFLATED) as archive:
    for name in ("manifest.toml", "main.js", "input_schema.json"):
        archive.write(root / name, arcname=name)
PY
```

这样 zip 根目录直接包含 manifest、入口和 JSON Schema。完整 SDK 调用代码见
[使用 Python SDK 运行 PoA](../App-SDK/Python-SDK/run-poa.md)。已经有开启 code mode
的线程时，调用形式为：

```python
result = thread.run_poa("city_plan.poa", input={"city": "Singapore", "days": 3})
```

## 3. 输入校验与程序使用

`thread/codeMode/exec` 接受可选的 `input` 字段；它是业务 JSON 对象，包仍通过
`package` 字段提交。Python 的 `Poa.run(input=...)` 和 `thread.run_poa(..., input=...)`
封装了这一接口。程序在运行时读取 `POA_INPUT`（也可写 `globalThis.POA_INPUT`），
不需要自行解析 JSON 字符串。

有 schema 时，运行时先校验输入，再准备包内 MCP server 和执行程序。省略输入时使用
空对象 `{}`，因此包含必填字段的 schema 仍会报错。没有声明 schema 的包只接受空对象。
默认值应在程序中处理；可选字段不等于 nullable，`days?: number` 允许省略，但不允许
传入 `null`。

通过 `dev_poas` 注册为模型工具时，输入 schema 也会成为 `poa.<name>` 的参数说明。
模型根据它填写业务参数，运行时仍用包内完整 schema 校验。失败信息包含
`status: "invalid_input"` 和 `errors`；每项错误提供 `fieldPath`、`schemaPath`、
`fieldSchema`，可用于定位缺失或非法字段。Python 直接调用时，请求失败会以 SDK
异常返回，不会产生正常的 PoA 输出。

## 4. 编写规则与边界

目前只支持顶层 `interface`、`type` 及其 `export` 形式，不支持 `import`、泛型类型
声明、`interface extends` 或可执行语句。需要组合对象时使用交叉类型 `A & B`。
JSDoc 要紧贴对应类型或属性；普通说明文字放在标签之前，每个标签独占一行，同一个
JSDoc 内不要重复相同标签。未支持的标签会报错，不会静默忽略。

交叉类型按以下规则合并：

- 对象与对象：属性取并集，必填字段取并集。
- 同名属性：schema 相同则去重；基础类型相同但约束不同则生成 `allOf`；类型不同报错。
- `additionalProperties` 采用更严格的约束：`false` 优先；`true` 采用另一侧；两个
  schema 生成 `allOf`。
- 非对象的交叉要求基础类型相同，结果为 `allOf`；其他组合及递归交叉引用报错。

`InputParams` 不能直接或间接引用自身。TypeScript 在此处使用的是受限的声明语法，
完整 TypeScript 程序或类型库不能直接当作 input schema。

## 5. 完整字段与语法列表

以下对应关系以当前分支的 `codex-rs/poa/src/input_schema_ts.rs`、
`input_schema_ts_intersection.rs` 和 `input_schema.rs` 为准。输入校验使用项目固定的
`jsonschema 0.58.1`，默认方言为 Draft 2020-12。

### 5.1 TypeScript 类型与结构的解析关系

| TypeScript 写法 | 生成的 JSON Schema | 字段含义与实际行为 |
| --- | --- | --- |
| `interface InputParams { ... }` / `type InputParams = { ... }` | 根 `{"type":"object", ...}` | 根类型名称固定；支持 `export`。根必须直接解析为对象，也可以由对象交叉类型合并得到 |
| `string` | `{"type":"string"}` | 只接受 JSON 字符串；不会把数值转成字符串 |
| `number` | `{"type":"number"}` | 接受整数和小数；不会把 `"3"` 转成数值 |
| `boolean` | `{"type":"boolean"}` | 只接受 `true` / `false`，不接受 `0` / `1` 或 `"true"` |
| `null` | `{"type":"null"}` | 值必须是 JSON `null`；这与字段是否可以省略是两个独立条件 |
| `field: T` | `properties.field` + `required: ["field"]` | 属性的类型放进 `properties`；无 `?` 时把属性名加入所在对象的 `required` |
| `field?: T` | `properties.field`，不加入 `required` | 允许字段缺失，但字段存在时仍必须满足 T。全部属性可选时，不生成 `required` 字段 |
| `field: string \| null` | `anyOf: [{"type":"string"},{"type":"null"}]`，并加入 `required` | 字段必须出现，值可以是字符串或 `null` |
| `field?: string \| null` | 同上，但不加入 `required` | 允许字段缺失、字符串或 `null` 三种情况 |
| `"fast"` / `3` / `true` 等单个字面量 | `{"const":"fast"}` / `{"const":3}` / `{"const":true}` | 必须等于指定 JSON 值。单独的 `null` 使用 `type: "null"`，不是 `const` |
| `"fast" \| "slow"` 等直接字面量联合 | `{"enum":["fast","slow"]}` | 必须等于列表中的某个值；字符串、数值、布尔值、`null` 可以组成混合字面量联合 |
| `string \| SomeObject` 等其他联合 | `{"anyOf":[...各分支 schema...]}` | 至少满足一个分支；同一值满足多个分支也合法。引用字面量别名的联合仍走 `anyOf`，不会展开别名再折叠成 `enum` |
| 联合类型加 `@oneOf` | `{"oneOf":[...各分支 schema...]}` | 必须恰好满足一个分支；直接字面量联合的 `enum` 会改成多个 `const` 分支 |
| `T[]` / `Array<T>` | `{"type":"array","items":<T 的 schema>}` | 数组的每个元素都按 T 校验；没有长度标签时允许空数组 |
| `{ name: string }` / 命名 `interface` 的定义 | `type`、`properties`、`required`、`additionalProperties` | 每一层对象独立生成自己的属性、必填列表和额外字段规则；默认禁止额外字段 |
| 属性中引用 `Profile` 等命名类型 | `{"$ref":"#/$defs/Profile"}` | 定义存入根 `$defs`，引用位置不复制定义；只有从根可达的类型及其依赖会进入 `$defs` |
| `A & B`，两侧为对象 | 合并成一个对象 schema | 合并属性、必填字段和额外字段规则；不会简单生成两个封闭对象的 `allOf` |
| `A & B`，两侧为可识别的同类非对象 | `{"allOf":[<A 的 schema>,<B 的 schema>]}` | 值必须同时满足两侧约束。合并器将 `number` 和 `integer` 视为同一数值类别，但整数约束仍保留 |
| `[key: string]: T` | `additionalProperties: <T 的 schema>` | 只校验未在 `properties` 声明的字段值；不会生成 `patternProperties`，也不会再次约束已声明字段 |
| `(T)` | T 的 schema | 括号不产生额外 schema 层级 |
| `export interface` / `export type` | 与不带 `export` 相同 | `export` 不影响输入数据的字段或校验规则 |

`properties` 定义“字段存在时如何校验”，`required` 定义“哪些字段必须存在”，
`additionalProperties` 定义“未声明字段是否允许以及如何校验”。三者分别生效。
`$defs` 只存放可复用的 schema 定义，不代表业务输入中需要有同名字段。

### 5.2 JSDoc 标签的解析关系与含义

| JSDoc 写法 | 生成的 Schema 字段 | 标签值的解析规则 | 含义与生效范围 |
| --- | --- | --- | --- |
| 标签前的普通说明文字 | `description` | 去掉注释外壳与行首 `*`，非空正文行以换行连接 | 给模型和开发者解释用途；不参与值校验，也不使用 `@description` |
| `@schema https://json-schema.org/draft/2020-12/schema` | 根 `$schema` | 使用根 `InputParams` 注释中的原始标签值；不填写标签时使用 Draft 2020-12 | 指定 schema 方言。标在普通属性或其他类型上不会生成局部 `$schema`；建议使用示例中的完整 URI |
| `@title 城市计划` | `title: "城市计划"` | 非空字符串，整行作为值 | 展示标题，不改变类型、必填性或默认值 |
| `@minLength 1` | `minLength: 1` | 非负整数，按 Rust `u64` 解析 | 字符串至少含 1 个 Unicode 码点；不是 UTF-8 字节数，也不是 JS 的 UTF-16 `.length` |
| `@maxLength 120` | `maxLength: 120` | 同上 | 字符串最多 120 个 Unicode 码点；一个可见字符可能由多个码点组成 |
| `@pattern ^[A-Za-z]+$` | `pattern: "^[A-Za-z]+$"` | 非空字符串，直接保留正则文本 | 字符串匹配约束；不写 `^` / `$` 时允许子串匹配。不加 `/.../` 分隔符，底层使用 Rust JSON Schema 校验器的正则实现 |
| `@format uri` | `format: "uri"` | 非空字符串，保留格式名 | 运行时显式开启 format 校验；已知格式检查字符串内容，未知名称默认忽略。`uri` 不等于“只允许 HTTP URL” |
| `@contentEncoding base64` | `contentEncoding: "base64"` | 非空字符串，保留编码名 | 默认 Draft 2020-12 下只是编码注解；不校验 Base64 合法性、不解码、不转换为图片输入 |
| `@minimum 1` | `minimum: 1` | 按 JSON 数值解析，允许整数、小数或指数形式 | 数值必须 `>= 1`，包含下界 |
| `@maximum 7` | `maximum: 7` | 同上 | 数值必须 `<= 7`，包含上界 |
| `@exclusiveMinimum 0` | `exclusiveMinimum: 0` | 同上；不是布尔值 | 数值必须 `> 0`，不包含下界 |
| `@exclusiveMaximum 1` | `exclusiveMaximum: 1` | 同上；不是布尔值 | 数值必须 `< 1`，不包含上界 |
| `@multipleOf 0.5` | `multipleOf: 0.5` | 按 JSON 数值解析；schema 校验要求值大于 0 | 输入数值必须是 0.5 的整数倍，例如 `0`、`0.5`、`1`；这条规则本身不限制正负 |
| `@asType integer` | 将 `type: "number"` 改为 `type: "integer"` | 只接受字面标签值 `integer`，且当前生成的 schema 必须是 `type: "number"` | 限制值为整数，不四舍五入、不转型；直接写在 `number` 属性或 `type Count = number` 的声明上 |
| `@minItems 1` | `minItems: 1` | 非负整数，按 Rust `u64` 解析 | 数组至少 1 个元素，与每个元素的 `items` 校验同时生效 |
| `@maxItems 5` | `maxItems: 5` | 同上 | 数组最多 5 个元素 |
| `@uniqueItems` / `@uniqueItems true` / `@uniqueItems false` | `uniqueItems: true` / `false` | 省略值为 `true`；有值时只接受小写 `true` / `false` | 按完整 JSON 值比较是否重复；对象也会按内容比较，不是按引用或某个 ID 字段去重 |
| `@oneOf` | 将 `anyOf` 或 `enum` 改为 `oneOf` | 不带值，注释目标的 AST 节点必须直接是联合类型 | 用于联合属性或联合类型别名的声明；不能标在只引用联合别名的属性上 |
| `@additionalProperties false` / `true` | `additionalProperties: false` / `true` | 只识别小写 `false` / `true` | 只可标注直接生成 `type: "object"` 的目标；分别禁止 / 允许未声明字段，允许时不额外限制其值 |
| `@additionalProperties {"type":"string","minLength":1}` | `additionalProperties: {"type":"string","minLength":1}` | 标签值以 `{` 开头时，整行按 JSON 解析，再由 schema 校验器检查合法性 | 未声明字段的值必须是非空字符串；只写一行，不支持多行标签值 |
| `@additionalProperties NonEmptyString` | `additionalProperties: {"$ref":"#/$defs/NonEmptyString"}` | 其他非空值按同文件的类型名解析；名称必须已声明 | 未声明字段按被引用类型校验；该类型会加入可达 `$defs` |

标签值中不要额外加引号：`@title 城市计划` 的值就是 `城市计划`；数值标签写
`@minimum 1`，写成 `@minimum "1"` 会因为是 JSON 字符串而失败。`@minLength` 等长度
标签要求整数文本，不能写 `1.5`。数值标签不接受 `NaN`、`Infinity` 或非 JSON 数值。

### 5.3 标签放置位置与约束组合

JSDoc 绑定到紧邻的下一条属性或类型声明，`/** ... */` 才会解析；`// ...` 和
`/* ... */` 不会生成描述或标签。正文必须放在第一个标签之前：标签之后的普通文字
不会追加到 `description`，也不会续接上一行标签值。

同一个字段上的多个约束同时生效。例如 `number` 配合 `@asType integer`、
`@minimum 1`、`@maximum 7` 表示“1 到 7 之间的整数”，不会为输入补值或截断超界数值。
同一个 JSDoc 内重复标签会报错，即使重复值相同；可以在类型定义和引用属性上分别添加
约束，此时定义约束与引用处的约束都必须满足。

标签贴在数组属性上时，约束的是数组本身。要限制每个元素的字符串长度，应把长度
约束放进元素类型定义，再使用该类型的数组：

```ts
/** @minLength 1 */
type NonEmptyString = string;

export interface InputParams {
  /** @minItems 1 */
  tags: NonEmptyString[];
}
```

对应的关键字段为：

```json
{
  "properties": {
    "tags": {
      "type": "array",
      "items": {"$ref": "#/$defs/NonEmptyString"},
      "minItems": 1
    }
  },
  "$defs": {
    "NonEmptyString": {"type": "string", "minLength": 1}
  }
}
```

这段 JSON 只展示映射片段；完整生成结果还会包含根 `type`、`$schema`、`required`
和默认的 `additionalProperties: false`。

编译器对 `@asType`、`@oneOf`、`@additionalProperties` 会检查标注目标；其他长度、
数值、格式等标签主要检查标签值，并不会替开发者检查目标是否是对应类型。
例如在 `string[]` 属性上写 `@minLength 1`，不会约束数组长度或元素字符串长度。
应使用 `@minItems` 约束数组，使用元素类型定义上的 `@minLength` 约束字符串。

引用别名时，属性处生成的是 `$ref`，因此 `@asType integer` 不能直接贴在
`count: Count` 上，即使 `Count` 的定义是 `number`。应把标签贴在
`type Count = number` 的声明上。`@oneOf` 同理，应贴在
`type Choice = string | number` 的声明上，而不是 `choice: Choice` 上。

`oneOf` 检查的是命中的分支数量，不是“从多个选项里随便选一个”。例如数值 `1`
同时满足 `number` 和整数别名分支，使用 `oneOf` 时会失败；使用 `anyOf` 时合法。
交叉类型 `allOf` 则要求所有约束同时满足，不会自动解决冲突，例如最短长度 5 与
最长长度 3 的组合没有任何合法字符串。

### 5.4 当前解析边界

- 不支持 `any`、`unknown`、`undefined`、`never`、`void`、`bigint`、裸 `object` 等
  TypeScript 类型，也不支持 `Record<K, V>`、元组、映射类型、条件类型等构造。
  `Array<T>` 是当前唯一支持的泛型用法。
- 字段名支持普通标识符和字符串字面量，不支持数值字段名或计算属性名。类型名和同一
  对象中的字段名不能重复；字符串索引签名的键必须是 `string`，同一对象最多一个。
- 命名类型引用不会自动展开为根对象；`type InputParams = Profile` 直接生成 `$ref`，
  不能满足根必须显式解析出 `type: "object"` 的要求。应直接定义根对象，或使用对象
  交叉类型组合。
- 索引签名先生成额外字段 schema；同一对象再声明 `@additionalProperties` 时，标签值
  会覆盖索引签名生成的 `additionalProperties`。这不是 TypeScript 编译器的完整索引
  签名语义，不应依赖它约束已声明字段。
- 交叉合并只做有限的类型识别：识别显式 `type` 和同类 `allOf`，不对 `const`、
  `enum`、`$ref`、联合类型做完整类型推导或可满足性证明。应检查生成的 schema，
  不把编译通过理解为约束一定存在合法输入。
- 未支持的 JSDoc 标签会报错。例如 `@default`、`@examples`、`@description`、
  `@required`、`@nullable` 不受支持；分别通过程序默认值、普通正文、属性 `?` 和
  `T | null` 表达相关含义。
- `format` 不支持任意自定义校验函数；当前 PoA 校验器没有注册应用自定义格式。
  默认 Draft 2020-12 下，已知格式包括 `date`、`date-time`、`time`、`duration`、
  `email`、`idn-email`、`hostname`、`idn-hostname`、`ipv4`、`ipv6`、`uri`、
  `uri-reference`、`uri-template`、`iri`、`iri-reference`、`uuid`、`regex`、
  `json-pointer`、`relative-json-pointer`。未知或拼错的格式名默认忽略，不能当作业务校验。
- 默认 Draft 2020-12 下，`@contentEncoding base64` 甚至允许 `"not base64!"` 通过，
  只要其他字符串约束满足。需要检查编码合法性时应在程序中显式校验；输入仍是原始字符串。
  若用 `@schema` 切换旧方言，部分关键字的行为会随方言变化，不能套用默认方言的解释。

需要直接使用 JSON Schema 时，可以自行编写 `.json` 文件并在 manifest 中声明相对路径；
根 schema 必须显式包含 `"type": "object"`，不要同时放置会覆盖它的根目录
`input_schema.ts`。校验只接受或拒绝原始 JSON 数据，不会补默认值、删除额外字段、
去重数组、转换类型或把编码内容转换成 SDK 的图片输入对象。
