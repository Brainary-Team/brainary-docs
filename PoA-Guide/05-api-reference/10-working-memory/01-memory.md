# 声明、登记与句柄

Working Memory 在程序侧分三步使用：用一个 class 声明有哪些集合，把声明登记进 PoA 对象并载入整份数据，再用声明里的字段读写。四类集合的句柄就是声明里的字段，字段名就是集合名。句柄上的读写都在 SDK 本地完成，整份文档在提交点写回一次。程序获得的是 owner 侧句柄。

## 声明

集合写成一个 class 上的字段类型标注：字段名就是集合名，字段的类型说明它是哪一类集合，以及里面有哪些类别和值类型。schema 由打包期从这些类型里提取，程序运行时这个 class 上没有任何成员。

~~~typescript
// 标记类型 Graph、Tree、List、KvMap 与具名标量 Int 由 SDK 提供
interface Task {
  /** @minLength 1 */
  title: string;
  /** @asType integer
   * @minimum 0
   */
  priority?: number;
}

interface Evidence {
  note: string;
}

class MyWorkingMemory {
  // 第一个类型参数：类别名 → 该类别的值类型
  // 第二个类型参数：graph 是“边类别 → [起点类别, 终点类别]”，tree 是“类别 → 允许的子类别”
  incident_graph: Graph<
    { Task: Task; Evidence: Evidence },
    { has_evidence: ["Task", "Evidence"] }
  >;

  task_tree: Tree<
    { Plan: Task; Milestone: Task; Task: Task },
    { Plan: ["Milestone", "Task"]; Milestone: ["Task"]; Task: [] }
  >;

  evidence_list: List<Evidence>;

  retry_by_status_code: KvMap<Int, Int>;

  goal: string;
}
~~~

值类型用 interface 写，字段上的标量精度与取值范围用 JSDoc 标注，写法与 input_schema 相同。类别名是第一个类型参数里的键：类别名与值类型是两回事，多个类别可以引用同一个值类型，例如上面 task_tree 的 Plan、Milestone 与 Task 三个类别都引用 Task 这一个值类型。

两个类型参数的含义按集合种类定：graph 的第二个参数是边类别到起点与终点类别的映射；tree 的第二个参数是每个类别允许的子类别，写成空数组表示这个类别不接受子节点；list 只有一个参数，就是项的值类型；map 的两个参数分别是 key 与 value 的类型，key 只能是标量。

顶层也允许声明非集合的标量字段，例如 goal: string。

## 登记与载入

先登记声明，再载入整份数据，顺序不能颠倒。

~~~typescript
// PoA 与 Kind 由 SDK 提供；agent_id 由宿主提供
const poa = new PoA({ agent_id });
// schema 由打包期从上面那些类型提取
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
  goal: "排查移动端 401",
};
const mem = await poa.load_data<MyWorkingMemory>(Kind.WM, wm_data);
~~~

register_schema 把一份 schema 登记到 Kind.WM 名下，load_data 把一批数据按已登记的声明写入。未登记声明就载入数据一律被拒绝。同一个 kind 只能登记一次，第二次登记同样被拒绝，不覆盖也不合并；清掉数据也不解除登记。

数据这一侧相反，可以反复写：再调一次 load_data 就整份替换上一次，所以没有单独的更新调用。句柄上的写方法也走这条路 —— 它们只改本地影子状态，到提交点把整份文档写回，见下面的提交点。

数据必须带齐声明里的全部集合。声明把每个没带 ? 的字段都算必填，缺任何一个都会在发出之前被拒绝，所以上面那段把四个集合和那个标量字段逐个写了出来，空集合也要写出空容器。

数据与声明的符合性由 SDK 在发出之前检查。WM 不做这一层，它只做四项输入卫生：合法 UTF-8、可解析 JSON、最外层是对象、不超过字节上限。code mode 里绕过 SDK 直接调原生工具时这一层检查就没有了，不符声明的数据会被原样收下且不报错。

整份数据按集合名分组，每组的形状随集合种类而定：graph 是 { nodes, edges }，tree 是 { nodes }（树没有边，父子关系写在节点的 parent_id 上，根的 parent_id 是 null），list 与 map 都是 { entries }，但元素不同形：list 的元素是 item_id 与 value，map 的元素是 key 与 value。顶层的标量字段直接给值，上面的 goal 就是这样写的。

## 句柄

句柄就是声明里的那个字段，集合名只写在声明里，调用处不再传集合名。句柄的变量名由作者自取，本指南一律写成 mem。

