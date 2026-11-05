# BTree

[BTree](03-btree.md) 是一个 tree 集合的句柄，提供树节点的读写、层级查询和改父。句柄是[声明](01-memory.md)里声明为 tree 的那个字段，下面的示例都以 taskTree 指代它，取法是 `const taskTree = mem.task_tree;`。失败的分流方式见 [WmError](06-errors.md)。

### add_root(node)

**作用**：在当前 tree 中新建一个节点并把它设为根。

**输入类型**：node: [BNode](05-elements.md)，不含 parent_id。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.add_root(
  new BNode({ id: "plan-1", node_class: "Plan", props: { title: "排查移动端 401" } }),
);
~~~

已有根时建根违反单根，空树上才能建根。建根时父子类别相容检查不会触发，node_class 仍须是该树已声明的类别。

### add_child(parent_id, node)

**作用**：在当前 tree 中新建一个节点并挂到指定父节点下。

**输入类型**：parent_id: string；node: [BNode](05-elements.md)，不含 parent_id。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.add_child(
  "plan-1",
  new BNode({ id: "ms-1", node_class: "Milestone", props: { title: "收集证据" } }),
);
~~~

新增不会覆盖已有节点，id 已存在时返回 STRUCTURE_CONFLICT。parent_id 指向另一棵树或某个图的节点时返回 INVALID_STRUCTURE，不支持跨树移动。父的 child_classes 不含该节点的 node_class 时违反父子类别相容。

### set_parent(node_id, new_parent_id, expected_parent_id)

**作用**：把一个节点改挂到新的父节点下，移动作用于整棵子树。

**输入类型**：node_id: string；new_parent_id: string | null；expected_parent_id: string | null。

**参数约束**：三个参数全部必填。node_id 不接受 null；另两个都接受 null。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.set_parent("subtask-1", "ms-2", "ms-1");
~~~

移动作用于整棵子树，调用方不需要列出后代节点。expected_parent_id 核对的是该节点当下的父节点，与实际当前父不符时返回 STRUCTURE_CONFLICT，details 的 reason 取 tree_parent_mismatch，并返回实际父节点。两个 null 参数各有自己的含义，见下面的改父。

### update(node_id, changes)

**作用**：更新一个树节点的属性。

**输入类型**：node_id: string；changes: object，形状是 { set?: object, unset?: string[] }，set 的键是属性名、值是新值，unset 是属性名的数组。参数名取 node_id 而不是 id，与 BTree 自己的 get、parent、children 和 set_parent 一致。

**参数约束**：set 与 unset 至少一个非空。node_class 与 parent_id 都不在两者的作用范围内。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.update("ms-2", { set: { priority: 2 } });
~~~

更新粒度与图节点相同，是 props 的一层字段。把 parent_id 写进 set 或 unset 返回 INVALID_ARGUMENT，改父只能走 set_parent。

### delete(node_id)

**作用**：删除一个树节点。

**输入类型**：node_id: string。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await taskTree.delete("subtask-1");
~~~

不级联删除。删一个还有子节点的节点会留下无父的非根节点，返回 CONSTRAINT_VIOLATION。删一棵子树的做法见下面的删除。

### get(node_id)

**作用**：按 id 读取一个树节点。

**输入类型**：node_id: string。

**返回值类型**：Promise<[BNode](05-elements.md) | null>。

**示例**：

~~~js
const milestoneNode = await taskTree.get("ms-2");
if (milestoneNode !== null) text(milestoneNode.props.title);
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

### children(node_id, options?)

**作用**：读取一个节点的直接子节点。

**输入类型**：node_id: string；options?: object，可含 limit。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是树节点。

**示例**：

~~~js
const childPage = await taskTree.children("plan-1");
text(childPage.items.map((child) => child.id));
~~~

它等价于 query 按 parent_id 过滤，只返回一跳子集，不包含孙节点。一次调用返回全部直接子节点。

### query(options)

**作用**：按条件查询当前 tree 的节点。

**输入类型**：options: object，必须含 target，可含 filter 和 limit。

**参数约束**：target 取 tree。无 mode 字段。collection 由句柄自动注入，不作为业务查询选项传入。filter 出现 child_id 时不得同时给 parent_id 或 id。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是树节点，比图节点多一个 parent_id。

**示例**：

~~~js
const page = await taskTree.query({ target: "tree", filter: { node_class: "Milestone" } });
text(page.items.map((node) => node.parent_id));
~~~

filter 的可用键是 id、node_class、parent_id、child_id 和已声明的一层标量属性。不给 filter 返回该树的全部节点。细则见下面的查询。

## 树的形状

树上没有边实体。父子关系由节点的 parent_id 表达，树不声明边类别。

