# WmError、WmUsageError、WmRequestError、WmTransportError 与 WmErrorCode

本页描述 Working Memory 的异常类层级、错误码和捕获方式。登记、载入与四类集合句柄的失败分成三类：不带错误码的本地使用错误，带错误码的拒绝，以及请求发出之后的传输与帧失败。下面的示例以 incidentGraph 指代一个 graph 集合的句柄，以 node 和 edge 指代已构造好的 [BNode](05-elements.md) 与 [BEdge](05-elements.md)。

带码的拒绝有两个来源，catch 到的形状一样，产出它的位置不同。四类集合的读写全部在 SDK 的本地影子状态上执行，它们的拒绝由 SDK 在调用当场同步抛出，没有跨进程调用参与；register_schema 与 load_data 经工具边界到达 WM，它们的拒绝由 WM 带码返回。所以分流的判据是有没有 code 字段，不是请求有没有发出。

### WmError

**作用**：Working Memory 异常的基类，不直接抛出。

**类型定义**：

~~~ts
interface WmError extends Error {
  message: string;
}
~~~

它只用于把三个子类归到同一个 catch 分支。四个异常类当前不作为沙箱全局暴露，分流时按 code 是否存在判断，不要写 instanceof，也不要按 message 的内容判断。

**示例**：

~~~js
try {
  await incidentGraph.add_node(node);
} catch (error) {
  text(error.message);
}
~~~

### WmUsageError

**作用**：表示 SDK 在本地拒绝了不符合使用约定的调用，而且这次拒绝没有错误码。

**类型定义**：

~~~ts
interface WmUsageError extends WmError {
  message: string;
}
~~~

它没有错误码，catch 到的对象上没有 code 字段，这是与 WmRequestError 的区分点。两处本地失败抛这个类：访问句柄上未声明的字段；构造 [BEdge](05-elements.md) 时缺少端点。集合方法的其余本地拒绝都带码，走 WmRequestError。

**示例**：

~~~js
// mem 是 Working Memory 的句柄，取法见“声明、登记与句柄”
try {
  await mem.tsak_tree.add_child("plan-1", node);
} catch (error) {
  text("集合名未声明");
}
~~~

### WmRequestError

**作用**：表示一次带错误码的拒绝，无论这个码是 SDK 在本地抛出的，还是 WM 带回的。

**类型定义**：

~~~ts
interface WmRequestError extends WmError {
  code: WmErrorCode;
  details?: object;
}
~~~

约束失败一律抛这个类，没有一条拒绝走返回值。details 用于诊断。字段级定位只出现在 SDK 本地抛出的那批码上，定位路径从本次调用的参数算起，图与树形如 element.props.title，list 与 map 形如 value.title；WM 带回的码里没有字段级定位。

details 的整体形状不作为调用依据，可据以分支的只是明文规定必返的那几项，它是一份白名单，而不是“凡 details 里有的都能读”。当前在册的是：CONSTRAINT_VIOLATION 与 STRUCTURE_CONFLICT 的 reason、CONSTRAINT_VIOLATION 在三条内置检查上的 constraint_id、RESOURCE_EXHAUSTED 的 reason 与 metric、RESULT_TOO_LARGE 的 max_items 与 actual_at_least、SCHEMA_TOO_LARGE 与 DATA_TOO_LARGE 的 limit 与 actual_at_least、以及 INVALID_ARGUMENT 与 INVALID_STRUCTURE 的字段级定位路径。本页各处写“details 里返回究竟是哪一条”时说的都是这几项，不是允许按别的键写分支。

WM 带回的九个码里只有两个带内容：SCHEMA_TOO_LARGE 与 DATA_TOO_LARGE 的 details 恰好是 { limit, actual_at_least }，按 UTF-8 字节计。其余七个恒为空对象。INVALID_SCHEMA 与 INVALID_DATA 都不带 rule、不带字段级定位，也没有说明定位是否精确的那一位。

CONSTRAINT_VIOLATION 的两个字段各管一批约束。reason 管由 schema 声明带来的那批；constraint_id 管三条不带参数的内置检查，取值是 wm.edge_endpoints_exist（边的端点必须存在）、wm.node_delete_no_dangling_edges（删节点不留悬空边）、wm.tree_parent_exists（树节点的父节点必须存在）。这三条有意不进 reason 的取值集合，constraint_id 因此是它们唯一的判别手段，缺它只能看到一个 CONSTRAINT_VIOLATION，分不出是哪一条；声明带来的那批则不占 constraint_id 这个字段。

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

