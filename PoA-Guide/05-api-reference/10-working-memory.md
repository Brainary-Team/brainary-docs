# Working Memory

Working Memory 是 PoA 程序侧的结构化状态容器。作者先用一个 class 声明有哪些集合，再把声明登记进 PoA 对象并载入整份数据，
之后用声明里的字段读写。它的全部状态属于当前 Run，而不属于某个 cell、turn 或 thread。

这组文档按对象拆分，与本章其余部分同一体例：类型按“作用、类型定义、示例”说明，
函数和方法按“作用、输入类型、返回值类型、示例”说明。

## 文件索引

| 文件 | 内容 |
| --- | --- |
| [声明、登记与句柄](10-working-memory/01-memory.md) | 集合声明、register_schema 与 load_data、句柄与集合名，以及 Working Memory 的作用域、默认序与结果规模 |
| [BGraph](10-working-memory/02-bgraph.md) | 图的节点与边读写、一跳邻接和查询 |
| [BTree](10-working-memory/03-btree.md) | 树的读写、约束模型和层级查询 |
| [BList 与 BMap](10-working-memory/04-blist-bmap.md) | 有序项追加与键值读写 |
| [元素与返回类型](10-working-memory/05-elements.md) | BNode、BEdge、CommitResult、BPage 和三个项类型 |
| [异常](10-working-memory/06-errors.md) | WmError、WmUsageError、WmRequestError、WmTransportError 和错误码 |
| [memory.snapshot](10-working-memory/07-snapshot.md) | 快照创建、列举、只读查询和 SnapshotPage，属下一阶段 |

四类集合的句柄都是[声明](10-working-memory/01-memory.md)里的字段，字段名就是集合名。
失败分成三类：请求尚未发出的本地使用错误、后端拒绝的请求错误，以及请求发出之后的传输与帧失败，分流方式见 [异常](10-working-memory/06-errors.md)。
