# memory

[memory](01-memory.md) 是宿主注入的 Working Memory 句柄：取得四类集合的句柄、把多个写入合并成一次原子提交、创建与读取快照、关闭当前 Run。它的全部状态属于当前 Run，连接由宿主在业务 JS 执行前完成，程序拿到的是已连接的 owner 侧句柄，没有也不需要 connect 调用。

### graph(name)

**作用**：取得一个 graph 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BGraph](02-bgraph.md)。

**示例**：

~~~js
const incidentGraph = memory.graph("incident_graph");
~~~

### tree(name)

**作用**：取得一个 tree 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BTree](03-btree.md)。

**示例**：

~~~js
const taskTree = memory.tree("task_tree");
~~~

### list(name)

**作用**：取得一个 list 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BList](04-blist-bmap.md)。

**示例**：

~~~js
const evidenceList = memory.list("evidence_list");
~~~

### map(name)

**作用**：取得一个 map 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BMap](04-blist-bmap.md)。

**示例**：

~~~js
const retryByStatusCodeMap = memory.map("retry_by_status_code");
~~~

四个访问器都是同步的，不发请求，只查本地已校验的 schema，返回操作该集合的句柄而不是集合内容。名字未在 collections 下声明、或名字存在但种类不符，在调用集合方法之前就同步抛 [WmUsageError](06-errors.md)。同一个 Run 内对同一个名字多次取句柄返回同一个对象。没有列出全部集合名的方法。

### batch(fn)

**作用**：把回调内登记的全部变更合成一个批次，一次原子提交。

**输入类型**：fn: (b: BatchBuilder) => void。回调参数的类型见 [BatchBuilder](05-elements.md)。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await memory.batch((b) => {
  b.graph("incident_graph").add_node(node);
  b.list("evidence_list").append("ev-1", { note: "网关时间快 31 秒" });
});
~~~

回调内不发请求，也没有读方法。登记窗口与求值规则见下面的批次。

### close_run_memory()

**作用**：释放当前 Run 的租约并清理运行态。

**输入类型**：无。

**返回值类型**：Promise<object>，成功响应可回一次性的状态标记。

**示例**：

~~~js
await memory.close_run_memory();
~~~

只有发起本 Run 的那个 PoA cell 可以调用它，子 agent、共享方和观测者调用一律不受理。关闭后旧句柄与未提交的变更全部失效，已生成的快照不受影响。

## 作用域与实例

Run 是 Working Memory 的作用域单位，由一个 PoA cell 发起，覆盖该 cell 沿派发链派出的全部子 agent。它与 cell、turn、thread 都不等同，同一个 thread 可以参与多个 Run，多个 Run 并发活跃是预期状态，各持独立的作用域与独立实例。

一个 Run 对应一个实例。实例是 id 唯一性与快照序号的作用域：节点 id、边 id、list 的 item_id 在整个实例内唯一，单个集合不是唯一性边界。

关闭是推荐流程，不构成清理必然发生的保证。Run 结束没有可依赖的宿主信号，close_run_memory 也不保证一定被调用。运行代际标记只用于拒绝旧代际的请求，不自行检测 Run 结束，也不回收内存，清理仍需宿主的生命周期机制。宿主关闭前先停止接收新的变更、等待在途提交获得确定结果，再执行清理。

它关闭的是 Working Memory 的 Run 作用域，与 TUI 用来关闭后台 PoA 会话的斜杠命令无关。

## 注入与可用范围

包资源校验、schema 注入与 WM 连接都在业务 JS 执行前做完。当前只有 app-server 的 thread/codeMode/exec 启动的 PoA cell 有 memory；注册为 poa 工具的程序、普通 turn 和子 agent 当前没有 memory 入口。

可选工具组 memories 和 [tools](../13-tools-manager.md) 枚举里的 memory 工具类别是模型可见的工具，本页的 memory 是只有程序能调的沙箱全局，两者同名不同物。

WM schema 与 input_schema 是两份不同的 schema，用途与可见性都不同：后者注册为 poa 工具时成为对模型可见的参数说明，WM schema 对模型不可见。缺这份声明时，四个访问器在第一次调用处同步抛 WmUsageError。

## 版本与持久化

版本号由 SDK 携带。四类集合的写方法每次都强制带期望版本做 CAS 比较，该值不由调用方传入，方法签名里没有 revision 参数。它出现在 [CommitResult](05-elements.md)、[BPage](05-elements.md) 和快照侧的返回结果上。owner 路径的首写基线是 0，第一次写入前无需查询版本。版本只向前推进，序号中间允许有间隔。

当前使用内存存储。进程退出后运行态、快照和变更历史都会丢失。要跨进程保留数据，保存成功导出的快照文件，见 [memory.snapshot](07-snapshot.md)。