它不表示带码的拒绝，那是 WmRequestError；也不表示不带码的本地拒绝，那是 WmUsageError。因为没有任何一侧的业务判定参与，它没有错误码，catch 到的对象上没有 code 字段。两类成因：超出 IPC 帧上限（当前约为 64 MiB）、连接断开。它以 Promise 拒绝的形式交付。信封本身坏掉不走这个类，那是一次结果未知的带码失败，见下面的失败的分流。

只有整份提交会跨进程，所以只有提交点可能抛出这个类；四类集合的方法在本地执行，不经过传输。整份文档另受 5 MiB 上限约束，超限时得到的是 DATA_TOO_LARGE 这个带码拒绝，不是帧上限。

**示例**：

~~~js
try {
  await incidentGraph.add_node(node);
} catch (error) {
  if (error.code === undefined) text("本地拒绝或传输失败");
}
~~~

### WmErrorCode

**作用**：标识 WmRequestError 的拒绝原因。

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

上面是 WmErrorCode 当前的全部取值，逐码含义见下面的错误码。其中有十个当前版本不会出现，各自的条目里都标了出来，不必为它们写分支。当前版本没有这个集合之外的错误码；后续版本可能增补，所以收到未列出的 code 应按未知值记录，不要写进分支。

剩下十四个按产出位置分成两批。WM 带回九个：KIND_NOT_IMPLEMENTED、SCHEMA_TOO_LARGE、INVALID_SCHEMA、SCHEMA_ALREADY_REGISTERED、SCHEMA_NOT_REGISTERED、DATA_TOO_LARGE、INVALID_DATA，加上工具边界的 INVALID_ARGUMENT 与 INTERNAL_ERROR。SDK 在本地同步抛出五个：INVALID_STRUCTURE、STRUCTURE_CONFLICT、CONSTRAINT_VIOLATION、NOT_FOUND、RESULT_TOO_LARGE，以及本地那一层的 INVALID_ARGUMENT。INVALID_ARGUMENT 因此是同名两层，区分见下面该码的条目。

**示例**：

~~~js
try {
  await incidentGraph.add_node(node);
} catch (error) {
  if (error.code === "STRUCTURE_CONFLICT") text("这个 id 已经存在");
}
~~~

## 失败的分流

本地使用错误没有 code 字段，带码的拒绝有。写方法的失败全部走异常，返回值不承载失败。

第三类落在上面两类之外：请求发出之后的传输与帧失败抛 WmTransportError，它同样不带码，成因见上面该类的条目。没有 code 字段这一点本地使用错误与它相同，要进一步分辨得看失败是不是发生在整份提交那一步。

WM 带回的码经一个信封跨传输边界到达，产出这一侧已经实现，code 与 details 可以作为分支依据。形状是一个裸字符串，不是 Error 对象，所以取 e 本身、不要取 e.message：字节位置 0 是固定前缀 `WMERR/1:`，随后是一段紧凑 JSON，内含 code、message 和 details，三个键恒在，details 没有内容时是空对象。SDK 只解析带这个前缀的字符串，宿主自己抛的普通字符串原样当作普通 JS 异常，不会按“看起来像 JSON”推断。

前缀存在而 JSON 无法解析时抛 WmRequestError、code 取 INTERNAL_ERROR、details.cause 标 malformed_error_envelope；成功返回的形状坏了同理，cause 标 malformed_response。两种都表示结果未知，不表示没发生，SDK 不自动重试也不自动回滚。

message 不可据以分支。它最多 1024 个 UTF-8 字节，含截断标记在内，只含结构性类型名、行列号、计数、长度和实现方自己的字段名，不含传进去的字符串值、自创的键名、由这些键名拼成的路径，也不含解析器的原生文字。

各方法条目列出的是该方法的主要特定失败，不是完整错误码全集。

## 本地使用错误的抛出时点

这些失败都在调用点同步抛出，尚未创建可供 catch 方法处理的 Promise。可用 try/catch 包住调用或 await。

四类集合的读写方法实现为非 async 的、返回 Promise 的普通函数，同步段先做参数校验，之后才进入 Promise，同步段抛出时链尾的 catch 方法捕获不到。

## 错误码

声明与载入一类由 register_schema 与 load_data 这两个调用产出，全部由 WM 带回，新写一个程序时最先遇到的是它们：

