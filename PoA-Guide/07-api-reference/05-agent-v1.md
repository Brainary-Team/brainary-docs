# 子代理 v1 API

本页只描述 `multi_agent_v1__*`。派发 agent 的 v2 (`collaboration__*`) 明确不在本文档范围内。
Prelude 的 `spawnAgent`、`collectAll` 等封装最终调用本页工具。

## `multi_agent_v1__spawn_agent`

创建一个 v1 子代理。

- 输入：`{ message?: string, items?: UserInput[], agent_type?: string, model?: string, reasoning_effort?: string, service_tier?: string, fork_context?: boolean }`。`message` 与 `items` 二选一；`fork_context` 为 `true` 时复制当前 thread 历史。
- 返回：`{ agent_id: string, nickname: string | null }`。

```js
const a = await tools.multi_agent_v1__spawn_agent({ message: "分析 src 目录并返回 JSON" });
```

## `multi_agent_v1__wait_agent`

等待一个或多个代理进入终态。传多个目标时返回最先完成的目标。

- 输入：`{ targets: string[], timeout_ms?: number }`。
- 返回：`{ status: Record<string, AgentStatus>, timed_out: boolean }`。`AgentStatus` 为
  `pending_init | running | interrupted | shutdown | not_found | { completed: string | null } | { errored: string }`。

```js
const r = await tools.multi_agent_v1__wait_agent({ targets: [a.agent_id], timeout_ms: 120000 });
text(r.status[a.agent_id]);
```

## `multi_agent_v1__send_input`

向已有代理发送消息；`interrupt` 可让消息打断当前任务。

- 输入：`{ target: string, message?: string, items?: UserInput[], interrupt?: boolean }`。
- 返回：`{ submission_id: string }`。

```js
const submitted = await tools.multi_agent_v1__send_input({ target: a.agent_id, message: "补充失败原因", interrupt: true });
```

## `multi_agent_v1__resume_agent`

恢复已关闭的代理。

- 输入：`{ id: string }`。
- 返回：`{ status: AgentStatus }`。

```js
const state = await tools.multi_agent_v1__resume_agent({ id: a.agent_id });
```

## `multi_agent_v1__close_agent`

关闭目标代理及仍打开的后代，并释放并发额度。

- 输入：`{ target: string }`。
- 返回：`{ previous_status: AgentStatus }`。

```js
const closed = await tools.multi_agent_v1__close_agent({ target: a.agent_id });
```

`UserInput` 条目可使用 `text`、`image`、`local_image`、`audio`、`local_audio`、`skill` 或
`mention` 类型；具体字段由条目的 `type` 决定。发送前应处理 Promise reject，并为 `status`
中缺失的 agent id 保留兜底逻辑。
