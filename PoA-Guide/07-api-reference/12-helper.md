# helper

helper 是不依赖 Agent 状态的纯辅助函数。

### parse_json_reply(raw)

**作用**：从回复文本中提取并解析第一个 JSON 对象。

**输入类型**：raw: string。

**返回值类型**：{ value: unknown | null; error: string | null }。

**示例**：

~~~js
const parsed = parse_json_reply('结果：{"count": 2}');
if (!parsed.error) text(parsed.value);
~~~

### map_limit(items, limit, fn)

**作用**：限制异步映射的最大并发数，并保持输入顺序。

**输入类型**：items: T[]；limit: number；fn: (item: T, index: number) => Promise<U>。

**返回值类型**：Promise<U[]>。

**示例**：

~~~js
const values = await map_limit(["a", "bb", "ccc"], 2, async (s) => s.length);
text(values);
~~~

### shape_of(value, depth)

**作用**：返回值的字段和类型形状，便于探测未知结果。

**输入类型**：value: unknown；depth: number。两个参数均必填。

**返回值类型**：string。

**示例**：

~~~js
text(shape_of({ name: "worker", count: 3 }, 2));
~~~
