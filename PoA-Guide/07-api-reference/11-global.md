# global

global 提供定时器、耗时统计和流程控制函数。

### set_timeout(callback, delay_ms)

**作用**：安排一次延迟回调。

**输入类型**：callback: () => void；delay_ms: number，单位为毫秒。两个参数均必填。

**返回值类型**：number，定时器 ID。

**示例**：

~~~js
const timeout_id = set_timeout(() => text("done"), 100);
~~~

### clear_timeout(timeout_id)

**作用**：取消尚未执行的定时器。

**输入类型**：timeout_id: number，必填。

**返回值类型**：void。

**示例**：

~~~js
clear_timeout(timeout_id);
~~~

### timed(fn)

**作用**：执行函数、捕获异常并记录耗时。

**输入类型**：fn: () => T | Promise<T>。

**返回值类型**：Promise<{ ok: boolean; value?: T; error?: string; ms: number }>。

**示例**：

~~~js
const result = await timed(() => tools.clock_sleep({ duration_ms: 10 }));
text(result.ms);
~~~

### exit()

**作用**：立即结束当前 PoA 执行。

**输入类型**：无。

**返回值类型**：never；不会正常返回。

**示例**：

~~~js
if (!POA_INPUT.enabled) exit();
~~~
