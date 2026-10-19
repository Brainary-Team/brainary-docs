# AgentConfig、Capabilities、ModelSpec 与 RuntimeSpec

本页描述 [Agent](03agent.md) 的静态配置类型。配置在 Agent 创建时确定，执行时由运行时转换为对应的模型、
能力和环境设置。

### ModelSpec

**作用**：指定 Agent 使用的模型和推理参数。

**输入类型**：

~~~ts
interface ModelSpec {
  model_id: string;
  provider?: string;
  reasoningEffort?: string;
}
~~~

**返回值类型**：[ModelSpec](02-agent-config.md) 配置对象。

**示例**：

~~~js
const model = {
  model_id: "gpt-6-sol",
  provider: "openAI",
  reasoningEffort: "high",
};
~~~

### Capabilities

**作用**：声明 Agent 可以访问的 Brainary tool、MCP server 和 PoA 能力。

**输入类型**：

~~~ts
interface Capabilities {
  brainary_tool?: string[];
  mcp?: string[];
  poa?: string[];
}
~~~

**返回值类型**：[Capabilities](02-agent-config.md) 配置对象。

**示例**：

~~~js
const capabilities = {
  brainary_tool: ["exec_command", "multi_agent"],
  mcp: ["filesystem"],
  poa: ["review-package"],
};
~~~

### Sandbox

**作用**：限定 Agent 的文件系统访问级别。

**输入类型**：枚举值 "read_only" | "workspace_write" | "danger_full_access"。

**返回值类型**：[Sandbox](02-agent-config.md) 配置值。

**示例**：

~~~js
const sandbox = "workspace_write";
~~~

### ApprovalPolicy

**作用**：决定执行需要审批时的处理策略。

**输入类型**：枚举值 "auto_review" | "deny_all"。

**返回值类型**：[ApprovalPolicy](02-agent-config.md) 配置值。

**示例**：

~~~js
const approvalPolicy = "auto_review";
~~~

### RuntimeSpec

**作用**：指定 Agent 的工作目录、沙箱和审批策略。

**输入类型**：

~~~ts
interface RuntimeSpec {
  workspace?: string;
  sandbox?: Sandbox;
  approvalPolicy?: ApprovalPolicy;
}
~~~

**返回值类型**：[RuntimeSpec](02-agent-config.md) 配置对象。

**示例**：

~~~js
const runtime = {
  workspace: "/workspace/project",
  sandbox: "workspace_write",
  approvalPolicy: "auto_review",
};
~~~

### AgentConfig

**作用**：组合 Agent 的模型、系统提示、能力、运行时和可派发的 Agent。

**输入类型**：

~~~ts
interface AgentConfig {
  model: ModelSpec;
  system_prompt?: string;
  capabilities?: Capabilities;
  runtime?: RuntimeSpec;
  subagents?: Agent[];
  remote_agents?: RemoteAgent[];
  handoff?: Handle;
}
~~~

**返回值类型**：[AgentConfig](02-agent-config.md) 配置对象；创建 Agent 后以只读配置暴露。

**示例**：

~~~js
const config = {
  model: { model_id: "gpt-6" },
  system_prompt: "只输出可执行的审查结论",
  capabilities: { brainary_tool: ["exec_command"] },
  runtime: { sandbox: "read_only" },
};
~~~
