+++
title = 'Oh My Zsh自动补全'
date = 2019-01-22T11:04:04+08:00
draft = false
categories = [
    "Linux",
    "脚本语言"
]
+++

shell 有多种，大多数人接触比较多的是 bash， 不管是 mac 还是各个 linux 发行版，默认的 shell 基本都是 bash，
虽然 bash 功能已经丰富了，但对于极客们来说，界面不够炫，提示功能也不够强大。而 zsh 功能及其强大，只是配置过于复杂，
后来就有了 oh-my-zsh 开源项目，配置难度大大降低。

Github地址: https://github.com/robbyrussell/oh-my-zsh

### 安装

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

### 更改系统默认 shell
```shell
chsh -s /bin/zsh
```

### 更改zsh配置文件

```shell
vim ~/.zshrc
```

### 修改主题

```shell
ZSH_THEME="robbyrussell"
```

### 配置插件

oh-my-zsh 还支持插件，插件存放目录为：
```shell
~/.oh-my-zsh/plugins
```

这个目录中每个子目录都是一个插件，目录名即为插件名，默认不开启，需要在`~/.zshrc`中该配置开启，比如:
```toml
plugins=(
git
git-flow
docker
kubectl
brew
npm
helm
zsh-autosuggestions
zsh-syntax-highlighting
)
```

这些插件可以给你常用的命令做用法提示，使用 tab 键触发。我这里再推荐另外三个不是内置的插件，
需要将它们单独下载到`~/.oh-my-zsh/plugins`并且加到上面的 plugins 配置列表中以启用插件：

| 插件  | 功能  | 地址  |
|---|---|---|
| zsh-autosuggestions  | 自动提示输入提示  |  https://github.com/zsh-users/zsh-autosuggestions |
|  zsh-syntax-highlighting |  高亮命令输入 |  https://github.com/zsh-users/zsh-syntax-highlighting |
|  zsh-history-substring-search |  查找匹配前缀的历史输入 |  https://github.com/zsh-users/zsh-history-substring-search |
