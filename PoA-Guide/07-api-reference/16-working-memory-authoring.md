# Working Memory 声明与接入（设计预览）

本页说明一种新的 Working Memory 接入方式：schema 由程序代码声明，在运行时注册，不再随 `.poa` 包携带一份独立的 JSON 文件。本页当前只有设计职责，没有可调用的 PoA API；现行方式见 [memory](14-working-memory/01-memory.md)。

---

## 1. 与现行方式的差别

| | 现行 | 本页这一版 |
| --- | --- | --- |
| schema 写在哪 | 包里的一份 JSON 文件 | 程序源码里的一个 class |
| 谁产出它 | 作者手写那份 JSON | 由 class 导出：`MyWorkingMemory.to_schema()` |
| 什么时候送达 | 业务 JS 执行之前，宿主读包取得 | 业务 JS 执行之中，程序调 `register_schema` 送出 |
| 入口对象 | 宿主注入的全局 `memory` | 程序自己构造的 `PoA` 实例 |

两处不变：声明文档的格式不变，仍是 §3 那一份；这组调用与后端之间不走 `tools.` 分发路径，是直接交互。

一处连带变化要先说清楚：**`memory` 不再是「业务 JS 执行之前就已连接」的全局**。schema 在程序运行当中才送达，集合在注册完成之前不存在。[memory](14-working-memory/01-memory.md) 的「注入与可用范围」一节、以及 [Working Memory 入口页](14-working-memory.md)开头那句「由宿主在业务 JS 执行前注入……程序拿到的是已连接的 owner 侧句柄」，描述的都是现行方式，在本页这一版里不成立。

---

## 2. 四步

~~~js
// 节点类型：先声明，再被引用
const Task = wm.node({
  title:    wm.string(),
  priority: wm.integer().optional(),
});

const Evidence = wm.node({
  note: wm.string(),
});

// 工作记忆本体：每个集合声明在一个显式的字段位上
class MyWorkingMemory {
  static fields = {
    task_tree:     wm.tree(Task),
    evidence_list: wm.list(Evidence),
    incident_graph: wm.graph(Task),
    retry_by_code: wm.map(wm.integer(), wm.integer()),
  };
}

const schema = MyWorkingMemory.to_schema();

const poa = new PoA();
poa.register_schema(Kind.WM, schema);
poa.load_data(Kind.WM, wm_data);
~~~

声明、导出、注册、灌数据。下面逐步说明。

示例里 `wm.*` 这组构造器叫什么名字、`Kind.WM` 在 JavaScript 里怎么写，都还没有确定，见 §5。

### 2.1 声明

`static fields` 下的每个键是一个集合名，值由一个构造器给出，由构造器决定它属于四类集合里的哪一类。

**四类集合各有一个构造器**，对应 graph、tree、list 和 map。类型信息必须落在一个显式的声明位上：普通 JavaScript class 在运行时不携带字段的类型信息，构造函数的形参名拿不到类型，所以字段不写在构造函数里。

**顶层只放集合，不放标量字段。** 声明文档的 `properties` 是集合声明的唯一位置，每个键都是一个集合名，没有「标量字段」这一类。一个单独的字符串要存进 Working Memory，得装进某个集合。

### 2.2 导出

`to_schema()` 把 `static fields` 收集成一份完整的声明文档，形状见 §3。它是一个纯函数，不发请求、不碰后端。

### 2.3 注册

`register_schema(Kind.WM, schema)` 把声明送给后端。

**后端按自己的规则全量校验这份文档，不因为它由 SDK 导出就跳过任何一项。** 这不是冗余：导出的那份文档是普通数据，程序在送出之前可以任意改写它，所以它到达后端时是不可信输入。校验不通过时整份拒绝，不会有一部分集合先建起来。

### 2.4 灌数据

`load_data(Kind.WM, wm_data)` 按已登记的声明把初始数据写进去。数据不符合声明时被拒绝。

### 2.5 顺序

注册必须在灌数据之前，这个顺序不是建议。同一个 Kind 能否重复注册、重复注册时的结果如何，尚未确定。

---

