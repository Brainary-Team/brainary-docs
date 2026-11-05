# 声明、登记与句柄

Working Memory 的作者面分三段：用一个 class 声明有哪些集合、把声明登记进 PoA 对象并灌入整份数据、再用声明里的字段读写。四类集合的句柄就是声明里的字段，字段名就是集合名。它的全部状态属于当前 Run，程序拿到的是 owner 侧句柄。

## 声明

集合写在一个 class 的 static fields 里，类上的 to_schema() 把这份声明导出成一份 schema。

~~~js
// 类型构造器 wm 由 SDK 提供，它的名字与各构造器名尚未确定，见下面的当前未确定
const Task = wm.type("Task", { title: wm.string(), priority: wm.integer().optional() });
const Evidence = wm.type("Evidence", { note: wm.string() });

class MyWorkingMemory {
  static fields = {
    incident_graph: wm.graph({
      nodes: [
        { name: "Task", value_type: Task },
        { name: "Evidence", value_type: Evidence },
      ],
      edges: [{ name: "has_evidence", from_class: "Task", to_class: "Evidence" }],
    }),
    task_tree: wm.tree({
      nodes: [
        { name: "Plan", value_type: Task, child_classes: ["Milestone", "Task"] },
        { name: "Milestone", value_type: Task, child_classes: ["Task"] },
        { name: "Task", value_type: Task },
      ],
    }),
    evidence_list: wm.list({ item_type: Evidence }),
    retry_by_status_code: wm.map({ key_type: wm.integer(), value_type: wm.integer() }),
  };
}
~~~

值类型的第一个参数是类型名，它是导出的 schema 里 $defs 的键，自引用类型靠它才能导出成引用。类别名是另一回事，写在 nodes[].name 上：类别名与值类型名是两样东西，两个不同类别可以引用同一个值类型 —— 上面 task_tree 的 Plan、Milestone 与 Task 三个类别就都引 Task 这一个值类型。

四个集合构造器的参数闭集如上：graph 的 nodes 与 edges 都必填，edges 可以是空数组；tree 只声明 nodes，child_classes 挂在每个节点类别上，省略按不接受子节点处理；list 只有 item_type；map 的 key_type 与 value_type 都必填，key_type 只能是标量。

顶层也允许声明非集合的标量字段，例如 goal: wm.string()。

## 登记与灌入

先登记声明，再灌入整份数据，次序不能反。

~~~js
// PoA 与 Kind 由 SDK 提供；agent_id 由宿主提供
const poa = new PoA({ agent_id });
await poa.register_schema(Kind.WM, MyWorkingMemory.to_schema());

const wm_data = {
  incident_graph: {
    nodes: [{ id: "n-task", node_class: "Task", props: { title: "排查移动端 401" } }],
    edges: [],
  },
  task_tree: {
    nodes: [{ id: "plan-1", node_class: "Plan", parent_id: null, props: { title: "排查移动端 401" } }],
  },
  evidence_list: { entries: [{ item_id: "ev-1", value: { note: "移动端复现" } }] },
  retry_by_status_code: { entries: [{ key: 401, value: 2 }] },
};
const mem = await poa.load_data(Kind.WM, wm_data);   // 句柄由哪个调用交出尚未确定，见下面的当前未确定
~~~

register_schema 把一份 schema 登记到 Kind.WM 名下，load_data 把一批数据按已登记的声明存进去。未登记声明就灌入数据一律被拒绝。同一个 kind 只能登记一次，第二次登记也一律被拒，不覆盖也不合并。

整份数据按集合名分组，每组的形状随集合种类而定：graph 是 { nodes, edges }，tree 是 { nodes }（树没有边，父子关系写在节点的 parent_id 上，根的 parent_id 是 null），list 与 map 都是 { entries }，但元素不同形 —— list 的元素是 item_id 与 value，map 的元素是 key 与 value。

## 句柄

句柄就是声明里的那个字段，集合名只写在声明里，调用处不再传集合名。句柄的变量名由作者自取，本指南一律写成 mem。

~~~js
// BNode 由 SDK 提供；mem 是上一段灌入后拿到的句柄
await mem.task_tree.add_child(
  "plan-1",
  new BNode({ id: "task-1", node_class: "Task", props: { title: "核对 token" } }),
);
const childPage = await mem.task_tree.children("plan-1");
await mem.evidence_list.append("ev-2", { note: "网关时间快 31 秒" });
await mem.retry_by_status_code.put(503, 1, "upsert");
~~~

字段的种类在声明里已经定下：声明为 graph 的字段是 [BGraph](02-bgraph.md)，tree 是 [BTree](03-btree.md)，list 是 [BList](04-blist-bmap.md)，map 是 [BMap](04-blist-bmap.md)。句柄把集合名作为协议字段 collection 注入每次请求，业务代码不重复传入集合名。

