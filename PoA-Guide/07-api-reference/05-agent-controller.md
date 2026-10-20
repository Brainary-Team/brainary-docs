# AgentController

[AgentController](05-agent-controller.md) 负责对已经创建的 [Agent](03agent.md) 做生命周期和上下文控制。

### compact(agent_id, compact_template?)

**作用**：压缩目标 Agent 的上下文，并按模板保留关键信息。

**输入类型**：agent_id: string；compact_template?: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agentController.compact(agent.agent_id, "保留结论、待办和文件路径");
~~~

### close(agent_id)

**作用**：关闭指定 Agent 并释放运行资源。

**输入类型**：agent_id: string。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agentController.close(agent.agent_id);
~~~

### steer(agent_id, message?, input_items?)

**作用**：向运行中的 Agent 发送转向输入，使它调整当前任务。

**输入类型**：agent_id: string；message?: string；input_items?: IOInput[]。

**参数约束**：message 和 input_items 必须且只能传入一个；仅传入 input_items 时，message 使用 undefined 占位。

**返回值类型**：Promise<void>。

**示例**：

~~~js
await agentController.steer(
  agent.agent_id,
  "停止扫描，先返回已发现的问题",
);
~~~
