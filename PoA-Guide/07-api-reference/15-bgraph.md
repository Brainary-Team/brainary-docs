# BGraph

[BGraph](15-bgraph.md) 是一个 graph 集合的句柄，提供节点与边的读写、一跳邻接和查询。句柄由 [memory](14-memory.md) 的 graph 访问器取得，下面的示例都以 incidentGraph 指代它。写方法成功提交时推进一次版本，读方法不推进；失败的分流方式见 [WmError](19-wm-errors.md)。

### add_node(node)

**作用**：在当前 graph 中新建一个节点。

**输入类型**：node: [BNode](18-wm-elements.md)。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.add_node(
  new BNode({ id: "node-1", node_class: "Task", properties: { title: "排查移动端 401" } }),
);
~~~

新增不会覆盖已有节点。id 在该操作之前的状态中已存在时返回 STRUCTURE_CONFLICT；node_class 未在当前 graph 声明、properties 不符声明的值类型、或写入未声明的属性返回 INVALID_ARGUMENT；违反 schema 声明的 unique_property 返回 CONSTRAINT_VIOLATION。

### add_edge(edge)

**作用**：在当前 graph 中新建一条边。

**输入类型**：edge: [BEdge](18-wm-elements.md)。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.add_edge(
  new BEdge({
    id: "edge-1",
    edge_class: "has_evidence",
    from_id: "node-1",
    to_id: "node-2",
    properties: {},
  }),
);
~~~

端点节点必须存在，端点类别必须正好是该边类别声明的 from_class 与 to_class，端点在最终状态中不存在时返回 CONSTRAINT_VIOLATION。端点属于另一个 graph 或 tree 时返回 INVALID_STRUCTURE。这两项与多重边唯一性都按最终状态校验，规则见下面的提交与约束。

### update_node(id, update)

**作用**：更新一个节点的属性。

**输入类型**：id: string；update: object，用 set 表达要写入的属性、用 unset 表达要移除的属性。

**参数约束**：set 与 unset 至少一个非空。node_class 与 parent_id 都不在两者的作用范围内。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.update_node("node-1", { set: { priority: 2 } });
~~~

字段缺失不表示删除，移除属性要写进 unset。粒度是 properties 的一层字段，嵌套对象与数组只能整体替换。把 parent_id 写进 set 或 unset 返回 INVALID_ARGUMENT；目标在该操作之前的状态中不存在时返回 NOT_FOUND_OR_DENIED。

### delete_node(id)

**作用**：删除一个节点。

**输入类型**：id: string。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.delete_node("node-1");
~~~

不级联删边。删除后仍留下关联边时返回 CONSTRAINT_VIOLATION；悬空边这一条按最终状态校验，同一批次同时删边删点是允许的。

### update_edge(id, update)

**作用**：更新一条边的属性。

**输入类型**：id: string；update: object，用 set 表达要写入的属性、用 unset 表达要移除的属性。

**参数约束**：set 与 unset 至少一个非空。edge_class、from_id 和 to_id 都不在两者的作用范围内。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.update_edge("edge-1", { set: { weight: 3 } });
~~~

形状与粒度同 update_node。改类别或改端点不在本方法的能力内，需要删除后重建。

### delete_edge(id)

**作用**：删除一条边。

**输入类型**：id: string。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.delete_edge("edge-1");
~~~

### neighbors(node_id, options?)

**作用**：读取一个节点的一跳邻接。本方法是 graph 专属。

**输入类型**：node_id: string；options?: object，可含 direction、edge_classes、node_classes、include、cursor 和 limit。

**参数约束**：direction 取 out、in 或 both。edge_classes 与 node_classes 取当前 graph 内已声明的类别名，它们是查询过滤器而不是 schema 字段，未声明的名字返回 INVALID_ARGUMENT。include 的 nodes 与 edges 默认都是 true。

**返回值类型**：Promise<[BPage](18-wm-elements.md)>，items 的元素是 [BNeighborItem](18-wm-elements.md)。

**示例**：

