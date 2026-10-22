# BTree

[BTree](03-btree.md) 是一个 tree 集合的句柄，提供树节点的读写、层级查询和改父。句柄由 [memory](01-memory.md) 的 tree 访问器取得，下面的示例都以 taskTree 指代它。写方法成功提交时推进一次版本，读方法不推进；失败的分流方式见 [WmError](06-errors.md)。

### add_root(node)

**作用**：在当前 tree 中新建一个节点并把它设为根。

**输入类型**：node: [BNode](05-elements.md)，不含 parent_id。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.add_root(
  new BNode({ id: "plan-1", node_class: "Plan", properties: { title: "排查移动端 401" } }),
);
~~~

已有根时建根违反单根，这一条按最终状态判，同一批次删旧根再建新根是允许的。建根时父子类别相容检查空转，node_class 仍须是该树已声明的类别。

### add_child(parent_id, node)

**作用**：在当前 tree 中新建一个节点并挂到指定父节点下。

**输入类型**：parent_id: string；node: [BNode](05-elements.md)，不含 parent_id。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.add_child(
  "plan-1",
  new BNode({ id: "ms-1", node_class: "Milestone", properties: { title: "收集证据" } }),
);
~~~

新增不会覆盖已有节点，id 在该操作之前的状态中已存在时返回 STRUCTURE_CONFLICT。parent_id 指向另一棵树或某个图的节点时返回 INVALID_STRUCTURE，不支持跨树移动。父的 child_classes 不含该节点的 node_class 时违反父子类别相容。

### set_parent(node_id, new_parent_id, expected_parent_id)

**作用**：把一个节点改挂到新的父节点下，移动作用于整棵子树。

**输入类型**：node_id: string；new_parent_id: string | null；expected_parent_id: string | null。

**参数约束**：三个参数全部必填。node_id 不接受 null；另两个都接受 null。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.set_parent("subtask-1", "ms-2", "ms-1");
~~~

移动作用于整棵子树，调用方不需要列出后代节点。expected_parent_id 核对的是节点在当前操作之前的父节点，与实际当前父不符时返回 STRUCTURE_CONFLICT 并回带实际父节点。两个 null 参数各有自己的含义，见下面的改父。

### update(node_id, update)

**作用**：更新一个树节点的属性。

**输入类型**：node_id: string；update: object，用 set 表达要写入的属性、用 unset 表达要移除的属性。

**参数约束**：set 与 unset 至少一个非空。node_class 与 parent_id 都不在两者的作用范围内。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.update("ms-2", { set: { priority: 2 } });
~~~

更新粒度与图节点相同，是 properties 的一层字段。把 parent_id 写进 set 或 unset 返回 INVALID_ARGUMENT，改父只能走 set_parent。

### delete(node_id)

**作用**：删除一个树节点。

**输入类型**：node_id: string。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.delete("subtask-1");
~~~

不级联删除。树侧拒的是最终状态下留下无父的非根节点，返回 CONSTRAINT_VIOLATION。同一批次内重复删同一个目标的去重规则与 [BGraph](02-bgraph.md) 的 delete_node 相同。删一棵子树的做法见下面的删除。

### get(node_id)

**作用**：按 id 读取一个树节点。

**输入类型**：node_id: string。

**返回值类型**：Promise<[BNode](05-elements.md) | null>。

**示例**：

~~~js
const milestoneNode = await taskTree.get("ms-2");
if (milestoneNode !== null) text(milestoneNode.properties.title);
~~~

### parent(node_id)

**作用**：读取一个节点的父节点。

**输入类型**：node_id: string。

**返回值类型**：Promise<[BNode](05-elements.md) | null>。

**示例**：

~~~js
const parentNode = await taskTree.parent("ms-2");
text(parentNode?.id);
~~~

### root()

**作用**：读取当前 tree 的根节点。

**输入类型**：无。

**返回值类型**：Promise<[BNode](05-elements.md) | null>。

**示例**：

~~~js
const rootNode = await taskTree.root();
text(rootNode?.id);
~~~

### children(node_id)

**作用**：读取一个节点的直接子节点。

**输入类型**：node_id: string。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是树节点。

**示例**：

~~~js
const childPage = await taskTree.children("plan-1");
text(childPage.items.map((child) => child.id));
~~~

