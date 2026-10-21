# WmError、WmUsageError、WmRequestError 与 WmErrorCode

本页描述 Working Memory 的异常类层级、错误码和捕获方式。[memory](14-memory.md) 及四类集合句柄的失败主要分成两类：请求尚未发出的本地使用错误，和请求已发出、由后端拒绝的请求错误。还有一类落在这两者之外。超出传输帧上限时表现为传输层错误，它不是本页的本地使用错误，也不带后端错误码，见 [memory](14-memory.md) 的 batch。

下面的示例以 incidentGraph 指代一个 graph 集合的句柄，以 node 和 edge 指代已构造好的 [BNode](18-wm-elements.md) 与 [BEdge](18-wm-elements.md)。

### WmError

**作用**：Working Memory 异常的基类，不直接抛出。

**类型定义**：

~~~ts
interface WmError extends Error {
  message: string;
}
~~~

它只用于把两个子类归到同一个 catch 分支。三个异常类当前不作为沙箱全局暴露，分流时按 code 是否存在和 message 的内容判断，不要写 instanceof。

**示例**：

~~~js
try {
  await incidentGraph.add_node(node);
} catch (error) {
  text(error.message);
}
~~~

### WmUsageError

**作用**：表示 SDK 在本地拒绝了不符合使用约定的调用，请求尚未发出。

**类型定义**：

~~~ts
interface WmUsageError extends WmError {
  message: string;
}
~~~

它没有后端错误码，catch 到的对象上没有 code 字段，这是与 [WmRequestError](19-wm-errors.md) 的区分点。四处本地失败抛这个类：取集合句柄时集合名未在 schema 的 collections 下声明、或声明成了别的种类；在 batch 回调窗口内误用 [memory](14-memory.md) 上的普通集合句柄；在回调窗口内调用 memory 的 snapshot 方法、close_run_memory 或嵌套 batch；构造 [BEdge](18-wm-elements.md) 时缺少端点。

这些失败都在调用点同步抛出，尚未创建可供 catch 方法处理的 Promise。可用 try/catch 包住调用或 await。四类集合的读写方法实现为非 async 的、返回 Promise 的普通函数，同步段先做参数校验与登记态拦截，之后才进入 Promise，同步段抛出时链尾的 catch 方法捕获不到。batch 本身创建 Promise 的确切时点尚未确定，不要据回调内的同步抛出推定外层的 catch 方法一定无法捕获。外层统一写成 try/catch 包住 await memory.batch(...)，可同时处理同步抛出和 Promise 拒绝。

**示例**：

~~~js
try {
  await memory.list("incident_graph").append("ev-1", {});
} catch (error) {
  text("集合种类不符");
}
~~~

### WmRequestError

**作用**：表示请求已经发出，但 WM 拒绝了它或执行失败。

**类型定义**：

~~~ts
interface WmRequestError extends WmError {
  code: WmErrorCode;
  details?: object;
}
~~~

约束失败一律抛这个类。唯一的例外是 REVISION_CONFLICT，它作为 [CommitResult](18-wm-elements.md) 返回且 ok 为 false，不作为异常抛出。版本冲突走提交结果，其他约束失败走异常。

details 用于诊断，在协议层可定位到字段，图与树形如 changes 下的 element.properties.title，list 与 map 形如 changes 下的 value.title。code 与 details 怎么跨传输边界传到 JS 面尚未确定，两者都还不能作为已可用的能力依赖，也不能提前写成稳定的分支条件。调用方应优先按 message 记录错误，并保留原始异常和请求上下文。

各方法条目列出的是该方法的主要特定失败，不是完整错误码全集。任何副作用请求还可能因运行代际失效返回 STALE_OWNER，或遇到资源与后端故障；STALE_OWNER 不出现在任何方法条目里。

**示例**：

~~~js
try {
  await incidentGraph.add_edge(edge);
} catch (error) {
  text(error.message);
}
~~~

### WmErrorCode

**作用**：标识 [WmRequestError](19-wm-errors.md) 的后端拒绝原因。

**类型定义**：

