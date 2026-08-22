---
title: Ubuntu创建分页文件
toc: true
categories:
  - CS
tags: [Linux, 服务器]
date: 2026-06-08 20:20:00
updated: 2026-06-08 20:20:00
---

本文将介绍如何在 Ubuntu 系统中创建[分页文件](https://wiki.archlinuxcn.org/wiki/Swap)。

## 为什么需要

当系统的物理内存不足时，如果没有分页文件作为临时缓冲，系统可能会触发 OOM Killer 来终止一些进程以释放内存，这可能导致服务中断等异常情况。

分页文件则可以作为物理内存耗尽前的缓冲，将当前内存中不活跃的数据“交换”到文件系统中，从而给活跃进程留出更多的内存空间。

<!-- more -->

## 如何使用分页文件

### 创建分页文件

在根目录创建一个名为 `swapfile` 的 1GB 分页文件：

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
```

随后使用 `swapon` 命令来挂载该分页文件：

```bash
sudo swapon /swapfile
```

最后验证是否生效：

```bash
swapon --show
free -h
```

### 开机自动挂载分页文件

要想让分页文件在系统启动时自动挂载，需要编辑 `/etc/fstab` 文件，添加一行 `/swapfile none swap sw 0 0`：

```bash
echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab
```

### 优化内核参数

可以通过调节 `vm.swappiness` 参数来控制系统使用分页文件的倾向。默认值为 60，可用范围是 0-100，值越大则越倾向于使用分页文件。

由于分页文件的效率显著低于物理内存，要想让系统减少对分页文件的依赖，可以将 `vm.swappiness` 设置为较低的值，例如 10：

```bash
sudo sysctl vm.swappiness=10
```

要想让这个参数配置永久生效，需要写入到 `/etc/sysctl.conf` 文件中：

```bash
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
```

### 删除或修改分页文件

对于一个正在被使用的分页文件，要想删除它或者修改它的大小，首先需要对其进行卸载：

```bash
sudo swapoff /swapfile
```

随后移除该文件：

```bash
sudo rm /swapfile
```

移除后，如需重新创建分页文件，可以按照前面介绍的步骤重新进行操作。

需要注意，如果更换了分页文件的路径，还需要更新 `/etc/fstab` 文件中的相关配置，以确保系统能够正确地自动挂载新的分页文件。
