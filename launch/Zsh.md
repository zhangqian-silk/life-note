# Zsh 环境配置（Linux / macOS）

> 目标：在 Linux / macOS 上完成 Zsh、Oh My Zsh、Powerlevel10k、常用插件、fzf、zoxide、Nerd Font，以及 nvm、Node.js LTS、npm 和 pnpm 的完整配置。

按 [新设备开发环境配置](./Environment.md) 执行时，先完成其第 1–3 节，再执行本篇第 1–13 节。验收完成后，返回该文档第 5 节继续安装 Codex 和 Claude Code。

已有 nvm、Node.js、Shell 配置块时直接复用；示例中的完整 `.zshrc` 只作为合并参考，不覆盖基础配置、代理、凭据入口或其他工具的初始化。Git clone 目标目录已存在时先检查其远端和状态，不重复 clone。

## 最终目录结构

```text
~
├── .oh-my-zsh/
│   └── custom/
│       ├── themes/powerlevel10k/
│       └── plugins/
│           ├── zsh-autosuggestions/
│           ├── zsh-syntax-highlighting/
│           └── zsh-completions/
├── .local/
│   ├── bin/
│   │   ├── fzf -> ~/.local/opt/fzf/bin/fzf
│   │   └── zoxide
│   ├── opt/fzf/
│   └── share/zsh/fzf/
├── .nvm/
│   └── current -> versions/node/vxx
├── .zshrc
└── .p10k.zsh
```

## 1. 安装基础依赖

复用新设备配置流程中已安装的基础工具，补齐缺少的依赖。

Debian / Ubuntu：

```shell
sudo apt update
sudo apt install -y git curl wget zsh
```

Fedora / RHEL：

```shell
sudo dnf install -y git curl wget zsh
```

Arch Linux：

```shell
sudo pacman -S --noconfirm git curl wget zsh
```

macOS 使用系统自带的 `/bin/zsh`。完成 Homebrew 安装后补充本篇工具依赖：

```shell
brew install git wget python
```

确认 zsh：

```shell
zsh --version
command -v zsh
```

## 2. 设置默认 Shell

当前用户：

```shell
chsh -s "$(command -v zsh)"
```

指定用户：

```shell
sudo chsh -s "$(command -v zsh)" username
```

Linux 也可使用：

```shell
sudo usermod --shell "$(command -v zsh)" username
```

确认结果：

```shell
if command -v getent >/dev/null 2>&1; then
  getent passwd "$USER"
elif [[ "$(uname -s)" == Darwin ]]; then
  dscl . -read "/Users/$(id -un)" UserShell
fi
```

默认 shell 通常需要重新登录后生效。

## 3. 安装 oh-my-zsh

使用 unattended 模式安装，避免安装脚本自动进入 zsh：

```shell
RUNZSH=no CHSH=no KEEP_ZSHRC=yes \
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended
```

安装目录：

```text
~/.oh-my-zsh
```

## 4. 安装 Powerlevel10k

```shell
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"
```

主题名称：

```shell
powerlevel10k/powerlevel10k
```

## 5. 安装 zsh 插件

```shell
CUSTOM="${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/plugins"
mkdir -p "$CUSTOM"

git clone https://github.com/zsh-users/zsh-autosuggestions "$CUSTOM/zsh-autosuggestions"
git clone https://github.com/zsh-users/zsh-syntax-highlighting "$CUSTOM/zsh-syntax-highlighting"
git clone https://github.com/zsh-users/zsh-completions "$CUSTOM/zsh-completions"
```

插件列表后续写入 `.zshrc`：

```shell
plugins=(
  git
  zsh-completions
  zsh-autosuggestions
  web-search
  jsontools
  copypath
  zsh-syntax-highlighting
)
```

## 6. 安装 fzf

将 fzf upstream 仓库放在 `~/.local/opt/fzf`，命令入口放在 `~/.local/bin/fzf`：

```shell
mkdir -p "$HOME/.local/opt" "$HOME/.local/bin" "$HOME/.local/share/zsh/fzf"

git clone --depth 1 https://github.com/junegunn/fzf.git "$HOME/.local/opt/fzf"
ln -sfn "$HOME/.local/opt/fzf/bin/fzf" "$HOME/.local/bin/fzf"
```

复制 zsh 加载脚本到本地维护目录：

```shell
cp "$HOME/.local/opt/fzf/shell/completion.zsh" "$HOME/.local/share/zsh/fzf/completion.zsh"
cp "$HOME/.local/opt/fzf/shell/key-bindings.zsh" "$HOME/.local/share/zsh/fzf/key-bindings.zsh"
```

