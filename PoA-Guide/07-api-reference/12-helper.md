# helper

helper 是不依赖 Agent 状态的纯辅助函数。

### parseJsonReply({raw})

**作用**：从回复文本中提取并解析第一个 JSON 对象。

**输入类型**：命名参数对象，包含 raw: string。

**返回值类型**：{ value: unknown | null; error: string | null }。

**示例**：

~~~js
const parsed = parseJsonReply({ raw: '结果：{"count": 2}' });
if (!parsed.error) text(parsed.value);
~~~

### mapLimit({items, limit, fn})

**作用**：限制异步映射的最大并发数，并保持输入顺序。

**输入类型**：命名参数对象，包含 items: T[]；limit: number；fn: (item: T, index: number) => Promise<U>。

**返回值类型**：Promise<U[]>。

**示例**：

~~~js
const values = await mapLimit({
  items: ["a", "bb", "ccc"],
  limit: 2,
  fn: async (s) => s.length,
});
text(values);
~~~

### shapeOf({value, depth})

**作用**：返回值的字段和类型形状，便于探测未知结果。

**输入类型**：命名参数对象，包含 value: unknown；depth: number。两个参数均必填。

**返回值类型**：string。

**示例**：

~~~js
text(shapeOf({ value: { name: "worker", count: 3 }, depth: 2 }));
~~~
