# 新设备开发环境配置（Linux / macOS）

目标：让 agent 在一台新设备上完成基础开发环境、Codex、Claude Code 和 Magpie 的安装、接入与验证。默认使用普通用户，适用于 macOS 和常见 Linux 发行版；Windows 原生环境见 [Windows 专区](./windows/index.md)。

默认组件：Zsh、Git、SSH、curl、jq、ripgrep、Python 3、Node.js LTS / npm、Docker / Compose、Codex CLI、Claude Code、Magpie。Go、Java、OpenCloudOS 专项设置、Shell 主题和额外语言工具链按项目需要另装。

## 1. 给执行 agent 的约定

按本文顺序逐节执行，每节完成后验证结果。命令失败时先处理失败原因，再继续依赖它的步骤；不要把所有代码块拼成一个脚本运行。

- 先识别系统、CPU 架构、当前用户、Shell 和现有版本。已有可用组件可以复用，避免重复安装不同来源的同名程序。
- 系统包通过系统包管理器安装；Node 使用用户级 nvm；Codex 和 Claude Code **按本文要求通过 npm 安装**；Magpie 使用官方发行版本。
- 保留已有代理、PATH、Git 身份、SSH 密钥及 agent 配置。重跑时检查是否已安装、是否已有配置块；只合并必要字段，不覆盖整个配置文件。
- 凭据通过用户提供的安全渠道、交互输入或登录流程取得，不写入本仓库、报告、命令历史或完整配置输出。以下变量名和示例域名都不是可用凭据。
- 有 sudo 或浏览器登录交互时，让用户完成对应步骤；继续完成其他独立工作，并在最终报告中列出待完成项。不要把“安装成功”当成“模型调用成功”。

需要提前确定的信息：

|信息|用途|获取方式|
|---|---|---|
|系统、架构、普通用户名、sudo 是否可用|选择安装命令|本机检测|
|`GIT_USER_NAME`、`GIT_USER_EMAIL`|Git 提交身份|用户提供；已有值优先保留|
|网络代理（如需）、内部 CA（如需）|访问软件源与模型服务|用户或设备管理配置|
|Magpie 模式：新建 / 恢复备份 / 连接已有网关|选择第 7 节的接入路径|默认本机新建|
|Relay 地址、API 协议、认证方式、模型列表|配置内部 provider|服务管理员提供|
|DeepSeek API key、Factory 等订阅账号|配置用户实际拥有的 provider|用户提供或交互登录|
|Codex 与 Claude Code 各自使用的模型 ID|选择默认模型|从 `magpie models` 的实际结果中选|

检测命令：

```bash
uname -s
uname -m
id -un
printf '%s\n' "$SHELL"
if [ -f /etc/os-release ]; then cat /etc/os-release; fi
for tool in zsh git ssh curl jq rg python3 node npm docker codex claude magpie; do
  command -v "$tool" || true
done
```

开始修改前，把已有的 `~/.zshrc`、`~/.zprofile`、`~/.gitconfig`、`~/.codex/config.toml`、`~/.codex/auth.json`、`~/.claude/settings.json`、`~/.claude.json` 和已有 Magpie 配置备份到**仓库之外**。备份目录权限设为 `700`，含凭据文件设为 `600`。只备份存在的文件，记录备份位置。

### 网络与代理（需要时）

先复用设备已有代理和证书。需要临时设置代理时，由用户提供不含口令的 `SETUP_PROXY_URL`，在当前安装 Shell 中加载：

```bash
: "${SETUP_PROXY_URL:?提供当前设备可用的代理地址}"
export HTTP_PROXY="$SETUP_PROXY_URL"
export HTTPS_PROXY="$SETUP_PROXY_URL"
export http_proxy="$SETUP_PROXY_URL"
export https_proxy="$SETUP_PROXY_URL"
setup_no_proxy="${NO_PROXY:-${no_proxy:-}}"
export NO_PROXY="localhost,127.0.0.1,::1${setup_no_proxy:+,$setup_no_proxy}"
export no_proxy="$NO_PROXY"
```