使用 fzf 仓库提供的安装器下载当前平台的可执行文件，不让安装器重写 Shell 配置：

```shell
"$HOME/.local/opt/fzf/install" --bin
```

若旧环境里的 fzf Shell 脚本需要兼容修复，先核对当前 upstream 版本；不要把 Linux 的 `sed -i` 参数直接用于 macOS。

确认：

```shell
"$HOME/.local/bin/fzf" --version
```

## 7. 安装 zoxide

```shell
curl -sSfL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh
```

确认：

```shell
"$HOME/.local/bin/zoxide" --version
```

## 8. 安装 nvm、Node.js、npm 和 pnpm

按 [nvm 官方安装说明](https://github.com/nvm-sh/nvm#installing-and-updating) 在当前 Shell 加载 nvm，并安装 Node.js LTS。`NVM_SYMLINK_CURRENT` 让后面的 Zsh PATH 始终指向当前使用的 Node 版本：

```bash
export NVM_DIR="$HOME/.nvm"
export NVM_SYMLINK_CURRENT=true
if [ ! -s "$NVM_DIR/nvm.sh" ]; then
  curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.41.0/install.sh \
    | PROFILE=/dev/null bash
fi
. "$NVM_DIR/nvm.sh"
nvm install --lts
nvm alias default 'lts/*'
nvm use default
npm install -g pnpm
```

npm 随 Node 一起安装，全局包放在 nvm 管理的用户目录中。后续通过 npm 安装 Codex 和 Claude Code 时，继续使用这套 Node/npm。

```bash
ls -ld "$NVM_DIR/current"
node --version
npm --version
pnpm --version
npm config get prefix
```

## 9. 迁移已有 Shell 配置

写入新的 `.zshrc` 之前，必须先检查原有 shell 配置，把本机已有的环境变量、代理、别名、函数和路径配置迁移过去。

先归档当前配置。下面的归档块在 Bash 中执行，避免 Zsh 对空 glob 报错；归档可能包含私有环境变量，只保存在本机，不粘贴到报告或提交到仓库：

```shell
umask 077
mkdir -p "$HOME/.config/shell-migration"
chmod 700 "$HOME/.config/shell-migration"

for file in \
  "$HOME/.bashrc" \
  "$HOME/.bash_profile" \
  "$HOME/.profile" \
  "$HOME/.zshrc" \
  "$HOME/.zprofile" \
  "$HOME/.zshenv" \
  /etc/profile \
  /etc/bashrc \
  /etc/profile.d/*.sh
do
  [ -r "$file" ] || continue
  {
    echo
    echo "===== $file ====="
    sed -n '1,240p' "$file"
  } >> "$HOME/.config/shell-migration/before-zsh.txt"
done
```

重点检查这些内容：

```shell
grep -nEi \
  'http_proxy|https_proxy|all_proxy|no_proxy|proxy|export PATH|PATH=|alias |function |NVM_DIR|PNPM_HOME|CONDA|JAVA_HOME|GOPATH|GOROOT|PYENV|RUSTUP|EDITOR|LANG|LC_|SSH_AUTH_SOCK|GPG_TTY' \
  "$HOME/.config/shell-migration/before-zsh.txt"
```

迁移规则：

- `http_proxy`、`https_proxy`、`all_proxy`、`no_proxy` 等网络代理，放到 `.zshrc` 的 `Local environment` 区域。
- 原有 `PATH` 追加项，合并进 zsh 的 `path=(...)` 数组，避免重复拼接字符串。
- 原有 `alias` 和函数，放到 oh-my-zsh 加载之后。
- 原有语言环境变量，如 `JAVA_HOME`、`GOPATH`、`GOROOT`、`RUSTUP_HOME`、`PYENV_ROOT`，按原值迁移。
- 原有 `EDITOR`、`LANG`、`LC_*`、`SSH_AUTH_SOCK`、`GPG_TTY` 等通用变量，确认仍需要后迁移。
- nvm、pnpm、conda 等工具如果已经在新 `.zshrc` 中统一配置，不要重复 source。

示例：如果原来的 bash 中有代理配置：

```shell
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1,.example.com,10.0.0.0/8"
```

新的 `.zshrc` 里也要保留到 `Local environment` 区域。

## 10. 写入 `.zshrc`

写入前先备份：

```shell
cp -a "$HOME/.zshrc" "$HOME/.zshrc.bak.$(date +%Y%m%d%H%M%S)" 2>/dev/null || true
```

写入 `~/.zshrc`：

```shell
# Powerlevel10k instant prompt is disabled for stable rendering in remote terminals.

# ----- Homebrew (macOS) -----
if [[ -x /opt/homebrew/bin/brew ]]; then
  eval "$(/opt/homebrew/bin/brew shellenv)"
elif [[ -x /usr/local/bin/brew ]]; then
  eval "$(/usr/local/bin/brew shellenv)"
fi

# ----- Paths -----
export CONDA_AUTO_ACTIVATE_BASE=false
export PNPM_HOME="$HOME/.local/share/pnpm"
export NVM_DIR="$HOME/.nvm"
export NVM_SYMLINK_CURRENT=true

path=(
  "$HOME/.local/bin"
  "$PNPM_HOME"
  "$NVM_DIR/current/bin"
  "/usr/local/bin"
  "$HOME/bin"
  $path
)
typeset -U path
export PATH

# ----- Oh My Zsh -----
export ZSH="$HOME/.oh-my-zsh"
if [[ -o interactive && ( -t 0 || -t 1 ) ]]; then
  ZSH_THEME="powerlevel10k/powerlevel10k"
else
  ZSH_THEME=""
fi
ZSH_DISABLE_COMPFIX="true"

if [[ -d "$ZSH/custom/plugins/zsh-completions/src" ]]; then
  fpath=("$ZSH/custom/plugins/zsh-completions/src" $fpath)
fi

zstyle ':completion:*' menu select
zstyle ':completion:*' group-name ''
zstyle ':completion:*' verbose yes
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'
zstyle ':completion:*:cd:*' tag-order local-directories path-directories
zstyle ':completion:*' select-prompt '%SScrolling active: current selection at %p%s'
zstyle ':completion:*' list-prompt '%SAt %p: hit TAB for more, or the character to insert%s'

setopt AUTO_LIST
setopt AUTO_MENU
setopt LIST_TYPES
setopt COMPLETE_IN_WORD
setopt AUTO_PARAM_SLASH

plugins=(
  git
  zsh-completions
  zsh-autosuggestions
  web-search
  jsontools
  copypath
  zsh-syntax-highlighting
)

source "$ZSH/oh-my-zsh.sh"
bindkey '^I' expand-or-complete

# ----- fzf -----
export FZF_BASE="$HOME/.local/opt/fzf"
if [[ -d "$FZF_BASE" ]]; then
  path=("$FZF_BASE/bin" $path)
  typeset -U path
  [[ -o interactive && -t 0 && -t 1 && -r "$HOME/.local/share/zsh/fzf/completion.zsh" ]] && source "$HOME/.local/share/zsh/fzf/completion.zsh"
  [[ -o interactive && -t 0 && -t 1 && -r "$HOME/.local/share/zsh/fzf/key-bindings.zsh" ]] && source "$HOME/.local/share/zsh/fzf/key-bindings.zsh"
fi

# ----- nvm -----
nvm() {
  unset -f nvm
  [[ -s "$NVM_DIR/nvm.sh" ]] && source "$NVM_DIR/nvm.sh"
  nvm "$@"
}

# ----- Powerlevel10k -----
[[ -o interactive && ( -t 0 || -t 1 ) && -r "$HOME/.p10k.zsh" ]] && source "$HOME/.p10k.zsh"

# ----- Local environment migrated from previous shell -----
[[ -r "$HOME/.config/life-note/proxy.env" ]] && source "$HOME/.config/life-note/proxy.env"
# Put host-specific settings from ~/.bashrc, ~/.bash_profile, ~/.profile and
# /etc/profile.d/*.sh here. Keep values consistent with the previous shell.
#
# Proxy example:
# export http_proxy="http://127.0.0.1:7890"
# export https_proxy="http://127.0.0.1:7890"
# export no_proxy="localhost,127.0.0.1,.example.com,10.0.0.0/8"

# Keep zoxide last so its shell hook stays registered after all framework setup.
command -v zoxide &>/dev/null && eval "$(zoxide init zsh)"
```

交互终端读取 `.zshrc`。非交互脚本、systemd / launchd 服务需要显式加载 `nvm.sh` 或使用实际的可执行文件路径；nvm 的加载代码统一维护在本篇的 Shell 配置中。

## 11. 写入 `.p10k.zsh`

写入前先备份：

```shell
cp -a "$HOME/.p10k.zsh" "$HOME/.p10k.zsh.bak.$(date +%Y%m%d%H%M%S)" 2>/dev/null || true
```

以下命令从本仓库根目录运行，备份后写入配套的 Powerlevel10k 配置：

```shell
cat launch/config/p10k.zsh.part-* > "$HOME/.p10k.zsh"
chmod 0644 "$HOME/.p10k.zsh"
```

这份配置拆分保存在 `launch/config/p10k.zsh.part-*`，按文件名顺序拼接为 `~/.p10k.zsh`。配置基于 Powerlevel10k `rainbow` 模板，保留 `dir`、`vcs`、运行环境、云环境、上下文和时间等模块，并覆盖为「海上生明月」配色：

```text
mainColor    #4D5CCB
bgColor      #181F34
surfaceColor #101523
linkColor    #9ABBFE
accentColor  #E4D29A
onMainColor  #FFFFFF
textColor    #A8B3D7
```

Git 状态配色：

```text
clean      #9ABBFE
modified   #E4D29A
untracked  #7F91E8
conflicted #B66A86
```

`root@host` 使用 `#101523` 背景和 `#E4D29A` 文字，避免默认 root 红色模块过于显眼。

## 12. 安装 Nerd Font

Powerlevel10k 配套使用 `MesloLGS NF`。字体安装在显示终端的设备上；通过 SSH 使用服务器时，在发起 SSH 的本机终端设置字体。

macOS：

```shell
brew install --cask font-meslo-lg-nerd-font
```

Linux 桌面：

```shell
mkdir -p ~/.local/share/fonts
cd ~/.local/share/fonts

curl -fLo "MesloLGS_NF_Regular.ttf" \
  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Regular.ttf
curl -fLo "MesloLGS_NF_Bold.ttf" \
  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold.ttf
curl -fLo "MesloLGS_NF_Italic.ttf" \
  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Italic.ttf
curl -fLo "MesloLGS_NF_Bold_Italic.ttf" \
  https://github.com/romkatv/powerlevel10k-media/raw/master/MesloLGS%20NF%20Bold%20Italic.ttf

fc-cache -fv
```

VS Code 终端字体：

```json
{
  "terminal.integrated.fontFamily": "MesloLGS NF"
}
```

## 13. 验证

先检查配置语法、配套工具和 Node 环境：

```shell
zsh -n "$HOME/.zshrc"
zsh -n "$HOME/.p10k.zsh"

zsh -ic 'echo ok'
zsh -ic 'command -v fzf; fzf --version'
zsh -ic 'command -v zoxide; zoxide --version'
zsh -ic 'command -v node; node -v; npm -v; pnpm -v'
test -r "$HOME/.p10k.zsh"
```

重新进入 zsh：

```shell
exec zsh
```

在实际交互终端中确认 Powerlevel10k 与插件已加载：

```zsh
print -r -- "$ZSH_THEME"
whence -w p10k
print -rl -- $plugins
```

前两条命令的期望输出：

```text
powerlevel10k/powerlevel10k
p10k: function
```

插件列表应包含第 5 节配置的插件。在终端中确认提示符配色、Git 状态和字体图标显示正常，自动建议、Tab 补全、语法高亮及 fzf 历史搜索可用，`z` 命令可跳转已访问的目录。

## 14. 更新

oh-my-zsh：

```shell
omz update
```

Powerlevel10k：

```shell
git -C "$HOME/.oh-my-zsh/custom/themes/powerlevel10k" pull --ff-only
```

插件：

```shell
for dir in \
  "$HOME/.oh-my-zsh/custom/plugins/zsh-autosuggestions" \
  "$HOME/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting" \
  "$HOME/.oh-my-zsh/custom/plugins/zsh-completions"
do
  git -C "$dir" pull --ff-only
done
```

fzf：

```shell
git -C "$HOME/.local/opt/fzf" pull --ff-only
"$HOME/.local/opt/fzf/install" --bin
cp "$HOME/.local/opt/fzf/shell/completion.zsh" "$HOME/.local/share/zsh/fzf/completion.zsh"
cp "$HOME/.local/opt/fzf/shell/key-bindings.zsh" "$HOME/.local/share/zsh/fzf/key-bindings.zsh"
```

nvm、Node.js、pnpm：

```shell
nvm install --lts
nvm alias default 'lts/*'
export NVM_SYMLINK_CURRENT=true
nvm use default
npm install -g pnpm
```
