# 使用 Skills

Skill 是一个以 `SKILL.md` 为入口的可复用说明包。它告诉 Agent 在某类任务中应遵循什么流程，以及需要读取哪些配套资源。

Python SDK 有两种使用方式：通过发现目录为 Agent 配置可自主选择的 Skill 池，或者使用 `SkillInput(name, path)` 强制某个 turn 使用指定 Skill。

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

## 单次强制指定 Skill

```python
from pathlib import Path

from openai_codex import Codex, CodexConfig, SkillInput, TextInput

workspace = Path("/absolute/path/to/workspace")
skill_path = workspace / ".agents/skills/review-changes/SKILL.md"

config = CodexConfig(codex_bin="/absolute/path/to/brainary-codex")

with Codex(config=config) as codex:
    agent = codex.thread_start(cwd=str(workspace))
    result = agent.run(
        [
            TextInput("审查当前未提交改动。"),
            SkillInput("review-changes", str(skill_path.resolve())),
        ]
    )
    print(result.final_response)
```

## 配置可自主选择的 Skill 池

当前运行时会从多个作用域发现 Skills：

| 位置 | 作用范围 | 修改方式 |
| --- | --- | --- |
| 从项目根到当前 `cwd` 路径上的 `.agents/skills` | 项目根的 Skills 对整个项目生效；中间目录的 Skills 只对该目录及其子目录中的 `cwd` 生效 | 通过 `thread_start(cwd=...)` 改变扫描范围；通过 `project_root_markers` 改变项目根识别规则；`.agents/skills` 目录名固定 |
| 项目配置层的 `.codex/skills` | 只对该配置层所属项目内的 Agent 生效，项目外的 Agent 不可见 | 随项目 `.codex` 配置层的位置变化；`skills` 子目录名固定，可增删其中的 Skill 目录 |
| `$HOME/.agents/skills` | 对当前操作系统用户启动的所有项目和 Agent 生效，其他用户不可见 | 可修改 app-server 进程的 `$HOME`，但会影响整个进程环境；通常只增删该目录中的 Skills |
| `$CODEX_HOME/skills` | 对共用该 `$CODEX_HOME` 的所有 Agent 生效，使用其他 `$CODEX_HOME` 的 Agent 不可见 | 可修改 `CODEX_HOME`，但会同时迁移 `config.toml` 等实例数据；不能只迁移这个 Skill 目录 |
| `$CODEX_HOME/skills/.system` | 对共用该 `$CODEX_HOME` 且启用了内置 Skills 的所有 Agent 生效，禁用内置 Skills 的 Agent 不可见 | 跟随 `CODEX_HOME`，不能单独改址；使用 `[skills.bundled] enabled = false` 禁用整个内置作用范围 |
| 管理员配置目录（Unix 通常为 `/etc/codex/skills`） | 对本机或容器内读取该系统配置层的所有用户和 Agent 生效，其他机器或容器不可见 | 由系统管理员部署或修改；普通 Python 应用不能通过 thread 配置改址 |
| 已启用插件中的 `skills/` | 只对该插件的启用范围生效：用户级启用覆盖该用户的所有项目，项目级启用只覆盖对应项目 | 跟随插件安装目录；通过插件配置启用或禁用整个插件，也可用 `[[skills.config]]` 禁用其中的单个 Skill |

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

from openai_codex import Codex, CodexConfig

workspace = Path("/absolute/path/to/workspace").resolve()
config = CodexConfig(codex_bin="/absolute/path/to/brainary-codex")

with Codex(config=config) as codex:
    agent = codex.thread_start(
        cwd=str(workspace),
        developer_instructions=(
            "根据任务和可用 Skill 的 description 自主选择合适的 Skill；"
            "没有匹配项时直接完成任务。"
        ),
    )

    result = agent.run("分析 logs/ci.log 中最近一次构建失败。")
    print(result.final_response)
```

## 启用、禁用和维护 Skill

发现目录中的 Skill 默认启用。为了让 Agent 能够自主选择，应保留自动 Skill 说明；下面的配置还会关闭不需要的内置 Skills：

```toml
# $CODEX_HOME/config.toml
[skills]
include_instructions = true

[skills.bundled]
enabled = false
```

`include_instructions = true` 让每个 turn 获得当前可用 Skill 的目录说明。`skills.bundled.enabled` 只控制运行时内置 Skills，不影响项目目录或用户目录中的 Skills。

需要暂时下线某个 Skill 时，可以在 `$CODEX_HOME/config.toml` 中按 `SKILL.md` 的绝对路径禁用：

```toml
[[skills.config]]
path = "/absolute/path/to/workspace/.agents/skills/diagnose-failure/SKILL.md"
enabled = false
```

也可以按名称配置：

```toml
[[skills.config]]
name = "diagnose-failure"
enabled = false
```

本地 Skill 推荐按路径配置，以免同名 Skill 来自用户目录、项目目录或插件时产生歧义。重新启用时删除对应的禁用项，或者把 `enabled` 改为 `true`。持久配置作用于使用同一 `$CODEX_HOME` 的 Agent；直接修改配置文件后，应新建 Codex 客户端或 thread 以确保新配置生效。

只想为某个 Agent 临时改变启用状态时，可以在 `thread_start(config=...)` 中提供 session 级覆盖，不修改全局文件：

```python
disabled_skill = (
    workspace / ".agents/skills/diagnose-failure/SKILL.md"
).resolve()

release_agent = codex.thread_start(
    cwd=str(workspace),
    config={
        "skills": {
            "config": [
                {
                    "path": str(disabled_skill),
                    "enabled": False,
                }
            ]
        }
    },
)
```

这适合从当前 Agent 已发现的 Skill 池中临时排除少量 Skill。禁用项较多时，应直接调整项目级 Skill 池，避免维护大量配置项。

## 同时传入文本、图片和 Skill

turn 输入可以是多项列表：

```python
from openai_codex import LocalImageInput, SkillInput, TextInput

result = agent.run(
    [
        TextInput("按照界面审查 Skill 分析这张截图。"),
        LocalImageInput("/absolute/path/to/screenshot.png"),
        SkillInput("ui-review", "/absolute/path/to/ui-review/SKILL.md"),
    ]
)
```

支持的公开输入类型包括 `TextInput`、`ImageInput`、`LocalImageInput`、`SkillInput` 和 `MentionInput`。单个字符串等价于 `TextInput(...)`。
