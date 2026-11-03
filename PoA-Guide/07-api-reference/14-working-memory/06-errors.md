# WmError、WmUsageError、WmRequestError、WmTransportError 与 WmErrorCode

本页描述 Working Memory 的异常类层级、错误码和捕获方式。[memory](01-memory.md) 及四类集合句柄的失败分成三类：请求尚未发出的本地使用错误，请求已发出、由后端拒绝的请求错误，以及请求已发出之后的传输与帧失败。下面的示例以 incidentGraph 指代一个 graph 集合的句柄，以 node 和 edge 指代已构造好的 [BNode](05-elements.md) 与 [BEdge](05-elements.md)。

### WmError

**作用**：Working Memory 异常的基类，不直接抛出。

**类型定义**：

~~~ts
interface WmError extends Error {
  message: string;
}
~~~

它只用于把三个子类归到同一个 catch 分支。四个异常类当前不作为沙箱全局暴露，分流时按 code 是否存在和 message 的内容判断，不要写 instanceof。

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

它没有后端错误码，catch 到的对象上没有 code 字段，这是与 WmRequestError 的区分点。四处本地失败抛这个类：取集合句柄时集合名未在 schema 顶层 properties 下声明、或声明成了别的种类；在 batch 回调窗口内误用 memory 上的普通集合句柄；在回调窗口内调用 memory 的 snapshot 方法、close_run_memory 或嵌套 batch；构造 [BEdge](05-elements.md) 时缺少端点。

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

约束失败一律抛这个类。唯一的例外是 REVISION_CONFLICT，它作为 [CommitResult](05-elements.md) 返回且 ok 为 false，不作为异常抛出。details 用于诊断，在协议层可定位到字段，图与树形如 changes 下的 element.props.title，list 与 map 形如 changes 下的 value.title。

**示例**：

~~~js
try {
  await incidentGraph.add_edge(edge);
} catch (error) {
  text(error.message);
}
~~~

### WmTransportError

**作用**：表示请求已经发出之后的传输或帧失败。

**类型定义**：

~~~ts
interface WmTransportError extends WmError {
  message: string;
}
~~~

它不表示 WM 的业务拒绝，那是 WmRequestError；也不表示本地预检失败，那是 WmUsageError。因为没有后端业务判定参与，它没有错误码，catch 到的对象上没有 code 字段。三类成因：超出 IPC 帧上限（当前约为 64 MiB）、连接断开、信封不可解析。它以 Promise 拒绝的形式交付；在 batch 回调里发生时 batch 本身同样拒绝。

请求发出之前命中的任何一道预检不走这个类。SDK 本地预检与发出前的最终字节检查都抛 WmUsageError，判据是请求根本没上线。

**示例**：

~~~js
try {
  await incidentGraph.add_node(node);
} catch (error) {
  if (error.code === undefined) text("本地拒绝或传输失败");
}
~~~

### WmErrorCode

**作用**：标识 WmRequestError 的后端拒绝原因。

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
  | "PAGINATION_EXPIRED"
  | "INTERNAL_ERROR"
  | "UNSUPPORTED_ACTION"
  | "UNSUPPORTED_FEATURE"
  | "UNSUPPORTED_VERSION";
~~~

这十九个是协议当前的全集，逐码含义见下面的错误码。另有两个码不在这个集合里但在规范面有效：UNSUPPORTED_CAPABILITY 与 INSTANCE_UNRECOVERABLE 随后续阶段恢复，当前阶段没有可达输入，因此不列入上面的类型。除这两个之外协议没有集合外的错误码，收到未列出的 code 应按未知值记录，不要写进分支。

**示例**：

~~~js
const result = await incidentGraph.add_node(node);
if (!result.ok) text({ conflict_at: result.revision });
~~~

## 失败的分流

本地使用错误没有 code 字段，请求错误有。版本冲突既不走本地也不走异常，它经 CommitResult 返回。

还有一类落在这三者之外：请求发出之后的传输与帧失败抛 WmTransportError，它同样不带后端错误码，成因见上面该类的条目与 [memory](01-memory.md) 的批次。没有 code 字段这一点本地使用错误与它相同，要进一步分辨得看请求有没有发出。

code 与 details 怎么跨传输边界传到 JS 面尚未确定，两者都还不能作为已可用的能力依赖，也不能提前写成稳定的分支条件。调用方应优先按 message 记录错误，并保留原始异常和请求上下文。

各方法条目列出的是该方法的主要特定失败，不是完整错误码全集。任何副作用请求还可能因运行代际失效返回 STALE_OWNER，或遇到资源与后端故障；STALE_OWNER 不出现在任何方法条目里。

## 本地使用错误的抛出时点

这些失败都在调用点同步抛出，尚未创建可供 catch 方法处理的 Promise。可用 try/catch 包住调用或 await。

四类集合的读写方法实现为非 async 的、返回 Promise 的普通函数，同步段先做参数校验与登记态拦截，之后才进入 Promise，同步段抛出时链尾的 catch 方法捕获不到。batch 本身创建 Promise 的确切时点尚未确定，不要据回调内的同步抛出推定外层的 catch 方法一定无法捕获。外层统一写成 try/catch 包住 await memory.batch(...)，可同时处理同步抛出和 Promise 拒绝。

batch 回调内的任何异常都会丢弃整批登记内容，不会部分提交，异常原样外抛不包装；这覆盖被拦截的方法调用、参数校验失败和业务代码主动抛出三种情形。一个批次里只有下标最小的失败项会被返回。

## 错误码

版本一类只有一个码：

- REVISION_CONFLICT：写入、快照或 schema 切换的期望版本过期。处理方式是重新读取、重算变更再提交。它经 CommitResult 返回，不抛异常。