树是单根的有根树，空树合法。非空时恰好有一个 parent_id 为 null 的节点，不允许游离节点，父链不成环。把一棵非空树清空同样合法。

## 约束与判定顺序

树的约束分三个层次。单父是构造保证，parent_id 是单值系统字段，一个节点有两个父这种状态在数据模型上不存在，既不需要检查也不需要唯一索引。parent_id 的引用完整性是内置强制项，每个非根节点的 parent_id 必须指向该树内存在的节点，它同时覆盖挂到不存在的父上和删掉一个还有子节点的父两个方向。另有四条写入期谓词，分别是单根、无环、深度上限、父子类别相容。

五条判定在每次写入时按这一次写入之后的状态求值，并按固定顺序评估：

- 引用完整性
- 单根
- 无环
- 深度上限
- 父子类别相容

顺序本身有后果，一个输入同时违反两条时返回前一条检查对应的错误码。无环排在深度之前，因为环上的节点到根没有路径，深度无定义。深度是根到该节点的边数，根为 0，上限由宿主能力声明带出，不进 schema。深度超限归在 CONSTRAINT_VIOLATION，不归 RESOURCE_EXHAUSTED，它是确定性的输入性失败。违反以上任一条都返回 CONSTRAINT_VIOLATION，协议层在 details 里返回究竟是哪一条，details 的可用性见 [WmRequestError](06-errors.md)。但承载它的字段有两个，对应上面那条分层：四条写入期谓词各在 reason 的取值集合里占一个值，引用完整性作为内置强制项有意不进那个集合，由 details 的约束标识返回。

两条判别规则跟着这个顺序一起读。成环与深度超限是两种不同的违规，不是同一种的两种程度：父链上溯正常终止于根、只是步数超过上限的那种输入属深度违规，不报成环；深度超限时 details 另外返回上限、实测深度和最深那个节点的 id：违规的节点可能不是任何一次调用的目标，没有它就无法定位。自环按成环处理，不另算一种。

单个方法另有自己的检查顺序，只执行该操作适用的检查：字段形状与必填、集合归属、目标状态前提、父指针比对、空操作。这五步评估的是调用自身与它执行之前的状态，五条判定评估的是它执行之后的状态。两者分两段：先逐项评估这五步，全部通过后再对写入之后的状态评估一次五条判定。

这五步之前还有一步，它在本页描述的 owner 路径上不会触发，但次序是语义的一部分而不是实现细节：对持有集合可见集的调用方，先判所选集合在不在可见集内，不在就返回 NOT_FOUND_OR_DENIED 并就此结束，不再往下执行任何一步。它必须排在最前，否则“元素级不存在统一用 NOT_FOUND”这条规定会造出一个探测面：不可见集合里的不存在元素返回 NOT_FOUND、存在元素返回 NOT_FOUND_OR_DENIED，两者可区分。宿主注入给 PoA 程序的是 owner 侧句柄，可见集是全集，所以本页其余各处不再重复这一步。

把 add_child 的 parent_id 传成新节点自己的 id，一个调用就能同时造出自环和零根两种违规，按固定顺序只返回一个码。在空树上这么做，写入之后的状态是自环且零根，单根先判，回的是零根那一条。要单独触发成环，需要在非空树上这么做，此时根还在、单根通过，无环才会命中。

## 改父

树节点必须在创建时就有位置，没有先建后挂这条路。节点位置由所选方法及其参数确定，不需要自行组装父节点字段。

set_parent 的 expected_parent_id 传 null 表示被移动的节点当前是根，new_parent_id 传 null 表示把该节点升为根。两者都受单根约束：一棵树在任何时刻只能有一个根，所以把旧根挂到新根下、或把别的节点升为根，都不是一次调用能完成的，做法见下面的换根。schema 里还须有一个类别把旧根的 node_class 列进 child_classes，否则这棵树上插不了新根；这一条由声明决定，运行期无法修改。

expected_parent_id 比的是父指针，不是版本。它核对该节点当下的父节点是否与调用方的预期一致，不符时返回 STRUCTURE_CONFLICT，details 的 reason 取 tree_parent_mismatch，并返回实际父节点；调用方不能仅凭本参数推定并发安全。父指针比对先于空操作判定：比对相符之后，若 new_parent_id 也等于实际当前父（两者同为 null 也算相等），才按 INVALID_ARGUMENT 拒绝。这个顺序是有意的：做成两个参数相同即拒的静态校验，会掩盖本该报的父节点不符。

被移动的节点是根时，按固定顺序先报单根那一条，不报成环。归属按调用之前的状态与写入之后的状态的并集判定，只要该 id 在其中任一侧出现过且属于另一棵树或某个图，就返回 INVALID_STRUCTURE。该 id 在两侧都不存在时才落到引用完整性。

