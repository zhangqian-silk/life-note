# 代理软件与订阅配置（Linux / macOS）

通过用户提供的订阅链接配置本机代理，供浏览器、终端、Magpie 和 Docker 使用。先完成 [新设备开发环境配置](./Environment.md) 第 2 节对应系统的基础工具安装，再执行本篇；完成后返回主文档第 3 节。

|设备|代理软件|订阅方式|
|---|---|---|
|macOS / Linux 桌面|Clash Verge Rev|导入远程订阅，启用配置和代理|
|无图形界面的 Linux 服务器|Mihomo|将订阅 URL 写入 `proxy-providers`，自动获取与更新节点|

执行顺序：先完成第 1 或第 2 节的客户端安装，再完成第 3 节的终端配置和第 4 节的代理请求、订阅节点检查，然后返回主文档第 3 节。Magpie 与 Docker 的服务代理设置，分别在主文档安装对应软件后执行；最后再做联合验收。

准备 `PROXY_SUBSCRIPTION_URL`，通过安全渠道提供或在本机静默输入。订阅 URL 通常包含账号 token，按凭据管理，不写入仓库或交接报告。已有可用客户端时，复用该客户端并更新订阅。

## 1. 桌面：Clash Verge Rev

### 安装

从 [Clash Verge Rev 官方发布页](https://github.com/clash-verge-rev/clash-verge-rev/releases/latest) 选择匹配系统与架构的安装包，安装方式见 [官方文档](https://www.clashverge.dev/install)。

macOS 使用 Homebrew：

```bash
brew install --cask clash-verge-rev
open -a 'Clash Verge'
```

该 cask 见 [Homebrew](https://formulae.brew.sh/cask/clash-verge-rev)。Linux 桌面下载实际的 `.deb` / `.rpm` 文件后，按当前发行版执行一条命令：

```bash
: "${CLASH_VERGE_DEB:?提供已下载的 deb 安装包路径}"
sudo apt-get install "$CLASH_VERGE_DEB"
```

```bash
: "${CLASH_VERGE_RPM:?提供已下载的 rpm 安装包路径}"
sudo dnf install "$CLASH_VERGE_RPM"
```

### 导入订阅并启用

按 [官方订阅导入说明](https://www.clashverge.dev/guide/profile.html) 配置：

1. 打开客户端的订阅页面，粘贴用户提供的订阅链接并导入。使用供应商提供的 Clash / Mihomo 配置订阅。
2. 等待配置下载成功，选中并启用它，确认代理组中出现节点。
3. 选择可用节点或订阅中的自动选择组，使用规则模式。
4. 在客户端设置中确认混合端口。本篇统一使用 `7890`；若采用其他端口，后面的 `SETUP_PROXY_URL` 同步调整。
5. 开启系统代理与开机启动。按第 3 节的终端小节显式配置代理，再完成第 4 节的代理请求检查。

客户端负责保存订阅与定时更新；更新链接时修改现有订阅。图形界面操作需要用户配合时，agent 继续完成终端和服务的配置。

## 2. Linux 服务器：Mihomo

### 安装官方二进制

从 [Mihomo 官方发布页](https://github.com/MetaCubeX/mihomo/releases/latest) 获取稳定版。本篇核对的版本为 `v1.19.34`；下面按当前设备架构下载，不需要安装 Go：

```bash
(
  set -e
  proxy_version=v1.19.34
  case "$(uname -m)" in
    x86_64) proxy_arch=amd64-v1 ;;
    aarch64|arm64) proxy_arch=arm64 ;;
    *) printf '%s\n' '请从官方发布页选择当前架构的 Mihomo 二进制'; exit 1 ;;
  esac
  proxy_download_dir="$(mktemp -d)"
  curl -fL --retry 3 \
    "https://github.com/MetaCubeX/mihomo/releases/download/$proxy_version/mihomo-linux-$proxy_arch-$proxy_version.gz" \
    -o "$proxy_download_dir/mihomo.gz"
  gzip -d "$proxy_download_dir/mihomo.gz"
  mkdir -p "$HOME/.local/bin"
  install -m 0755 "$proxy_download_dir/mihomo" "$HOME/.local/bin/mihomo"
)
"$HOME/.local/bin/mihomo" -v
```

### 根据订阅链接生成配置

Mihomo 的 [proxy-providers](https://wiki.metacubex.one/en/config/proxy-providers/) 可直接引用远程订阅，订阅内容支持 [YAML、URI 列表与 Base64](https://wiki.metacubex.one/en/config/proxy-providers/content/)。以下配置从订阅读取节点，每小时更新一次，通过自动选择组提供代理，回环和私有网段直连。

在 Bash 中执行下面的块。已有 `~/.config/mihomo/config.yaml` 时，在原配置中合并或更新订阅与代理组；此块只新建配置，不覆盖已有文件：

```bash
python3 - <<'PY'
import getpass
import json
import os
from pathlib import Path
import secrets
from urllib.parse import urlsplit

url = os.environ.get('PROXY_SUBSCRIPTION_URL') or getpass.getpass('代理订阅链接：')
parsed = urlsplit(url)
if parsed.scheme != 'https' or not parsed.netloc:
    raise SystemExit('需要有效的 HTTPS 订阅链接')

root = Path.home() / '.config' / 'mihomo'
root.mkdir(parents=True, exist_ok=True, mode=0o700)
config = {
    'mixed-port': 7890,
    'allow-lan': False,
    'bind-address': '127.0.0.1',
    'mode': 'rule',
    'log-level': 'warning',
    'external-controller': '127.0.0.1:9090',
    'secret': secrets.token_urlsafe(32),
    'proxy-providers': {
        'subscription': {
            'type': 'http',
            'url': url,
            'interval': 3600,
            'proxy': 'DIRECT',
            'health-check': {
                'enable': True,
                'url': 'https://www.gstatic.com/generate_204',
                'interval': 300,
            },
        },
    },
    'proxy-groups': [{
        'name': 'PROXY',
        'type': 'url-test',
        'use': ['subscription'],
        'url': 'https://www.gstatic.com/generate_204',
        'interval': 300,
    }],
    'rules': [
        'IP-CIDR,127.0.0.0/8,DIRECT,no-resolve',
        'IP-CIDR,10.0.0.0/8,DIRECT,no-resolve',
        'IP-CIDR,172.16.0.0/12,DIRECT,no-resolve',
        'IP-CIDR,192.168.0.0/16,DIRECT,no-resolve',
        'IP-CIDR6,::1/128,DIRECT,no-resolve',
        'IP-CIDR6,fc00::/7,DIRECT,no-resolve',
        'MATCH,PROXY',
    ],
}
# JSON is valid YAML and preserves special characters in the subscription URL.
path = root / 'config.yaml'
fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_EXCL, 0o600)
with os.fdopen(fd, 'w') as output:
    json.dump(config, output, ensure_ascii=False, indent=2)
    output.write('\n')
print('已生成 Mihomo 配置')
PY
unset PROXY_SUBSCRIPTION_URL
"$HOME/.local/bin/mihomo" -t -d "$HOME/.config/mihomo"
```

订阅用于维护节点，路由规则使用本机配置。将内部域名的 `DOMAIN-SUFFIX,...,DIRECT` 规则补在 `MATCH,PROXY` 之前，并同步加入第 3 节的直连列表。订阅需要特定 User-Agent 时，按供应商要求配置 provider 的 `header`。

首次下载订阅使用直连。如果安装包或订阅地址在当前网络不可达，先复用已有网络出口，或由用户提供离线安装包 / 节点配置，待代理可用后再更新订阅。

### 保持后台运行

按 [Mihomo 官方服务说明](https://wiki.metacubex.one/en/startup/service/) 使用 systemd；这里采用普通用户服务，只监听本机端口，不启用 TUN：

```bash
mkdir -p "$HOME/.config/systemd/user"
if [ ! -e "$HOME/.config/systemd/user/mihomo.service" ]; then
cat > "$HOME/.config/systemd/user/mihomo.service" <<'UNIT'
[Unit]
Description=Mihomo subscription proxy

[Service]
ExecStart=%h/.local/bin/mihomo -d %h/.config/mihomo
Restart=on-failure
RestartSec=5
UMask=0077

[Install]
WantedBy=default.target
UNIT
fi
systemctl --user daemon-reload
systemctl --user enable --now mihomo.service
systemctl --user is-active mihomo.service
```

已有同名服务时复用其配置。需要登出后继续运行的服务器，由管理员启用该用户的 linger。无 systemd 的环境可在独立终端执行 `~/.local/bin/mihomo -d ~/.config/mihomo`，由已有进程管理器维护。

修改订阅 URL 后，先重新执行 `mihomo -t`，通过后用 `systemctl --user restart mihomo.service` 应用。正常的节点更新由 provider 的 `interval` 完成。

## 3. 连接终端、Magpie 与 Docker

### 当前终端与后续 Zsh 会话

将实际本机代理地址设为 `SETUP_PROXY_URL`，例如 `http://127.0.0.1:7890`。混合端口同时支持 HTTP 与 SOCKS，本篇使用 HTTP 代理方式：

```bash
export SETUP_PROXY_URL=http://127.0.0.1:7890
export HTTP_PROXY="$SETUP_PROXY_URL"
export HTTPS_PROXY="$SETUP_PROXY_URL"
export http_proxy="$SETUP_PROXY_URL"
export https_proxy="$SETUP_PROXY_URL"
proxy_no_proxy="${NO_PROXY:-${no_proxy:-}}"
export NO_PROXY="localhost,127.0.0.1,::1${proxy_no_proxy:+,$proxy_no_proxy}"
export no_proxy="$NO_PROXY"
```

把实际内部域名与地址合并到 `NO_PROXY`。curl、npm、Git 等可使用这些变量；不同工具对网段写法的支持不同，内部服务同时登记实际域名 / IP。Magpie 的回环地址应始终直连。

需要通过代理执行联网的 sudo 命令时，显式传递代理变量。例如 Ubuntu / Debian 更新包索引：

```bash
sudo env HTTP_PROXY="$HTTP_PROXY" HTTPS_PROXY="$HTTPS_PROXY" \
  http_proxy="$http_proxy" https_proxy="$https_proxy" \
  NO_PROXY="$NO_PROXY" no_proxy="$no_proxy" apt-get update
```

下载 Docker 仓库密钥、安装系统包等命令同样处理；Fedora 换成相应的 `dnf` 命令。后台服务仍按下面的服务配置设置代理。

将这些代理变量保存到本机 `~/.config/life-note/proxy.env`，合并进已有文件；该文件只记录本机代理地址与直连列表，不保存订阅链接。后续 [Zsh 配置](./Zsh.md) 会加载它：

```bash
python3 - <<'PY_ENV'
import os
from pathlib import Path
import shlex

path = Path.home() / '.config/life-note/proxy.env'
path.parent.mkdir(parents=True, exist_ok=True, mode=0o700)
start = '# >>> life-note proxy >>>'
end = '# <<< life-note proxy <<<'
names = ['SETUP_PROXY_URL', 'HTTP_PROXY', 'HTTPS_PROXY', 'http_proxy',
         'https_proxy', 'NO_PROXY', 'no_proxy']
block = start + '\n' + '\n'.join(
    'export ' + name + '=' + shlex.quote(os.environ[name]) for name in names
) + '\n' + end + '\n'
old = path.read_text() if path.exists() else ''
if start in old:
    before, _, rest = old.partition(start)
    _, found, after = rest.partition(end)
    if not found:
        raise SystemExit('原代理配置块不完整，先修复后再更新')
    updated = before + block + after.lstrip('\n')
else:
    updated = old.rstrip('\n') + '\n' + block
with path.open('a'):
    pass
path.chmod(0o600)
path.write_text(updated)
PY_ENV
. "$HOME/.config/life-note/proxy.env"
```

### Magpie 后台服务

完成主文档中的 Magpie 安装后，为它显式配置本机代理，再重启服务：

```bash
: "${SETUP_PROXY_URL:?先加载本机代理地址}"
magpie settings set proxy "$SETUP_PROXY_URL"
magpie daemon restart
```

根据内部 Relay 的域名和网段配置代理客户端的直连规则。后台服务的代理设置不依赖当前终端是否打开。

### Linux Docker Engine

完成 Docker 安装后，将代理加入其 systemd 服务环境。已有同类配置时在原文件中合并；新设备可以新建：

```bash
: "${SETUP_PROXY_URL:?先加载本机代理地址}"
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf > /dev/null <<EOF_DOCKER_PROXY
[Service]
Environment="HTTP_PROXY=$SETUP_PROXY_URL"
Environment="HTTPS_PROXY=$SETUP_PROXY_URL"
Environment="NO_PROXY=$NO_PROXY"
EOF_DOCKER_PROXY
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo docker run --rm hello-world
```

这里配置的是 Docker Engine 拉取镜像的网络。容器内部需要联网时，按项目单独配置；容器中的 `127.0.0.1` 指向容器自身。参见 [Docker daemon proxy](https://docs.docker.com/engine/daemon/proxy/)。

### macOS Docker Desktop

在 Docker Desktop 的代理设置中选择系统代理，让它使用 Clash Verge Rev 已启用的系统代理，再执行 `docker run --rm hello-world`。代理位置与模式参见 [Docker Desktop 设置](https://docs.docker.com/desktop/settings-and-maintenance/settings/#proxies)，不套用 Linux 的 systemd 配置。

## 4. 验收与交接

先验证显式代理请求，再验证终端变量确实生效：

```bash
: "${SETUP_PROXY_URL:?先加载本机代理地址}"
curl --noproxy '' --proxy "$SETUP_PROXY_URL" --fail --max-time 30 \
  https://www.gstatic.com/generate_204 -o /dev/null -w '%{http_code}\n'
curl --noproxy '' --fail --max-time 30 https://registry.npmjs.org/ \
  -o /dev/null -w '%{http_code}\n'
```

第一条应返回 `204`；第二条应返回 `200`。再访问实际需要的 Git 服务与内部 Relay，确认公网通过代理、内网按直连列表访问。

Mihomo 还应检查订阅已加载有效节点。下面只输出节点数量，不输出订阅 URL 或控制接口密钥：

```bash
python3 - <<'PY'
import json
from pathlib import Path
import urllib.request

config = json.loads((Path.home() / '.config/mihomo/config.yaml').read_text())
request = urllib.request.Request(
    'http://127.0.0.1:9090/providers/proxies',
    headers={'Authorization': 'Bearer ' + config['secret']},
)
opener = urllib.request.build_opener(urllib.request.ProxyHandler({}))
with opener.open(request, timeout=10) as response:
    provider = json.load(response)['providers']['subscription']
count = len(provider['proxies'])
assert count > 0, '订阅没有可用节点'
print('订阅节点数量：', count)
PY
```

最后重新打开 Zsh，重复请求检查；完成 Magpie 和 Docker 配置后，分别验证实际模型调用与镜像拉取。交接报告记录客户端版本、订阅更新结果、运行方式、本机代理地址和验证结果，隐藏订阅 URL 与节点凭据。

## 5. 更新与停用

Clash Verge Rev 使用客户端的订阅更新和软件更新功能。Mihomo 更换官方二进制后重新执行配置检查并重启服务，保留现有订阅配置与缓存。

停用时，先关闭客户端的系统代理，或执行 `systemctl --user disable --now mihomo.service`，再移除本次添加的终端代理变量、Magpie 代理设置和 Docker 服务代理配置，恢复原值。
