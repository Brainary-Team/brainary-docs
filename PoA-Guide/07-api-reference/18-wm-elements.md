# BNode、BEdge、BatchBuilder 与读写返回类型

本页描述 Working Memory 的构造类型、批次构建器和读写返回类型。构造类型用于写入 [BGraph](15-bgraph.md) 与 [BTree](16-btree.md)，返回类型由四类集合的读写方法和 [memory](14-memory.md) 的 batch 返回。下面的示例以 incidentGraph、taskTree、evidenceList 和 retryByStatusCodeMap 指代四类集合的句柄，它们由 memory 上同名的访问器取得。

### BNode

**作用**：构造一个图节点或树节点。

**类型定义**：

~~~ts
interface BNode {
  id: string;
  node_class: string;
  parent_id?: string | null;
  properties: object;
}
~~~

实例由 new BNode({ id, node_class, properties }) 构造。普通 JS 对象表达不了类别归属，节点必须经该包装类构造。

id 由调用方提供，必须是非空字符串，在整个实例内唯一，图的节点与树的节点不能重用同一个 id。WM 不解析 id 的内部结构，也不做大小写或空白归一，T1 与 t1 是两个 id。

node_class 取所选集合内声明的节点类别名，不取值类型名，两个不同类别可以引用同一个值类型。类别由集合名和类别名共同定位，别的集合里存在同名类别不会使本次写入有效。

properties 的字段由该类别引用的值类型声明。更新粒度是 properties 的一层字段，嵌套对象与数组只能整体替换，需要逐项更新的数据应声明为 list。

parent_id 只出现在树节点上，由 WM 维护。它不在构造器参数内，建点时的位置由 BTree 的 add_root 或 add_child 给出，改父走 BTree 的 set_parent，写进 update 的 set 或 unset 会返回 INVALID_ARGUMENT。它也不可用作排序字段。写入时不传它，读回树节点时它必定带回。没有 parent_id，query 的结果之间就没有任何关系信息，调用方无法还原树的形状。图节点不携带该字段。

**示例**：

~~~js
const node = new BNode({
  id: "node-1",
  node_class: "Task",
  properties: { title: "排查移动端 401" },
});
~~~

### BEdge

**作用**：构造一条图边。边只存在于 graph，树不声明边。

**类型定义**：

~~~ts
interface BEdge {
  id: string;
  edge_class: string;
  from_id: string;
  to_id: string;
  properties?: object;
}
~~~

实例由 new BEdge({ id, edge_class, from_id, to_id, properties }) 构造。

id 的规则与 BNode 相同，节点与边各用一个命名空间。edge_class 取所选 graph 内声明的边类别名。from_id 与 to_id 取节点实例 id，不取类别名，两者在 BEdge 上必填。缺端点时在构造处同步抛 [WmUsageError](19-wm-errors.md)，不等到写入期。

该边类别声明了 value_type 时 properties 必填，未声明时必须是空对象或省略。边的两个端点必须属于所选 graph。端点存在与端点类别相容按一个批次的最终状态校验，批次内部允许先登记边再登记端点。

edge_class、from_id 和 to_id 都不在 update_edge 的 set 与 unset 范围内，改类别或改端点需要删除后重建。

**示例**：

~~~js
const edge = new BEdge({
  id: "edge-1",
  edge_class: "has_evidence",
  from_id: "node-1",
  to_id: "node-2",
  properties: {},
});
~~~

### BatchBuilder

**作用**：在 [memory](14-memory.md) 的 batch 回调内选择集合，并登记这一批次的写入。

**类型定义**：

~~~ts
interface BatchBuilder {
  graph(name: string): BatchGraph;
  tree(name: string): BatchTree;
  list(name: string): BatchList;
  map(name: string): BatchMap;
}
~~~

四个访问器是同步的。它们校验集合名与集合种类后返回绑定当前批次和该集合的专用句柄，并为每项操作注入协议字段 structure。取句柄失败时同步抛 [WmUsageError](19-wm-errors.md)，此时不发请求。

