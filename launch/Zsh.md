# Zsh 环境配置

> 目标：在 Linux 服务器上配置一套稳定、可维护的 zsh 开发环境，包括 zsh、oh-my-zsh、Powerlevel10k、常用插件、fzf、zoxide、nvm、Node.js、pnpm。

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

Debian / Ubuntu：

```shell
sudo apt update
sudo apt install -y git curl wget zsh
```

CentOS / RHEL / OpenCloudOS：

```shell
sudo yum install -y git curl wget zsh
```

Arch Linux：

```shell
sudo pacman -S --noconfirm git curl wget zsh
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

或者：

```shell
sudo usermod --shell "$(command -v zsh)" username
```

确认结果：

```shell
getent passwd "$USER"
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

对本地副本做兼容处理：

```shell
if ! grep -q '__fzf_completion_options/ zle on/' "$HOME/.local/share/zsh/fzf/completion.zsh"; then
  sed -i '/eval \$__fzf_completion_options/i\
  __fzf_completion_options=${__fzf_completion_options/ zle on/}\
  __fzf_completion_options=${__fzf_completion_options/ zle off/}' \
    "$HOME/.local/share/zsh/fzf/completion.zsh"
fi

if ! grep -q '__fzf_key_bindings_options/ zle on/' "$HOME/.local/share/zsh/fzf/key-bindings.zsh"; then
  sed -i '/eval \$__fzf_key_bindings_options/i\
  __fzf_key_bindings_options=${__fzf_key_bindings_options/ zle on/}\
  __fzf_key_bindings_options=${__fzf_key_bindings_options/ zle off/}' \
    "$HOME/.local/share/zsh/fzf/key-bindings.zsh"
fi
```

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

## 8. 安装 nvm、Node.js、pnpm

安装 nvm：

```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
```

加载 nvm，安装 Node.js LTS：

```shell
export NVM_DIR="$HOME/.nvm"
export NVM_SYMLINK_CURRENT=true
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"

nvm install --lts
nvm alias default 'lts/*'
nvm use default
```

安装 pnpm：

```shell
npm install -g pnpm
```

确认当前 Node 软链：

```shell
readlink -f "$NVM_DIR/current"
node -v
npm -v
pnpm -v
```

## 9. 写入 `.zshrc`

写入前先备份：

```shell
cp -a "$HOME/.zshrc" "$HOME/.zshrc.bak.$(date +%Y%m%d%H%M%S)" 2>/dev/null || true
```

写入 `~/.zshrc`：

```shell
# Powerlevel10k instant prompt. Keep this before anything that can print output.
if [[ -r "${XDG_CACHE_HOME:-$HOME/.cache}/p10k-instant-prompt-${(%):-%n}.zsh" ]]; then
  source "${XDG_CACHE_HOME:-$HOME/.cache}/p10k-instant-prompt-${(%):-%n}.zsh"
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
if [[ -t 1 ]]; then
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
[[ -t 1 && -r "$HOME/.p10k.zsh" ]] && source "$HOME/.p10k.zsh"

# Keep zoxide last so its shell hook stays registered after all framework setup.
command -v zoxide &>/dev/null && eval "$(zoxide init zsh)"
```

## 10. 写入 `.p10k.zsh`

写入前先备份：

```shell
cp -a "$HOME/.p10k.zsh" "$HOME/.p10k.zsh.bak.$(date +%Y%m%d%H%M%S)" 2>/dev/null || true
```

从仓库写入固定配置：

```shell
install -m 0644 launch/config/p10k.zsh "$HOME/.p10k.zsh"
```

这份配置基于 Powerlevel10k `rainbow` 模板，保留 `dir`、`vcs`、运行环境、云环境、上下文和时间等模块，并覆盖为「海上生明月」配色：

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

## 11. 安装 Nerd Font

Powerlevel10k 需要本地终端使用 Nerd Font。推荐安装 `MesloLGS NF`。

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

## 12. 验证

```shell
zsh -n "$HOME/.zshrc"
zsh -n "$HOME/.p10k.zsh"

zsh -ic 'echo ok'
zsh -ic 'command -v fzf; fzf --version'
zsh -ic 'command -v zoxide; zoxide --version'
zsh -ic 'command -v node; node -v; npm -v; pnpm -v'
```

重新进入 zsh：

```shell
exec zsh
```

确认主题：

```shell
echo "$ZSH_THEME"
```

期望输出：

```text
powerlevel10k/powerlevel10k
```

## 13. 更新

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
