# 全局 Primitive

这些值由 code-mode 运行时直接注入 `globalThis`，不通过 `tools` 调用。函数返回
`Promise` 时使用 `await`。除 `ALL_TOOLS`、`POA_INPUT` 和 `tools` 外，下面每个条目都给出
输入、返回值和最小示例。

## 输出

### `text(value)`

把字符串、数字、布尔值、`null` 或可序列化对象追加到当前 PoA 的输出。

- 输入：`value: string | number | boolean | null | unknown`。
- 返回：无有意义的返回值；输出会在 cell 结束时汇总。

```js
text({ ok: true, count: 3 });
```

### `image(value)`、`audio(value)`、`generatedImage(value)`

分别追加图片、音频和图像生成结果。接受 data URL、对应内容对象或宿主返回的内容块；
`generatedImage` 接受 `{ image_url, output_hint? }`。

- 输入：`image(value)`、`audio(value)`；`generatedImage({ image_url: string, output_hint?: string })`。
- 返回：无有意义的返回值。

```js
image({ image_url: "data:image/png;base64,...", detail: "high" });
audio({ audio_url: "data:audio/wav;base64,..." });
generatedImage({ image_url: "data:image/png;base64,...", output_hint: "保存为封面" });
```

## 会话状态和流程控制

### `store(key, value)` / `load(key)`

在当前 code-mode 会话保存和读取 JSON 值。状态只属于当前会话，模型不会自动看到它。

- `store` 输入：`key: string`、`value: unknown`；返回无有意义的值。
- `load` 输入：`key: string`；返回保存的值，未找到时为 `undefined`。

```js
store("summary", { count: 3 });
const summary = load("summary");
text(summary);
```

### `exit()`

立即结束当前脚本。它通过内部哨兵异常实现；不要在会吞掉异常的 `catch` 中调用。

- 输入：无。
- 返回：不会正常返回；顶层运行结果表示脚本提前结束。

```js
if (!POA_INPUT) exit();
```

### `setTimeout(callback, delayMs?)` / `clearTimeout(timeoutId?)`

提供沙箱内的一次性定时器。

- `setTimeout` 输入：`callback: () => void`、`delayMs?: number`（毫秒）。返回定时器 ID。
- `clearTimeout` 输入：`timeoutId?: number`；返回无有意义的值。

```js
const id = setTimeout(() => text("done"), 100);
clearTimeout(id);
```

### `notify(value)` / `yield_control()`

`notify` 尝试立即向线程历史注入一条文本或 JSON；`yield_control` 尝试把已累积的输出
提前交给模型并继续执行。它们没有稳定的业务返回值，客户端不保证 PoA 用户能看到中间输出。

- 输入：`notify(value: string | number | boolean | null | unknown)`；`yield_control()` 无参数。
- 返回：无有意义的返回值。

```js
notify({ progress: 0.5 });
yield_control();
```

## 运行时探测对象

### `ALL_TOOLS`

当前 cell 实际可调用工具的快照。每项至少包含 `{ name: string, description: string }`；
描述中可能带当前工具的 TypeScript 声明。工具面在 cell 启动后不变化。

```js
const names = ALL_TOOLS.map(({ name }) => name).sort();
text(names);
```

### `tools`

按工具名索引的对象。调用形式为 `await tools.<name>(args)`；参数必须按 `ALL_TOOLS`
对应描述传入，失败通常表现为 Promise reject。

```js
const result = await tools.clock__curr_time({});
text(result.current_time);
```

### `POA_INPUT`

由 PoA 调用方注入的业务 JSON。没有输入时不会注入该全局变量；输入 schema 和 SDK 用法见
[POA INPUT SDK](07-poa-input-sdk.md)。

## `ask()`

`ask` 只在 `interactive` 或 `unavailable` 用户输入模式下注入；默认的 `disabled` 模式中
不存在该函数。完整协议、入口限制、错误和示例见 [ask 交互 API](08-ask.md)。
