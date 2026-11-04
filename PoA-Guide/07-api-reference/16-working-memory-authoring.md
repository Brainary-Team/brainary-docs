# Working Memory 声明与接入（设计预览）

Working Memory 是 PoA 程序侧的结构化状态容器，按声明好的集合存放数据。本页说明怎么声明它、怎么把声明和初始数据交给后端。本页当前只有设计职责，没有可调用的 PoA API。

要用 Working Memory，程序里做四件事：在代码里声明要哪些集合，把声明导出成一份 schema，注册给后端，再灌入初始数据。集合在注册完成之前不存在。

---

## 1. 四步

~~~js
// 节点类型：先声明，再被引用
const Task = wm.node("Task", {
  title:    wm.string(),
  priority: wm.integer().optional(),
});

const Evidence = wm.node("Evidence", {
  note: wm.string(),
});

// 工作记忆本体
class MyWorkingMemory {
  static fields = {
    task_tree:      wm.tree(Task),
    evidence_list:  wm.list(Evidence),
    incident_graph: wm.graph(Task),
    retry_by_code:  wm.map(wm.integer(), wm.integer()),
  };
}

const schema = MyWorkingMemory.to_schema();

const poa = new PoA();
poa.register_schema(Kind.WM, schema);
poa.load_data(Kind.WM, wm_data);
~~~

示例里 `wm.*` 这组构造器叫什么名字、`Kind.WM` 在 JavaScript 里怎么写，都还没有确定，见 §4。

### 1.1 声明

`static fields` 下的每个键是一个集合名，值由一个构造器给出，由构造器决定它属于四类集合里的哪一类 —— graph、tree、list、map 各有一个构造器。字段写在 `static fields` 上而不写在构造函数里，是因为 JavaScript 运行时拿不到构造函数形参的类型。

**顶层只放集合，不放标量字段。** 一个单独的字符串要存进 Working Memory，得先装进某个集合。

### 1.2 导出

`to_schema()` 把 `static fields` 收集成一份完整的声明文档，形状见 §2。它是一个纯函数，不发请求、不碰后端。

### 1.3 注册

`register_schema(Kind.WM, schema)` 把声明送给后端。后端按自己的规则全量校验这份文档，不因为它由 SDK 导出就跳过任何一项；校验不通过时整份拒绝，不会有一部分集合先建起来。

### 1.4 灌数据

`load_data(Kind.WM, wm_data)` 按已登记的声明把初始数据写进去，数据不符合声明时被拒绝。**它必须在注册之后调用。**

---

## 2. 导出的那份声明长什么样

上面那段代码导出的就是下面这份文档。

~~~json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "poa://incident-401/wm.schema.json",
  "x-wm-format": "wm.schema.v1",

  "type": "object",
  "additionalProperties": false,
  "required": ["task_tree", "evidence_list", "incident_graph", "retry_by_code"],
  "properties": {

    "task_tree": {
      "$ref": "https://brainary.dev/wm/v1/tree",
      "x-wm-tree": {
        "nodes": [
          {"name": "Task", "value_type": {"$ref": "#/$defs/Task"}, "child_classes": ["Task"]}
        ]
      }
    },

    "evidence_list": {
      "$ref": "https://brainary.dev/wm/v1/list",
      "x-wm-list": {"item_type": {"$ref": "#/$defs/Evidence"}}
    },

    "incident_graph": {
      "$ref": "https://brainary.dev/wm/v1/graph",
      "x-wm-graph": {
        "nodes": [{"name": "Task", "value_type": {"$ref": "#/$defs/Task"}}],
        "edges": []
      }
    },

    "retry_by_code": {
      "$ref": "https://brainary.dev/wm/v1/map",
      "x-wm-map": {
        "key_type": {"type": "integer"},
        "value_type": {"type": "integer"}
      }
    }
  },

  "$defs": {
    "Task": {
      "type": "object",
      "additionalProperties": false,
      "required": ["title"],
      "properties": {
        "title": {"type": "string"},
        "priority": {"type": "integer"}
      }
    },
    "Evidence": {
      "type": "object",
      "additionalProperties": false,
      "required": ["note"],
      "properties": {
        "note": {"type": "string"}
      }
    }
  }
}
~~~

四条读法：

- 它是一份**合法的 JSON Schema 2020-12 文档**，可以被通用工具解析。
- **顶层键是一个闭集**：必填 `$schema`、`$id`、`x-wm-format`、`type`、`properties`、`required`、`additionalProperties`；可选 `$defs`、`x-wm-constraints`、`title`、`description`。闭集之外的键一律拒绝，不静默忽略。
- **`properties` 下的每个键是一个集合名。** 它属于哪一类集合，由 `$ref` 指向的保留命名空间决定；该类集合自己的参数写在同名的 `x-wm-*` 键里。元素类型要么引用 `$defs` 里的定义，要么直接内联 —— 示例里 map 的键和值就是内联标量。
- **`required` 必须恰好列出 `properties` 下的全部键**，不多不少。声明过的集合在状态里必然存在，空集合写成空内容而不是缺键。

示例里这棵树只有一个节点类别，子节点同属该类别；这个图只声明了节点类别，边的列表是空的。

`x-wm-format` 与 `$schema` 分管两件事：`$schema` 是 JSON Schema 方言，`x-wm-format` 是 Working Memory 对这份文档的额外约定（顶层键闭集、关键词范围、`x-wm-*` 族的形状），它不描述数据形状。

---

## 3. 本页不覆盖的

集合声明好之后，在它上面能做什么，见各集合自己那一页：[BGraph](14-working-memory/02-bgraph.md)、[BTree](14-working-memory/03-btree.md)、[BList 与 BMap](14-working-memory/04-blist-bmap.md)、[memory.snapshot](14-working-memory/07-snapshot.md)，元素与异常类型见 [元素与返回类型](14-working-memory/05-elements.md) 和 [异常](14-working-memory/06-errors.md)。

---

## 4. 当前的接入边界

本页描述的接口还没有实现，下列各项也还没有确定。本页没写到的不等于不存在，不要据本页推定。

- **`wm.*` 这组构造器的名字尚未确定。** 示例里的写法是占位，用来说明形状。
- **`Kind` 在 JavaScript 里的写法尚未确定。** 示例写作 `Kind.WM`，它还有另一个取值 `Kind.LTM`，对应的能力目前没有定义。
- **导出时 `$id` 取什么、由谁给出，尚未确定。** 它必须是一个绝对 URI，示例里那个取值是占位。
- **`PoA` 这个类的构造参数尚未确定。** 它预期持有一个 `agent_id`，而该字段的来源、类型与唯一性规则都还没有定下来；本指南另一处的 `Agent.agent_id` 是宿主派给子 agent 的标识，与此处不是同一个东西。
- **同一个 Kind 能否重复注册、重复注册时的结果如何，尚未确定。**
- **注册完成之后怎么取到集合句柄，尚未确定。**
- **这组调用走哪条宿主通道尚未确定**，相应地，集合从哪一刻起可用、连接由谁发起，也都还没有定。
- **后端承担的职责范围尚未确定。** 已经明确的是 schema 的存储，以及 schema 和对应数据的增删改查；版本、快照、批次、分页这些能力如何安排还没有结论。

---

## 相关文档

- [Working Memory](14-working-memory.md)：这一组文档的入口页与共同约定
- [Episodic Memory](15-episodic-memory.md)：另一份设计预览
