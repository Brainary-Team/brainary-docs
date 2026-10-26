# memory.snapshot

[memory.snapshot](07-snapshot.md) 提供 Working Memory 的快照创建、列举和只读查询。它挂在 [memory](01-memory.md) 上，三个方法都返回 Promise，都不在 [BatchBuilder](05-elements.md) 上提供，在 batch 回调窗口内调用会同步抛 [WmUsageError](06-errors.md)。创建、列举和只读查询都不推进当前 Run 的版本。

### SnapshotPage

**作用**：承载快照侧的分页结果，由 list 和 query 共用。

**类型定义**：

~~~ts
interface SnapshotPage {
  entries: unknown[];
  next_cursor?: unknown;
}
~~~

快照侧不使用 [BPage](05-elements.md)。这个分页外壳不带 revision，快照分页绑定的是不可变的快照 id，不是一个读版本。list 返回的 entries 是快照元数据条目，query 返回的 entries 是查询结果，两者共用同一个外壳。

**示例**：

~~~js
const page = await memory.snapshot.list();
text(page.entries.length);
~~~

### create(association?)

**作用**：捕获当前已提交状态，生成一个不可变快照。

**输入类型**：association?: object，由接入侧自定义的业务关联信息。

**参数约束**：association 是有大小上限的 JSON 对象，上限由宿主能力声明提供。

**返回值类型**：Promise<object>。

**示例**：

~~~js
const created = await memory.snapshot.create({ kind: "check", round_ref: "round-3" });
text(created);
~~~

后端返回的成功信息含快照 id、捕获时绑定的 schema 标识、实际捕获的状态版本、实例内的快照序号，以及一个明确区分导出成功和失败的导出结果。期望版本过期时协议返回 REVISION_CONFLICT，不会把较新状态当作指定版本的快照。

### list(options?)

**作用**：列举当前 Run 关联的快照。

**输入类型**：options?: object。

**返回值类型**：Promise<[SnapshotPage](07-snapshot.md)>，entries 按快照序号排序。

**示例**：

~~~js
const page = await memory.snapshot.list();
text(page.entries);
~~~

条目包含快照 id、捕获版本、序号及关联信息，接入侧还需自行保存对应的 schema 标识。本方法只列当前 Run 关联的快照，不提供跨 Run 发现。

### query(snapshot_id, options)

**作用**：按已知快照 id 只读查询历史数据。

**输入类型**：snapshot_id: string；options: object，目标、过滤和排序字段沿用四类集合的 query。

**参数约束**：options 必须含 target 和 structure。快照路径上没有集合句柄，所以 structure 不会被自动注入，必须由调用方给出；缺少它返回 INVALID_ARGUMENT。target 取 nodes 或 edges 时 structure 必须是一个 graph 名。

**返回值类型**：Promise<[SnapshotPage](07-snapshot.md)>，entries 是查询结果。

**示例**：

~~~js
const page = await memory.snapshot.query(snapshot_id, {
  target: "nodes",
  structure: "incident_graph",
});
text(page.entries.length);
~~~

示例里的 snapshot_id 来自接入侧索引或本 Run 的列举结果。本方法不装载、也不改写当前 Run。结果必须按快照捕获时的 schema 解释，历史数据不按当前 schema 重新解释。map 的历史不另建一套接口，用的就是本方法。

## 快照 id 与关联信息

快照 id 由 WM 生成且不可猜测，供历史读取使用，不得自行构造或按序号枚举。序号是实例内的快照序号，列举按它排序。

association 由 WM 只校验形状和大小、保存并返回，不解释业务含义。它是接入侧的业务字段，不是 WM 解释执行的控制参数。检查轮次建议记录类别、稳定的轮次引用和结果，适用时另记判据引用与判据版本。

同一状态可以生成多个不同快照。响应丢失后重试可能多拍一份，WM 不去重，也不标记重试产物，同一个业务轮次可能对应多条快照，不能假定一轮一快照。

## 持久化与导出

[memory](01-memory.md) 当前使用内存存储，进程退出后运行态与内存中的快照都会丢失，只有此前成功导出的文件和接入侧记录能留存。

导出失败不使快照创建失败，但会在响应中显式标出。这时快照仍可在本 WM 进程内读取，却没有得到可跨进程留存的文件，快照创建成功不等于数据已持久保存。收到创建成功而导出失败时，应按该快照不可跨进程使用处理。

接入侧应在每次快照成功后持久记录快照 id、schema 标识、序号和业务关联键，导出成功时还应记录文件位置，导出失败须显式记录。

## 跨 Run 读取

当前不提供装载动作。历史数据只能用本页的只读查询读取，不能把它当成可写 Run 的恢复流程。跨进程导出的文件没有 SDK 导入动作，PoA 程序也不能直接读取导出的文件。

新 Run 无法通过列举发现旧 Run 的快照，旧快照的 id 只能来自接入侧索引，索引本身不等于快照内容已持久化。业务需要跨 Run 查找快照时，必须由宿主保存快照 id、schema 标识和关联信息，并提供额外的读取通道。

query 按宿主注入的授权范围检查，可以跨 Run，但只在 WM 进程仍持有对应快照时有效。持有一个快照 id 不等于获得授权。

## 当前未确定

三个方法的完整 JS 返回形状尚未确定，列举与查询的分页参数位置、以及导出结果的字段名也尚未确定。上面只说明已有依据的语义，不能把示例里的字段访问当作稳定的调用面。

create 的成功响应里各项对应的 JS 字段名尚未确定，不要按字段名写分支。创建成功与 REVISION_CONFLICT 这个失败分支如何统一成一个 JS 签名也尚未确定。

## 相关文档

- [memory](01-memory.md)：集合访问、批量提交和 Run 关闭。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
- [Episodic Memory（设计预览）](../../11-episodic-memory.md)：WM 快照与未来情景记录的关系，以及尚未确定的接入方式。
