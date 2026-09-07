---
title: Ubuntu自定义欢迎消息
toc: true
categories:
  - CS
tags: [Linux, 服务器]
date: 2025-11-01 19:11:00
updated: 2025-11-01 19:11:00
---

本文将介绍如何自定义 Ubuntu 系统远程 SSH 登录时显示的欢迎消息。

## 动态消息

默认情况下，欢迎消息可能形如：

```txt
Welcome to Ubuntu xx.xx LTS (GNU/Linux xxxxxx)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of xxxxxx

  System load:  x.x                Processes:             xxx
  Usage of /:   xx.x% of xx.xxGB   Users logged in:       x
  Memory usage: xx%                IPv4 address for eth0: xx.xx.xx.xx

...

0 updates can be applied immediately.

Last login: xxxxxx from xx.xx.xx.xx
```

这些消息大部分是是动态生成的，即每一次显示的消息都可能是不一样的。

它们是 `/etc/update-motd.d/` 目录下的多个脚本文件按照一定的顺序运行后生成的。

<!-- more -->

运行以下命令：

```bash
cd /etc/update-motd.d/
ls -l
```

可见到类似如下的输出：

```txt
total ...
-rwxr-xr-x 1 root root ... 00-header
-rwxr-xr-x 1 root root ... 10-help-text
lrwxrwxrwx 1 root root ... 50-landscape-sysinfo -> /usr/share/landscape/landscape-sysinfo.wrapper
-rwxr-xr-x 1 root root ... 50-motd-news
-rwxr-xr-x 1 root root ... 85-fwupd
-rwxr-xr-x 1 root root ... 90-updates-available
-rwxr-xr-x 1 root root ... 91-contract-ua-esm-status
-rwxr-xr-x 1 root root ... 91-release-upgrade
-rwxr-xr-x 1 root root ... 92-unattended-upgrades
-rwxr-xr-x 1 root root ... 95-hwe-eol
-rwxr-xr-x 1 root root ... 97-overlayroot
-rwxr-xr-x 1 root root ... 98-fsck-at-reboot
-rwxr-xr-x 1 root root ... 98-reboot-required
```

这些脚本文件的名称顺序（升序）就表示它们的执行顺序。

要想禁用任一脚本文件，只需将其文件权限中的可执行权限去掉即可，例如：

```bash
sudo chmod -x 10-help-text
```

当然，我们也可以在该目录下创建自己的脚本文件，以实现个性化。

## 静态消息

如果想要修改静态消息，可以运行以下命令：

```bash
sudo nano /etc/motd
```

这个文件中的文本内容会直接显示在动态消息的下方。

## 备注

推荐一个可以生成大号 ASCII 艺术字的网站 [patorjk.com/software/taag](http://patorjk.com/software/taag)。把它生成的艺术字放在 `/etc/motd` 文件中，可以让欢迎消息更酷炫。

```txt

██████╗ ██████╗ ████████╗███████╗
██╔══██╗██╔══██╗╚══██╔══╝██╔════╝
██████╔╝██████╔╝   ██║   ███████╗
██╔═══╝ ██╔══██╗   ██║   ╚════██║
██║     ██║  ██║   ██║   ███████║
╚═╝     ╚═╝  ╚═╝   ╚═╝   ╚══════╝

Welcome to Primitive Rhodes Island Terminal Service !
```
