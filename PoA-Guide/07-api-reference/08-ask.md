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
};
type AskResult =
  | { status: "answered"; answers: AskAnswer[] }
  | { status: "unavailable"; reason: "nonInteractive" };
```

`answers` 与 `questions` 等长且按位置对应。选择题的 `selected` 是选项 label；自由文本题
的 `selected` 必须为 `null`。`text` 是自由文本内容，或者选择选项时附加的用户备注。
未回答的题返回 `{ selected: null, text: null }`。

## 模式

运行时使用 `CodeModeUserInputMode`：

| 模式 | `ask` 是否存在 | 行为 |
| --- | --- | --- |
| `disabled` | 否 | 不注入 `ask`；默认模式 |
| `interactive` | 是 | 发出用户输入事件并等待回答 |
| `unavailable` | 是 | 不联系用户，立即返回 `unavailable` |

一个 cell 同时只能等待一个交互；在等待 `ask` 时调用 nested tool，或再次调用 `ask`，会
拒绝 Promise。取消、cell 终止或连接断开会拒绝 Promise，而不是伪装成 `unavailable`。

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
