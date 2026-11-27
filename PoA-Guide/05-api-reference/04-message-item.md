# MessageItem

MessageItem 是发给 agent 的一个消息项，用于 [Agent.run](01-agent.md#runmessage-message_items-timeout_ms)、[append_message](03-agent-executor.md#append_messageagent_id-message-message_items-immediately) 和 [SpawnSpec](03-agent-executor.md#spawnspec) 的 message_items。多个消息项按数组顺序发送。

### ImageDetail

**作用**：图片的细节级别。

**类型定义**：

~~~ts
type ImageDetail = "auto" | "low" | "high" | "original";
~~~

**示例**：

~~~js
const item = { type: "image", image_url: "data:image/png;base64,iVBORw0KGgo...", detail: "high" };
~~~

### TextElement

**作用**：文本中需要按特殊元素显示的一段，用 UTF-8 字节范围标出。

**类型定义**：

~~~ts
interface TextElement {
  byte_range: { start: number; end: number };
  placeholder?: string | null;
}
~~~

| 字段 | 说明 |
| --- | --- |
| byte_range | 这一段在 text 中的 UTF-8 字节范围，start 含、end 不含 |
| placeholder | 可选，这一段的显示文字 |

### MessageItem

**作用**：消息项的类型，由 type 字段区分。

**类型定义**：

~~~ts
type MessageItem =
  | { type: "text"; text: string; text_elements?: readonly TextElement[] }
  | { type: "image"; image_url: string; detail?: ImageDetail | null }
  | { type: "audio"; audio_url: string }
  | { type: "skill"; name: string; path: string }
  | { type: "mention"; name: string; path: string };
~~~

| type | 字段 |
| --- | --- |
| text | text：文本；text_elements：可选，见 [TextElement](#textelement) |
| image | image_url：图片的 data URL；detail：可选，见 [ImageDetail](#imagedetail) |
| audio | audio_url：音频的 data URL |
| skill | name：skill 名称；path：该 skill 的 SKILL.md 路径 |
| mention | name：显示名称；path：引用目标，格式为 `app://<connector-id>` 或 `plugin://<plugin-name>@<marketplace-name>` |

图片和音频以 data URL 内嵌在消息中。需要发送本地文件时，先读出文件内容，再转为 data URL。

**示例**：

~~~js
import { AgentExecutor } from "@brainary/poa-sdk";

const handle = await new AgentExecutor().spawn_agent({
  message_items: [
    { type: "text", text: "按 code-review skill 检查截图和录音中提到的问题" },
    { type: "image", image_url: "data:image/png;base64,iVBORw0KGgo...", detail: "original" },
    { type: "audio", audio_url: "data:audio/wav;base64,UklGRiQ..." },
    { type: "skill", name: "code-review", path: "/workspace/skills/code-review/SKILL.md" },
  ],
  nickname: "reviewer",
});
~~~