批次专用句柄只提供写方法，这些方法同步执行，只登记意图，不返回提交结果；提交结果只由 batch 本身返回。BatchGraph 上有 add_node、update_node、delete_node、add_edge、update_edge、delete_edge；BatchTree 上有 add_root、add_child、set_parent、update、delete；BatchList 上有 append、replace、delete；BatchMap 上有 put、delete。四类集合的读方法一律不挂批次专用句柄，回调窗口内不发任何请求。

批次专用句柄与 memory 上的普通集合句柄是不同对象，不共用普通句柄的缓存。BatchBuilder 本身不提供扁平的 add_node、append、delete 等写方法，集合名只在取句柄时传入，不追加为每个写方法的首参。

BatchBuilder 与批次专用句柄仅在所属回调同步执行期间有效。登记窗口在回调同步返回时结束，不延续到 batch 返回的 Promise 结算。把构建器或专用句柄保存到回调外再使用，以及跨批次复用，都会同步抛 WmUsageError。

**示例**：

~~~js
const result = await memory.batch((b) => {
  b.graph("incident_graph").add_node(node);
  b.map("retry_by_status_code").put(401, 1, "upsert");
});
~~~

### CommitResult

**作用**：表示一次提交的结果。

**类型定义**：

~~~ts
interface CommitResult {
  ok: boolean;
  revision: number;
  generated?: object;
}
~~~

ok 表示本次提交是否落地。接受一次调用不等于提交成功，每个写方法和 batch 都返回这个类型。

revision 在成功时是提交后的新版本，在版本冲突时是服务端的当前版本，也就是调用方重读、重算、重提所需的版本依据。写成功推进一次版本，读和快照创建不推进，一个批次无论含多少个写方法整批成功只推进一次。

generated 只出现在 [BList](17-blist-bmap.md) 的 append 上，带出服务端分配的 position。其余方法没有这一位，不是空对象。版本冲突时未分配位置，整个 generated 都缺省，读它前必须先判 ok。一个批次包含多个 append 时 generated 如何对应各项尚未确定，当前批次里的 append 不能依赖这个字段。

约束失败一律抛 [WmRequestError](19-wm-errors.md)。只有 REVISION_CONFLICT 以 ok 为 false 回到本类型，ok 为 false 只有版本冲突这一个含义。SDK 不自动合并、也不自动重试。

**示例**：

~~~js
const result = await incidentGraph.add_node(node);
text({ ok: result.ok, revision: result.revision });
~~~

### BPage

**作用**：承载当前实例的分页读结果。

**类型定义**：

~~~ts
interface BPage {
  items: unknown[];
  next_cursor?: unknown;
  revision: number;
}
~~~

三个字段是字段全集，没有第四位，也没有 complete 字段。是否还有后续页由 next_cursor 判断，有值表示还有下一页。revision 是这一页的读版本，与 CommitResult 的 revision 同口径。

items 的元素类型由调用的方法与查询目标决定：[BGraph](15-bgraph.md) 的 query 按 target 返回 [BNode](18-wm-elements.md) 或 [BEdge](18-wm-elements.md)，[BTree](16-btree.md) 的 query 与 children 返回带 parent_id 的 BNode，[BList](17-blist-bmap.md) 的 query 返回 [BListItem](18-wm-elements.md)，[BMap](17-blist-bmap.md) 的 query 返回 [BMapItem](18-wm-elements.md)，BGraph 的 neighbors 返回 [BNeighborItem](18-wm-elements.md)。

游标由 WM 生成，对调用方不透明，只能原样传回。它至少编码了目标、规范化过滤器、排序键和读版本，并在作用域内防篡改。调用方不得自行构造、解析或偏移游标，这与 offset 式分页有本质区别。游标钉住发出它的那次读的版本，继续分页恒在同一读版本上进行，翻页期间发生的提交不使它失效。

