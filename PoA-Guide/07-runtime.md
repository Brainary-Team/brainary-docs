# 运行环境

除了从 `@brainary/poa-sdk` 导入的 API，PoA 程序还可以直接使用运行时提供的标准 Web API。它们是全局函数和对象，无需导入，行为遵循 Web 标准。

## 全局 API

| 类别 | API |
| --- | --- |
| 定时器 | setTimeout、clearTimeout、setInterval、clearInterval、queueMicrotask |
| 计时 | performance、Performance、PerformanceEntry、PerformanceMark、PerformanceMeasure |
| 随机数 | crypto.getRandomValues、crypto.randomUUID |
| 编码 | TextEncoder、TextDecoder、TextEncoderStream、TextDecoderStream、atob、btoa |
| URL | URL、URLSearchParams、URLPattern |
| 数据 | Blob、File |
| 流 | ReadableStream、WritableStream、TransformStream 及其 Reader、Writer 和 Controller，ByteLengthQueuingStrategy、CountQueuingStrategy、CompressionStream、DecompressionStream |
| 事件与取消 | EventTarget、Event、CustomEvent、ErrorEvent、PromiseRejectionEvent、AbortController、AbortSignal、DOMException |
| 控制台与错误 | console、reportError |

读写文件、执行命令和调用外部服务，分别使用 SDK 的 [FileExecutor](05-api-reference/05-executors.md#fileexecutor)、[TerminalExecutor](05-api-reference/05-executors.md#terminalexecutor) 和 [McpExecutor](05-api-reference/05-executors.md#mcpexecutor)。

## 定时器

- setTimeout 和 setInterval 返回 number 类型的 ID，传给 clearTimeout、clearInterval 即可取消。
- 顶层代码执行完毕后，程序会等到所有定时器触发或被取消才结束。
- setInterval 会一直重复执行，直到调用 clearInterval。
- 在回调中调用 [exit()](05-api-reference/08-global.md#exit) 会立即结束程序，其余定时器不再触发。

~~~js
import { text } from "@brainary/poa-sdk";

await new Promise((resolve) => setTimeout(resolve, 1000));  // 等待 1 秒

let ticks = 0;
const interval = setInterval(() => {
  ticks += 1;
  text(`第 ${ticks} 次`);
  if (ticks === 3) clearInterval(interval);
}, 500);
~~~

## 计时

performance.now() 返回程序开始运行以来经过的毫秒数，带小数部分。

~~~js
import { TerminalExecutor, text } from "@brainary/poa-sdk";

const start = performance.now();
const result = await new TerminalExecutor().exec_command("pwd");
text({ output: result.output, ms: Math.round(performance.now() - start) });
~~~

## 随机数

- crypto.randomUUID() 返回一个随机的 UUID v4 字符串。
- crypto.getRandomValues(array) 用随机值填满一个整数类型数组（Int8Array、Uint8Array、Uint8ClampedArray、Int16Array、Uint16Array、Int32Array、Uint32Array、BigInt64Array 或 BigUint64Array）并返回该数组。数组最多 65536 字节。
- 传入其他类型的数组时抛出 name 为 TypeMismatchError 的 DOMException，超过 65536 字节时抛出 name 为 QuotaExceededError 的 DOMException。

~~~js
const id = crypto.randomUUID();
const bytes = crypto.getRandomValues(new Uint8Array(16));
~~~

## console

- console.log 和 console.info 把格式化后的内容追加到程序输出，与 [text](05-api-reference/07-io.md#textvalue) 一样出现在运行结果中。
- console.warn 和 console.error 也追加到程序输出，内容前分别加上 `[warn] ` 和 `[error] `。
- console.debug 不产生输出。

只给开发者看的日志请用 [Logger](05-api-reference/08-global.md#logger)，它不进入程序输出。

## 未捕获的错误

- 顶层代码抛出的异常使程序失败。
- 回调中抛出的异常和未处理的 Promise rejection 同样使程序失败。全局注册了 `error` 或 `unhandledrejection` 监听器，并在其中调用 `event.preventDefault()` 时，程序继续运行。顶层代码抛出的异常不经过这些监听器。
- reportError(error) 把 error 当作回调中未捕获的异常处理。

~~~js
import { Logger } from "@brainary/poa-sdk";

addEventListener("unhandledrejection", (event) => {
  event.preventDefault();
  Logger.warn("未处理的 rejection", { reason: String(event.reason) });
});
~~~
