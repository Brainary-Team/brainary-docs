# global

global 提供定时器、耗时统计和流程控制函数。

以下示例假定已从 `/global` 入口导入，这些函数也可从根入口 `@brainary/poa-sdk` 导入：

~~~js
import { clearTimeout, exit, setTimeout, timed } from "@brainary/poa-sdk/global";
import { POA_INPUT, text } from "@brainary/poa-sdk/io";
~~~

### setTimeout(callback, delayMs?)

**作用**：安排一次延迟回调。需要等待回调执行时，应 await 一个 Promise。

**输入类型**：callback: () => void，必填；delayMs?: number，单位为毫秒。

**返回值类型**：TimeoutId，即 number 类型的定时器 ID。

**示例**：

~~~js
const timeout_id = setTimeout(() => text("done"), 100);
await new Promise((resolve) => setTimeout(resolve, 200));
~~~

### clearTimeout(timeout_id?)

**作用**：取消尚未执行的定时器。

**输入类型**：timeout_id?: TimeoutId | null，可以省略或传入 null。

**返回值类型**：void。

**示例**：

~~~js
clearTimeout(timeout_id);
~~~

### timed(fn)

**作用**：执行同步或异步函数并记录耗时。

**输入类型**：fn: () => T | PromiseLike<T>，必填。

**返回值类型**：Promise<TimedResult<T>>，即 { value: T; ms: number }，ms 为耗时毫秒数。fn 抛出的异常会继续抛出。

**示例**：

~~~js
import { TerminalExecutor } from "@brainary/poa-sdk/executor";

const { value, ms } = await timed(() => new TerminalExecutor().exec_command("pwd"));
text({ output: value.output, ms });
~~~

### exit()

**作用**：立即结束当前 PoA 执行。

**输入类型**：无。

**返回值类型**：never；不会正常返回。

**示例**：

~~~js
if (!POA_INPUT.enabled) exit();
~~~
