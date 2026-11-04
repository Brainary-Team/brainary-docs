# BList 与 BMap

[BList](04-blist-bmap.md) 与 [BMap](04-blist-bmap.md) 分别是一个 list 集合和一个 map 集合的句柄。句柄由 [memory](01-memory.md) 的 list 与 map 访问器取得，下面的示例以 evidenceList 和 retryByStatusCodeMap 指代它们。写方法成功提交时推进一次版本，读方法不推进；失败的分流方式见 [WmError](06-errors.md)。

这两类集合的 value 不用包装类。一个 list 只有一个 item_type，一个 map 只有一个 value_type，没有类别需要区分，写方法直接接收原值；声明为内联标量时，value 就是裸标量本身。

## BList

### append(item_id, value)

**作用**：在 list 末尾追加一个项。

**输入类型**：item_id: string，非空；value: unknown，符合该 list 声明的 item_type。

**返回值类型**：Promise<[CommitResult](05-elements.md)>，成功时 generated 带出服务端分配的 position。

**示例**：

~~~js
const result = await evidenceList.append("ev-1", { note: "网关时间快 31 秒" });
if (!result.ok) throw new Error("追加未提交，需重读后重新计算");
text(result.generated.position);
~~~

这是四类集合里唯一会带 generated 的方法，position 由服务端分配，不能自行指定。版本冲突时未分配位置，整个 generated 都缺省，读它前必须像上面那样先判 ok；冲突路径上直接读会拿到一个 TypeError，不是一个可处理的失败。item_id 在该操作之前的状态中已存在时返回 STRUCTURE_CONFLICT；value 不符声明的 item_type 返回 INVALID_STRUCTURE，item_id 不是非空字符串则返回 INVALID_ARGUMENT，两者不是同一个码。

### replace(item_id, value)

**作用**：按 item_id 替换一个项的完整 value。

**输入类型**：item_id: string；value: unknown，符合该 list 声明的 item_type。

**参数约束**：按 item_id 定位，不接收数组下标或 position，也不接收排序字段。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await evidenceList.replace("ev-1", { note: "复测后时钟已同步" });
~~~

replace 替换完整 value，item_id 与 position 保持不变。它与先删再追加是两种不同的改法，后者会让同一个 item_id 拿到新的 position，语义是删掉再追加到末尾。目标不存在时返回 NOT_FOUND。

### delete(item_id)

**作用**：按 item_id 删除一个项。

**输入类型**：item_id: string。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await evidenceList.delete("ev-1");
~~~

删除后该 item_id 不再命中，它占用过的 position 不会再次分配。目标不存在时返回 NOT_FOUND。

### BList.query(options)

**作用**：按条件分页查询当前 list 的项。

**输入类型**：options: object，必须含 target，可含 filter、sort、cursor 和 limit。

**参数约束**：target 取 list。structure 由句柄自动注入，不作为业务查询选项传入。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是 [BListItem](05-elements.md)。

**示例**：

~~~js
const page = await evidenceList.query({ target: "list", filter: { item_id: "ev-1" } });
if (page.items.length > 0) text(page.items[0].value);
~~~

按 item_id 读取单项仍返回分页体，没有匹配项时 items 是空数组，这不是错误。读取整个 list 需要翻完各页。

## BList 的项与顺序

list 的每个项有三样东西：由调用方提供的非空字符串 item_id、符合 item_type 的 value、由 WM 分配的非负整数 position，字段说明见 [BListItem](05-elements.md)。服务端不生成 item_id。list 没有按下标的读取方法，也没有独立的单项读方法，replace 与 delete 全靠这个键定位，调用方需要保存它。

当前只支持末尾追加、按 item_id 替换完整 value、按 item_id 删除。不支持任意位置插入、链表指针、自动去重或隐式重排。

position 分配是生成器，按书写顺序递增，取每个 list 的单调高水位加一。同一批次里两个 append 的相对次序由调用登记顺序决定，这一类不是顺序无关的。只有成功提交的追加才分配，失败或回滚不消耗；删除后不复用并允许出现间隔；在实例、快照和恢复后保持不变。

三条前置条件都按该操作之前的前缀状态求值，由此确定三种批次组合的结果：同一批次两次追加同一个 item_id 被拒绝，并指向第二项；先删再追加通过，该项拿到新的 position；先追加再替换通过。list 允许删除后重新使用同一个 item_id，而图与树对批次开始时已有的元素 id 不允许在同一批次中删除后重新创建。