~~~ts
type WmErrorCode =
  | "REVISION_CONFLICT"
  | "INVALID_ARGUMENT"
  | "INVALID_SCHEMA"
  | "INVALID_STRUCTURE"
  | "STRUCTURE_CONFLICT"
  | "SCHEMA_CONFLICT"
  | "CONSTRAINT_VIOLATION"
  | "NOT_FOUND"
  | "NOT_FOUND_OR_DENIED"
  | "FORBIDDEN"
  | "UNAUTHENTICATED"
  | "STALE_OWNER"
  | "WM_BUSY"
  | "RESOURCE_EXHAUSTED"
  | "BACKEND_UNAVAILABLE"
  | "INSTANCE_UNRECOVERABLE"
  | "INTERNAL_ERROR"
  | "UNSUPPORTED_ACTION"
  | "UNSUPPORTED_CAPABILITY"
  | "UNSUPPORTED_FEATURE"
  | "UNSUPPORTED_VERSION";
~~~

这是协议当前已定名的全集。其中走 PoA 调用面的是前十二个，后面几个与连接建立、后端故障或版本协商相关，程序侧一般只需记录。分页游标因读版本被压实而失效时另有一个错误码，它还没有定名，不在上面的清单里，见 [BPage](18-wm-elements.md)。

REVISION_CONFLICT 表示写入、快照或 schema 切换的期望版本过期，处理方式是重新读取、重算变更再提交，它经 CommitResult 返回，不抛异常。INVALID_ARGUMENT 表示请求字段缺失、类型不符、参数组合非法，或类别未在所选集合声明。

INVALID_STRUCTURE 表示集合不存在、集合种类与操作不符，已授权的写入目标、边端点、树父节点或邻接起点属于另一集合，以及 list 的 value 不符声明的 item_type、map 的 key 或 value 不符声明的类型。STRUCTURE_CONFLICT 表示对前缀状态的断言不成立，包括元素或条目的 id 已存在、put 的写入模式与 key 的存在性不匹配、set_parent 的期望父节点与实际当前父不符。CONSTRAINT_VIOLATION 表示图、树或 schema 约束在最终状态下不满足。

NOT_FOUND 表示已授权 list 的项或 map 的 key 不存在。NOT_FOUND_OR_DENIED 与 FORBIDDEN 表示目标不存在或当前调用方无权访问，不得利用两者的错误差异探测未授权对象。UNAUTHENTICATED 表示可信认证上下文缺失、过期或无法验证，由宿主刷新。STALE_OWNER 表示请求携带的运行代际已失效，应停止旧执行，不能继续用旧句柄重试。

RESOURCE_EXHAUSTED 表示批次、查询或存储资源达到上限，应缩小请求。UNSUPPORTED_CAPABILITY 表示当前后端未声明所需能力，例如分页依赖保留历史读版本、list 与 map 的写入依赖原子操作；具体部署是否可能返回此码，由绑定时要求的后端能力决定。

类别定位用集合名和类别名共同完成。类别未在所选集合声明返回 INVALID_ARGUMENT，其他集合存在同名类别也不回退。不可见对象先按授权规则返回 NOT_FOUND_OR_DENIED，不通过归属错误泄漏其存在性。普通查询的 filter 只在所选集合内筛选，未命中仍返回合法空结果，不改报集合归属错误。list 与 map 侧的目标不存在使用 NOT_FOUND，图与树侧使用 NOT_FOUND_OR_DENIED。两者要不要统一尚未确定，调用方按具体方法分别处理。

**示例**：

~~~js
const result = await incidentGraph.add_node(node);
if (!result.ok) text({ conflict_at: result.revision });
~~~

超时或断线只表示提交结果未知。原 WM 进程与 Run 仍存在时，确认流程是先读当前版本，若它已超过本次请求的期望版本，再用 query 按节点 id、边 id、item_id 或 map key 确认落地的是否为本次提交，确认未落地才用新的版本基线重发。按原期望版本盲目重发是安全的，但没有信息量，首次已落地必然回 REVISION_CONFLICT，此时不能把它读成写入失败，必须回到第一步。SDK 不提供幂等键，也不会自动重试或自动合并变更，快照创建同样没有去重保证。WM 进程已退出时运行态与内存快照均已丢失，不能按上述流程重读原 Run，新 Run 的初始版本不能用于判断旧提交是否成功。

batch 回调内的任何异常都会丢弃整批登记内容，不会部分提交，异常原样外抛不包装；这覆盖被拦截的方法调用、参数校验失败和业务代码主动抛出三种情形。一个批次里只有下标最小的失败项会被返回。

## 相关文档

- [memory](14-memory.md)：集合访问、批量提交和 Run 关闭。
- [CommitResult](18-wm-elements.md)：提交结果、分页体和构造类型。