按需合并内部域名与网段；本机 Magpie 的回环地址必须绕过代理。不要把旧设备的代理地址直接搬过来。Magpie 后台服务和 Docker Engine 的网络环境需分别在其设置中配置，不能假设它们继承当前终端环境。内部 CA 通过系统信任存储配置，不通过关闭 TLS 校验解决下载问题。

## 2. 安装系统基础工具

以下分支只执行与当前系统匹配的一支。全新设备缺少 sudo 权限时，先由管理员完成系统包安装。

### Ubuntu / Debian

```bash
sudo apt-get update
sudo apt-get install -y \
  ca-certificates curl git openssh-client zsh jq ripgrep \
  python3 python3-pip python3-venv build-essential pkg-config unzip
```

### Fedora

```bash
sudo dnf install -y \
  ca-certificates curl git openssh-clients zsh jq ripgrep \
  python3 python3-pip gcc gcc-c++ make pkgconf-pkg-config unzip
```

其他 Linux 发行版先根据 `/etc/os-release` 找到对应包管理器和官方安装方式，不套用 Ubuntu 的仓库地址。

### macOS

先检查 Command Line Tools：

```bash
xcode-select -p
```

如果尚未安装，执行 `xcode-select --install`，等待用户完成系统安装窗口，再继续。系统自带 `/bin/zsh`，可以直接使用。

