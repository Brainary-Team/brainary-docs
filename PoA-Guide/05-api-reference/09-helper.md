# helper

以下示例从 `/helper` 入口导入，也可以从根入口 `@brainary/poa-sdk` 导入：

~~~js
import { map_limit, parse_json_reply, shape_of } from "@brainary/poa-sdk/helper";
import { text } from "@brainary/poa-sdk/io";
~~~

### parse_json_reply(raw)

**作用**：从 agent 的回复中解析 JSON。依次尝试：

1. 把整段回复解析为 JSON。
2. 解析回复中第一个 Markdown 代码块（` ```json ` 或 ` ``` `）的内容。
3. 解析回复中第一个可以完整解析的 JSON 对象或数组。
4. 解析从第一个 `{` 到最后一个 `}` 之间的内容。

**输入类型**：raw: string | null | undefined，可以直接传入 [AgentResult](01-agent.md#agentresult).result。

**返回值类型**：JsonReply<T>，T 默认为 JsonValue。成功时 error 为 null；失败时 value 为 null，error 为错误信息。raw 不是字符串时返回失败。

~~~ts
type JsonReply<T = JsonValue> =
  | { value: T; error: null }
  | { value: null; error: string };
~~~

T 只声明类型，不校验解析结果的结构。

**示例**：

~~~js
const parsed = parse_json_reply('结果如下：{"count": 2}');
if (parsed.error === null) text(parsed.value);  // {"count":2}
~~~

### map_limit(items, limit, fn)

**作用**：对 items 的每一项调用 fn，最多 limit 个调用同时进行，结果按 items 的顺序返回。

**输入类型**：items: readonly T[]；limit: number，正整数；fn: (item: T, index: number) => R | PromiseLike<R>。三个参数均必填。

**返回值类型**：Promise<Awaited<R>[]>。

**异常**：

- limit 不是正整数：抛出 RangeError。
- 任一 fn 抛出异常：返回的 Promise 以该异常拒绝，尚未开始的项不再处理。

**示例**：

~~~js
const lengths = await map_limit(["a", "bb", "ccc"], 2, async (s) => s.length);
text(lengths);  // [1,2,3]
~~~

### shape_of(value, depth?)

**作用**：返回值的结构描述，用于查看未知数据的结构。对象列出各字段，数组只描述第一个元素，展开到 depth 层。

**输入类型**：value: unknown，必填；depth?: number，非负整数，默认 1。

**返回值类型**：string。

**异常**：depth 不是非负整数时抛出 RangeError。

**示例**：

~~~js
text(shape_of({ name: "worker", tags: ["a"] }, 2));
// object{name: string, tags: array(1) of string}
~~~
