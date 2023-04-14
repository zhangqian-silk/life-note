# OpenCloudOS 配置方式

## 换源

### Docker 源

- 修改配置文件（`/etc/docker/daemon.json`）

```json
{
    "registry-mirrors": [
        "https://mirror.ccs.tencentyun.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://registry.docker-cn.com"
    ]
}
```

- 刷新配置

```shell
sudo systemctl daemon-reload
```

## git 配置

- 安装 git
  
    ```shell
    sudo yum -y install git
    ```

## shell 配置

### 安装 on-my-zsh

- 安装 zsh
  
    ```shell
    sudo yum -y install zsh
    ```

- 设置 zsh 为默认 shell
  
  - usermod
  
    ```shell
    sudo usermod --shell /bin/zsh [username]
    ```

  - chsh
  
    ```shell
    sudo chsh --shell /bin/zsh [username]
    ```

  - 手动修改 passwd 文件（`/etc/passwd`）

    ```yaml
    root:x:0:0:root:/root:/bin/zsh
    ```

- 安装 oh-my-zsh

    ```shell
    sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
    ```

### 主题设置

- 修改配置文件（`/root/.zshrc`）来配置主题（官方主题预览：<https://github.com/ohmyzsh/ohmyzsh/wiki/Themes>）

    ```yaml
    ZSH_THEME="robbyrussell"
    ```

- 使用 powerlevel10k 主题

    ```shell
    git clone --depth=1 https://gitee.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
    ```

    ```yaml
    ZSH_THEME="powerlevel10k/powerlevel10k"
    ```

- 下载字体文件（字体文件参考：<https://github.com/romkatv/powerlevel10k#fonts>）
- 将字体文件放置在对应目录中（`/usr/share/fonts/`）
- 刷新字体缓存，查看安装结果

    ```shell
    fc-cache
    fc-list
    ```

- 重新启动终端，按照 powerlevel10k 提示进行配置，最终会生成配置文件（`/root/.p10k.zsh`）
- 配置颜色时，可通过 `for code ({000..255}) print -P -- "$code: %F{$code}text color%f"` 获取所有颜色代码

### 插件设置

- 安装三方插件后，需要同时修改配置文件（`/root/.zshrc`）才能生效

  - 手动修改

    ```yaml
    plugins=(
        git
        otherPlugin1
        otherPlugin2
    )
    ```

  - 命令行修改

    ```shell
    sed -i 's/^plugins=(\(.*\)/plugins=(otherPlugin1 \1/' ~/.zshrc
    ```

- 历史信息补全插件
  
    ```shell
    git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
    ```

- 语法高亮插件
  
    ```shell
    git clone --depth=1 https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
    ```
