# helper

helper 是不依赖 Agent 状态的纯辅助函数。

以下示例假定已从 `/helper` 入口导入，这些函数也可从根入口 `@brainary/poa-sdk` 导入：

~~~js
import { map_limit, parse_json_reply, shape_of } from "@brainary/poa-sdk/helper";
import { text } from "@brainary/poa-sdk/io";
~~~

### parse_json_reply(raw)

**作用**：从回复文本中提取并解析 JSON，支持纯 JSON、Markdown JSON 代码块和夹在文字中的 JSON 对象或数组。

**输入类型**：raw: string | null | undefined，可直接传入 AgentResult.result。

**返回值类型**：JsonReply<T>，T 默认为 JsonValue。成功时 error 为 null，失败时 value 为 null。

~~~ts
type JsonReply<T = JsonValue> =
  | { value: T; error: null }
  | { value: null; error: string };
~~~

**示例**：

~~~js
const parsed = parse_json_reply('结果：{"count": 2}');
if (parsed.error === null) text(parsed.value);
~~~

### map_limit(items, limit, fn)

**作用**：限制异步映射的最大并发数，并按输入顺序返回结果。

**输入类型**：items: readonly T[]；limit: number，正整数；fn: (item: T, index: number) => R | PromiseLike<R>。三个参数均必填。

**返回值类型**：Promise<Awaited<R>[]>。

任一回调失败时，返回的 Promise 被拒绝。

**示例**：

~~~js
const values = await map_limit(["a", "bb", "ccc"], 2, async (s) => s.length);
text(values);
~~~

### shape_of(value, depth?)

**作用**：返回值的类型结构字符串，便于探测未知结果。对象递归到指定层数，数组描述首元素。

**输入类型**：value: unknown，必填；depth?: number，非负整数，默认 1。

**返回值类型**：string。

**示例**：

~~~js
text(shape_of({ name: "worker", tags: ["a"] }, 2));
// "object{name: string, tags: array(1) of string}"
~~~