~~~js
const page = await incidentGraph.neighbors("node-1", { direction: "out" });
text(page.items.length);
~~~

direction 三者有意义的前提是边恒有向，这由 schema 固定，没有对应的声明字段。带上边才能还原每个邻居是经哪条边、从哪个方向到达的，只需要邻居 id 时应显式传入 include 的 edges 为 false，否则返回体比必要的大。查询可能需要扫描所选图的边，大图上应控制查询范围。

### query(options)

**作用**：按条件分页查询当前 graph 的节点或边。

**输入类型**：options: object，必须含 target，可含 filter、sort、cursor 和 limit。

**参数约束**：target 取 nodes 或 edges。structure 由句柄自动注入，不作为业务查询选项传入。

**返回值类型**：Promise<[BPage](18-wm-elements.md)>，target 为 nodes 时项是 [BNode](18-wm-elements.md)，target 为 edges 时项是 [BEdge](18-wm-elements.md)。

**示例**：

~~~js
const condition = { target: "nodes", filter: { node_class: "Evidence" } };
let page = await incidentGraph.query(condition);
while (page.next_cursor) {
  page = await incidentGraph.query({ ...condition, cursor: page.next_cursor });
}
~~~

filter 按 id、node_class 或 edge_class、以及已声明属性做精确匹配。sort 的可用字段是已声明的一层标量字段，并自动追加元素 id 作为末级排序键。graph 未指定 sort 时的默认顺序尚未固定，需要稳定顺序时必须显式传入 sort。

## 集合归属与类别定位

句柄把集合名作为协议字段 structure 注入每次请求，业务代码不重复传入集合名。WM 再校验集合、类别和权限，不按类别名推断集合。

节点类别只在当前 graph 的声明中查找，其他 graph 或 tree 声明了同名类别不使本次写入有效。边的两个端点必须属于所选 graph。邻接查询同样只在句柄指定的 graph 内执行：WM 校验 node_id 属于该 graph，不因节点 id 在实例内唯一而自动切换集合。所选集合不是 graph，或已授权的 node_id 属于另一个集合时返回 INVALID_STRUCTURE，与取句柄时的拒绝口径一致，且只在授权可见后才报告归属错误。

节点 id 的唯一性作用域是整个实例，节点与边各用一个命名空间。

查询只在所选集合内筛选，未命中返回合法空结果，不改报集合归属错误。

## 提交与约束

端点存在、端点类别相容和多重边唯一性都是谓词，按一个批次的最终状态校验。同一批次里可以先登记边再登记两个端点；立即提交形态下最终状态就是这一次调用之后的状态，仍要先有点再有边。

不支持多重边，同一边类别下同一个有序节点对至多一条边。这一条也按最终状态判：同一批次先删旧边再加一条同类别同端点的新边允许，同一批次新增两条同类别同端点的边不允许。

同一批次内重复删同一个目标按去重处理，不返回 NOT_FOUND_OR_DENIED，删除在最终状态语义下幂等；单次删除一个不存在的目标仍然报错。delete_edge 与 delete_node 的去重规则相同。

已有元素不能在同一批次内通过删除再创建更换类别。更换类别需要分别提交删除与新建，并处理关联边和两个提交之间的中间状态。

## 当前未确定

update_node、delete_node、update_edge、delete_edge 的参数形状，以及 query 与 neighbors 的查询选项形状，都尚未确定。上面给出的是这些方法需要表达的信息和当前的调用形态，还不能作为稳定调用签名，写业务代码时应把这几处集中封装在一处。

更复杂的删除序列的结果尚未确定。对批次开始时就不存在的目标，以及删除、重建后再次删除这类组合，都不能按上面的去重规则推导结果。

## 相关文档

- [memory](14-memory.md)：集合访问、批量提交和 Run 关闭。
- [BNode、BEdge 与读写返回类型](18-wm-elements.md)：构造类型、提交结果和分页体。
- [WmError](19-wm-errors.md)：异常类层级、错误码和捕获方式。
