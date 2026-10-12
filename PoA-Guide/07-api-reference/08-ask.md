# `ask()` 人机交互 API

## 函数签名

```ts
declare function ask(request: AskRequest): Promise<AskResult>;
```

`request` 必须是对象并包含 `questions` 数组。数组长度为 1–3；单个问题不能使用
`question` 简写。

```ts
type AskQuestionOption = { label: string; description: string };
type AskQuestion = {
  question: string;
  options?: AskQuestionOption[];
};
type AskRequest = { questions: AskQuestion[] };
```

### 问题规则

- `question` 不能是空白字符串。
- 省略 `options` 表示自由文本。
- 提供 `options` 时必须有 2–3 项；空数组不是自由文本的别名。
- option 的 `label` 不能重复，不能以 `user_note: ` 开头，也不能使用 `None of the above`。
- 单字段最多 4 KiB，整个请求序列化后最多 8 KiB。

## 返回结构

```ts
type AskAnswer = {
  selected: string | null;
  text: string | null;
  rejected?: boolean;
};
type AskResult =
  | { status: "answered"; answers: AskAnswer[] }
  | { status: "unavailable"; reason: "nonInteractive" };
```

`answers` 与 `questions` 等长且按位置对应。选择题的 `selected` 是选项 label；自由文本题
的 `selected` 必须为 `null`。`text` 是自由文本内容，或者选择选项时附加的用户备注。
未回答的题返回 `{ selected: null, text: null }`，且不含 `rejected: true`。选择题中用户明确选择“None of the above”时返回 `rejected: true`；即使没有备注，`selected` 和 `text` 也可能同为 `null`。不要只用这两个字段判断用户是否拒绝全部选项。自由文本题不会返回 `rejected: true`。

4 KiB 单字段和 8 KiB 整请求上限只限制脚本发出的请求，不限制用户填写的答案文本。

## 模式

运行时使用 `CodeModeUserInputMode`：

| 模式 | `ask` 是否存在 | 行为 |
| --- | --- | --- |
| `disabled` | 否 | 不注入 `ask`；默认模式 |
| `interactive` | 是 | 发出用户输入事件，由客户端回答或由客户端自动提交空答案 |
| `unavailable` | 是 | 不联系用户，立即返回 `unavailable` |

| 入口 | 模式 | 说明 |
| --- | --- | --- |
| TUI `/poa <PATH>`、`/poa fg <PATH>` | `interactive` | 可显示脚本或子 agent 的问题；当前仅支持本地 embedded app-server |
| TUI `/poa bg <PATH>` | `unavailable` | 不等待当前用户回答 |
| `brainary poa <PATH>`、`brainary exec --poa <PATH>` | `unavailable` | 非交互运行 |
| app-server `thread/codeMode/exec` | 由 `userInputMode` 指定 | 省略、传 `null` 或 `disabled` 时不注入 `ask` |
| 当前 Python SDK `run_poa` | `disabled` | 公共方法未发送 `userInputMode`，不能直接使用交互式 `ask()` |

app-server 客户端在发送 `interactive` 或 `unavailable` 前，应检查 `initialize` 响应的 `capabilities.userInputMode`。旧服务端可能接受却忽略该字段，随后脚本会遇到 `ask is not defined`。`interactive` 请求使用 `isBlocking: false`；服务端没有自动超时，客户端负责展示、决定宽限期并在无人回答时提交空答案。客户端若不响应，请求可能持续等待；客户端错误或断开也可能在服务端被折叠为空答案，脚本不能据此推断用户有意跳过。

一个 cell 同时只能等待一个交互；等待 `ask` 时调用 nested tool，或等待工具时调用 `ask`，以及等待一个 `ask` 时再次调用 `ask`，都会在调用点同步抛错，尚未创建可供 `.catch()` 处理的 Promise。可用 `try/catch` 包住调用或 `await`；前一次结束后，同一 cell 可以顺序多次 `ask()`。取消、cell 终止或连接断开属于异常终止，不应当作 `unavailable` 处理。

## 子 agent 自主提问

交互式 PoA 派出的 v1 或 v2 子 agent 可以在自己的线程调用模型工具 `request_user_input`；这与脚本全局函数 `ask()` 是两条不同的调用路径。子 agent 及其后代继承创建它的 PoA 的模式，不能自行从 `disabled` 或 `unavailable` 升级到 `interactive`：

| 继承模式 | 子 agent 调用 `request_user_input` |
| --- | --- |
| `interactive` | 向客户端发送 `item/tool/requestUserInput`，`isBlocking: false`；回答回到子 agent 的当前 turn |
| `unavailable` | 不发用户请求，工具向模型返回“无人可答”的可恢复错误 |
| `disabled` | 不发用户请求，工具向模型返回“提问已关闭”的可恢复错误 |

请求携带子 agent 自己的 `threadId` 和 `turnId`，而不是 PoA 父线程的 ID。客户端应按请求中的子线程路由与回答，关注 `thread/status/changed` 或 `thread/list` / `thread/read` 中的 `waitingOnUserInput`；该状态清除是异步的。新建子线程的自动附着只是 best-effort，漏掉时可用 `thread/list` 和 `ancestorThreadId` 查找。父 cell 结束或中断不会自动撤回子 agent 的待答问题；需中断其自己的 turn。请求结束时客户端还应处理 `serverRequest/resolved`，清理对应的待答界面。跨进程恢复不保证保留 PoA 派生来源。普通非 PoA 子 agent 的提问限制不受此机制改变。

## 示例

```js
const result = await ask({
  questions: [
    {
      question: "部署到哪个环境？",
      options: [
        { label: "staging", description: "预发布环境" },
        { label: "production", description: "生产环境" },
      ],
    },
    { question: "是否需要记录备注？" },
  ],
});

if (result.status === "unavailable") {
  text("当前运行没有交互终端");
} else {
  const [environment, note] = result.answers;
  text({ environment: environment.selected, note: note.text });
}
```

用户选择 production 并填写“窗口已审批”时，结果形如：

```json
{
  "status": "answered",
  "answers": [
    { "selected": "production", "text": null },
    { "selected": null, "text": "窗口已审批" }
  ]
}
```

后台或 CLI 的非交互模式返回：

```json
{ "status": "unavailable", "reason": "nonInteractive" }
```