旧运行代际返回 STALE_OWNER 时，调用方应停止旧执行，不能继续用旧句柄重试。

## 集合名

集合名取 schema 的 collections 下的声明名。四类集合共用一个命名空间，不得重名。集合名允许非 ASCII，含连字符、空格或中文都可用，调用时写 schema 中的原始名称。本文档对函数和方法的 snake_case 约定不约束集合名。

## 批次

回调同步执行期间，memory 上的普通集合句柄进入登记态，在回调里误用它们的读写方法同步抛 WmUsageError。回调窗口内调用 memory 的 snapshot 方法、close_run_memory 或嵌套 batch 一律不受理，三者同样同步抛 WmUsageError。窗口在回调同步返回时结束，之后普通集合句柄恢复正常调用。

批次专用句柄上的写方法同步执行，只登记意图，不返回提交结果；提交结果只由 batch 本身返回。

每次立即提交都是一次跨进程往返，并推进一次版本。写 20 个节点，立即提交要 20 次往返；batch 只要 1 次，整批原子，成功时只推进一次版本。

提交须同时满足批次大小限制和传输帧大小限制，这是两道独立的闸门。批次上限由宿主能力声明提供，超出时返回 RESOURCE_EXHAUSTED。传输帧上限由宿主和传输层决定，当前实现约为 64 MiB；超出帧上限是传输层失败，不在 [WmErrorCode](06-errors.md) 的清单里，SDK 错误码不保证可用。SDK 不自动拆批，自动拆分会破坏整批原子性；需要拆分时由调用方处理，拆分后的批次之间不保证原子性。不含任何变更的批次在后端返回 INVALID_ARGUMENT，不推进版本。

批次中三类判定的求值时点不同：

- 谓词按最终状态求值，与书写顺序无关，包括多重边唯一性、边端点存在与端点类别相容、删点不留悬空边、unique_property 以及树的五条判定。
- 前置条件按该操作之前的前缀状态逐项求值，包括 id 与 item_id 不存在、replace 与 delete 的目标存在、put 的 mode 比对、set_parent 的期望父节点。
- 生成器按书写顺序求值，只有 [BList](04-blist-bmap.md) 的 append 分配 position 属于这一类。

与操作顺序无关只对谓词成立。批次里登记的变更是有序序列，同一组操作换个书写顺序可能得到不同结果。候选状态在批次内部允许暂时违反任何谓词，先登记边再登记端点、一个批次删一棵子树、在现有树上插一层新根都靠这一条成立。批次级的净效果为空不做检测。

## 完整示例

下面这段串起一次完整动线：取句柄、一次原子提交、判定提交结果、版本冲突后确认再重发、翻完全部分页。

~~~js
const incidentGraph = memory.graph("incident_graph");
const evidenceList = memory.list("evidence_list");

const node = new BNode({
  id: "node-1",
  node_class: "Task",
  properties: { title: "排查移动端 401" },
});
const changes = (b) => {
  b.graph("incident_graph").add_node(node);
  b.list("evidence_list").append("ev-1", { note: "网关时间快 31 秒" });
};

const committed = await memory.batch(changes);
if (!committed.ok) {
  const probe = await evidenceList.query({ target: "list", filter: { item_id: "ev-1" } });
  if (probe.items.length === 0) await memory.batch(changes);
}

const condition = { target: "nodes", sort: { title: "asc" } };
let page = await incidentGraph.query(condition);
const ids = page.items.map((item) => item.id);
while (page.next_cursor) {
  page = await incidentGraph.query({ ...condition, cursor: page.next_cursor });
  ids.push(...page.items.map((item) => item.id));
}
text(ids);
~~~

ok 为 false 只有版本冲突这一个含义，处理方式是重新读取、重算变更再提交。上面先用 query 确认这一批是否已落地，未落地才重发。按原期望版本盲目重发是安全的，但首次已落地必然再回一次冲突，没有信息量。

## 当前未确定

WM schema 怎么写、它在包里声明在哪，本指南尚未覆盖，当前由接入侧和宿主侧提供。

批次上限的具体字段名尚未确定。close_run_memory 的完整返回形状、重复关闭的结果、以及不受理时返回什么都尚未确定，不要围绕某个特定异常或错误码写兜底分支。

上面示例里 query 的 filter 与 sort 形状同样尚未确定，见 [BGraph](02-bgraph.md)，应把这几处集中封装在一处。

## 相关文档

- [BGraph](02-bgraph.md)：图的节点与边读写、一跳邻接和查询。
- [BTree](03-btree.md)：树的读写、约束模型和层级查询。
- [BList 与 BMap](04-blist-bmap.md)：有序项追加与键值读写。
- [BNode、BEdge 与读写返回类型](05-elements.md)：构造类型、批次构建器、提交结果和分页体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
- [memory.snapshot](07-snapshot.md)：快照创建、列举和只读查询。
