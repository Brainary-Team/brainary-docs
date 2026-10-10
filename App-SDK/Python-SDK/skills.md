# 使用 Skills

Skill 是一个以 `SKILL.md` 为入口的可复用说明包。它告诉 Agent 在某类任务中应遵循什么流程，以及需要读取哪些配套资源。

Python SDK 有两种使用方式：通过发现目录为 Agent 配置可自主选择的 Skill 池，或者使用 `SkillInput(name, path)` 为某个 turn 显式选择已发现且启用的 Skill。

## 创建 Skill

在工作区创建：

```text
.agents/
└── skills/
    └── review-changes/
        ├── SKILL.md
        └── references/
            └── checklist.md
```

`SKILL.md` 示例：

```markdown
---
name: review-changes
description: Review a Git diff and report correctness risks with file and line evidence.
---

# Review changes

1. Read the repository instructions first.
2. Inspect the diff and the affected call sites.
3. Report correctness, security, and compatibility problems before style issues.
4. Cite the file and line for every finding.
5. If there are no findings, state that explicitly and list remaining test gaps.
```

`name` 和 `description` 会用于 Skill 元数据；目录名也会在缺少显式名称时参与默认命名。

## 单次显式指定 Skill

```python
from pathlib import Path

from brainary import Brainary, BrainaryConfig, SkillInput, TextInput

workspace = Path("/absolute/path/to/workspace")
skill_path = workspace / ".agents/skills/review-changes/SKILL.md"

config = BrainaryConfig(brainary_bin="/absolute/path/to/brainary")

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(cwd=str(workspace))
    result = agent.run(
        [
            TextInput("审查当前未提交改动。"),
            SkillInput("review-changes", str(skill_path.resolve())),
        ]
    )
    print(result.final_response)
```

## 配置可自主选择的 Skill 池

通过 `thread_start(cwd=...)` 指定项目工作目录，运行时会发现项目根到该 `cwd`
路径上各级 `.agents/skills` 中的 Skill。其他作用域的发现位置和持久管理方式见
[配置文件使用说明](configuration.md#skills-管理)。

例如，为一个工程 Agent 配置包含代码审查、故障诊断和发布说明三项能力的 Skill 池：

```text
workspace/
├── .agents/
│   └── skills/
│       ├── review-changes/
│       │   └── SKILL.md
│       ├── diagnose-failure/
│       │   ├── SKILL.md
│       │   └── references/
│       │       └── runbook.md
│       └── write-release-notes/
│           ├── SKILL.md
│           └── templates/
│               └── release.md
└── src/
```

每个 `SKILL.md` 的 `description` 应描述触发条件，而不是只写笼统能力。例如：

```yaml
---
name: diagnose-failure
description: Diagnose failed builds, tests, or runtime incidents from logs; identify the root cause and propose evidence-backed fixes.
---
```

Agent 会先看到可用 Skill 的名称和描述，再按任务语义决定是否选择。多个 Skill 的 `description` 如果高度重叠，模型的选择会不稳定；应让每个 Skill 的边界清晰、互斥，并在正文中说明何时不应使用。

Python 侧不需要逐项注册这个池。只要把 Agent 的 `cwd` 指向上述工作区，运行时就会发现 `.agents/skills` 下的三个 Skill：

```python
from pathlib import Path

from brainary import Brainary, BrainaryConfig

workspace = Path("/absolute/path/to/workspace").resolve()
config = BrainaryConfig(brainary_bin="/absolute/path/to/brainary")

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(
        cwd=str(workspace),
        developer_instructions=(
            "根据任务和可用 Skill 的 description 自主选择合适的 Skill；"
            "没有匹配项时直接完成任务。"
        ),
    )

    result = agent.run("分析 logs/ci.log 中最近一次构建失败。")
    print(result.final_response)
```

## 用代码配置 Skill 池

在 `thread_start(config=...)` 中设置 `skills`，可以为这个 Agent 控制自动目录说明、
内置 Skills 和单项启用状态：

```python
from pathlib import Path

from brainary import Brainary, BrainaryConfig

workspace = Path("/absolute/path/to/workspace").resolve()
disabled_skill = workspace / ".agents/skills/diagnose-failure/SKILL.md"
config = BrainaryConfig(brainary_bin="/absolute/path/to/brainary")

with Brainary(config=config) as brainary:
    agent = brainary.thread_start(
        cwd=str(workspace),
        config={
            "skills": {
                "include_instructions": True,
                "bundled": {"enabled": False},
                "config": [
                    {"path": str(disabled_skill.resolve()), "enabled": False},
                ],
            }
        },
    )
    result = agent.run("根据当前改动生成发布说明。")
    print(result.final_response)
```

| 字段 | 含义 |
| --- | --- |
| `include_instructions` | 默认为 `True`，向 turn 提供可用 Skills 的自动目录说明；设为 `False` 只关闭此说明，不等于禁用 Skill |
| `bundled.enabled` | 默认为 `True`；设为 `False` 排除运行时内置 Skills，保留项目、用户等其他作用域的 Skills |
| `config[].path` | 按 `SKILL.md` 的绝对路径选择 Skill |
| `config[].name` | 按名称选择 Skill，会匹配当前发现的所有同名 Skill |
| `config[].enabled` | 控制匹配 Skill 的启用状态，必须显式提供布尔值 |

每项规则必须且只能指定 `path` 或 `name`。本地 Skill 推荐按路径配置；按名称禁用
可以改用 `{"name": "diagnose-failure", "enabled": False}`。重新启用时传入对应的
`enabled=True` 规则；如果已有较低优先级的禁用规则，仅省略当前覆盖不会重新启用。

`skills.config` 用于筛选已发现的 Skill，不会把任意路径注册为新的 Skill 池。
`SkillInput` 同样只能选择已发现且启用的 Skill，不会绕过禁用规则。以上覆盖不会
写入配置文件；多个 Agent 可以复用配置字典，跨进程恢复或分叉 thread 时需再次传入。
配置文件中的启用、禁用和维护操作见[配置文件使用说明](configuration.md#skills-管理)。

## 同时传入文本、图片和 Skill

将 `ui-review` 放入同一工作区的 `.agents/skills` 后，turn 输入可以是多项列表：

```python
from brainary import LocalImageInput, SkillInput, TextInput

result = agent.run(
    [
        TextInput("按照界面审查 Skill 分析这张截图。"),
        LocalImageInput("/absolute/path/to/screenshot.png"),
        SkillInput(
            "ui-review",
            str(workspace / ".agents/skills/ui-review/SKILL.md"),
        ),
    ]
)
```

支持的公开输入类型包括 `TextInput`、`ImageInput`、`LocalImageInput`、`SkillInput` 和 `MentionInput`。单个字符串等价于 `TextInput(...)`。