该读版本被压实后游标失效，走一个独立于 REVISION_CONFLICT 的错误码，并回带当前仍可用的最旧版本。收到该码时的恢复路径是丢弃本地视图、重新发起首次读、从新的版本继续，不是重读同一页。那个错误码还没有定名，不在 [WmErrorCode](19-wm-errors.md) 的清单里。游标的有效期与保留预算两个数值也尚未确定，在拿到数之前分页会不会中断只能定性，不能定量。

单项读取也返回这个分页体。按唯一键查询时 items 的长度上界是 1，没有匹配项时 items 是空数组，这不是错误。[BTree](16-btree.md) 的 get、parent 和 root 不套本类型，不返回读版本。

**示例**：

~~~js
let page = await incidentGraph.query({ target: "nodes" });
while (page.next_cursor) {
  page = await incidentGraph.query({ target: "nodes", cursor: page.next_cursor });
}
~~~

### BListItem

**作用**：表示 [BList](17-blist-bmap.md) 查询结果中的一个项。

**类型定义**：

~~~ts
interface BListItem {
  item_id: string;
  position: number;
  value: unknown;
}
~~~

三个字段在查询结果中必返。item_id 由调用方提供，服务端不生成该值。list 没有按下标的读取方法，也没有独立的单项读方法，replace 与 delete 全靠这个键定位，调用方需要保存它。

position 由 WM 分配：同一 list 从 0 开始，只有成功提交的追加才分配，失败或回滚不消耗，删除后不复用并允许出现间隔，在实例、快照和恢复后保持不变。

value 符合该 list 声明的 item_type。item_type 声明为内联标量时，value 就是裸标量本身，不需要包装类。

**示例**：

~~~js
const page = await evidenceList.query({ target: "list" });
text(page.items.map((item) => item.position));
~~~

### BMapItem

**作用**：表示 [BMap](17-blist-bmap.md) 查询结果中的一个项。

**类型定义**：

~~~ts
interface BMapItem {
  key: string | number | boolean;
  value: unknown;
}
~~~

两个字段在查询结果中必返，返回的是规范化后的 key 表示。

key 的类型由该 map 声明的 key_type 决定，不接受 null，也不得通过隐式类型转换改变。整数 key 使用无前导零的十进制表示，写 401 不写 "401"，写成字符串按 INVALID_STRUCTURE 拒绝。字符串 key 不做 Unicode 归一化，按原始 Unicode 标量序列比较，大小写不折叠。布尔 key 只接受 true 和 false。节点 id 与 item_id 不同，WM 对它们不做任何归一。

value 符合该 map 声明的 value_type，对象值直接传入，不需要包装类。

**示例**：

~~~js
const page = await retryByStatusCodeMap.query({ target: "map", filter: { key: 429 } });
text(page.items[0].value);
~~~

### BNeighborItem

**作用**：表示 [BGraph](15-bgraph.md) 的 neighbors 结果中的一对相邻节点和边。

**类型定义**：

~~~ts
interface BNeighborItem {
  node?: BNode;
  edge?: BEdge;
}
~~~

它与 query 的项不同形。相邻节点与对应的边来自同一读版本，并分别经过授权过滤。带上边才能还原每个邻居是经哪条边、从哪个方向到达的。

include.nodes 与 include.edges 的默认值都是 true，省略 include 或省略其中一个键一律按 true 处理。代价是返回体比必要的大，只需要邻居 id 时应显式传入 include 的 edges 为 false。把 nodes 传成 false 时返回项里就没有 node，两个字段因此都标成可选。

这个类型的字段全集尚未确定，未列出的字段不能作为调用依据。

**示例**：

~~~js
const page = await incidentGraph.neighbors("node-1", { direction: "out" });
text(page.items.map((item) => item.node?.id));
~~~

## 相关文档

- [memory](14-memory.md)：集合访问、批量提交和 Run 关闭。
- [WmError](19-wm-errors.md)：异常类层级、错误码和捕获方式。
