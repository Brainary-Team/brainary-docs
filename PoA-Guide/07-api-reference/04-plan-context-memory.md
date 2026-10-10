# 计划、目标、上下文、权限与记忆

## 计划与目标

### `update_plan`

覆盖当前计划；最多一个条目可处于 `in_progress`。

- 输入：`{ explanation?: string, plan: Array<{ step: string, status: "pending" | "in_progress" | "completed" }> }`。
- 返回：宿主确认结果（动态对象）。

```js
await tools.update_plan({ plan: [{ step: "读取输入", status: "in_progress" }, { step: "输出结果", status: "pending" }] });
```

### `create_goal`、`get_goal`、`update_goal`

目标工具用于需要持久化工作目标的会话。

- `create_goal` 输入 `{ objective: string, token_budget?: number }`；返回新目标快照。
- `get_goal` 输入 `{}`；返回目标、状态、token/时间用量和剩余预算；没有目标时返回空目标。
- `update_goal` 输入 `{ status: "complete" | "blocked" }`；返回更新后的目标。

```js
await tools.create_goal({ objective: "生成 API 文档" });
const goal = await tools.get_goal({});
text(goal);
```

## 上下文和权限

### `get_context_remaining`

- 输入：`{}`。
- 返回：`{ tokens_left: number | null }`。

```js
text(await tools.get_context_remaining({}));
```

### `request_permissions`

向用户请求额外文件系统或网络权限；实际返回可能只是请求集合的子集。

- 输入：`{ environment_id?: string, reason?: string, permissions: { file_system?: { read?: string[], write?: string[] }, network?: { enabled?: boolean } } }`。
- 返回：宿主定义的已授予权限对象；拒绝或 `approval_policy = "never"` 时通常为空集合。

```js
const granted = await tools.request_permissions({
  permissions: { file_system: { read: ["/tmp/input.csv"] } },
  reason: "读取用户选择的 CSV",
});
text(granted);
```

## 时间

### `clock__curr_time`

- 输入：`{}`。
- 返回：`{ current_time: string }`，格式为 `YYYY-MM-DD HH:MM:SS UTC`。

```js
text((await tools.clock__curr_time({})).current_time);
```

## 记忆

### `memories__list`

- 输入：`{ path?: string, cursor?: string, max_results?: number }`。
- 返回：`{ entries: Array<{ entry_type: "file" | "directory", path: string }>, next_cursor?: string | null, path?: string | null, truncated: boolean }`。

```js
const page = await tools.memories__list({ path: "projects" });
text(page.entries);
```

### `memories__read`

- 输入：`{ path: string, line_offset?: number, max_lines?: number }`。
- 返回：`{ content: string, path: string, start_line_number: number, truncated: boolean }`。

```js
const note = await tools.memories__read({ path: "projects/plan.md", max_lines: 40 });
text(note.content);
```

### `memories__search`

- 输入：`{ queries: string[], path?: string, cursor?: string, max_results?: number, case_sensitive?: boolean, normalized?: boolean, context_lines?: number, match_mode?: { type: "any" } | { type: "all_on_same_line" } | { type: "all_within_lines", line_count: number } }`。
- 返回：`{ queries, path?, match_mode, matches: Array<{ path, content, content_start_line_number, match_line_number, matched_queries }>, next_cursor?, truncated }`。

```js
const hits = await tools.memories__search({ queries: ["PoA", "schema"], match_mode: { type: "all_on_same_line" } });
text(hits.matches);
```

### `memories__add_ad_hoc_note`

创建临时 memory 笔记。只有用户明确要求记住、忘记或更新信息时使用。

- 输入：`{ filename: string, note: string }`；文件名格式为 `YYYY-MM-DDTHH-MM-SS-<slug>.md`。
- 返回：`{}`。

```js
await tools.memories__add_ad_hoc_note({ filename: "2026-10-09T12-00-00-api.md", note: "PoA API 已拆分。" });
```