- KIND_NOT_IMPLEMENTED：kind 取了 "LTM"。当前只受理 "WM"，这是一次确定性拒绝，不会静默接受，改 kind 之外没有别的处理方式。message 只说明当前受理哪个 kind，不回显任何声明或数据内容。
- SCHEMA_TOO_LARGE：声明原文的字节数超过上限，缺省 5 MiB，可由运维配置。量的是 schema 这一个字符串自身的 UTF-8 字节数，不是 JS 的 String.length，也不是整个入参 JSON 转义之后的长度；边界是闭区间，恰好等于上限受理，多一个字节拒。不可重试，只能改声明。details 必返 limit 与 actual_at_least。
- SCHEMA_ALREADY_REGISTERED：同一个 kind 第二次登记声明。第二次一律拒绝，不覆盖也不合并，提交一份完全相同的声明同样被拒。message 只提 kind，不回显已登记的那份声明的任何内容，`$id` 也不回显：WM 不读声明内部的键，而且同一个桶可能被另一个 Run 占着，见[作用域与实例](01-memory.md)，报出存着的那份声明会把另一个 Run 的内容泄出来。
- SCHEMA_NOT_REGISTERED：该 kind 下还没有登记过声明，而这个数据操作要求先登记。读与删同样走这条拒绝：什么都没声明时的答案不是空结果。message 只说明 kind 与“必须先登记”，不回显任何声明内容，也不说明登记过什么。
- INVALID_DATA：整份载入的数据不是合法 UTF-8、不是可解析的 JSON、最外层不是对象，或者第一份文档之后还跟着第二份。这些连同字节上限就是输入卫生的全部内容，所以这个码不表示数据与声明相符或不符 —— WM 不解析声明，也不按声明校验数据，数据与声明的符合性由 SDK 在发出前检查。message 只含结构性元数据，例如出错的行列与类型名，不含作者的键名与值。
- DATA_TOO_LARGE：整份载入的数据超过上限，缺省 5 MiB，可由运维配置，度量与边界同 SCHEMA_TOO_LARGE。不可重试，只能减小数据。details 必返 limit 与 actual_at_least 两项。

输入与状态断言一类判的是调用内容和它对状态的假设。除 INVALID_SCHEMA、SCHEMA_CONFLICT 以及 INVALID_ARGUMENT 的工具边界那一层之外，其余都由 SDK 在方法调用当场同步抛出：

- INVALID_ARGUMENT：同名两层，两层都是确定性失败，资源状况改变也不会成功。SDK 本地那一层管集合方法的用法：字段缺失、类型不符、参数组合非法、类别未在所选集合声明、节点 id 与边 id 与 list 的 item_id 超过 256 个 UTF-8 字节、limit 取 0 或负数或非整数或超过 10000。它带字段级定位路径，根是本地方法的参数根。WM 的工具边界那一层只说入参不符合工具的输入形状，例如 schema 或 data 不是字符串、缺 kind，它的 details 是空对象，没有字段级定位。
- INVALID_STRUCTURE：集合不存在、集合种类与操作不符，写入目标、边端点、树父节点或邻接起点属于另一集合，以及 list 的 value 不符声明的 item_type、map 的 key 或 value 不符声明的类型。item_id 不在这一类里：它不由 item_type 约束，它的类型错判给 INVALID_ARGUMENT。
- STRUCTURE_CONFLICT：断言“目标不存在”而它实际存在，按调用时的状态求值，包括元素或条目的 id 已存在、put 以 create 模式遇到已存在的 key。set_parent 的期望父节点与实际当前父不符也取这个码，details 的 reason 为 tree_parent_mismatch，并返回实际父节点。反方向的那一支不在这里：put 以 replace 模式遇到不存在的 key 属于“目标不存在”，取 NOT_FOUND。
- CONSTRAINT_VIOLATION：图、树或 schema 约束在本次写入之后不满足。
- INVALID_SCHEMA：声明不是一份完整的 JSON 对象，由 register_schema 返回，不由单次集合读写的参数触发。判据与 INVALID_DATA 同一套：合法 UTF-8、可解析 JSON、最外层是对象、后面不跟第二份文档。它不判声明的内容对不对 —— WM 不解析声明，所以收到这个码不等于声明的形状有问题，只等于那串文本不是一份 JSON 对象。details 是空对象：没有 rule，也没有指向声明内部的定位路径。不可重试，改声明。
- SCHEMA_CONFLICT：与 schema 切换有关。当前版本不会出现。

存在性与授权一类不区分不存在和无权，以免被用来探测：

