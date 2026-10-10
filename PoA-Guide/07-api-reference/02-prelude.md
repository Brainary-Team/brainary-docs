# Prelude 编排 API（仅 v1）

Prelude 是普通 JavaScript，不是运行时内置能力。把仓库中的
`workflow-demos/lib/prelude.js` 放在 PoA 源码前面，才能使用这些函数。本页只记录
当前 v1 `multi_agent_v1__*` 后端；派发 agent 的 v2 接口不属于本页。

## Agent 编排

### `spawnAgent(message, options?)`

创建一个 v1 子 agent 并返回跨调用句柄。

- 输入：`message: string`；`options?: { meta?: unknown }`。
- 返回：`{ key: string, label: string, meta?: unknown }`；`key` 是 `agent_id`。

```js
const worker = await spawnAgent("分析 README，只返回三条结论", { meta: { file: "README.md" } });
```

### `spawnMany(specs, concurrency = 3)`

限流创建多个 v1 agent。

- 输入：`specs: Array<{ message: string, meta?: unknown }>`、并发上限 `concurrency: number`。
- 返回：按输入顺序排列的句柄数组 `Array<Handle>`。

```js
const handles = await spawnMany(files.map((file) => ({ message: "检查 " + file, meta: { file } })), 3);
```

### `collectAll(handles, timeoutMs = 300000, pollMs = 15000)`

逐个等待 v1 agent 到达终态。超时、失败或没有最终文本时值为 `null`。

- 输入：`handles: Handle[]`、`timeoutMs`、`pollMs`。
- 返回：`Map<string, string | null>`，键是 `handle.key`。

```js
const replies = await collectAll(handles, 120000);
text([...replies]);
```

### `runBatch(specs, options?)`

`spawnMany` 加 `collectAll` 的便捷组合。

- 输入：`specs` 同 `spawnMany`；`options?: { concurrency?: number, timeoutMs?: number }`。
- 返回：按输入顺序的 `Array<{ handle: Handle, reply: string | null }>`。

```js
const rows = await runBatch([{ message: "列出此目录的入口文件", meta: { dir: "." } }], { concurrency: 2 });
```

### `sendAndWait(handle, message, options?)`

向已有 v1 agent 发送后续任务，并等待与上次不同的新答复。

- 输入：`handle: Handle`、`message: string`；`options?: { timeoutMs?: number, pollMs?: number }`。
- 返回：新的答复字符串；超时或没有观察到变化时为 `null`。

```js
const second = await sendAndWait(worker, "请补充一条风险", { timeoutMs: 60000 });
```

### `closeAll(handles)`

尽力关闭一批 v1 agent，忽略单个关闭错误。

- 输入：`handles: Handle[]`。
- 返回：无有意义的值。

```js
await closeAll(handles);
```

## 纯函数与能力探测

### `shellLines(command, options?)`

调用 `tools.exec_command`，把本次返回的 `output` 按行 trim 并删除空行；可用 `validate`
过滤数据行。它不会自动检查 `exit_code`，长命令也不会自动续接 `session_id`。

- 输入：`command: string`；`options?: { validate?: (line: string) => boolean }`。
- 返回：`string[]`。

```js
const dirs = await shellLines("find . -maxdepth 1 -type d", { validate: (line) => SAFE_NAME.test(line) });
```

### `parseJsonReply(raw)`、`mapLimit(items, limit, fn)`

`parseJsonReply` 从第一个 `{` 到最后一个 `}` 贪婪提取 JSON，返回
`{ value, error }`。`mapLimit` 最多同时运行 `limit` 个异步回调并保持输入顺序。

- 输入：`parseJsonReply(raw: string)`；`mapLimit(items: T[], limit: number, fn: (item, index) => Promise<U>)`。
- 返回：`{ value: unknown | null, error: string | null }`；`Promise<U[]>`。

```js
const parsed = parseJsonReply(reply);
const values = await mapLimit(paths, 4, async (path) => path.length);
```

### `SAFE_NAME`

正则 `/^[A-Za-z0-9._-]+$/`，用于过滤可安全拼接到 prompt 的名称。

```js
if (!SAFE_NAME.test(name)) throw new Error("invalid name");
```

### `AGENT_BACKEND` / `requireAgents()`

`AGENT_BACKEND` 是启动时探测出的 `"v1"` 或 `null`。`requireAgents()` 返回 `"v1"`，
没有可用 v1 工具时抛出并列出当前工具名。

```js
if (requireAgents() !== "v1") throw new Error("v1 agent backend required");
```

### `hasTool(name)` / `callTool(name, args?)`

`hasTool` 检查工具是否出现在 `ALL_TOOLS`；`callTool` 统一处理缺失和异常，不抛出。

- 返回：`hasTool(name): boolean`；`callTool` 返回
  `{ name, status: "ok" | "absent" | "error", value, error, ms }`。

```js
const result = await callTool("clock__curr_time", {});
if (result.status === "ok") text(result.value);
```

### `shapeOf(value, depth = 1)` / `timed(fn)`

`shapeOf` 返回运行时值的结构描述字符串；`timed` 捕获函数异常并记录耗时。

- 返回：`shapeOf` 返回 `string`；`timed` 返回 `{ ok, value, error, ms }`。

```js
text(shapeOf(await tools.clock__curr_time({})));
text(await timed(() => tools.exec_command({ cmd: "pwd" })));
```

## v1 原生工具映射

Prelude 的 agent 函数只依赖：`multi_agent_v1__spawn_agent`、`multi_agent_v1__wait_agent`、
`multi_agent_v1__send_input`、`multi_agent_v1__resume_agent` 和 `multi_agent_v1__close_agent`。
参数和返回结构见 [v1 子代理 API](05-agent-v1.md)。