没有 Homebrew 时，使用 [Homebrew 官方安装方式](https://brew.sh/)：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

依据安装器打印的实际路径加载 Homebrew；Apple Silicon 通常为 `/opt/homebrew`，Intel 通常为 `/usr/local`：

```bash
if [ -x /opt/homebrew/bin/brew ]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [ -x /usr/local/bin/brew ]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi
brew install git jq ripgrep python pkgconf
```

将安装器要求的 `brew shellenv` 行合并到 `~/.zprofile`，避免重复追加。后面第 4 节也会让非登录 Zsh 加载 Homebrew。不要使用 `sudo brew install`。

## 3. Git 与 SSH

如果尚未设置提交身份，使用用户确认的值；不要从设备登录名猜测姓名和邮箱：

```bash
git config --global --get user.name
git config --global --get user.email
```

缺少配置时再执行：

```bash
: "${GIT_USER_NAME:?先设置用户提供的 Git 姓名}"
: "${GIT_USER_EMAIL:?先设置用户提供的 Git 邮箱}"
git config --global user.name "$GIT_USER_NAME"
git config --global user.email "$GIT_USER_EMAIL"
```

全新环境可设置 `git config --global init.defaultBranch main`；已有仓库的分支不受它影响。GitHub、公司 Git 服务分别按实际授权方式连接，优先复用可用的 SSH agent 或凭据管理器。

使用 SSH 且没有现有密钥时，执行下面的交互命令。目标文件已存在时不覆盖；口令由用户输入：

```bash
ssh-keygen -t ed25519 -C "$GIT_USER_EMAIL"
```

将生成的 `.pub` 公钥登记到目标 Git 服务，再用目标仓库执行 `git ls-remote <仓库地址>` 验证访问。不要复制旧设备的私钥到仓库，也不要关闭 SSH 主机密钥校验。

## 4. Node.js、npm 与 Zsh

### 安装 nvm 和当前 LTS

本文核对时使用 nvm `v0.41.0`。安装前可查看 [nvm 官方说明](https://github.com/nvm-sh/nvm#installing-and-updating)，已有 nvm 时直接加载，不重复安装：

```bash
export NVM_DIR="$HOME/.nvm"
if [ ! -s "$NVM_DIR/nvm.sh" ]; then
  curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.41.0/install.sh \
    | PROFILE=/dev/null bash
fi
. "$NVM_DIR/nvm.sh"
nvm install --lts
nvm alias default 'lts/*'
nvm use default
node --version
npm --version
npm config get prefix
```

npm 随 Node 一起安装，不再通过 apt、dnf 或 Homebrew 安装另一份 Node/npm。Node 至少应满足当前两个 CLI 的 `engines` 要求；第 5 节会再检查。nvm 管理的全局包应位于当前用户目录下。

### 让新终端与 agent 能找到工具

把下面的块**合并一次**到 `~/.zshrc`，保留文件中已有配置。同名功能已配置时调整现有块，不再重复添加：

```zsh
# >>> life-note development environment >>>
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

path=("$HOME/.local/bin" $path)
typeset -U path
export PATH

export NVM_DIR="$HOME/.nvm"
[[ -s "$NVM_DIR/nvm.sh" ]] && source "$NVM_DIR/nvm.sh"
# <<< life-note development environment <<<
```

检查后，在支持 `chsh` 的正常登录环境中切换默认 Shell：

```bash
zsh -n "$HOME/.zshrc"
chsh -s "$(command -v zsh)"
zsh -lic 'command -v node; node --version; npm --version'
```

`chsh` 可能要求用户密码；目录服务管理的账号应按管理员要求修改。macOS 的 `/bin/zsh` 已可用，不必另装 Zsh。非交互脚本、systemd / launchd 服务不会自动加载 `.zshrc`，需要显式加载 `nvm.sh` 或配置实际的可执行文件路径；不要把 nvm 加载代码塞进 `.zshenv`。

如需 Oh My Zsh、Powerlevel10k、fzf、zoxide，再按 [Zsh 进阶配置](./Zsh.md) 选择安装。它们不影响本篇的基础验收。

## 5. 通过 npm 安装 Codex 与 Claude Code

先读取注册表中的版本和 Node 要求，确认当前 LTS 满足要求，再安装：

```bash
npm view @openai/codex version engines --json
npm view @anthropic-ai/claude-code version engines --json
npm install -g @openai/codex@latest @anthropic-ai/claude-code@latest
codex --version
claude --version
npm ls -g --depth=0
```

这两条 npm 包名来自 [OpenAI 官方 Codex CLI 文档](https://developers.openai.com/codex/cli/) 和 [Claude Code 官方 npm 安装说明](https://code.claude.com/docs/en/setup#install-with-npm)。安装保留默认 optional dependencies，不加 `--omit=optional`，也不使用 `sudo npm install -g`。如果当前 npm 版本遇到 Claude 文档列出的兼容问题，按其安装故障页面处理后重试，保持 npm 安装这一目标。

如果执行路径来自旧的 Homebrew、独立安装器或系统 Node，先检查 `command -v codex`、`command -v claude` 和 PATH 优先级，确保实际运行的是这次 npm 安装的版本。以后切换 nvm 的 Node 主版本时，需在新版本下重新安装这两个全局包。

先完成二进制安装即可；模型登录和默认模型选择在第 7 节统一通过 Magpie 完成。

## 6. Python 与 Docker

### Python 3

系统安装完成后检查解释器，项目依赖放在各自的虚拟环境里：

```bash
python3 --version
python3 -m pip --version
```

以下示例在目标项目根目录执行，已有 `.venv` 时复用，不重复创建：

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
deactivate
```

把 `.venv/` 放入项目的 `.gitignore`。不要用 `sudo pip` 修改系统 Python；macOS 也不要替换系统自带的 Python 路径。

### Ubuntu / Debian：Docker Engine

新设备从 Docker 官方仓库安装 Engine、Buildx 和 Compose 插件。已有 Docker、Podman 或 containerd 的设备先确认其用途和兼容性，不自动删除已有容器或运行时。

在 Bash 中执行此块；它仅支持 Ubuntu / Debian，其他发行版走下一分支：

```bash
. /etc/os-release
case "$ID" in
  ubuntu) docker_distro=ubuntu; docker_codename="${UBUNTU_CODENAME:-$VERSION_CODENAME}" ;;
  debian) docker_distro=debian; docker_codename="$VERSION_CODENAME" ;;
  *) printf '%s\n' '请使用当前发行版的 Docker 官方安装文档'; exit 1 ;;