- NOT_FOUND：已授权集合内的元素不存在，四类集合统一用这个码，覆盖 graph 的节点与边、tree 的节点、list 的 item_id、map 的 key，也含 put 以 replace 模式遇到不存在的 key。它只用于“目标必须存在”这一类前置条件，也就是写入与 set_parent；读路径取不到一律是合法空结果，返回空页或 null，不取这个码。
- NOT_FOUND_OR_DENIED 与 FORBIDDEN：作用域、实例、集合或可见性层面的找不到与无权访问。元素级的“目标不存在”不在这两个码里，一律取 NOT_FOUND。不得利用两者的错误差异探测未授权对象。两个码当前版本都不会出现。
- UNAUTHENTICATED：可信认证上下文缺失、过期或无法验证，由宿主刷新。当前版本不会出现。
- STALE_OWNER：请求携带的运行代际已失效。当前版本不会出现。

资源与能力一类由请求规模或部署形态决定，重试同一个请求不会变好：

- RESULT_TOO_LARGE：一次读的命中项数超过上限，由 SDK 在本地读的当场同步抛出。它不可重试，同一次读换个时候做还是同样的结果，处理方式是收窄 filter 或先减少数据；指定 limit 绕不过它，因为上限针对的是命中项数而不是返回项数。details 必返 max_items 与 actual_at_least 两项，前者是上限 10000，后者恒为 10001：数到这个界限就停，不再求精确总数，所以它只表示已确认超过上限，不是实际命中数，不得当作计数展示。上限的取值见[默认序与结果规模](01-memory.md)。
- RESOURCE_EXHAUSTED：只用于“同一个请求在资源恢复后可能成功”的那一类，三种成因是单实例的内存驻留上限、进程名额或进程预算用尽、读版本名额或读工作预算不足。请求大小超限不在这里，那是请求自身的形状问题，整份文档超限取 DATA_TOO_LARGE，其余取 INVALID_ARGUMENT。details 返回 reason 与 metric 两项，前者说明是哪一类资源，后者说明是哪一个上限；重试条件按 metric 区分，不是一概可以重试。当前版本不会出现。
- UNSUPPORTED_ACTION、UNSUPPORTED_FEATURE 与 UNSUPPORTED_VERSION：动作未知、字段取值当前不受理、或声明格式与方言的版本不受支持，属于请求与声明本身的问题，改客户端或改声明。三个码当前版本都不会出现。

运行期故障一类，程序侧一般只需记录并把失败如实上报：

- WM_BUSY：实例当前无法受理请求。当前版本不会出现。
- INTERNAL_ERROR：未归类的内部失败。WM 侧取这个码的情形是工具无法编码自己的输出；SDK 侧在信封或成功返回的形状坏了时也取它，并在 details.cause 里标出是哪一种。结果未知时按下面的确认流程处理。

类别定位用集合名和类别名共同完成。类别未在所选集合声明返回 INVALID_ARGUMENT，其他集合存在同名类别也不回退。不可见对象先按授权规则返回 NOT_FOUND_OR_DENIED，不通过归属错误泄漏其存在性；这一步在当前版本不会触发，程序获得的是 owner 侧句柄，该码不会出现。普通查询的 filter 只在所选集合内筛选，未命中仍返回合法空结果，不改报集合归属错误。

## 提交结果未知时的确认流程

这一节只适用于整份提交那一步。四类集合的方法在本地执行、不跨进程，没有结果未知这个中间态；连接中断、超时和进程退出只会落在提交点上，而它们都只表示提交结果未知，不表示提交失败。

整份提交是替换而不是追加，所以重发同一份文档与只发一次等效，不会因为上一次其实已经生效而撞上 STRUCTURE_CONFLICT 或 NOT_FOUND。这是整份提交与元素级提交的主要差别：元素级提交必须先重读再决定，整份提交可以直接重发。

重发之后仍确认不了的是本地影子状态与 WM 里那份文档是否一致 —— 作者面没有整份读取的入口。SDK 不提供幂等键，不自动重试、也不自动回滚；一个 batch 在提交之前崩溃时这一批写全部丢失，影子状态里的改动不会被补发。该 turn 已结束或 WM 进程已退出时，那份文档连同声明一起丢弃，不能按上述办法重发。

## 相关文档

- [声明、登记与句柄](01-memory.md)：集合声明、登记与载入、句柄，以及默认序与结果规模。
- [CommitResult](05-elements.md)：提交结果、读结果体和构造类型。
