# BList 与 BMap

[BList](04-blist-bmap.md) 与 [BMap](04-blist-bmap.md) 分别是一个 list 集合和一个 map 集合的句柄。句柄是[声明](01-memory.md)里声明为 list 与 map 的那两个字段，下面的示例以 evidenceList 和 retryByStatusCodeMap 指代它们，取法是 `const evidenceList = mem.evidence_list;` 与 `const retryByStatusCodeMap = mem.retry_by_status_code;`。失败的分流方式见 [WmError](06-errors.md)。

这两类集合的 value 不用包装类。一个 list 只有一个 item_type，一个 map 只有一个 value_type，没有类别需要区分，写方法直接接收原值；声明为内联标量时，value 就是裸标量本身。

## BList

### append(item_id, value)

**作用**：在 list 末尾追加一个项。

**输入类型**：item_id: string，非空；value: unknown，符合该 list 声明的 item_type。

**返回值类型**：Promise<[CommitResult](05-elements.md)>，成功时返回体上带出 SDK 分配的 position。

**示例**：

~~~js
const result = await evidenceList.append("ev-1", { note: "网关时间快 31 秒" });
text(result.position);
~~~

这是四类集合里唯一会带 position 的方法，该值由 SDK 分配，不能自行指定；replace 与 delete 的返回体没有这个字段。追加失败时不分配位置，失败一律以异常抛出，所以得到返回值就意味着该字段存在。

item_id 已存在时返回 STRUCTURE_CONFLICT；value 不符声明的 item_type 返回 INVALID_STRUCTURE，item_id 不是非空字符串则返回 INVALID_ARGUMENT，两者不是同一个码。

### replace(item_id, value)

**作用**：按 item_id 替换一个项的完整 value。

**输入类型**：item_id: string；value: unknown，符合该 list 声明的 item_type。

**参数约束**：按 item_id 定位，不接收数组下标或 position，也不接收排序字段。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await evidenceList.replace("ev-1", { note: "复测后时钟已同步" });
~~~

replace 替换完整 value，item_id 与 position 保持不变。它与先删再追加是两种不同的改法，后者会让同一个 item_id 得到新的 position，语义是删掉再追加到末尾，而且那是两次调用、中间该项不存在。目标不存在时返回 NOT_FOUND。

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

**作用**：按条件查询当前 list 的项。

**输入类型**：options: object，必须含 target，可含 filter 和 limit。

**参数约束**：target 取 list。collection 由句柄自动注入，不作为业务查询选项传入。

**返回值类型**：Promise<[BPage](05-elements.md)>，项是 [BListItem](05-elements.md)。

**示例**：

~~~js
const page = await evidenceList.query({ target: "list", filter: { item_id: "ev-1" } });
if (page.items.length > 0) text(page.items[0].value);
~~~

按 item_id 读取单项仍返回 [BPage](05-elements.md)，没有匹配项时 items 是空数组，这不是错误。不带 filter 的 query 一次返回整个 list。

## BList 的项与顺序

list 的每个项由三部分组成：由调用方提供的非空字符串 item_id、符合 item_type 的 value、由 SDK 分配的非负整数 position，字段说明见 [BListItem](05-elements.md)。item_id 不由 SDK 生成。list 没有按下标的读取方法，也没有独立的单项读方法，replace 与 delete 全靠这个键定位，调用方需要保存它。

当前只支持末尾追加、按 item_id 替换完整 value、按 item_id 删除。不支持任意位置插入、链表指针、自动去重或隐式重排。

position 按调用先后递增，取每个 list 的单调高水位加一，所以两次 append 的相对次序就是调用次序。只有成功的追加才分配，失败不消耗；删除后不复用并允许出现间隔；在实例、快照和恢复后保持不变。高水位随整份文档一起写回，重建影子状态之后继续沿用，不按当前最大 position 重算 —— 那样会在删掉尾号之后把已删的号重新发出去。

三条前置条件都按调用时的状态求值：追加一个已存在的 item_id 被拒绝；先删再追加通过，该项得到新的 position；先追加再替换通过。list 允许删除后重新使用同一个 item_id。

默认按 position 升序。limit 的语义与命中项数上限见[默认序与结果规模](01-memory.md)。

元素级的“目标不存在”四类集合统一使用 NOT_FOUND，图的节点与边、树的节点、list 的 item_id、map 的 key 都走这个码。

## BMap

### put(key, value, mode)

**作用**：按 key 写入一个完整 value。

**输入类型**：key: string | number | boolean，符合该 map 声明的 key_type；value: unknown，符合 value_type；mode: "create" | "replace" | "upsert"。

**参数约束**：三个参数全部必填，mode 必须明确给出。create 只允许新 key，replace 只允许已有 key，upsert 两者都允许。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await retryByStatusCodeMap.put(503, 1, "upsert");   // 新建与覆盖都允许
await retryByStatusCodeMap.put(429, 1, "create");                  // 429 已存在时被拒
await retryByStatusCodeMap.put(503, 2, "replace");                 // 503 不存在时被拒
~~~

模式不匹配一定报错，不会静默覆盖或静默创建。两支取的不是同一个码：create 遇到已存在的 key 返回 STRUCTURE_CONFLICT，语义是“已经存在”；replace 遇到不存在的 key 返回 NOT_FOUND，语义是“目标不存在”，与 replace、delete 这类要求目标存在的操作一致。key 或 value 不符 schema 声明返回 INVALID_STRUCTURE，整数 key 写成字符串属于这一类，不做隐式转换。

### delete(key)

**作用**：按 key 删除一条记录。

**输入类型**：key: string | number | boolean，符合该 map 声明的 key_type。

**参数约束**：按 key 定位，不需要传入 value 或写入模式。

**返回值类型**：Promise<[CommitResult](05-elements.md)>。

**示例**：

~~~js
const result = await retryByStatusCodeMap.delete(503);
~~~

目标不存在时返回 NOT_FOUND，重复删同一个 key 时第二次就落在这个码上，删除不幂等。

### BMap.query(options)

**作用**：按条件查询当前 map 的记录。

**输入类型**：options: object，必须含 target，可含 filter 和 limit。

**参数约束**：target 取 map。collection 由句柄自动注入，不作为业务查询选项传入。

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

mode 比对是前置条件，按调用时的状态求值：先 create 再 replace 同一个 key 通过，第二次看到的是第一次的结果；两次 create 同一个 key 被拒绝；同一个 key 两次 upsert 不同 value 时后者生效。

默认按规范化 key 升序，比较按 key_type 声明的标量类型进行。整数 key 按数值比，9 排在 401 之前，不按字典序。limit 的语义与命中项数上限见[默认序与结果规模](01-memory.md)。

map 的历史不另建一套接口，对快照使用同样的 query，见 [memory.snapshot](07-snapshot.md)。

## 当前未确定

上面七个方法的签名已稳定，参数个数、位置与名字可以照着写，但写方法仍有一些参数细节尚未确定，遇到本页未写明的取值规则不要自行推定。replace 与 append 的参数形状完全相同。

## 相关文档

- [声明、登记与句柄](01-memory.md)：集合声明、登记与载入、句柄，以及默认序与结果规模。
- [BListItem 与 BMapItem](05-elements.md)：项类型、提交结果和读结果体。
- [WmError](06-errors.md)：异常类层级、错误码和捕获方式。