esac
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL "https://download.docker.com/linux/$docker_distro/gpg" \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF_DOCKER
Types: deb
URIs: https://download.docker.com/linux/$docker_distro
Suites: $docker_codename
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF_DOCKER
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run --rm hello-world
sudo docker compose version
```

软件源方式参见 [Ubuntu](https://docs.docker.com/engine/install/ubuntu/) / [Debian](https://docs.docker.com/engine/install/debian/) 官方文档。默认通过 `sudo docker` 使用；若用户需要免 sudo，再按 [Linux post-install](https://docs.docker.com/engine/install/linux-postinstall/) 配置。`docker` 组具有相当于 root 的主机权限，不把放宽 socket 权限作为修复办法。

### Fedora：Docker Engine

```bash
sudo curl -fsSL https://download.docker.com/linux/fedora/docker-ce.repo \
  -o /etc/yum.repos.d/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run --rm hello-world
sudo docker compose version
```

参见 [Fedora 官方安装说明](https://docs.docker.com/engine/install/fedora/)。其他 Linux 发行版从 [Docker Engine 安装入口](https://docs.docker.com/engine/install/) 选择对应分支。

### macOS：Docker Desktop

按 [Docker Desktop for Mac](https://docs.docker.com/desktop/setup/install/mac-install/) 选择与 Apple Silicon / Intel 匹配的安装器。已配置 Homebrew 时也可按 [Homebrew cask](https://formulae.brew.sh/cask/docker-desktop) 执行：

```bash
brew install --cask docker-desktop
open -a Docker
```

首次启动需要用户完成安装窗口中的设置。等待 Engine 就绪后执行：

```bash
docker version
docker compose version
docker run --rm hello-world
```

macOS 只有 Docker CLI 不能运行本地容器，还需要 Docker Desktop 提供的 Linux 虚拟机与 Engine。

## 7. 安装并接入 Magpie

### 7.1 官方程序与后台服务

这里的 Magpie 是 [yetone/magpie](https://github.com/yetone/magpie)，统一管理 agent 模型的本地网关。使用官方二进制，provider 和订阅通过配置或插件接入；不要求安装 Go 或自行编译。

按 [Magpie 官方入门文档](https://usemagpie.ai/docs/start) 安装。普通设备使用官方安装器；无图形界面的 Linux 服务器可明确选择终端构建：

```bash
# 默认构建：macOS / Linux
curl -fsSL https://usemagpie.ai/install.sh | sh
```

服务器若要选择终端构建，用下面的命令**代替**上一条：

```bash
curl -fsSL https://usemagpie.ai/install.sh | sh -s -- --tui
```

按安装器提示更新 PATH，并检查：

```bash
magpie --version
magpie daemon install --web
magpie daemon start
magpie daemon status
curl --noproxy '*' -fsS http://127.0.0.1:3425/
```

Linux 使用用户级 systemd，macOS 使用 launchd。Linux 服务器如需登出后继续运行，由管理员按实际要求启用该用户的 linger。没有用户级服务管理器的容器可在独立终端运行 `magpie serve --web`，交给容器或已有进程管理器维护，不再启动第二份抢占端口的实例。

网关默认是本机 `127.0.0.1:3425`，Web 默认端口为 `3430`。**使用 `magpie daemon status` 打印的实际登录链接**，其中带有 Web 登录 key，不将它提交到仓库。Web 的 key 与模型网关调用凭据不是同一个概念。

### 7.2 选择 provider 的来源

以下三种路径按设备用途选择；恢复已有配置后不用再重新添加相同 provider。

**路径 A：新建本机配置（默认）。** 打开服务状态打印的 Web 登录链接，在 Providers 页面添加已有 API key、内部 Relay 和订阅。现成 preset 可直接添加，例如已有 DeepSeek key 时，在 Bash 中静默输入后添加（受保护的凭据注入可跳过输入部分）：

```bash
if [ -z "${DEEPSEEK_API_KEY:-}" ]; then
  printf 'DeepSeek API key: ' >&2
  IFS= read -r -s DEEPSEEK_API_KEY
  printf '\n' >&2
