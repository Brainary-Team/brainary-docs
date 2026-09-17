# Brainary API Key 使用前配置说明

Brainary 当前不提供账号登录入口，也没有可以直接使用的默认云模型服务。第一次使用前，需要在 `config.toml` 中填写模型名称、服务地址和接口类型，再通过环境变量提供 API Key。不需要创建或配置 `auth.json`。

> *API Key 属于敏感凭据。请从你的管理员或模型服务厂商获取；不要把真实 Key 发给他人，也不要放进截图、Issue、聊天记录或 Git 仓库。*
> 
> 

## 一、先判断厂商接口是否兼容

先查看模型服务商的文档，确认它支持下面至少一种接口：

- `wire_api = "responses"`：接口地址通常以 `/v1/responses` 结尾。

- `wire_api = "anthropic"`：接口地址通常以 `/v1/messages` 结尾。

只提供 Chat Completions（例如 `/v1/chat/completions`）的服务，目前不能直接接入 Brainary。

## 二、配置模型服务的 API Key

API Key 不能单独使用，还需要告诉 Brainary 使用哪个模型、连接哪个地址，以及使用哪种接口。

### 1\. 创建配置目录

macOS / Linux：

```Bash
mkdir -p ~/.codex
```

Windows PowerShell：

```PowerShell
New-Item -ItemType Directory -Force "$HOME\.codex"
```

### 2\. 编辑 `config.toml`

macOS / Linux：

```Bash
nano ~/.codex/config.toml
```

Windows PowerShell：

```PowerShell
notepad "$HOME\.codex\config.toml"
```

在文件中添加厂商配置。如果文件中已有其他设置，请保留原内容。

#### Anthropic Messages 兼容厂商模板

```Python
model = "厂商提供的模型ID"
model_provider = "vendor"

[model_providers.vendor]
name = "厂商名称"
base_url = "https://厂商提供的接口地址/v1"
env_key = "VENDOR_API_KEY"
wire_api = "anthropic"
requires_openai_auth = false
```

#### Responses 兼容厂商模板

```Python
model = "厂商提供的模型ID"
model_provider = "vendor"

[model_providers.vendor]
name = "厂商名称"
base_url = "https://厂商提供的接口地址/v1"
env_key = "VENDOR_API_KEY"
wire_api = "responses"
requires_openai_auth = false
```

需要替换的内容：

- `model`：厂商提供的准确模型 ID。

- `name`：用于识别该厂商的显示名称。

- `base_url`：厂商提供的兼容接口根地址；末尾是否包含 `/v1` 以厂商文档为准。

- `env_key`：保存 API Key 的环境变量名称，可以使用厂商建议的名称。

- `wire_api`：根据厂商支持的协议填写 `anthropic` 或 `responses`。

`model_provider = "vendor"` 必须与 `[model_providers.vendor]` 中的 `vendor` 完全一致。你也可以把 `vendor` 换成便于识别的英文标识，例如 `company` 或其他喜欢的字符。

使用上面的自定义 Provider 配置时，`requires_openai_auth = false` 会让 Brainary 直接读取 `env_key` 指定的环境变量，因此不需要 `auth.json`。

### 3\. 配置厂商 API Key 环境变量

环境变量名称必须与 `config.toml` 的 `env_key` 完全一致。以上面使用的 `VENDOR_API_KEY` 为例：

macOS / Linux 当前终端：

```Bash
export VENDOR_API_KEY="替换为厂商APIKey"
```

Windows PowerShell 当前窗口：

```PowerShell
$env:VENDOR_API_KEY = "替换为厂商APIKey"
```

然后从同一个终端窗口启动 Brainary。如果改用了其他环境变量名称，例如 `COMPANY_API_KEY`，上面的命令和 `env_key` 都要同步改成该名称。

如果希望每次打开终端后都可用，可以通过操作系统、公司密钥管理工具或 Shell 启动配置持久设置该环境变量。直接写入 Shell 配置文件虽然方便，但会以明文保存 Key，请根据所在组织的安全要求选择。

### 4\. 一个完整示例

下面只是展示配置结构，示例地址不能直接使用。请把模型 ID、服务地址和环境变量名称替换成服务商提供的真实信息。

`~/.codex/config.toml`：

```Python
model = "your-model-id"
model_provider = "company"

[model_providers.company]
name = "Company Model Service"
base_url = "https://api.example.com/v1"
env_key = "COMPANY_API_KEY"
wire_api = "responses"
requires_openai_auth = false
```