未指定 sort 时按 position 升序。可排序字段是 item_type 已声明的一层标量字段，加上系统字段 position 和 item_id，并自动追加 item_id 为稳定末级排序键。字段值按声明类型比较，整数按数值，字符串按原始 Unicode 标量序列。缺失或显式 null 排在非空值之后，同值再按 item_id 排序。item_type 是内联标量时该 list 没有字段名，此时可排序的是 value 本身。排序只是读取视图，不改变 position，也不产生新版本。服务端可以按能力拒绝代价过高的排序。

list 侧的目标不存在使用 NOT_FOUND，图与树侧使用 NOT_FOUND_OR_DENIED，调用方按具体方法分别处理。

## BMap

### put(key, value, mode)

**作用**：按 key 写入一个完整 value。

**输入类型**：key: string | number | boolean，符合该 map 声明的 key_type；value: unknown，符合 value_type；mode: "create" | "replace" | "upsert"。

**参数约束**：三个参数全部必填，mode 必须明确给出。create 只允许新 key，replace 只允许已有 key，upsert 两者都允许。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await retryByStatusCodeMap.put(503, 1, "upsert");
~~~

模式不匹配一定报错，不会静默覆盖或静默创建。create 撞上已存在的 key、replace 撞上不存在的 key，都返回 STRUCTURE_CONFLICT；协议层在 details 里回带究竟是哪一种，details 的可用性见 [WmRequestError](06-errors.md)。key 或 value 不符 schema 声明返回 INVALID_STRUCTURE，整数 key 写成字符串属于这一类，不做隐式转换。

### delete(key)

**作用**：按 key 删除一条记录。

**输入类型**：key: string | number | boolean，符合该 map 声明的 key_type。

**参数约束**：按 key 定位，不需要传入 value 或写入模式。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await retryByStatusCodeMap.delete(503);
~~~

目标不存在时返回 NOT_FOUND。同一批次内重复删同一个 key 的去重规则与 [BGraph](02-bgraph.md) 的 delete_node 相同。

### BMap.query(options)

**作用**：按条件分页查询当前 map 的记录。

**输入类型**：options: object，必须含 target，可含 filter、sort、cursor 和 limit。

**参数约束**：target 取 map。structure 由句柄自动注入，不作为业务查询选项传入。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是 [BMapItem](05-elements.md)。

**示例**：

~~~js
const page = await retryByStatusCodeMap.query({ target: "map", filter: { key: 429 } });
if (page.items.length > 0) text(page.items[0].value);
~~~

按 key 精确查询用 filter 的 key，批量读取可以用已声明 value 字段的精确条件。单项查询也通过 items 取值，未匹配时 items 是空数组。

## BMap 的 key 与写入

map 的 key 类型由 schema 的 key_type 声明，比较规则与 value 字段说明见 [BMapItem](05-elements.md)。单项读取用 query 按 key 过滤，没有单独的读取方法。

map 不支持对 value 内字段做局部更新。写入用 put 替换完整 value，删除用 delete，不接受 set 与 unset。替换时须提供完整对象，包括必填字段，不能只传要改的那一个字段；对象值直接传入，不需要包装类。

mode 比对是前置条件，按该操作之前的前缀状态求值，由此确定三种批次组合的结果：先 create 再 replace 同一个 key 通过，第二项看到第一项的结果；两次 create 同一个 key 被拒绝；同一个 key 两次 upsert 不同 value 时后者生效。

未指定 sort 时按规范化 key 升序，比较按 key_type 声明的标量类型进行。整数 key 按数值比，9 排在 401 之前，不按字典序。也可以按声明的 value 字段排序，value 是内联标量时按 value 本身，并自动追加规范化 key 为稳定末级排序键。

map 的历史不另建一套接口，对快照使用同样的 query，见 [memory.snapshot](07-snapshot.md)。

## 当前未确定

BList 的 replace 与 delete、BMap 的 delete，以及两个 query 的选项形状，都尚未确定。上面给出的是这些方法需要表达的信息和当前的调用形态，还不能作为稳定调用签名。

一个批次包含多个 append 时 generated 如何对应各项尚未确定，批次里的 append 当前不能依赖这个字段。

list 与 map 侧的目标不存在用 NOT_FOUND、图与树侧用 NOT_FOUND_OR_DENIED，两者要不要统一尚未确定。

## 相关文档

- [memory](01-memory.md)：集合访问、批量提交和 Run 关闭。
- [BListItem 与 BMapItem](05-elements.md)：项类型、提交结果和分页体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