~~~js
// BNode 由 SDK 提供；mem 是上一段载入后得到的句柄
await mem.task_tree.add_child(
  "plan-1",
  new BNode({ id: "task-1", node_class: "Task", props: { title: "核对 token" } }),
);
const childPage = await mem.task_tree.children("plan-1");
await mem.evidence_list.append("ev-2", { note: "网关时间快 31 秒" });
await mem.retry_by_status_code.put(503, 1, "upsert");
~~~

字段的种类在声明里已经确定：声明为 graph 的字段是 [BGraph](02-bgraph.md)，tree 是 [BTree](03-btree.md)，list 是 [BList](04-blist-bmap.md)，map 是 [BMap](04-blist-bmap.md)。集合名绑在句柄上，业务代码不重复传入它。它只用于 SDK 本地寻址与分派，不是工具参数 —— WM 侧那四个整份接口没有 collection 这一位。

句柄对象上只有声明过的集合字段。访问未声明的名字抛 [WmUsageError](06-errors.md)，不会得到 undefined：集合名拼错时报的是这个名字没有声明，而不是在调用方法时得到一个原生 TypeError。

顶层的标量字段没有元素级的读写入口，只随整份数据一起载入，句柄上没有单独读写它们的方法。

## 作用域与实例

Run 是 Working Memory 的作用域单位，由一个 PoA cell 发起。它与 cell、turn、thread 都不等同，同一个 thread 可以参与多个 Run。

一个 Run 对应一个实例。实例是 id 唯一性的作用域，但四类 id 的作用域不是同一个：节点 id 在该实例的全部 graph 与 tree 之间共用一个空间，图节点与树节点因此不能重用同一个 id；边 id 是另一个实例级空间；list 的 item_id 只在该 list 内唯一；map 的 key 只在该 map 内唯一。

存储按 thread、turn 和 kind 分桶，还不是一个 Run 一桶 —— 宿主交给工具的是 turn 的 id，拿不到 Run 级标识。下面两条限制都由此而来。收口要靠宿主交出 Run 级标识，不要自己加参数绕开它，那修不了桶的身份。

同一个模型 turn 里起两个 Run 会共用一份状态。第二个 Run 的登记被当作重复登记拒掉；它第一次 load_data 会覆盖另一个 Run 的文档，而且报告的是“覆盖了一份已有文档”，尽管它自己从没写过；它即使没登记过也能读、能清那份文档，因为准入判的是这个桶里有没有声明，而声明是另一个 Run 放进去的；两个 Run 属于不同包时，这就是把一个包的文档交给了另一个包。

turn 结束会连数据一起丢掉。turn stop 与 turn abort 都丢弃该桶的条目，声明和数据一起丢。一个在自己的 turn 停止之后仍在派发的 cell 会在运行中失去数据：它下一次操作因没有声明被拒，照提示重新登记只会得到一个空槽，再读就是一份普通的空记忆，而不是一次可以辨认的丢失。

子 agent 不在这条作用域里。设计上 Run 要覆盖该 cell 沿派发链派出的全部子 agent，当前的分桶做不到；子 agent 侧的入口不在本页范围内，见下面的可用范围。

## 可用范围

本页只覆盖一条执行路径：app-server 的 thread/codeMode/exec 启动的 PoA cell。注册为 poa 工具后由模型在普通 turn 里执行一个完整程序是另一条路径，它的作用域锚点与身份派生都与本页这条不同，本页不覆盖，也不要默认两条同构；子 agent 的 Working Memory 入口也不在本页范围内。

可选工具组 memories 和 [tools](../06-brainary-tool.md) 枚举里的 memory 工具类别是模型可见的工具，与本页的 Working Memory 同名不同物。

WM schema 与 input_schema 是两份不同的 schema，用途与可见性都不同：后者注册为 poa 工具时成为对模型可见的参数说明，WM schema 对模型不可见。

## 持久化

当前使用内存存储。进程退出后运行态和变更历史都会丢失，跨进程保留数据要靠导出快照，见 [memory.snapshot](07-snapshot.md)。

## 集合名

集合名就是声明里的字段名。四类集合共用一个命名空间，不得重名。中文这类非 ASCII 字母可以直接用作字段名；含连字符或空格的名字不是合法标识符。本文档对函数和方法的 snake_case 约定不约束集合名。

## 默认序与结果规模

四类集合的 query 和 [BGraph](02-bgraph.md) 的 neighbors 一次调用返回全部命中项。[BPage](05-elements.md) 的 next_cursor 是一个可选字段，当前版本不会出现。