## 3. 导出的那份声明长什么样

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
        "edges": [{"name": "blocks", "from_class": "Task", "to_class": "Task"}]
      }
    },

    "retry_by_code": {
      "$ref": "https://brainary.dev/wm/v1/map",
      "x-wm-map": {
        "key_type": {"type": "integer"},
        "value_type": {"type": "integer", "minimum": 0}
      }
    }
  },

  "$defs": {
    "Task": {
      "type": "object",
      "additionalProperties": false,
      "required": ["title"],
      "properties": {
        "title": {"type": "string", "minLength": 1},
        "priority": {"type": "integer", "minimum": 0, "maximum": 9}
      }
    },
    "Evidence": {
      "type": "object",
      "additionalProperties": false,
      "required": ["note"],
      "properties": {
        "note": {"type": "string", "minLength": 1}
      }
    }
  }
}
~~~

四条读法：

- 它是一份**合法的 JSON Schema 2020-12 文档**，可以被通用工具解析。
- **顶层键是一个闭集**：必填 `$schema`、`$id`、`x-wm-format`、`type`、`properties`、`required`、`additionalProperties`；可选 `$defs`、`x-wm-constraints`、`title`、`description`。闭集之外的键一律拒绝，不静默忽略。
- **`properties` 下的每个键是一个集合名。** 它属于哪一类集合，由 `$ref` 指向的保留命名空间决定；该类集合自己的参数写在同名的 `x-wm-*` 键里。元素类型放 `$defs`，由 `#/$defs/` 本地引用。
- **`required` 必须恰好列出 `properties` 下的全部键**，不多不少。声明过的集合在状态里必然存在，空集合写成空内容而不是缺键。

`x-wm-format` 与 `$schema` 分管两件事：`$schema` 是 JSON Schema 方言，`x-wm-format` 是 Working Memory 对这份文档的额外约定（顶层键闭集、关键词范围、`x-wm-*` 族的形状）。它不描述数据形状。

---

## 4. 这一版不改什么

- **声明文档的格式不变。** §3 那一份与现行规定同一套，这一版换的是谁产出它、怎么送达。
- **四类集合的读写方法不变。** [BGraph](14-working-memory/02-bgraph.md)、[BTree](14-working-memory/03-btree.md)、[BList 与 BMap](14-working-memory/04-blist-bmap.md) 和 [memory.snapshot](14-working-memory/07-snapshot.md) 描述的方法面不在本页射程内。
- **元素与异常类型不变。** 见 [元素与返回类型](14-working-memory/05-elements.md) 和 [异常](14-working-memory/06-errors.md)。

---

## 5. 当前的接入边界

本页描述的接口还没有实现，下列各项也还没有确定。遇到本页未写明的取值规则不要自行推定。

- **`wm.*` 这组构造器的名字尚未确定。** 示例里的写法是占位，用来说明形状。
- **`Kind` 在 JavaScript 里的写法尚未确定。** 示例写作 `Kind.WM`，它还有另一个取值 `Kind.LTM`，对应的能力目前没有定义，调用它应当得到一个明确的拒绝而不是被静默接受。
- **`PoA` 这个类的构造参数尚未确定。** 它预期持有一个 `agent_id`，而该字段的来源、类型与唯一性规则都还没有定下来；本指南另一处的 `Agent.agent_id` 是宿主派给子 agent 的标识，与此处不是同一个东西。
- **注册完成之后怎么取用集合，尚未确定。** 现行的按名取句柄（`memory.tree("task_tree")`）在这一版里不保留，替代形态还没有定。
- **这组调用走哪条宿主通道尚未确定**，相应地，`memory` 从哪一刻起可用、连接由谁发起，也都还没有定。
- **后端在这一版里承担的职责范围尚未确定。** 已经明确的是 schema 的存储，以及 schema 和对应数据的增删改查；版本、快照、批次、分页这些现行方法面上的能力在这一版里如何安排，还没有结论，不要据本页推定它们被取消。

---

## 相关文档

- [memory](14-working-memory/01-memory.md)：现行的接入方式与作用域规则
- [Working Memory](14-working-memory.md)：这一组文档的入口页与共同约定
- [Episodic Memory](15-episodic-memory.md)：另一份设计预览
