# 执行与文件工具

这些工具通过 `tools.<name>` 调用。工具是否进入当前 cell 由环境和 feature 决定，调用前可用
`ALL_TOOLS.some((item) => item.name === "工具名")` 探测。

## `exec_command`

在选定环境中运行命令；短命令返回退出码，仍在运行时返回 `session_id`。

输入：

```ts
{
  cmd: string;
  environment_id?: string;
  workdir?: string;
  shell?: string;
  login?: boolean;
  tty?: boolean;
  yield_time_ms?: number;
  max_output_tokens?: number;
  sandbox_permissions?: "use_default" | "with_additional_permissions" | "require_escalated";
  additional_permissions?: { file_system?: { read?: string[]; write?: string[] }; network?: { enabled?: boolean } };
  justification?: string;
  prefix_rule?: string[];
}
```
返回：`{ output: string, exit_code?: number, session_id?: number, chunk_id?: string, wall_time_seconds: number, original_token_count?: number }`。

```js
const r = await tools.exec_command({ cmd: "printf 'hello\\n'", workdir: "." });
if (r.exit_code !== 0) throw new Error(r.output);
text(r.output);
```

## `write_stdin`

向 `exec_command` 返回的运行中会话写入 stdin，或省略 `chars` 轮询新输出。

- 输入：`{ session_id: number, chars?: string, yield_time_ms?: number, max_output_tokens?: number }`。
- 返回：与 `exec_command` 相同的输出对象；仍运行时继续带 `session_id`。

```js
const next = await tools.write_stdin({ session_id: r.session_id, chars: "yes\\n" });
text(next.output);
```

## `shell_command`

一次性运行 shell 脚本，没有可续接会话。

- 输入：`{ command: string, workdir?: string, timeout_ms?: number, login?: boolean, sandbox_permissions?: string, additional_permissions?: object, justification?: string, prefix_rule?: string[] }`。
- 返回：宿主定义的执行结果（通常包含输出和退出码）；建议先 `JSON.stringify` 探测。

```js
const r = await tools.shell_command({ command: "pwd", workdir: ".", timeout_ms: 10000 });
text(r);
```

## `apply_patch`

应用文件补丁。它是 freeform 工具，入参是裸补丁字符串，不是对象。

- 输入：`string`。
- 返回：宿主定义的补丁结果，失败时 Promise reject。

```js
await tools.apply_patch("*** Begin Patch\\n*** Update File: note.txt\\n@@\\n-old\\n+new\\n*** End Patch");
```

## `view_image`

读取执行环境中的本地图片，返回可转发给 `image()` 的 data URL。

- 输入：`{ path: string, detail?: "high" | "original", environment_id?: string }`。
- 返回：`{ detail: "high" | "original", image_url: string }`。

```js
const picture = await tools.view_image({ path: "./out/chart.png", detail: "original" });
image(picture.image_url);
```
