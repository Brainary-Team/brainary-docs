# Brainary API Key 使用前配置说明

API Key 是模型服务给你的访问凭证。把它和模型信息配置好，Brainary 才能连接对应服务。

本文默认你已经可以运行 `brainary` 命令。第一次配置，按下面 4 步操作即可；厂商直连和中转服务都使用同一套步骤。目前不需要执行登录命令，也不需要手动创建 `auth.json`。

> API Key 不要发给他人、放进截图或提交到 Git 仓库。下面的 Key 和地址都是占位示例，需要替换。

## 第 1 步：准备 4 项信息

向管理员或模型服务商获取：

| 需要什么 | 怎么确认 |
| --- | --- |
| 模型 ID | 服务商提供的准确名称，不是模型的宣传名称 |
| 服务地址 | 用于程序连接的 API 根地址，不是服务商的网站首页 |
| 接口类型 | 确认是 `responses` 还是 `anthropic` |
| API Key | 你自己的访问凭证 |

不知道接口类型时，可以先判断API_key：“这个服务支持 Responses 接口，还是 Anthropic Messages 接口？API 根地址和模型 ID 是什么？”只支持 Chat Completions 的服务，当前不能直接接入。

## 第 2 步：填写配置文件

在终端中运行与你的系统对应的命令，打开配置文件：

如果文件已经有内容，不要清空或直接追加整段模板，先看[已有配置怎么修改](#已有配置怎么修改)。设置过 `CODEX_HOME` 的用户，使用[该变量指定的目录](#我设置过-codex_home)。

**macOS / Linux：**

```bash
mkdir -p ~/.codex
nano ~/.codex/config.toml
```

**Windows PowerShell：**

```powershell
New-Item -ItemType Directory -Force "$HOME\.codex"
notepad "$HOME\.codex\config.toml"
```

文件为空时，复制下面这一份模板。只需要修改 `model`、`base_url`、`wire_api` 三项，其余先保持原样：

```toml
model = "替换为模型ID"
model_provider = "vendor"

[model_providers.vendor]
name = "我的模型服务"
base_url = "https://api.example.com/v1"
env_key = "VENDOR_API_KEY"
wire_api = "responses"
requires_openai_auth = false
```

| 修改哪一项 | 填什么 |
| --- | --- |
| `model` | 第 1 步拿到的模型 ID |
| `base_url` | 第 1 步拿到的 API 根地址 |
| `wire_api` | Responses 接口填 `"responses"`；Anthropic Messages 接口填 `"anthropic"` |

服务地址不要包含最后的 `/responses` 或 `/messages`，程序会自动追加。例如，服务商给的是 `https://api.example.com/v1/messages`，这里就填 `https://api.example.com/v1`。

保存文件：使用 `nano` 时，按 `Ctrl+O`、`Enter` 保存，再按 `Ctrl+X` 退出；使用记事本时，按 `Ctrl+S` 保存后关闭。

## 第 3 步：在当前终端设置 Key

回到刚才的终端，将下面的占位文字替换成你的真实 Key，再执行命令。保留外面的引号。

**macOS / Linux：**

```bash
export VENDOR_API_KEY="替换为你的真实APIKey"
```

**Windows PowerShell：**

```powershell
$env:VENDOR_API_KEY = "替换为你的真实APIKey"
```

`VENDOR_API_KEY` 是给这个 Key 起的变量名，要与配置里的 `env_key` 完全一致。按本文操作时，不需要改这个名称。

这一步只对当前终端及其启动的程序有效。不要关闭或更换窗口，直接继续第 4 步。命令可能留在终端历史中，不要分享含真实 Key 的历史记录。

## 第 4 步：启动并验证

如果 Brainary 已经在运行，先退出，再从设置了 Key 的同一个终端启动：

```bash
brainary
```

进入界面后，输入 `/status` 核对模型，再发送：“请只回复：配置成功”。能收到正常回复，说明基本连接已经通了。

也可以在终端里直接验证一次：

```bash
brainary exec --skip-git-repo-check "请只回复：配置成功"
```

这里的 `--skip-git-repo-check` 允许在普通目录验证，不要求当前目录是 Git 仓库。工具调用、图片等功能是否可用，还取决于所选模型和服务。

## 没成功时，看这里

| 遇到的情况 | 先检查什么 |
| --- | --- |
| 找不到 `brainary` 命令 | 先确认已安装程序，并且终端能找到它；这一步与 Key 无关 |
| 提示环境变量不存在 | 是否执行过第 3 步？是否换了终端？变量名是否为 `VENDOR_API_KEY`？ |
| 返回 `401` 或“未授权” | Key 是否填错、过期，是否有当前模型的使用权限？特殊认证方式见下文 |
| 返回 `404` 或“接口不存在” | 地址和接口类型是否对应？地址末尾是否多写了 `/responses` 或 `/messages`？ |
| 提示“模型不存在” | `model` 是否使用了服务商提供的准确模型 ID？ |
| 提示 `Brainary credentials are not configured` | 配置是否保存到了正确目录？`model_provider = "vendor"` 是否写在顶层，且厂商配置中有 `requires_openai_auth = false`？ |

### 已有配置怎么修改

- 找到原有的 `model`、`model_provider`，修改它们的值；没有时，加到文件第一个 `[分组名]` 之前。
- 找到 `[model_providers.vendor]`，修改里面的设置；没有这个分组时，才把模板中的这个分组及其内容加到末尾。
- 保留其他配置，不要重复添加同名设置或同名分组。`model_provider` 的值必须与分组名最后一段一致，例如都使用 `vendor`。

### 换个终端后又不能用了

第 3 步设置的是临时变量。新开终端后，重新执行第 3 步，再启动 Brainary。

如果想长期保存，可以请管理员通过系统或密钥管理工具设置环境变量。直接写进 Shell 启动文件会明文保存 Key，需按所在组织的要求处理。

### 服务商要求使用 `x-api-key`

仅当服务商明确要求这个认证方式时，才需要看这一项。

本文模板默认把 Key 放在 `Authorization: Bearer` 请求头中；选择 `anthropic` 也不会自动改成 `x-api-key`。如果服务商要求额外发送 `x-api-key`，在配置文件末尾添加：

```toml
[model_providers.vendor.env_http_headers]
"x-api-key" = "VENDOR_API_KEY"
```

右侧填的是变量名，不是真实 Key；变量仍按第 3 步设置。已有这个分组时，直接修改原分组。

这会同时发送 Bearer 和 `x-api-key`。如果服务商要求只发其中一种，或使用其他签名方式，请让管理员按服务要求配置，不能只靠切换 `wire_api` 解决。

### 我设置过 `CODEX_HOME`

`CODEX_HOME` 用来指定配置和数据放在哪里。没有设置时，默认使用 `~/.codex`；设置过时，请编辑指定目录里的 `config.toml`，不要继续修改默认目录中的文件。

查看当前设置：macOS / Linux 执行 `echo "$CODEX_HOME"`；Windows PowerShell 执行 `$env:CODEX_HOME`。没有输出表示使用默认目录。

自定义目录必须先创建，再将它的 absolute path 设为 `CODEX_HOME`。目录可以叫 `.brainary`，但当前版本不会自动查找它，也不识别 `BRAINARY_HOME`。切换目录不会自动搬走旧配置和历史记录。

如果启动时还使用了 `--profile <name>`，也要检查该目录里的 `<name>.config.toml`，它可能覆盖刚修改的配置。

