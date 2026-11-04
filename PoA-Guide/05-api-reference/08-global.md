# global

以下示例从 `/global` 入口导入，也可以从根入口 `@brainary/poa-sdk` 导入：

~~~js
import { clearTimeout, exit, setTimeout, timed } from "@brainary/poa-sdk/global";
import { POA_INPUT, text } from "@brainary/poa-sdk/io";
~~~

### setTimeout(callback, delayMs?)

**作用**：在 delayMs 毫秒后执行一次 callback。

**输入类型**：callback: () => void，必填；delayMs?: number，单位毫秒，默认 0。

程序运行到末尾时，尚未触发的回调不再执行。需要等待回调时，await 一个在回调中 resolve 的 Promise。

**返回值类型**：TimeoutId，即 number，用于 clearTimeout。

**异常**：callback 不是函数时抛出 TypeError。

**示例**：

~~~js
await new Promise((resolve) => setTimeout(resolve, 1000));  // 等待 1 秒

const timeout_id = setTimeout(() => text("超时"), 60_000);
~~~

### clearTimeout(timeout_id?)

**作用**：取消尚未触发的回调。timeout_id 省略、为 null 或对应的回调已触发时不做任何操作。

**输入类型**：timeout_id?: TimeoutId | null。

**返回值类型**：void。

**示例**：

~~~js
clearTimeout(timeout_id);
~~~

### timed(fn)

**作用**：执行 fn 并测量耗时。fn 可以是同步或异步函数。

**输入类型**：fn: () => T | PromiseLike<T>，必填。

**返回值类型**：Promise<TimedResult<T>>。fn 抛出的异常原样抛出。

~~~ts
interface TimedResult<T> {
  value: T;   // fn 的返回值
  ms: number; // 耗时，单位毫秒
}
~~~

**示例**：

~~~js
import { TerminalExecutor } from "@brainary/poa-sdk/executor";

const { value, ms } = await timed(() => new TerminalExecutor().exec_command("pwd"));
text({ output: value.output, ms });
~~~

### exit()

**作用**：立即结束程序，本次运行按成功处理。已追加的输出照常返回。

**输入类型**：无。

**返回值类型**：never。

exit() 通过抛出异常结束程序。在 try 块中调用时，catch 须把不处理的异常重新抛出，否则程序会继续运行。

**示例**：

~~~js
if (POA_INPUT.dry_run) {
  text("dry run，不执行修改");
  exit();
}
~~~
