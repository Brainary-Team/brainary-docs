# Working Memory

Working Memory 是 PoA 程序侧的结构化状态容器。全局 `memory` 由宿主在业务 JS 执行前注入，不走 `tools.` 分发路径，
程序拿到的是已连接的 owner 侧句柄。它的全部状态属于当前 Run，而不属于某个 cell、turn 或 thread。

这组文档按对象拆分，与本章其余部分同一体例：类型按“作用、类型定义、示例”说明，
函数和方法按“作用、输入类型、返回值类型、示例”说明。

## 文件索引

| 文件 | 内容 |
| --- | --- |
| [memory](14-working-memory/01-memory.md) | 集合访问器、batch、close_run_memory，以及 Working Memory 的作用域与版本规则 |
| [BGraph](14-working-memory/02-bgraph.md) | 图的节点与边读写、一跳邻接和查询 |
| [BTree](14-working-memory/03-btree.md) | 树的读写、约束模型和层级查询 |
| [BList 与 BMap](14-working-memory/04-blist-bmap.md) | 有序项追加与键值读写 |
| [元素与返回类型](14-working-memory/05-elements.md) | BNode、BEdge、BatchBuilder 与批次专用句柄、CommitResult、BPage 和三个项类型 |
| [异常](14-working-memory/06-errors.md) | WmError、WmUsageError、WmRequestError 和错误码 |
| [memory.snapshot](14-working-memory/07-snapshot.md) | 快照创建、列举、只读查询和 SnapshotPage |

四类集合的句柄都由 [memory](14-working-memory/01-memory.md) 上的同名访问器取得，
集合名取 schema 的 collections 下的声明名。写方法成功提交时推进一次版本，读方法不推进；
失败分成请求尚未发出的本地使用错误和后端拒绝的请求错误两类，分流方式见 [异常](14-working-memory/06-errors.md)。
