# global

以下示例从 `/global` 入口导入，也可以从根入口 `@brainary/poa-sdk` 导入：

~~~js
import { Logger, exit } from "@brainary/poa-sdk/global";
import { POA_INPUT, text } from "@brainary/poa-sdk/io";
~~~

定时器、计时和随机数等标准 Web API 由运行时直接提供，无需导入，见 [运行环境](../07-runtime.md)。

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

### Logger

**作用**：记录开发者日志，分 debug、info、warn、error 四个级别。日志随会话保存，在 CLI 和 TUI 中显示，不进入程序输出和模型上下文。

~~~ts
interface Logger {
  debug(message: string, data?: unknown): void;
  info(message: string, data?: unknown): void;
  warn(message: string, data?: unknown): void;
  error(message: string, data?: unknown): void;
}

type LogLevel = "debug" | "info" | "warn" | "error";
~~~

**输入类型**：message: string，必填；data?: unknown，可选，须能被 JSON 序列化。

**返回值类型**：void。Logger.error 只记录日志，不抛出异常。

**异常**：message 不是字符串，或 data 不能被 JSON 序列化时抛出异常。

**示例**：

~~~js
Logger.info("开始处理", { files: 3 });

try {
  JSON.parse(raw);
} catch (error) {
  Logger.error("解析失败", { error: String(error) });
}
~~~