读和排序都在 SDK 本地完成。默认序是固定的，不由调用方选：graph 按元素 id 升序，tree 按 id 升序，list 按 position 升序，map 按规范化 key 升序。graph 与 tree 的 id 按原始 Unicode 标量序列比较，graph 的边按边 id；map 的整数 key 按数值比较，9 排在 401 之前。邻接读取也有稳定序。需要其他顺序时在客户端自行排序。

limit 省略时返回全部命中项，给出时按默认序取前 N 项。合法范围是 1 到 10000：取 0、负数、非整数或超过 10000 都返回 INVALID_ARGUMENT，不会被截到边界上。

limit 取的是默认序的前 N 项。要“按某个字段排序的前 N 项”，必须省略 limit、取回全部后自行排序再截断。写 limit: 10 再在客户端排序，得到的是 id 最小的 10 项按该字段重排，它不报错，但不是按该字段排序的前 10 项。

命中项数的上限是 10000。它针对的是命中项数而不是返回项数，所以指定 limit 不能绕过它：limit 为 10 而 filter 命中 5 万项时同样被拒绝。超限是确定性拒绝，返回 RESULT_TOO_LARGE，处理方式是收窄 filter 或先减少数据。

整份文档另有一个字节上限，与上面的命中项数上限各管一侧：提交时那份文档的 JSON 文本超过上限会被拒绝，缺省 5 MiB，可由运维配置，返回 DATA_TOO_LARGE，见 [WmRequestError](06-errors.md)。它量的是那一个字符串自身的 UTF-8 字节数，不是 JS 的 String.length。传输帧上限由宿主和传输层决定，当前实现约为 64 MiB，远在 5 MiB 的文档上限之上，所以帧超限这一支实际碰不到。

两项上限互不替代。命中项数在 10000 以内而读回来的对象很大时 RESULT_TOO_LARGE 不会触发，字节一侧也没有对应的错误码 —— 那一侧的闸门只在提交时才有。读取大对象时应当先用 filter 收窄范围，不能只按项数估算。

## 提交点

句柄上的写方法不各自跨进程。它们只改本地影子状态，到提交点才把整份文档写回一次。

提交点是一个 batch 的结束：batch 里的 N 次写只改影子状态，batch 结束时整份写回一次。不放进 batch 的单发调用各自成一个隐式 batch，所以写一处就写回一次 —— 把本该一个 batch 的写拆成多个单发，就把一次整份提交变成了 N 次，并发读者会看到中间状态。

batch 不是原子的，也不回滚：回调里每个操作仍逐个检查，不放行批内的非法中间态；本地失败不会自动撤回已经改掉的影子状态；提交之前崩溃时这一批写全部丢失。

## 不提供的能力

当前不提供把多个写入合并成一次原子提交 —— batch 给的是一个提交边界，不是原子性。BatchBuilder 这个登记器不提供。

## 完整示例

下面的示例覆盖一次完整流程：声明、登记与载入、按字段读写、新建一条边、取回全部命中项后在客户端排序。

~~~typescript
// PoA、Kind、标记类型、BNode 与 BEdge 由 SDK 提供；agent_id 由宿主提供
interface Task { title: string }
interface Evidence { note: string }

class MyWorkingMemory {
  incident_graph: Graph<
    { Task: Task; Evidence: Evidence },
    { has_evidence: ["Task", "Evidence"] }
  >;
  evidence_list: List<Evidence>;
}

const poa = new PoA({ agent_id });
await poa.register_schema(Kind.WM, MyWorkingMemory.to_schema());
const mem = await poa.load_data<MyWorkingMemory>(Kind.WM, {
  incident_graph: { nodes: [], edges: [] },
  evidence_list: { entries: [] },
});

// 边不能先于端点存在：两个端点先写，边后写
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

const page = await mem.incident_graph.query({ target: "nodes", filter: { node_class: "Task" } });
const titles = page.items.map((item) => item.props.title).sort();
text(titles);
~~~

上面的写都没放进 batch，所以每个 await 各自成一个隐式 batch、各自整份写回一次；任何一次失败都以异常抛出，前面几次写回的内容不会回滚，中间状态对并发读可见。最后的 query 在本地执行，不带 limit，取回全部命中项之后在客户端按 title 排序；默认序只给 id 升序。

## 相关文档

- [BGraph](02-bgraph.md)：图的节点与边读写、一跳邻接和查询。
- [BTree](03-btree.md)：树的读写、约束模型和层级查询。
- [BList 与 BMap](04-blist-bmap.md)：有序项追加与键值读写。
- [BNode、BEdge 与读写返回类型](05-elements.md)：构造类型、提交结果和读结果体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
- [memory.snapshot](07-snapshot.md)：快照创建、列举和只读查询。
