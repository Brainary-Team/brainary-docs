# MessageItem

MessageItem 是发给 agent 的一个消息项，用于 [Agent.run](01-agent.md#runmessage-items-output_schema-timeout_ms-thread_id) 的 items 和 [SpawnSpec](03-agent-executor.md#spawnspec) 的 message_items。多个消息项按数组顺序发送。

### ImageDetail

**作用**：图片的细节级别。

**类型定义**：

~~~ts
type ImageDetail = "auto" | "low" | "high" | "original";
~~~

**示例**：

~~~js
const item = { type: "local_image", path: "/workspace/diagram.png", detail: "high" };
~~~

### MessageItem

**作用**：消息项的类型，由 type 字段区分。

**类型定义**：

~~~ts
type MessageItem =
  | { type: "text"; text: string }
  | { type: "image"; image_url: string; detail?: ImageDetail | null }
  | { type: "local_image"; path: string; detail?: ImageDetail | null }
  | { type: "audio"; audio_url: string }
  | { type: "local_audio"; path: string }
  | { type: "skill" | "mention"; name: string; path: string };
~~~

| type | 字段 |
| --- | --- |
| text | text：文本 |
| image | image_url：图片的 data URI；detail：可选，见 [ImageDetail](#imagedetail) |
| local_image | path：本地图片文件的路径；detail：可选，见 [ImageDetail](#imagedetail) |
| audio | audio_url：音频的 data URI |
| local_audio | path：本地音频文件的路径 |
| skill | name：skill 名称；path：该 skill 的 SKILL.md 路径 |
| mention | name：显示名称；path：引用目标，格式为 `app://<connector-id>` 或 `plugin://<plugin-name>@<marketplace-name>` |

**示例**：

~~~js
import { AgentExecutor } from "@brainary/poa-sdk";

const handle = await new AgentExecutor().spawn_agent({
  message_items: [
    { type: "text", text: "按 code-review skill 检查截图和录音中提到的问题" },
    { type: "local_image", path: "/workspace/screenshot.png", detail: "original" },
    { type: "local_audio", path: "/workspace/recording.wav" },
    { type: "skill", name: "code-review", path: "/workspace/skills/code-review/SKILL.md" },
  ],
  agent_name: "reviewer",
});
~~~
