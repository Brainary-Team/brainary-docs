# Provider 扩展工具

以下工具由 provider、orchestrator 或 feature 注入，当前环境可能没有。先在 `ALL_TOOLS` 中探测；
不要因为工具名存在于其他环境就直接调用。动态工具的参数和返回 schema 以对应
`ALL_TOOLS[].description` 为准。

## `web__run`

执行联网搜索、打开页面或页面内查找。典型输入是：

```js
const web = ALL_TOOLS.find((item) => item.name === "web__run");
if (!web) throw new Error("web__run is unavailable");
const result = await tools.web__run({
  search_query: [{ q: "standalone web search" }],
  response_length: "short",
});
text(result);
```
- 输入：provider 定义的操作数组（常见字段有 `search_query`、`open`、`find`）和
  `response_length?: "short" | "medium" | "long"`。
- 返回：provider 定义的搜索结果或页面内容，通常是对象或字符串。

## `skills__list`

分页列出 skill。参数和返回值由 skill provider 声明。

- 输入：通常为 `{ source?: "orchestrator" | "executor", cursor?: string }`。
- 返回：通常包含 `{ skills, next_cursor? }`，每项包含名称、说明、来源和包标识。

```js
if (hasTool("skills__list")) text(await tools.skills__list({ source: "orchestrator" }));
```

## `skills__read`

读取 skill 的主说明或引用资源。

- 输入：通常为 `{ source, package, resource, cursor? }`，字段必须以运行时声明为准。
- 返回：分页文本资源，通常包含 `text` 和后续游标。

```js
const page = await tools.skills__read({
  source: "orchestrator",
  package: "example-skill",
  resource: "SKILL.md",
});
text(page);
```

## `image_gen__imagegen`

生成新图片，或基于最多五张本地/会话参考图编辑图片。

- 输入：通常包含文本提示和可选参考图片路径；完整输入 schema 从 `ALL_TOOLS` 读取。
- 返回：通常为 `{ image_url: string, output_hint?: string }` 或包含该字段的结果。

```js
const generated = await tools.image_gen__imagegen({ prompt: "一张简洁的城市地图插画" });
generatedImage(generated);
```

## `wait_for_environment`

等待一个状态为 `starting` 的执行环境启动。

- 输入：`{ environment_id: string }`。
- 返回：启动成功时 `{ environment_id: string, status: "ready" }`；失败时 Promise reject 或错误对象。

```js
const env = await tools.wait_for_environment({ environment_id: "env-123" });
text(env);
```

## `list_available_plugins_to_install`

列出当前可安装插件或连接器。

- 输入：通常为 `{}`。
- 返回：候选项数组，包含 ID、名称、类型和说明；以运行时返回 schema 为准。

```js
const choices = await tools.list_available_plugins_to_install({});
text(choices);
```

## `request_plugin_install`

按候选 ID 请求安装插件或连接器，并等待安装结果。

- 输入：通常为 `{ plugin_id: string }`；字段名和可选版本字段以运行时声明为准。
- 返回：安装结果或状态对象；用户拒绝、缺少依赖和安装失败都应处理。

```js
const installed = await tools.request_plugin_install({ plugin_id: "example-plugin" });
text(installed);
```

## 通用降级模式

provider 工具的 schema 可能随模型和连接器变化。使用 `hasTool` 探测，使用 `callTool` 统一捕获
缺失和失败，并把未知返回值先序列化：

```js
const result = await callTool("web__run", { search_query: [{ q: "PoA" }] });
if (result.status === "ok") text(JSON.stringify(result.value));
```