它等价于 query 按 parent_id 过滤，只返回一跳子集，不包含孙节点。有 next_cursor 时本页也不是全部直接子节点。

### query(options)

**作用**：按条件分页查询当前 tree 的节点。

**输入类型**：options: object，必须含 target，可含 filter、sort、cursor 和 limit。

**参数约束**：target 取 tree。无 mode 字段。structure 由句柄自动注入，不作为业务查询选项传入。filter 出现 child_id 时不得同时给 parent_id 或 id。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是树节点，比图节点多一个 parent_id。

**示例**：

~~~js
const page = await taskTree.query({ target: "tree", filter: { node_class: "Milestone" } });
text(page.items.map((node) => node.parent_id));
~~~

filter 的可用键是 id、node_class、parent_id、child_id 和已声明的一层标量属性。不给 filter 返回该树的全部节点。细则见下面的查询与分页。

## 树的形状

树上没有边实体。父子关系由节点的 parent_id 表达，树不声明边类别。

树是单根的有根树，空树合法。非空时恰好有一个 parent_id 为 null 的节点，不允许游离节点，父链不成环。把一棵非空树清空同样合法。

## 约束与判定顺序

树的约束分三个层次。单父是构造保证，parent_id 是单值系统字段，一个节点有两个父这种状态在数据模型上不存在，既不需要检查也不需要唯一索引。parent_id 的引用完整性是内置强制项，每个非根节点的 parent_id 必须指向该树内存在的节点，它同时覆盖挂到不存在的父上和删掉一个还有子节点的父两个方向。另有四条写入期谓词，分别是单根、无环、深度上限、父子类别相容。

五条判定全部按最终状态求值，并按固定顺序评估：

- 引用完整性
- 单根
- 无环
- 深度上限
- 父子类别相容

顺序本身有后果，一个输入同时违反两条时返回前一条检查对应的错误码。无环排在深度之前，因为环上的节点到根没有路径，深度无定义。深度是根到该节点的边数，根为 0，上限由宿主能力声明带出，不进 schema。深度超限归在 CONSTRAINT_VIOLATION，不归 RESOURCE_EXHAUSTED，它是确定性的输入性失败。违反以上任一条都返回 CONSTRAINT_VIOLATION，协议层在 details 里回带究竟是哪一条，details 的可用性见 [WmRequestError](06-errors.md)。

单个方法另有自己的检查顺序，只执行该操作适用的检查：字段形状与必填、集合归属、目标状态前提、CAS、空操作、批次级禁令。这六步的作用域是单个方法和它自己的前缀状态，五条判定的作用域是整批和最终状态。两者不是一条流水线，而是分两段：先按书写顺序逐项评估六步，全部通过后再对最终状态整批评估一次五条判定。

把 add_child 的 parent_id 传成新节点自己的 id，一个调用就能同时造出自环和零根两种违规，按固定顺序只返回一个码。在空树上这么做，最终状态是自环且零根，单根先判，回的是零根那一条。要单独撞到成环，得在非空树上这么做，此时根还在、单根通过，无环才会命中。

## 改父

树节点必须在创建时就有位置，没有先建后挂这条路。节点位置由所选方法及其参数确定，不需要自行组装父节点字段。

set_parent 的 expected_parent_id 传 null 表示被移动的节点当前是根，用于在现有树上插一层新根，它有两个前提：同批次须另有一个操作产生新根，且 schema 里须有一个类别把旧根的 node_class 列进 child_classes。后一条意味着在哪棵树上能插新根由 schema 决定，运行期改不了。new_parent_id 传 null 表示把该节点升为根，同批次须让旧根消失。

expected_parent_id 与版本基线的作用不同。SDK 用期望版本检查提交版本，本参数检查父节点是否符合预期，调用方不能仅凭本参数推定并发安全。CAS 相符之后 new_parent_id 仍等于实际当前父（两者同为 null 也算相等）时返回 INVALID_ARGUMENT。这个顺序是刻意的，CAS 优先于空操作判定；做成两个参数相同即拒的静态校验会掩盖本该报的父节点不符。

被移动的节点是根时，按固定顺序先报单根那一条，不报成环。归属按起始状态与候选状态的并集判定，只要该 id 在其中任一侧出现过且属于另一棵树或某个图，就返回 INVALID_STRUCTURE。该 id 在两侧都不存在时才落到引用完整性。

