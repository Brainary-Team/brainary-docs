# BGraph

[BGraph](15-bgraph.md) 是一个 graph 集合的句柄，提供节点与边的读写、一跳邻接和查询。句柄由
[memory](14-memory.md) 的 graph 访问器取得，下面的示例都以 incidentGraph 指代它。

集合由句柄指定。句柄把集合名作为协议字段 structure 注入每次请求，业务代码不重复传入集合名；WM 再校验集合、类别
和权限，不按类别名推断集合。节点类别只在当前 graph 的声明中查找，仅在其他 graph 或 tree 声明了同名类别不会使本次
写入有效。边的两个端点必须属于所选 graph。节点 id 的唯一性作用域是整个实例，节点与边各用一个命名空间。

写方法成功提交时推进一次版本，读方法不推进。写方法的返回值表明本次提交是否落地，约束失败一律抛
[WmRequestError](19-wm-errors.md)，只有 REVISION_CONFLICT 以 ok 为 false 回到
[CommitResult](18-wm-elements.md)。各条目只说明该方法的特定失败，完整错误码见 [WmErrorCode](19-wm-errors.md)。

update_node、delete_node、update_edge、delete_edge 的参数形状，以及 query 与 neighbors 的查询选项形状，都尚未
确定。下面给出这些方法需要表达的信息和当前的调用形态，但它们还不能作为稳定调用签名；写业务代码时应把这几处集中
封装在一处。

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

新增不会覆盖已有节点：id 在该操作之前的状态中已存在时返回 STRUCTURE_CONFLICT，不按最终状态判。node_class 未在
当前 graph 声明、properties 不符声明的值类型、或写入未声明的属性返回 INVALID_ARGUMENT。违反 schema 声明的
unique_property 返回 CONSTRAINT_VIOLATION。已有元素不能在同一批次内通过删除再创建更换类别；更换类别需要分别提交
删除与新建，并处理关联边和两个提交之间的中间状态。

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

端点节点必须存在、端点类别必须正好是该边类别声明的 from_class 与 to_class，这两项都是谓词，按一个批次的最终状态
校验，所以同一批次里可以先登记边再登记两个端点。立即提交形态下最终状态就是这一次调用之后的状态，仍要先有点再有边。
端点在最终状态中不存在时返回 CONSTRAINT_VIOLATION。不支持多重边：同一边类别下，同一个有序节点对至多一条边，按
最终状态判，所以同一批次先删旧边再加一条同类别同端点的新边允许，同一批次新增两条同类别同端点的边不允许。端点属于
另一个 graph 或 tree 时返回 INVALID_STRUCTURE，且只在授权可见后才报告归属错误。

### update_node(id, update)

**作用**：更新一个节点的属性。

**输入类型**：id: string；update: object，用 set 表达要写入的属性、用 unset 表达要移除的属性。

**参数约束**：set 与 unset 至少一个非空。node_class 与 parent_id 都不在两者的作用范围内。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.update_node("node-1", { set: { priority: 2 } });
~~~

更新用 set 与 unset 明确表达，字段缺失不表示删除。粒度是 properties 的一层字段，嵌套对象与数组只能整体替换。改类别
不在本方法的能力内；把 parent_id 写进 set 或 unset 返回 INVALID_ARGUMENT。目标在该操作之前的状态中不存在时返回
NOT_FOUND_OR_DENIED。

### delete_node(id)

**作用**：删除一个节点。

**输入类型**：id: string。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await incidentGraph.delete_node("node-1");
~~~

不级联删边。删除后仍留下关联边时返回 CONSTRAINT_VIOLATION；悬空边这一条是谓词、按最终状态校验、与操作顺序无关，
所以同一批次同时删边删点是允许的。同一批次内重复删同一个目标按去重处理，不返回 NOT_FOUND_OR_DENIED，删除在最终
状态语义下幂等；单次删除一个不存在的目标仍然报错。更复杂的删除序列的结果尚未确定：对批次开始时就不存在的目标，
以及删除、重建后再次删除这类组合，都不能按上面的去重规则推导结果。

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

同批次重复删同一目标的去重规则与 delete_node 相同。

### neighbors(node_id, options?)

**作用**：读取一个节点的一跳邻接。本方法是 graph 专属。

**输入类型**：node_id: string；options?: object，可含 direction、edge_classes、node_classes、include、cursor
和 limit。

**返回值类型**：Promise<[BPage](18-wm-elements.md)>，items 的元素是 [BNeighborItem](18-wm-elements.md)。

**示例**：

~~~js
const page = await incidentGraph.neighbors("node-1", { direction: "out" });
text(page.items.length);
~~~

direction 取 out、in 或 both，三者有意义的前提是边恒有向，这由 schema 固定、没有对应的声明字段。edge_classes 与
node_classes 取当前 graph 内已声明的类别名，它们是查询过滤器而不是 schema 字段，未声明的名字返回
INVALID_ARGUMENT。include 的 nodes 与 edges 默认都是 true。

邻接查询只在句柄指定的 graph 内执行：WM 校验 node_id 属于该 graph，不因节点 id 在实例内唯一而自动切换集合，也不
查找其他图的同名类别；所选集合不是 graph，或已授权的 node_id 属于另一个集合时返回 INVALID_STRUCTURE。查询可能需要
扫描所选图的边，大图上应控制查询范围。

### query(options)

**作用**：按条件分页查询当前 graph 的节点或边。

**输入类型**：options: object，必须含 target，可含 filter、sort、cursor 和 limit。

**参数约束**：target 取 nodes 或 edges。structure 由句柄自动注入，不作为业务查询选项传入。

**返回值类型**：Promise<[BPage](18-wm-elements.md)>，target 为 nodes 时项是 [BNode](18-wm-elements.md)，
target 为 edges 时项是 [BEdge](18-wm-elements.md)。

**示例**：

~~~js
const page = await incidentGraph.query({ target: "nodes", filter: { node_class: "Evidence" } });
text(page.items.map((node) => node.id));
~~~

filter 按 id、node_class 或 edge_class、以及已声明属性做精确匹配，只在所选集合内筛选；未命中返回合法空结果，不改报
集合归属错误。sort 的可用字段是已声明的一层标量字段，并自动追加元素 id 作为末级排序键。graph 未指定 sort 时的
默认顺序尚未固定，需要稳定顺序时必须显式传入 sort。集合名不存在或种类不符返回 INVALID_STRUCTURE，与取句柄时的
拒绝口径一致。

翻页时把上一页的 next_cursor 原样传回，并沿用同一组查询条件：

~~~js
const condition = { target: "nodes", filter: { node_class: "Evidence" } };
let page = await incidentGraph.query(condition);
while (page.next_cursor) {
  page = await incidentGraph.query({ ...condition, cursor: page.next_cursor });
}
~~~

## 相关文档

- [memory](14-memory.md)：集合访问、批量提交和 Run 关闭。
- [BNode、BEdge 与读写返回类型](18-wm-elements.md)：构造类型、提交结果和分页体。
- [WmError](19-wm-errors.md)：异常类层级、错误码和捕获方式。
