# BNode、BEdge 与读写返回类型

本页描述 Working Memory 的构造类型和读写返回类型。构造类型用于写入 [BGraph](02-bgraph.md) 与 [BTree](03-btree.md)，返回类型由四类集合的读写方法返回。下面的示例以 incidentGraph、evidenceList 和 retryByStatusCodeMap 指代三类集合的句柄，取法是 `const incidentGraph = mem.incident_graph;`，另两个同理，句柄本身是[声明](01-memory.md)里的字段。

### BNode

**作用**：构造一个图节点或树节点。

**类型定义**：

~~~ts
interface BNode {
  id: string;
  node_class: string;
  parent_id?: string | null;
  props: object;
}
~~~

实例由 new BNode({ id, node_class, props }) 构造；普通 JS 对象表达不了类别归属，节点必须经该包装类构造。node_class 取所选集合内声明的节点类别名，不取值类型名。props 的字段由该类别引用的值类型声明。parent_id 只出现在树节点上，由 SDK 维护，不在构造器参数内。

**示例**：

~~~js
const node = new BNode({
  id: "node-1",
  node_class: "Task",
  props: { title: "排查移动端 401" },
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
  props?: object;
}
~~~

实例由 new BEdge({ id, edge_class, from_id, to_id, props }) 构造。edge_class 取所选 graph 内声明的边类别名。from_id 与 to_id 取节点实例 id，不取类别名，两者必填；参数缺失或不是合法 ID 时在构造处同步抛带码错误。端点是否存在、是否属于当前 graph，在 add_edge 时检查。该边类别声明了属性类型时 props 必填，未声明时必须是空对象或省略。

**示例**：

~~~js
const edge = new BEdge({
  id: "edge-1",
  edge_class: "has_evidence",
  from_id: "node-1",
  to_id: "node-2",
  props: {},
});
~~~

### 写方法的返回值

图与树的写方法、list 的 replace/delete 和 map 的 delete 返回 `Promise<void>`。list 的 append 返回 `{ position }`，map 的 put 返回 `{ created }`。写方法的失败一律以异常抛出，不靠返回值传递。所有写入都在本地影子状态上完成，整份文档在提交点写回，见[提交点](01-memory.md)。

**示例**：

~~~js
await incidentGraph.add_node(node);
const appended = await evidenceList.append("ev-1", { note: "网关时间快 31 秒" });
text(appended.position);
const put = await retryByStatusCodeMap.put(503, 1, "upsert");
text(put.created);
~~~

### BPage

**作用**：承载当前实例的读结果。

**类型定义**：

~~~ts
interface BPage {
  items: unknown[];
}
~~~

`items` 是当前版本的字段全集，包含全部命中项；当前版本不提供游标分页字段。

**示例**：

~~~js
const page = await incidentGraph.query({ target: "nodes" });
text(page.items.length);
~~~

### BListItem

**作用**：表示 [BList](04-blist-bmap.md) 查询结果中的一个项。

**类型定义**：

~~~ts
interface BListItem {
  item_id: string;
  position: number | bigint;
  value: unknown;
}
~~~

三个字段在查询结果中必返。item_id 由调用方提供，不由 SDK 生成。position 由 SDK 分配，同一 list 从 0 开始。value 符合该 list 声明的 item_type；item_type 声明为内联标量时，value 就是裸标量本身，不需要包装类。

**示例**：

~~~js
const page = await evidenceList.query();
text(page.items.map((item) => item.position));
~~~

### BMapItem

**作用**：表示 [BMap](04-blist-bmap.md) 查询结果中的一个项。

**类型定义**：

~~~ts
interface BMapItem {
  key: string | number | bigint | boolean;
  value: unknown;
}
~~~

两个字段在查询结果中必返，返回的是规范化后的 key 表示。key 的类型由该 map 声明的 key_type 决定，不接受 null，也不得通过隐式类型转换改变。value 符合该 map 声明的 value_type，对象值直接传入，不需要包装类。

**示例**：

~~~js
const page = await retryByStatusCodeMap.query({ filter: { key: 429 } });
if (page.items.length > 0) text(page.items[0].value);
~~~

### BNeighborItem

**作用**：表示 [BGraph](02-bgraph.md) 的 neighbors 结果中的一对相邻节点和边。

**类型定义**：

~~~ts
interface BNeighborItem {
  node?: BNode;
  edge?: BEdge;
}
~~~

它与 query 的项不同形。相邻节点与对应的边来自同一次读。include 的 nodes 与 edges 默认都是 true，把 nodes 传成 false 时返回项里就没有 node，两个字段因此都标成可选。

**示例**：

~~~js
const page = await incidentGraph.neighbors("node-1", { direction: "out" });
text(page.items.map((item) => item.node?.id));
~~~

## id 与类别

id 由调用方提供，必须是非空字符串，长度不超过 256 个 UTF-8 字节，超长返回 INVALID_ARGUMENT。这个上限同样适用于边 id 与 [BList](04-blist-bmap.md) 的 item_id。节点 id 在该实例的全部 graph 与 tree 之间共用一个空间，图的节点与树的节点因此不能重用同一个 id；边 id 是另一个实例级空间；item_id 只在该 list 内唯一，map 的 key 只在该 map 内唯一，逐条见[作用域与实例](01-memory.md)。id 的内部结构不被解析，也不做大小写或空白归一，T1 与 t1 是两个 id。

类别由集合名和类别名共同定位，别的集合里存在同名类别不会使本次写入有效。两个不同类别可以引用同一个值类型。缺端点这类构造期失败在构造处就抛出，不等到写入期。

props 的更新粒度是一层字段，嵌套对象与数组只能整体替换，需要逐项更新的数据应声明为 list。edge_class、from_id 和 to_id 都不在 update_edge 的 set 与 unset 范围内，改类别或改端点需要删除后重建。

边的两个端点必须属于所选 graph。端点存在与端点类别相容在写这条边时校验，所以两个端点要先分别写入，才能写这条边。

## 树节点的 parent_id

parent_id 由 SDK 维护，不在 BNode 的构造器参数内。建点时的位置由 [BTree](03-btree.md) 的 add_root 或 add_child 给出，改父走 set_parent，写进 update 的 set 或 unset 会返回 INVALID_ARGUMENT。它也不可用作排序字段。

写入时不传它，读回树节点时它必定带回。没有 parent_id，query 的结果之间就没有任何关系信息，调用方无法还原树的形状。图节点不携带该字段。

## 提交结果

写方法的返回类型按上面的定义区分：大多数写方法返回 `void`，list append 返回位置，map put 返回是否新建。一次调用不等于一次提交：写方法只改本地影子状态，整份写回发生在提交点，见[提交点](01-memory.md)。带码的拒绝一律抛 [WmRequestError](06-errors.md)，失败不通过返回值传递。SDK 不自动合并、也不自动重试。整份数据的读回、恢复和删除由已登记的 owner 提供，不属于集合句柄上的写方法。

一个 batch 里写多处只写回一次，往返次数与调用次数无关。不放进 batch 的单发调用各自成一个隐式 batch、各自写回一次，中间状态对并发读可见；中途失败时前面已经写回的内容不会回滚，由调用方决定是补齐还是回退。

## 读结果

items 的元素类型由调用的方法与查询目标决定：[BGraph](02-bgraph.md) 的 query 按 target 返回 BNode 或 BEdge，[BTree](03-btree.md) 的 query 与 children 返回带 parent_id 的 BNode，[BList](04-blist-bmap.md) 的 query 返回 BListItem，[BMap](04-blist-bmap.md) 的 query 返回 BMapItem，BGraph 的 neighbors 返回 BNeighborItem。

一次调用返回全部命中项，默认序与 limit 的语义、命中项数上限见[默认序与结果规模](01-memory.md)。

单项读取也返回这个结果体。按唯一键查询时 items 的长度上界是 1，没有匹配项时 items 是空数组，这不是错误。BTree 的 get、parent 和 root 不套本类型。

## key 的规范化

整数 key 使用无前导零的十进制表示，写 401 不写 "401"，写成字符串按 INVALID_STRUCTURE 拒绝。字符串 key 不做 Unicode 归一化，按原始 Unicode 标量序列比较，大小写不折叠。布尔 key 只接受 true 和 false。

节点 id 与 item_id 不同，它们不做任何归一。

## 相关文档

- [声明、登记与句柄](01-memory.md)：集合声明、登记与载入、句柄，以及默认序与结果规模。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