输入与状态断言一类判的是请求内容和它对状态的假设：

- INVALID_ARGUMENT：请求字段缺失、类型不符、参数组合非法，或类别未在所选集合声明。
- INVALID_STRUCTURE：集合不存在、集合种类与操作不符，已授权的写入目标、边端点、树父节点或邻接起点属于另一集合，以及 list 的 value 不符声明的 item_type、map 的 key 或 value 不符声明的类型。
- STRUCTURE_CONFLICT：对前缀状态的断言不成立，包括元素或条目的 id 已存在、put 的写入模式与 key 的存在性不匹配、set_parent 的期望父节点与实际当前父不符。
- CONSTRAINT_VIOLATION：图、树或 schema 约束在最终状态下不满足。
- INVALID_SCHEMA 与 SCHEMA_CONFLICT：与 schema 声明本身和 schema 切换有关，不由单次集合读写的参数触发。

存在性与授权一类不区分不存在和无权，以免被用来探测：

- NOT_FOUND：已授权集合内的元素不存在，四类集合统一用这个码，覆盖 graph 的节点与边、tree 的节点、list 的 item_id、map 的 key，也含 put 以 replace 模式撞上不存在的 key。它只用于「目标必须存在」这一类前置条件，也就是写入与 set_parent；读路径取不到一律是合法空结果，返回空页或 null，不取这个码。
- NOT_FOUND_OR_DENIED 与 FORBIDDEN：作用域、实例、集合或可见性层面的找不到与无权访问。元素级的「目标不存在」不在这两个码里，一律取 NOT_FOUND。不得利用两者的错误差异探测未授权对象。
- UNAUTHENTICATED：可信认证上下文缺失、过期或无法验证，由宿主刷新。
- STALE_OWNER：请求携带的运行代际已失效。应停止旧执行，不能继续用旧句柄重试。

资源与能力一类由请求规模或部署形态决定，重试同一个请求不会变好：

- RESOURCE_EXHAUSTED：批次、查询或存储资源达到上限，应缩小请求。details 回带 reason 与 metric 两样，前者说撞了哪一类资源，后者说撞了哪一个上限。
- PAGINATION_EXPIRED：读句柄不能继续用。details 的 reason 取两个值之一，compacted 表示句柄所钉的读版本已被压实、那段历史不再存在，ttl_expired 表示句柄到期而读版本可能仍在。两种成因都要停止使用旧句柄，详见下面的分页游标失效。
- UNSUPPORTED_ACTION、UNSUPPORTED_FEATURE 与 UNSUPPORTED_VERSION：动作未知、字段取值当前不受理、或声明格式与方言的版本不受支持，属于请求与声明本身的问题，改客户端或改声明。

后端与实例一类是运行期故障，程序侧一般只需记录并把失败如实上报：

- WM_BUSY：实例当前无法受理请求。
- INTERNAL_ERROR：未归类的内部失败。结果未知时按下面的确认流程处理。

类别定位用集合名和类别名共同完成。类别未在所选集合声明返回 INVALID_ARGUMENT，其他集合存在同名类别也不回退。不可见对象先按授权规则返回 NOT_FOUND_OR_DENIED，不通过归属错误泄漏其存在性。普通查询的 filter 只在所选集合内筛选，未命中仍返回合法空结果，不改报集合归属错误。

## 分页游标失效

读句柄不能继续用时返回 PAGINATION_EXPIRED，它与 REVISION_CONFLICT 是两件事：后者说你的期望版本过期了、重读即可，前者说这个分页句柄本身作废了。details 的 reason 取两个值，必须分开处理。compacted 表示句柄所钉的读版本已被压实，那段历史确实拿不回来了。ttl_expired 表示句柄到期而读版本可能仍在，它只说明旧句柄作废，不保证重新发起首次读仍然返回原内容或原版本。

两种成因下都要停止使用旧句柄。要重建一致的分页结果，做法是重新发起首次读，并且只组合同一个新读版本上的页，不得把新页拼到旧页上。旧页可以留作明确标识的过期展示缓存，不能当成已经恢复的一致视图。details 至少回带实例标识、原读版本，以及当前仍可用的最旧版本，也就是保留窗口的下界。

另有两类分页失败不走这个码：查询条件或作用域与句柄不匹配返回 INVALID_ARGUMENT，授权上下文失效返回 UNAUTHENTICATED 或 FORBIDDEN。SDK 不自动重试、也不改写句柄，恢复动作一律由调用方发起。

## 提交结果未知时的确认流程

超时或断线只表示提交结果未知。原 WM 进程与 Run 仍存在时，确认流程是先读当前版本，若它已超过本次请求的期望版本，再用 query 按节点 id、边 id、item_id 或 map key 确认落地的是否为本次提交，确认未落地才用新的版本基线重发。

按原期望版本盲目重发是安全的，但没有信息量，首次已落地必然回 REVISION_CONFLICT，此时不能把它读成写入失败，必须回到第一步。

SDK 不提供幂等键，也不会自动重试或自动合并变更，快照创建同样没有去重保证。WM 进程已退出时运行态与内存快照均已丢失，不能按上述流程重读原 Run，新 Run 的初始版本不能用于判断旧提交是否成功。

## 当前未确定

INVALID_SCHEMA、SCHEMA_CONFLICT、WM_BUSY、INTERNAL_ERROR、UNSUPPORTED_ACTION、UNSUPPORTED_FEATURE 和 UNSUPPORTED_VERSION 七个码的精确触发条件本指南尚未覆盖，上面给出的是它们所属的类别和调用方的处理方式。

## 相关文档

- [memory](01-memory.md)：集合访问、批量提交和 Run 关闭。
- [CommitResult](05-elements.md)：提交结果、分页体和构造类型。
