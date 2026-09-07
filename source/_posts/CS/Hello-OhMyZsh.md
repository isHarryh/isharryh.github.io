---
title: 开始上手OhMyZsh
toc: true
categories:
  - CS
tags: [Linux, 服务器]
date: 2025-07-11 22:28:00
updated: 2025-07-11 22:28:00
---

本文是作者在 Ubuntu 系统上配置 zsh 和 Oh My Zsh 的记录。

## 为什么需要

[zsh](https://www.zsh.org) 是一个功能强大的 Shell，提供了比 bash 更丰富的功能和更好的交互体验。

[Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh) 是一个开源的 zsh 配置框架，允许用户便捷地安装插件和主题，从而更加高效地使用 zsh。

## 安装步骤

### 安装 zsh

在 Ubuntu 中，使用以下命令安装：

```bash
sudo apt update
sudo apt install zsh -y
```

<!--more-->

对于其他操作系统，也有各自的安装方法，在此不一一列举。

> 查看当前系统中的所有可用的 Shell：
> 
> ```bash
> cat /etc/shells
> ```

安装完成后，我们一般需要进行 zsh 的初始配置。

**但是**由于我们采用了 Oh My Zsh 来管理 zsh 的配置，所以我们可以忽略这些繁琐的工作。

> 被忽略的工作：
> 
> 1. 使用 Change Shell 命令将 zsh 设置为默认 Shell：
>    ```bash
>    chsh -s /bin/zsh
>    ```
>    当然，也可以自动查找 zsh 的路径：
>    ```bash
>    chsh -s $(which zsh)
>    ```
> 2. 重新启动终端，或者使用 `exec zsh` 命令来激活 Zsh。
> 3. 如果是第一次使用（即用户目录下没有 `.zshrc` 文件），Zsh 会要求你进行初始配置。

### 安装 Oh My Zsh

首先确保安装了 Git，然后使用 curl 来安装 Oh My Zsh：

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

或者使用 wget 来安装：

```bash
sh -c "$(wget https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh -O -)"
```

跟随提示完成安装即可。过程中，脚本会提示将 zsh 设置为默认 Shell。

## 插件

Oh My Zsh 提供了丰富的插件，可以通过编辑 `~/.zshrc` 文件来启用或禁用插件：

```bash
nano ~/.zshrc
```

找到以 `plugins=` 开头的行，即可查看当前已启用的插件列表，多个插件名称之间需要使用空格或换行符分隔。

注意，修改了 `~/.zshrc` 文件后，需要重新加载配置文件才能使更改生效：

```bash
source ~/.zshrc
```

以下列举了一些常用的插件。

### Git 集成

`git` 是内置的 Git 集成插件，提供了许多 Git 命令的快捷方式。

### 快速跳转

`z` 是内置的快速跳转插件，可以通过输入部分目录名来快速跳转到对应目录。

使用示例：

```bash
cd /path/to/some/very/long/directory/fun
cd /path/to/other/directory/boo
z fun  # 此时会自动跳转到第一个目录
```

### 语法高亮

`zsh-syntax-highlighting` 插件提供了命令行语法高亮功能，可以帮助用户更好地识别命令的语法错误。

它不是内置插件，需要我们手动下载并安装：

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting 
```

然后在 `~/.zshrc` 中启用它即可。

### 自动补全

`zsh-autosuggestions` 插件提供了命令行自动补全功能，可以根据历史命令和当前输入的内容自动建议可能的命令。

它同样不是内置插件，需要手动下载并安装：

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

然后在 `~/.zshrc` 中启用它即可。

使用自动补全时，预测的内容会以灰色字体显示，按下方向右键“→”就可以接受当前的建议了。

## 备注

需要为所有用户配置相同的 zsh 配置？

不推荐使用符号链接来共享 root 用户的 `.zshrc` 文件，因为这可能会导致安全问题。

但是复制一份 `.zshrc` 文件到其他用户的 home 目录下是可以的。

以下命令可以将当前用户的配置文件迁移到 `other_user`：

```bash
U=other_user
cp ~/.zshrc /home/$U/
cp -r ~/.oh-my-zsh /home/$U/
chown -R $U /home/$U/.zshrc /home/$U/.oh-my-zsh
chsh -s $(which zsh) $U
```