句柄对象上只挂声明过的集合字段。访问未声明的名字抛 [WmUsageError](06-errors.md)，不会得到 undefined —— 集合名拼错时报的是这个名字没有声明，而不是在调用方法时拿到一个原生 TypeError。

顶层的标量字段在本阶段没有元素级的读写入口，只随整份数据一起灌入，句柄上没有单独读写它们的方法。

## 作用域与实例

Run 是 Working Memory 的作用域单位，由一个 PoA cell 发起，覆盖该 cell 沿派发链派出的全部子 agent。它与 cell、turn、thread 都不等同，同一个 thread 可以参与多个 Run，多个 Run 并发活跃是预期状态，各持独立的作用域与独立实例。

一个 Run 对应一个实例。实例是 id 唯一性的作用域：节点 id、边 id、list 的 item_id 在整个实例内唯一，单个集合不是唯一性边界。

Run 的生命周期由宿主管理。Run 结束没有可依赖的宿主信号，WM 不自行检测 Run 结束，也不回收内存，清理仍需宿主的生命周期机制。宿主关闭前先停止接收新的变更、等待在途提交获得确定结果，再执行清理。

## 可用范围

本页只覆盖一条执行路径：app-server 的 thread/codeMode/exec 启动的 PoA cell。执行 PoA 另有一条路径 —— 注册为 poa 工具后由模型在普通 turn 里执行一个完整程序 —— 它的作用域锚点与身份派生都与本页这条不同，当前单独待裁，本页不覆盖它，也不要默认两条同构。子 agent 的 Working Memory 入口同样单独待裁。这几处都不要读成「确定没有入口」，它们是尚未裁定，不是已有结论。

可选工具组 memories 和 [tools](../06-brainary-tool.md) 枚举里的 memory 工具类别是模型可见的工具，与本页的 Working Memory 同名不同物。

WM schema 与 input_schema 是两份不同的 schema，用途与可见性都不同：后者注册为 poa 工具时成为对模型可见的参数说明，WM schema 对模型不可见。

## 持久化

当前使用内存存储。进程退出后运行态、快照和变更历史都会丢失。要跨进程保留数据，保存成功导出的快照文件，见 [memory.snapshot](07-snapshot.md)。

## 集合名

集合名就是声明里的字段名。四类集合共用一个命名空间，不得重名。集合名允许非 ASCII，含连字符、空格或中文都可用，调用时写 schema 中的原始名称。本文档对函数和方法的 snake_case 约定不约束集合名。

## 默认序与结果规模

四类集合的 query 和 [BGraph](02-bgraph.md) 的 neighbors 一次调用返回全部命中项。[BPage](05-elements.md) 的 next_cursor 是一个可选位，本阶段不会出现。

默认序由服务端固定：graph 按元素 id 升序，tree 按 id 升序，list 按 position 升序，map 按规范化 key 升序。map 的整数 key 按数值比较，9 排在 401 之前。本阶段请在客户端自行排序。

limit 省略时返回全部命中项，给出时按默认序取前 N 项。limit 取 0、负数、非整数或超过上限都返回 INVALID_ARGUMENT。

limit 取的是默认序的前 N 项。要「按某个字段排序的前 N 项」，必须省略 limit、取回全部后自行排序再截断。写 limit: 10 再在客户端排序，拿到的是 id 最小的 10 项按该字段重排，它不报错，但不是按该字段排序的前 10 项。

命中项数的上限是 10000。它量的是命中数而不是返回数，所以给 limit 不能绕过它：limit 为 10 而 filter 命中 5 万项时照样被拒。超限是确定性拒绝，返回 RESULT_TOO_LARGE，处理方式是收窄 filter 或先减少数据。

请求的大小另有两道闸，与上面那道各管一头。请求文本有一个 UTF-8 字节上限，按即将发送的那一份文本的实际编码字节数计；SDK 在发出前命中时同步抛 [WmUsageError](06-errors.md)，请求根本没上线。传输帧上限由宿主和传输层决定，当前实现约为 64 MiB；超出帧上限是请求发出之后的传输层失败，抛 [WmTransportError](06-errors.md)，它没有错误码，以 Promise 拒绝的形式交付。

三道闸互不替代，中间有一条缝要知道：命中项数没超过 10000 而回传的字节数超了，在错误码面没有出口 —— 这一类由传输层异常承接，拿到的是 WmTransportError 而不是 RESULT_TOO_LARGE。读大对象时应当先靠 filter 把范围收窄，不要只盯着项数。

## 本阶段不提供

