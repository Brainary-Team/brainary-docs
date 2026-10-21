# memory

[memory](14-memory.md) 是宿主注入的 Working Memory 句柄：取得四类集合的句柄、把多个写入合并成一次原子提交、
创建与读取快照、关闭当前 Run。

memory 的全部状态属于当前 Run。Run 是 Working Memory 的作用域单位，由一个 PoA cell 发起，覆盖该 cell 沿派发链
派出的全部子 agent；它与 cell、turn、thread 都不等同，同一个 thread 可以参与多个 Run。多个 Run 并发活跃是预期
状态，各持独立的作用域与独立实例。

连接由宿主完成：包资源校验、schema 注入与 WM 连接都在业务 JS 执行前完成，程序拿到的是已连接的 owner 侧 memory，
不调用也不需要 connect。当前只有 app-server 的 thread/codeMode/exec 启动的 PoA cell 有 memory；注册为 poa 工具的
程序、普通 turn 和子 agent 当前没有 memory 入口。

当前使用内存存储。进程退出后，运行态、快照和变更历史都会丢失；需要在进程退出后保留数据时，应保存成功导出的快照
文件，见 [memory.snapshot](20-wm-snapshot.md)。

版本号由 SDK 携带。四类集合的写方法每次都强制带期望版本做 CAS 比较，但那个值不由调用方传入，所以任何方法签名里
都没有 revision 参数；它只出现在 [CommitResult](18-wm-elements.md) 和 [BPage](18-wm-elements.md) 上。owner 路径
的首写基线是 0，调用方无需在第一次写入前查询版本。版本只会向前推进，序号中间允许有间隔。

集合名取 schema 的 collections 下的声明名，四类集合共用一个命名空间、不得重名。集合名允许非 ASCII，含连字符、
空格或中文的集合名可用，调用时应使用 schema 中的原始名称；本文档对函数和方法的 snake_case 约定不约束集合名。
可选工具组 memories 和 [tools](13-skills-manager.md) 枚举里的 memory 工具类别与本页的 memory 全局是不同的东西：
前者是模型可见的工具，后者是只有程序能调的沙箱全局。

### graph(name)

**作用**：取得一个 graph 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BGraph](15-bgraph.md)。

**示例**：

~~~js
const incidentGraph = memory.graph("incident_graph");
~~~

### tree(name)

**作用**：取得一个 tree 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BTree](16-btree.md)。

**示例**：

~~~js
const taskTree = memory.tree("task_tree");
~~~

### list(name)

**作用**：取得一个 list 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BList](17-blist-bmap.md)。

**示例**：

~~~js
const evidenceList = memory.list("evidence_list");
~~~

### map(name)

**作用**：取得一个 map 集合的句柄。

**输入类型**：name: string，取 schema 的 collections 下的声明名。

**返回值类型**：[BMap](17-blist-bmap.md)。

**示例**：

~~~js
const retryByStatusCodeMap = memory.map("retry_by_status_code");
~~~

四个访问器都是同步的，不发请求，返回的是用于操作该集合的句柄而不是集合内容。它们只查本地已校验的 schema：名字在
collections 下未声明、或名字存在但种类不符，在调用集合方法之前就同步抛 [WmUsageError](19-wm-errors.md)。同一个
Run 内对同一个名字多次取句柄返回同一个对象，句柄可以缓存。没有列出全部集合名的方法，集合名从包里的 schema 定义
获取。

### batch(fn)

**作用**：把回调内登记的全部变更合成一个批次，一次原子提交。

**输入类型**：fn: (b: [BatchBuilder](18-wm-elements.md)) => void。

**返回值类型**：Promise<[CommitResult](18-wm-elements.md)>。

**示例**：

~~~js
const result = await memory.batch((b) => {
  b.graph("incident_graph").add_node(node);
  b.list("evidence_list").append("ev-1", { note: "网关时间快 31 秒" });
});
~~~

回调内不发请求，也没有读方法：批次专用句柄上的写方法同步、只登记意图、不返回提交结果，提交结果只由 batch 本身
返回。回调同步执行期间，memory 上的普通集合句柄进入登记态，在回调里误用它们的读写方法会同步抛 WmUsageError；
回调窗口内调用 memory 的 snapshot 方法、close_run_memory 或嵌套 batch 同样不受理，三者都同步抛 WmUsageError。
窗口在回调同步返回时结束，之后普通集合句柄恢复正常调用。

每次立即提交都是一次跨进程往返并推进一次版本；一次写入 20 个节点，立即提交需要 20 次往返，batch 只需 1 次并保证
整批原子性，整批成功只推进一次版本。提交需同时满足批次大小限制和传输帧大小限制，两者的上限由宿主能力声明提供，
超出批次上限返回 RESOURCE_EXHAUSTED，超出帧上限可能表现为传输层错误。不能依赖 SDK 自动拆批：自动拆分会破坏整批
原子性，需要拆分时由调用方处理，拆分后的批次之间不保证原子性。不含任何变更的批次在后端返回 INVALID_ARGUMENT，
也不会推进版本。

批次中三类判定的求值时点不同。谓词按最终状态求值、与书写顺序无关，包括多重边唯一性、边端点存在与端点类别相容、
删点不留悬空边、unique_property 以及树的五条判定；前置条件按该操作之前的前缀状态逐项求值，包括 id 与 item_id 不
存在、replace 与 delete 的目标存在、put 的 mode 比对、set_parent 的期望父节点；生成器按书写顺序求值，只有
[BList](17-blist-bmap.md) 的 append 分配 position 属于这一类。所以与操作顺序无关只对谓词成立，批次里登记的变更
是有序序列，同一组操作换个书写顺序可能得到不同结果。候选状态在批次内部允许暂时违反任何谓词，这是先登记边再登记
端点、一个批次删一棵子树、在现有树上插一层新根能成立的前提。批次级的净效果为空不做检测。

### close_run_memory()

**作用**：释放当前 Run 的租约并清理运行态。

**输入类型**：无。

**返回值类型**：Promise<object>，成功响应可回一次性的状态标记。

**示例**：

~~~js
await memory.close_run_memory();
~~~

关闭后旧句柄与未提交的变更全部失效，已生成的快照不受影响。只有发起本 Run 的那个 PoA cell 可以调用它；子 agent、
共享方和观测者调用一律不受理。它关闭的是 Working Memory 的 Run 作用域，与 TUI 用来关闭后台 PoA 会话的斜杠命令
不是同一件事。

Run 结束没有可依赖的宿主信号，本方法也不保证一定被调用：运行代际标记用于拒绝旧代际的请求，不会自行检测 Run 结束
或回收内存，清理仍需宿主的生命周期机制。所以关闭属于推荐流程，不是清理必然发生的保证。旧运行代际返回 STALE_OWNER
时，调用方应停止旧执行，不能继续用旧句柄重试。

## 相关文档

- [BGraph](15-bgraph.md)：图的节点与边读写、一跳邻接和查询。
- [BTree](16-btree.md)：树的读写、约束模型和层级查询。
- [BList 与 BMap](17-blist-bmap.md)：有序项追加与键值读写。
- [BNode、BEdge 与读写返回类型](18-wm-elements.md)：构造类型、批次构建器、提交结果和分页体。
- [WmError](19-wm-errors.md)：异常类层级、错误码和捕获方式。
- [memory.snapshot](20-wm-snapshot.md)：快照创建、列举和只读查询。