fi
: "${DEEPSEEK_API_KEY:?需要有效的 DeepSeek API key}"
magpie provider add deepseek "$DEEPSEEK_API_KEY"
unset DEEPSEEK_API_KEY
```

内部 Relay 要向管理员确认基础地址、协议（OpenAI Chat Completions / Responses / Anthropic Messages）、认证头和模型发现地址，不能只凭模型名猜测协议。已确认是 OpenAI 兼容接口时可参考以下模板，其他协议通过 Web 表单或 `magpie provider` 的当前帮助配置：

```bash
: "${RELAY_BASE_URL:?提供包含实际 API 前缀的 Relay 地址}"
: "${RELAY_API_KEY:?提供 Relay API key}"
magpie provider add relay "url=$RELAY_BASE_URL" "key=$RELAY_API_KEY"
unset RELAY_API_KEY
```

变量不要通过含明文密钥的 `export ...` 命令写入历史。可以用受保护的凭据注入；人工终端用静默输入。Magpie CLI 添加命令的参数会短暂出现在本机进程参数中，共享设备优先在 Web 页面输入凭据。重复配置时用 `magpie provider set <现有 id> ...`，`add` 会创建另一个 provider。

Factory 订阅使用 [Factory 社区认证插件](https://github.com/magpie-community/plugins/tree/main/packages/factory)，由用户完成登录：

```bash
magpie plugin add @magpie-community/opencode-factory-auth
magpie plugin login factory
```

ChatGPT 或 Claude 订阅可在 Magpie 中添加，也可分别完成 `codex login`、`claude` 的官方登录后让 Magpie 发现。只配置用户实际拥有的来源，不把某个 API key 当作订阅登录信息。

**路径 B：恢复用户提供的 Magpie 备份。** 在旧设备用 Magpie 的 Backup 导出，按需包含订阅登录；通过安全渠道把加密备份和口令交给新设备。在新设备上保留本机 agent 文件，再重新选择模型：

```bash
: "${MAGPIE_BACKUP:?提供本机已安全接收的 Magpie 备份路径}"
magpie restore --no-agents --no-library "$MAGPIE_BACKUP"
magpie daemon restart
```

由用户输入备份口令；恢复后检查插件是否就绪及订阅是否需要重新登录。不整目录覆盖旧设备的 HOME、`.codex` 或 `.claude`，避免带入机器路径与过期会话。

**路径 C：复用已有 Magpie 网关。** 新设备仍可运行本机 Magpie，把远端当作一个 provider，不必复制远端的订阅凭据。若远端网关只监听回环地址，可在独立终端建立 SSH 转发：

```bash
: "${MAGPIE_SSH_TARGET:?提供 user@host}"
ssh -N -o ExitOnForwardFailure=yes \
  -L 127.0.0.1:13425:127.0.0.1:3425 "$MAGPIE_SSH_TARGET"
```

保持该 SSH 进程运行，在另一个已加载 nvm / Magpie 的终端中配置 provider。认证使用管理员提供的**网关调用凭据**，下面假设远端使用 Bearer key：

```bash
: "${MAGPIE_GATEWAY_KEY:?通过安全渠道提供远端网关调用凭据}"
magpie provider add remote-magpie \
  url=http://127.0.0.1:13425/v1 "key=$MAGPIE_GATEWAY_KEY"