## 删除

删一个叶节点可以单独提交。删一棵有子节点的子树，把该子树全部节点的删除放进同一批次即可，谓词按最终状态求值、顺序无关，不必先删叶再删根。这需要先用一次不带 filter 的 query 翻完全部分页算出后代集，漏掉后代会在删除其父时触发引用完整性失败。

删中间节点并把子节点提升给祖父没有专门动作，一个批次可以原子完成：提交若干个 set_parent 改挂祖父，加一个 delete。

## 单项读方法

get、parent 和 root 是四类集合里唯一的三个单项读方法。它们在协议层归一到 query 加一个唯一键 filter：get 等价于按 id 过滤，parent 等价于按 child_id 过滤并依赖单父构造保证，root 等价于按 parent_id 为 null 过滤并依赖单根谓词。

取不到时返回 null，不抛异常，空树的 root 和根节点的 parent 都返回 null。认证或授权失败仍按异常处理，此时的 NOT_FOUND_OR_DENIED 只剩实例不可见、作用域无权这类成因，不含这个节点不存在。

这三个方法不套 [BPage](05-elements.md)，不返回读版本，需要版本信息时用 query。多次独立读取不保证观察到同一个版本，需要一致的树结构时应使用全树分页读取。它们的结果上界是 1，不算分页查询，不要求后端声明按版本读取的能力；children 与 query 落入分页定义，要求这项能力。

## 查询与分页

按 parent_id 过滤得到一跳子节点，等价于 children；parent_id 为 null 得到根，等价于 root；按 child_id 过滤得到父节点，等价于 parent；按 id 过滤得到单个节点，等价于 get。

child_id 是一条反向关系谓词，不是字段过滤，返回的那个父节点身上没有 child_id 字段。它禁止与 id 同用：同时给两者等于一次「这个是不是那个的父」的关系判定，不支持这种参数组合。child_id 可以与 node_class 及属性条件同用，此时这些条件约束的是返回的那个父节点。null 只匹配 null，不参与与非空值的比较，root 完全依赖这一条。目标节点不存在时按空结果处理，不是错误。

未指定 sort 时按 id 升序，按原始 Unicode 标量序列比较。指定 sort 时自动追加 id 为稳定末级排序键。可排序字段是 id、node_class 和已声明的一层标量属性，parent_id 不可排序。缺失或显式 null 排在非空值之后，同值再按 id 排。没有稳定末级键的分页结果可能重复或漏项，这与 list 的 position 加 item_id、map 的规范化 key 是同一条规则。

子节点无序。返回顺序由排序规定决定，默认按 id 升序，它是展示顺序，不承载业务含义。需要顺序的层级有两条出路：改用 [BList](04-blist-bmap.md)，或在节点属性上自带一个序号字段由调用方排序。

不带 filter 的 query 在单一读版本内返回整棵树的形状，也就是全部节点加全部父指针。它是一次分页读，一次调用拿不完全部节点，大树需要翻完 next_cursor。判叶要看数据性质：叶是 parent_id 未被任何节点引用的节点。它与 child_classes 为空这个类型性质是不同维度，不得混用，一个声明了 child_classes 的节点在数据上仍可能是叶。多次 query 不能用于拼接需要跨版本一致性的路径。

## 不提供的能力

本页方法不接受 mode，传入时返回 INVALID_ARGUMENT。

当前不提供路径查询、重排、子树删除、根列举、叶列举和移动节点，也没有 neighbors，那是 [BGraph](02-bgraph.md) 专属。祖先链、深度、叶集、子树节点集和子树高度可由调用方从一次不带 filter 的 query 结果计算，SDK 不为它们提供单独的方法。

## 当前未确定

update 与 delete 的参数形状、children 的分页参数、以及 query 的查询选项形状，都尚未确定。上面给出的是这些方法需要表达的信息和当前的调用形态，还不能作为稳定调用签名。

children 的 cursor 与 limit 传在哪里尚未确定，不能假定它可以读取后续页；需要明确表达分页时改用 query 按 parent_id 过滤并传入 cursor。

深度上限的具体数值待实测。

## 相关文档

- [memory](01-memory.md)：集合访问、批量提交和 Run 关闭。
- [BNode、BEdge 与读写返回类型](05-elements.md)：构造类型、提交结果和分页体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