macOS / Linux 当前终端：

```Bash
export COMPANY_API_KEY="替换为你的APIKey"
brainary
```

Windows PowerShell 当前窗口：

```PowerShell
$env:COMPANY_API_KEY = "替换为你的APIKey"
brainary
```

## 三、配置第三方中转服务

第三方中转服务的配置方式与模型厂商相同。下面以 Responses 兼容中转服务为例：

```Python
model = "中转服务提供的模型ID"
model_provider = "relay"

[model_providers.relay]
name = "第三方中转"
base_url = "https://中转服务地址/v1"
env_key = "RELAY_API_KEY"
wire_api = "responses"
requires_openai_auth = false
```

然后在启动 Brainary 的终端中设置：

```Bash
export RELAY_API_KEY="替换为你的APIKey"
brainary
```

如果中转服务提供 Anthropic Messages 接口，把 `wire_api` 改为 `"anthropic"`，并按服务商要求填写 `base_url`。只提供 `/v1/chat/completions` 的中转服务当前不能直接接入。

## 四、验证配置

关闭已经运行的 Brainary，然后重新打开终端。使用其他厂商时，先确认 API Key 环境变量在当前终端中有效。

在需要处理的项目目录中运行：

```Bash
brainary
```

进入 TUI 后发送一条简单消息。如果能够正常返回模型回复，说明模型、服务地址、协议和 API Key 均已配置成功。

也可以在项目目录中执行一次非交互请求：

```Bash
brainary exec "请只回复：配置成功"
```

## 五、常见问题

### 提示环境变量不存在

检查 `config.toml` 中的 `env_key` 是否与实际设置的环境变量名称完全一致。设置环境变量后，必须从能够读取该变量的同一个终端启动 Brainary。

macOS / Linux 可检查变量是否存在，但不要展示或分享完整值：

```Bash
test -n "$VENDOR_API_KEY" && echo "已设置" || echo "未设置"
```

Windows PowerShell：

```PowerShell
if ($env:VENDOR_API_KEY) { "已设置" } else { "未设置" }
```

### 请求返回 404 或接口不存在

通常是 `base_url` 或 `wire_api` 选择错误。确认厂商实际提供的是 `/v1/responses` 还是 `/v1/messages`，并检查 `base_url` 是否多写或漏写了路径。

### 请求返回 401 或未授权

确认 API Key 没有多余空格、没有被撤销或过期，并且属于当前配置的厂商。当前接入路径使用 Bearer 认证；如果厂商只接受其他认证头，即使接口协议兼容也可能无法直接使用。

### 模型不存在

`model` 必须填写厂商接口接受的准确模型 ID，不能只填写宣传名称。

### 提示 Brainary credentials are not configured

这通常表示仍在使用需要内置认证的 Provider。检查 `model_provider` 是否指向你在 `[model_providers.<名称>]` 中定义的配置，并确认该配置包含：

```Python
env_key = "你的APIKey环境变量名称"
requires_openai_auth = false
```

同时确认对应环境变量已设置，然后退出并重新启动 Brainary。

### 使用了自定义配置目录

如果环境中设置了 `CODEX_HOME`，Brainary 会从该目录读取 `config.toml`，而不是从默认的 `~/.codex` 目录读取。例如：

```Bash
export CODEX_HOME="/absolute/path/to/brainary-config"
```

这时配置文件应位于：

```Python
/absolute/path/to/brainary-config/config.toml
```

不确定是否设置过时，macOS / Linux 可执行：

```Bash
echo "$CODEX_HOME"
```

Windows PowerShell 可执行：

```PowerShell
$env:CODEX_HOME
```

如果输出为空，继续使用默认的 `.codex` 目录即可。

## 六、兼容名称说明

`CODEX_HOME` 和 `.codex` 是 Brainary 当前保留的底层兼容标识，不是界面上的产品名称。不要自行改成 `BRAINARY_HOME` 或 `.brainary`，否则程序无法读取配置。

API Key 环境变量名称由 `model_providers.<名称>.env_key` 决定，可以根据模型厂商、中转服务或组织规范命名。

## 七、安全注意事项

- 不要把包含真实 API Key 的配置提交到 Git 仓库。

- 不要在截图、Issue、文档或聊天中展示真实 API Key。

- 不要把真实 API Key 写进可共享的脚本。

- 使用环境变量命令时，注意终端历史可能保存输入内容。

- 如果怀疑 Key 已泄露，请立即到对应厂商后台撤销并重新生成。