unset MAGPIE_GATEWAY_KEY
```

远端使用其他认证方式时按管理员要求调整。后续模型 ID 以本机 `magpie models` 显示的远端 provider 命名空间为准。SSH 转发关闭后远端模型不可用，需要由登录会话或进程管理器保持连接。

需要内网 Web 访问时，由服务端执行 `magpie daemon web on --lan`，使用状态输出中的实际内网地址与登录 key。不同网卡绑定、端口和访问策略以该服务配置为准；新设备的 `127.0.0.1` 始终指向新设备自己。本文不固化某台设备的 IP、账号或登录链接。

### 7.3 为两个 CLI 选择模型

先查看当前可用模型，再设置用户选定的默认值；不要硬编码某个设备才存在的模型：

```bash
magpie models codex
magpie models claude
```

将实际选中的完整 ID 放入以下变量，格式通常是 `provider/model`，模型部分也可能包含 `/`：

```bash
: "${CODEX_MODEL_ID:?从 magpie models codex 中选择完整模型 ID}"
: "${CLAUDE_MODEL_ID:?从 magpie models claude 中选择完整模型 ID}"
magpie codex "$CODEX_MODEL_ID"
magpie claude "$CLAUDE_MODEL_ID"
magpie codex
magpie claude
```

让 Magpie 写入当前版本所需的 agent 配置，不手工拼接它管理的 Codex / Claude 认证字段，也不覆盖原有 MCP、skills、其他 provider 配置。检查启动环境中旧的 `OPENAI_BASE_URL`、`ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN` 等是否与新配置冲突，只清理已确认属于旧接入的设置。已启动的 agent 需重新打开才能读取更新。

如需模型名显示 provider 在前，可为选定模型设置完整名称，再让这两个 agent 保留自定义名称：

```bash
magpie model name "$CODEX_MODEL_ID" 'provider · 模型显示名'
magpie model suffix codex own
magpie model suffix claude own
```

将示例显示名改为该模型的实际 provider 与名称；Claude 使用另一个模型时也为它设置名称。Magpie 默认 provider 后缀为“模型名 · Provider”；`own` 让显式命名的模型按原样显示，新增且未命名的模型仍使用默认规则。显示名不会改变实际模型 ID。

## 8. 验收与交接

重新打开一个登录终端，确认版本和 PATH，而不只检查当前安装进程：

```bash
zsh -lic 'git --version; python3 --version; node --version; npm --version; codex --version; claude --version; magpie --version'
venv_check_dir="$(mktemp -d)"
python3 -m venv "$venv_check_dir"
"$venv_check_dir/bin/python" -m pip --version
magpie daemon status
curl --noproxy '*' -fsS http://127.0.0.1:3425/
```

Python 验证后只删除本次创建的 `$venv_check_dir` 目录。Linux 用 `sudo docker info`，macOS 用 `docker info`，确认有 Server 信息；`docker compose version` 与 `hello-world` 也应成功。

最后在新的空目录里用两个 CLI 分别发起一次最小模型请求，确认**实际生成回复**：

```bash
smoke_dir="$(mktemp -d)"
codex exec --skip-git-repo-check -C "$smoke_dir" '只回复 OK，不调用工具，也不修改文件。'
(cd "$smoke_dir" && claude -p '只回复 OK，不调用工具，也不修改文件。' --max-turns 1)
```

在 Magpie 中确认两次请求走了预期 provider / 模型。连接失败、登录缺失、配额不足或工具调用协议不兼容都应报告为待解决问题，不能仅凭 `/v1/models` 有返回就判定完成。

agent 最终报告只包含：系统与架构、安装版本与程序路径、配置变更和备份位置、Docker 是否就绪、Magpie 运行方式、两个默认模型 ID、两次调用结果、仍需用户处理的事项。报告不包含 key、token、备份口令或带 key 的 Web URL。

## 9. 更新与回退

同一 nvm Node 版本下，更新两个 CLI：

```bash
npm install -g @openai/codex@latest @anthropic-ai/claude-code@latest
magpie update
```

更新后重做第 8 节的关键验收。需要固定版本时用 `包名@明确版本`，先记录上一个可用版本；不默认开启 agent 的全部权限，也不顺带安装项目尚未需要的语言或工具。

撤销某个 agent 的 Magpie 接入：

```bash
magpie codex default
magpie claude default
```

先验证 agent 已恢复原有配置，再决定是否停用 `magpie daemon stop`。需要从备份恢复时只恢复本次修改的配置字段，不用旧备份覆盖后来新增的设置，不删除 Docker volume、SSH 密钥或用户项目。

安装命令于 2026-11-13 核对；在新设备执行时以各软件当前官方文档、实际帮助和系统支持范围为准。