## 删除

删一个叶节点一次调用就行。删一棵有子节点的子树要自底向上逐个删：先删全部叶，再删新成为叶的那一层，依此类推，直到删到子树的根。删除顺序不能颠倒：引用完整性是每次写入各自校验的，任何一次“删掉一个还有子节点的节点”都会被拒绝。

这需要先用一次不带 filter 的 query 取回全部节点算出后代集，漏掉后代会在删除其父时触发引用完整性失败。按 id 升序逐个删尤其容易出错：那正是 query 的默认顺序，它通常把父节点排在子节点之前，删除会确定性失败。

整棵子树的删除不是原子的：往返次数等于节点数，中间状态对并发读可见，中途失败时已删掉的那些节点不会恢复。

删中间节点并把子节点提升给祖父没有专门动作，做法是逐个 set_parent 把子节点改挂到祖父，最后 delete 那个中间节点。每一步单独看都合法，但整体不是原子的。

## 换根

在现有树上插一层新根、或把另一个节点换成根，都要整树拆建，因为单根在每次写入时都校验，而建根要求空树：先自底向上把整棵树删空，再建新根，再把原来的根挂到新根下，然后自顶向下重建其余节点。

这一串操作不是原子的，而且中途失败时树已经被拆掉，没有回滚。换根之前应当先用一次不带 filter 的 query 把全部节点与父指针取回本地保存，否则重建时无法得到原来的形状。

## 单项读方法

get、parent 和 root 是四类集合里唯一的三个单项读方法。它们在协议层归一到 query 加一个唯一键 filter：get 等价于按 id 过滤，parent 等价于按 child_id 过滤并依赖单父构造保证，root 等价于按 parent_id 为 null 过滤并依赖单根谓词。

取不到时返回 null，不抛异常，空树的 root 和根节点的 parent 都返回 null。认证或授权失败仍按异常处理，此时的 NOT_FOUND_OR_DENIED 只剩实例不可见、作用域无权这类成因，不含这个节点不存在。

这三个方法不套 [BPage](05-elements.md)，它们的结果上界是 1。多次独立读取不保证观察到同一份树结构，需要一致的树结构时用一次不带 filter 的 query 整棵读回。

## 查询

按 parent_id 过滤得到一跳子节点，等价于 children；parent_id 为 null 得到根，等价于 root；按 child_id 过滤得到父节点，等价于 parent；按 id 过滤得到单个节点，等价于 get。

child_id 是一条反向关系谓词，不是字段过滤，返回的那个父节点上没有 child_id 字段。它禁止与 id 同用：同时给两者等于一次“这个是不是那个的父”的关系判定，不支持这种参数组合。child_id 可以与 node_class 及属性条件同用，此时这些条件约束的是返回的那个父节点。null 只匹配 null，不参与与非空值的比较，root 完全依赖这一条。目标节点不存在时按空结果处理，不是错误。

默认按 id 升序，按原始 Unicode 标量序列比较。limit 的语义与命中项数上限见[默认序与结果规模](01-memory.md)。

子节点无序。返回顺序按 id 升序，它是展示顺序，不承载业务含义。需要顺序的层级有两种做法：改用 [BList](04-blist-bmap.md)，或在节点属性上自带一个序号字段由调用方排序。

不带 filter 的 query 一次返回整棵树的形状，也就是全部节点加全部父指针。判叶要看数据性质：叶是 parent_id 未被任何节点引用的节点。它与 child_classes 为空这个类型性质是不同维度，不得混用，一个声明了 child_classes 的节点在数据上仍可能是叶。

## 不提供的能力

本页方法不接受 mode，传入时返回 INVALID_ARGUMENT。

当前不提供路径查询、重排、子树删除、根列举、叶列举和移动节点，也没有 neighbors，那是 [BGraph](02-bgraph.md) 专属。祖先链、深度、叶集、子树节点集和子树高度可由调用方从一次不带 filter 的 query 结果计算，SDK 不为它们提供单独的方法。

## 当前未确定

上面十个方法的签名已稳定，参数个数、位置与名字可以照着写，但写方法仍有一些参数细节尚未确定，遇到本页未写明的取值规则不要自行推定。

children 的 options 里能不能再给 filter 尚未确定：协议层它展开成一次按 parent_id 过滤的 query，调用方自带的 filter 与那个自动注入的条件如何合并没有规定。要在子节点上再加筛选条件时改用 query，由调用方给出 parent_id。

深度上限的具体数值待实测。

## 相关文档

- [声明、登记与句柄](01-memory.md)：集合声明、登记与载入、句柄，以及默认序与结果规模。
- [BNode、BEdge 与读写返回类型](05-elements.md)：构造类型、提交结果和读结果体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