把多个写入合并成一次原子提交属下一阶段。

## 完整示例

下面这段串起一次完整动线：声明、登记与灌入、按字段读写、建一条边、取回全部命中项后在客户端排序。

~~~js
// PoA、Kind、类型构造器 wm、BNode 与 BEdge 由 SDK 提供；agent_id 由宿主提供
const Task = wm.type("Task", { title: wm.string(), priority: wm.integer().optional() });
const Evidence = wm.type("Evidence", { note: wm.string() });

class MyWorkingMemory {
  static fields = {
    incident_graph: wm.graph({
      nodes: [
        { name: "Task", value_type: Task },
        { name: "Evidence", value_type: Evidence },
      ],
      edges: [{ name: "has_evidence", from_class: "Task", to_class: "Evidence" }],
    }),
    evidence_list: wm.list({ item_type: Evidence }),
  };
}

const poa = new PoA({ agent_id });
await poa.register_schema(Kind.WM, MyWorkingMemory.to_schema());
const mem = await poa.load_data(Kind.WM, {
  incident_graph: { nodes: [], edges: [] },
  evidence_list: { entries: [] },
});                                            // 句柄由哪个调用交出尚未确定

// 边不能先于端点存在：两个端点各写一次，边再写一次，三次独立提交
await mem.incident_graph.add_node(
  new BNode({ id: "node-1", node_class: "Task", props: { title: "排查移动端 401" } }),
);
await mem.incident_graph.add_node(
  new BNode({ id: "node-2", node_class: "Evidence", props: { note: "网关时间快 31 秒" } }),
);
await mem.incident_graph.add_edge(
  new BEdge({ id: "edge-1", edge_class: "has_evidence", from_id: "node-1", to_id: "node-2", props: {} }),
);

const appended = await mem.evidence_list.append("ev-1", { note: "网关时间快 31 秒" });
text(appended.position);

const page = await mem.incident_graph.query({ target: "nodes" });
const titles = page.items.map((item) => item.id).sort();
text(titles);
~~~

上面每个 await 都是一次独立提交，任何一次失败都以异常抛出，前面几次已经落地的写入不会回滚；中间状态对并发读可见。最后的 query 不带 limit，取回全部命中项之后在客户端排序。

## 当前未确定

集合声明写成什么形状尚未确定。本页写的是在一个 class 的 static fields 里用类型构造器声明，另一个方向是只写 TypeScript 的类型标注、由构建期从类型里提取出 schema。两种形状在后面两节上没有差别：句柄仍然是声明里的那个字段，字段名仍然是集合名，登记与灌入的两个调用也不变。

哪一个调用交出句柄尚未确定。当前倾向是由 load_data 交出，上面的示例按这个写法示意，它的签名还没有定稿，不要当作稳定的调用面。同一个 Run 内不重新灌入就取回句柄的入口同样未确定。

类型构造器的名字与它们所在的命名空间尚未确定，示例里的 wm.type、wm.string、wm.integer、wm.graph、wm.tree、wm.list 和 wm.map 都只是示意写法。

agent_id 的类型、来源与唯一性尚未确定，示例里的 new PoA({ agent_id }) 只说明它是 PoA 上唯一要给的东西。

整份灌入时 list 的 position 从哪里来尚未确定：由调用方在 entries 的元素里提供，还是灌入时按数组顺序从 0 重排。平时这个值由服务端分配，上面的示例没有写它。

声明了的集合在灌入的那份数据里允不允许缺省尚未确定。当前导出的 schema 把全部集合都算必填，所以上面登记与灌入那段把声明的四个集合逐个写了出来，暂时不要省略其中任何一个。

顶层标量字段怎么读写尚未确定。本阶段它们只随整份灌入的数据一起写入，没有元素级的入口。

tree、list 与 map 的 query 是否仍需要 target 尚未确定。graph 侧按节点与边区分是确定的，它的 target 取 nodes 或 edges。

四类集合的方法签名已稳定，参数个数、位置与名字可以照着写，但写方法仍有一些参数细节没有定下来。query 统一取单个 options 对象，可含 filter 和 limit；filter 是按 id、key、类别名或已声明属性的精确匹配条件。四类集合的默认顺序都已确定，见上面的默认序与结果规模。

## 相关文档

- [BGraph](02-bgraph.md)：图的节点与边读写、一跳邻接和查询。
- [BTree](03-btree.md)：树的读写、约束模型和层级查询。
- [BList 与 BMap](04-blist-bmap.md)：有序项追加与键值读写。
- [BNode、BEdge 与读写返回类型](05-elements.md)：构造类型、提交结果和读结果体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
- [memory.snapshot](07-snapshot.md)：快照创建、列举和只读查询，属下一阶段。
