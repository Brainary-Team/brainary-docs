# MessageItem

MessageItem 表示发送给 Agent 的单个消息项，由 type 字段区分文本、图片、音频、skill 和 mention。
[SpawnSpec](01-agent-orchestration.md#spawnspec)、[Agent](03-agent.md) 和 [AgentExecutor](04-agent-executor.md) 的消息项数组均使用这个类型。

### ImageDetail

**作用**：指定消息中图片的详细程度，与底层消息协议保持一致。

**类型定义**：

~~~ts
export type ImageDetail = "auto" | "low" | "high" | "original";
~~~

**示例**：

~~~js
const item = { type: "image", image_url: "https://example.com/diagram.png", detail: "high" };
~~~

### MessageItem

**作用**：定义 Agent 消息项的类型和对应字段。

**类型定义**：

~~~ts
export type MessageItem =
  | { type: "text"; text: string }
  | { type: "image"; image_url: string; detail: ImageDetail | null }
  | { type: "local_image"; path: string; detail: ImageDetail | null }
  | { type: "audio"; audio_url: string }
  | { type: "local_audio"; path: string }
  | { type: "skill" | "mention"; name: string; path: string };
~~~

image 与 local_image 的 detail 字段必填，可以传入 null。其取值见 [ImageDetail](#imagedetail)。
skill 与 mention 均需提供 name 和 path。

**示例**：

~~~js
const message_items = [
  { type: "text", text: "结合图片和音频检查这份说明" },
  { type: "image", image_url: "https://example.com/diagram.png", detail: null },
  { type: "local_image", path: "/workspace/diagram.png", detail: null },
  { type: "audio", audio_url: "https://example.com/recording.wav" },
  { type: "local_audio", path: "/workspace/recording.wav" },
  { type: "skill", name: "code-review", path: "/workspace/skills/code-review/SKILL.md" },
  { type: "mention", name: "README", path: "/workspace/README.md" },
];
const handle = await orchestration.spawn_agent({
  message_items,
  agent_name: "reviewer",
  meta: "review-docs",
});
~~~
