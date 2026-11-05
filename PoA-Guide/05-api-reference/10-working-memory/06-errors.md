# WmError、WmUsageError、WmRequestError、WmTransportError 与 WmErrorCode

本页描述 Working Memory 的异常类层级、错误码和捕获方式。登记、灌入与四类集合句柄的失败分成三类：请求尚未发出的本地使用错误，请求已发出、由后端拒绝的请求错误，以及请求已发出之后的传输与帧失败。下面的示例以 incidentGraph 指代一个 graph 集合的句柄，以 node 和 edge 指代已构造好的 [BNode](05-elements.md) 与 [BEdge](05-elements.md)。

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

它没有后端错误码，catch 到的对象上没有 code 字段，这是与 WmRequestError 的区分点。两处本地失败抛这个类：访问句柄上未声明的字段；构造 [BEdge](05-elements.md) 时缺少端点。

**示例**：

~~~js
// mem 是 Working Memory 的句柄，取法见「声明、登记与句柄」
try {
  await mem.tsak_tree.add_child("plan-1", node);
} catch (error) {
  text("集合名未声明");
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

约束失败一律抛这个类，没有一条后端拒绝走返回值。details 用于诊断，在协议层可定位到字段，图与树形如 changes 下的 element.props.title，list 与 map 形如 changes 下的 value.title。

details 的整体形状不作为调用依据，可据以分支的只是协议明文规定必返的那几项，它是一份白名单而不是「凡 details 里有的都能读」。当前在册的是：CONSTRAINT_VIOLATION 与 STRUCTURE_CONFLICT 的 reason、CONSTRAINT_VIOLATION 在三条内置检查上的约束标识、RESOURCE_EXHAUSTED 的 reason 与 metric、RESULT_TOO_LARGE 的 max_items 与 actual_at_least、SCHEMA_TOO_LARGE 与 DATA_TOO_LARGE 的 limit 与 actual_at_least、INVALID_SCHEMA 的 rule 与定位路径及定位状态、以及字段级的定位路径。本页各处写「协议层在 details 里回带究竟是哪一条」时说的都是这几项，不是允许按别的键写分支。这条限制与下面那条「产出信封的那一侧还没有落地」是两件事：后者说当前连 code 都还拿不到，前者说即使拿到了也只有白名单内的项可依赖。

CONSTRAINT_VIOLATION 的两个位各管一批约束。reason 管由 schema 声明带来的那批；约束标识管三条不带参数的内置检查 —— 边的端点必须存在、删节点不留悬空边、树节点的父节点必须存在。这三条刻意不进 reason 的取值集合，约束标识因此是它们唯一的判别手段，缺它只能看到一个 CONSTRAINT_VIOLATION、分不出是哪一条；声明带来的那批则不占约束标识的位。

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

它不表示 WM 的业务拒绝，那是 WmRequestError；也不表示本地预检失败，那是 WmUsageError。因为没有后端业务判定参与，它没有错误码，catch 到的对象上没有 code 字段。三类成因：超出 IPC 帧上限（当前约为 64 MiB）、连接断开、信封不可解析。它以 Promise 拒绝的形式交付。

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
  | "KIND_NOT_IMPLEMENTED"
  | "SCHEMA_TOO_LARGE"
  | "SCHEMA_ALREADY_REGISTERED"
  | "SCHEMA_NOT_REGISTERED"
  | "INVALID_DATA"
  | "DATA_TOO_LARGE"
  | "INVALID_ARGUMENT"
  | "INVALID_SCHEMA"
  | "INVALID_STRUCTURE"
  | "STRUCTURE_CONFLICT"
  | "SCHEMA_CONFLICT"
  | "CONSTRAINT_VIOLATION"
  | "RESULT_TOO_LARGE"
  | "NOT_FOUND"
  | "NOT_FOUND_OR_DENIED"
  | "FORBIDDEN"
  | "UNAUTHENTICATED"
  | "STALE_OWNER"
  | "WM_BUSY"
  | "RESOURCE_EXHAUSTED"
  | "INTERNAL_ERROR"
  | "UNSUPPORTED_ACTION"
  | "UNSUPPORTED_FEATURE"
  | "UNSUPPORTED_VERSION";
~~~

这二十四个是协议当前的全集，逐码含义见下面的错误码。其中有十个本阶段不会出现，各自的条目里都标了出来，不必为它们写分支。另有两个码不在这个集合里但在规范面有效：UNSUPPORTED_CAPABILITY 与 INSTANCE_UNRECOVERABLE 随后续阶段恢复，当前阶段没有可达输入，因此不列入上面的类型。除这两个之外协议没有集合外的错误码，收到未列出的 code 应按未知值记录，不要写进分支。

**示例**：

~~~js
try {
  await incidentGraph.add_node(node);
} catch (error) {
  if (error.code === "STRUCTURE_CONFLICT") text("这个 id 已经存在");
}
~~~

## 失败的分流

本地使用错误没有 code 字段，请求错误有。写方法的失败全部走异常，返回值不承载失败。

还有一类落在这三者之外：请求发出之后的传输与帧失败抛 WmTransportError，它同样不带后端错误码，成因见上面该类的条目。没有 code 字段这一点本地使用错误与它相同，要进一步分辨得看请求有没有发出。

code 与 details 怎么跨传输边界传到 JS 面已经定下来，但产出这个信封的那一侧还没有落地，所以两者当前仍不能作为已可用的能力依赖，也不要提前写成稳定的分支条件。调用方应优先按 message 记录错误，并保留原始异常和请求上下文。

约定的形状是一个裸字符串，固定前缀 `WMERR/1:` 加一段紧凑 JSON，内含 code、message 和 details。SDK 只解析带这个前缀的字符串，宿主自己抛的普通字符串原样当作普通 JS 异常，不会靠「看起来像 JSON」去猜。前缀在而 JSON 解析不了时抛 WmRequestError、code 取 INTERNAL_ERROR、details.cause 标 malformed_error_envelope，语义是请求结果未知；SDK 不自动重试，是否重发由调用方按上面的确认流程决定。

各方法条目列出的是该方法的主要特定失败，不是完整错误码全集。任何副作用请求还可能遇到资源与后端故障。

## 本地使用错误的抛出时点

这些失败都在调用点同步抛出，尚未创建可供 catch 方法处理的 Promise。可用 try/catch 包住调用或 await。

四类集合的读写方法实现为非 async 的、返回 Promise 的普通函数，同步段先做参数校验，之后才进入 Promise，同步段抛出时链尾的 catch 方法捕获不到。

## 错误码

声明与灌入一类由 register_schema 与 load_data 这两个调用产出，写一个新程序时最先撞到的是它们：

- KIND_NOT_IMPLEMENTED：kind 取了 "LTM"。本阶段只受理 "WM"，这是一次确定性拒绝，不会静默接受，改 kind 之外没有别的处理方式。message 只说明本阶段受理哪个 kind，不回显任何声明或数据内容。
- SCHEMA_TOO_LARGE：声明原文的字节数超过上限，当前是 5 MiB。不可重试，只能改声明。details 必返 limit 与 actual_at_least。
- SCHEMA_ALREADY_REGISTERED：同一个 kind 第二次登记声明。第二次一律拒绝，不覆盖也不合并，提交一份完全相同的声明同样被拒。message 回显当前生效的那份声明的 $id —— 这是本册唯一会回显 $id 实际值的地方。
- SCHEMA_NOT_REGISTERED：该 kind 下还没有登记过声明，而这个数据操作要求先登记。读与删同样走这条拒绝 —— 什么都没声明时的答案不是空结果。message 只点出 kind 与「必须先登记」，不回显任何声明内容，也不说明登记过什么。
- INVALID_DATA：整份灌入的数据不是合法 UTF-8、不是可解析的 JSON、或者最外层不是对象。message 只含结构性元数据，例如出错的行列与类型名，不含作者的键名与值。
- DATA_TOO_LARGE：整份灌入的数据超过上限，当前是 5 MiB。不可重试，只能减小数据。details 取 limit 与 actual_at_least 两样。

输入与状态断言一类判的是请求内容和它对状态的假设：

- INVALID_ARGUMENT：请求字段缺失、类型不符、参数组合非法，或类别未在所选集合声明。单项或属性大小超过声明上限也取这个码，请求文本超过字节上限同样如此 —— 那是请求自身形状的问题，换个资源状况也不会成功。
- INVALID_STRUCTURE：集合不存在、集合种类与操作不符，已授权的写入目标、边端点、树父节点或邻接起点属于另一集合，以及 list 的 value 不符声明的 item_type、map 的 key 或 value 不符声明的类型、按未声明的字段排序。item_id 不在这一类里 —— 它不由 item_type 约束，它的类型错判给 INVALID_ARGUMENT。
- STRUCTURE_CONFLICT：断言「目标不存在」而它实际存在，按调用时的状态求值，包括元素或条目的 id 已存在、put 以 create 模式撞上已存在的 key。set_parent 的期望父节点与实际当前父不符也取这个码，details 的 reason 为 tree_parent_mismatch 并回带实际父节点。反方向的那一支不在这里：put 以 replace 模式撞上不存在的 key 是「目标不存在」，取 NOT_FOUND。
- CONSTRAINT_VIOLATION：图、树或 schema 约束在本次写入之后不满足。
- INVALID_SCHEMA：声明本身不符规定的形状，由 register_schema 返回，不由单次集合读写的参数触发。不可重试，改声明。details 必返三项：rule 是原因标识、取一个封闭的取值集合，可以与 code 一起作为稳定的分支依据；定位路径指向声明文档里出错的位置；定位状态说明那个路径是精确位置、还是因输出预算或文档未取全而省略。rule 的取值集合与定位状态的三个取值本指南尚未逐个列出。
- SCHEMA_CONFLICT：与 schema 切换有关。本阶段不会出现。

存在性与授权一类不区分不存在和无权，以免被用来探测：

- NOT_FOUND：已授权集合内的元素不存在，四类集合统一用这个码，覆盖 graph 的节点与边、tree 的节点、list 的 item_id、map 的 key，也含 put 以 replace 模式撞上不存在的 key。它只用于「目标必须存在」这一类前置条件，也就是写入与 set_parent；读路径取不到一律是合法空结果，返回空页或 null，不取这个码。
- NOT_FOUND_OR_DENIED 与 FORBIDDEN：作用域、实例、集合或可见性层面的找不到与无权访问。元素级的「目标不存在」不在这两个码里，一律取 NOT_FOUND。不得利用两者的错误差异探测未授权对象。两个码本阶段都不会出现。
- UNAUTHENTICATED：可信认证上下文缺失、过期或无法验证，由宿主刷新。本阶段不会出现。
- STALE_OWNER：请求携带的运行代际已失效。本阶段不会出现。

资源与能力一类由请求规模或部署形态决定，重试同一个请求不会变好：

- RESULT_TOO_LARGE：一次读的命中项数超过上限。它不可重试，同一个请求换个时候发还是同样的结果，处理方式是收窄 filter 或先减少数据；给 limit 绕不过它，因为闸门量的是命中数而不是返回数。details 必返 max_items 与 actual_at_least 两样，前者是上限，后者只表示已确认超过这个数、不是实际命中数，不得当作计数展示。上限的取值与那条「字节超了没有出口」的缝见 [默认序与结果规模](01-memory.md)。
- RESOURCE_EXHAUSTED：只用于「同一个请求在资源恢复后可能成功」的那一类，三种成因是单实例的内存驻留上限、进程名额或进程预算用尽、读版本名额或读工作预算不足。请求大小超限不在这里，那是请求自身的形状问题，取 INVALID_ARGUMENT 或走传输层失败。details 回带 reason 与 metric 两样，前者说撞了哪一类资源，后者说撞了哪一个上限；重试条件按 metric 分，不是一概可以重试。本阶段不会出现。
- UNSUPPORTED_ACTION、UNSUPPORTED_FEATURE 与 UNSUPPORTED_VERSION：动作未知、字段取值当前不受理、或声明格式与方言的版本不受支持，属于请求与声明本身的问题，改客户端或改声明。三个码本阶段都不会出现。

后端与实例一类是运行期故障，程序侧一般只需记录并把失败如实上报：

- WM_BUSY：实例当前无法受理请求。本阶段不会出现。
- INTERNAL_ERROR：未归类的内部失败。结果未知时按下面的确认流程处理。

类别定位用集合名和类别名共同完成。类别未在所选集合声明返回 INVALID_ARGUMENT，其他集合存在同名类别也不回退。不可见对象先按授权规则返回 NOT_FOUND_OR_DENIED，不通过归属错误泄漏其存在性 —— 这一步在本阶段空转，程序拿到的是 owner 侧句柄，该码不会出现。普通查询的 filter 只在所选集合内筛选，未命中仍返回合法空结果，不改报集合归属错误。

## 提交结果未知时的确认流程

连接中断、超时和进程退出都只表示提交结果未知，不表示提交失败。原 WM 进程与 Run 仍存在时，确认流程是用 query 按节点 id、边 id、item_id 或 map key 重读当前状态，看这一次写入的业务证据在不在，确认未落地才重发。

盲目重发不安全：重发一个已经落地的新增会撞上 STRUCTURE_CONFLICT，重发一个已经落地的删除会撞上 NOT_FOUND，而这两个码都不表示刚才那一次失败了。所以要先重读、再决定。

SDK 不提供幂等键，不自动重试、也不自动回滚。一串写入中途失败时，之前已经落地的那些留在原处，补齐还是回退由调用方决定。WM 进程已退出时运行态与内存快照均已丢失，不能按上述流程重读原 Run。

## 当前未确定

INVALID_SCHEMA 与 INTERNAL_ERROR 两个码的精确触发条件本指南尚未覆盖，上面给出的是它们所属的类别和调用方的处理方式。标了本阶段不会出现的那些同样没有逐条给出触发条件。

错误码表已按本阶段的码面对过一遍：声明与灌入那一组的六个码是补入的，另有十个码在条目里标了本阶段不会出现。那十个要不要从 WmErrorCode 的类型定义里去掉尚未确定，本页按保留处理。

## 相关文档

- [声明、登记与句柄](01-memory.md)：集合声明、登记与灌入、句柄，以及默认序与结果规模。
- [CommitResult](05-elements.md)：提交结果、读结果体和构造类型。
