---
title: Ubuntu系统开荒记录
toc: true
categories:
  - CS
tags: [日志, 操作系统]
date: 2025-02-26 21:22:00
updated: 2025-02-26 21:22:00
---

本文是作者自用的 Ubuntu 操作系统（版本 24.02）服务器的一次开荒记录。

## 初始化

### 使用 SSH 接入系统

在本地计算机上使用安全终端（Secure Shell，SSH）连接到远程服务器。

```bash
ssh <用户名>@<服务器地址>
```

上述命令是使用用户名和普通密码进行身份认证的，运行命令后需要输入密码。事实上，采用此身份验证方式具有一定的暴力破解风险，因此推荐采用密钥对进行身份验证。

### 更新系统软件

更新所有系统包，确保拥有最新的安全补丁和功能。

```bash
sudo apt update && sudo apt upgrade -y
sudo apt dist-upgrade -y
```

同时，确保常用的工具已安装：

```bash
sudo apt install vim htop curl wget git -y
```

### 配置 UFW 防火墙

Ubuntu 自带一个简易防火墙（Uncomplicated Firewall，UFW），可用于设置防火墙规则。先激活 UFW 并允许必要的功能：

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

如需使用 HTTP/HTTPS 等服务，应该开放额外的端口，例如：

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### 定制化 SSH 功能

为了提高 SSH 的安全性和个性化程度，可以采取如下做法：

#### 更改默认的 SSH 端口

更改默认的 SSH 端口（22）为其他端口，可能有利于提高安全性。打开 SSH 配置文件：

```bash
sudo nano /etc/ssh/sshd_config
```

找到 `Port` 字段，删除该行的注释符号 `#`（如有），然后修改该字段的值为其他端口（例如改为 `Port 2222`）。保存文件后，重启 SSH 服务：

```bash
sudo systemctl restart ssh
```

接着，在 UFW 中也进行相应的端口权限修改：

```bash
sudo ufw allow 2222/tcp
sudo ufw deny 22/tcp
```

下一次使用 SSH 连接服务器时，需要记得更换连接的端口。

#### 采用密钥对身份验证

使用密钥对登录来代替普通密码登录，可以显著提高安全性。

首先，在本地生成密钥对。以 RSA 加密算法为例（当前可以更换为更安全的 ed25519 加密算法），在本地运行：

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

默认情况下，在 `~/.ssh/` 目录中会生成两个文件：`id_rsa` 和 `id_rsa.pub`。前者是私钥，后者是公钥。

然后，将公钥上传到服务器。在本地运行：

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub <用户名>@<服务器地址>
```

> 当 `ssh-copy-id` 不可用时，可以将公钥内容手动复制到服务器的 `~/.ssh/authorized_keys` 文件中。在服务器端运行以下命令：
> 
> ```bash
> mkdir -p ~/.ssh
> chmod 700 ~/.ssh
> echo "<公钥内容>" >> ~/.ssh/authorized_keys
> chmod 600 ~/.ssh/authorized_keys
> ```

打开 SSH 配置文件，修改以下参数：

```ini
PubkeyAuthentication yes
PasswordAuthentication no
```

最后，重启 SSH 服务。我们可以新开一个终端来测试密钥对登录：

```bash
ssh -i <密钥路径> <用户名>@<服务器地址>
```

#### 延长会话存续时间

如需防止 SSH 会话在闲置一段时间后被自动关闭，我们可以延长会话的存续时间。打开 SSH 配置文件，修改以下参数：

```ini
ClientAliveInterval 600
ClientAliveCountMax 3
```

重启 SSH 服务，会话存续时间就被设置为了 1800 秒

### 创建新用户

出于安全考虑，应该尽可能避免使用 root 用户进行常规操作。我们可以创建一个新的管理员用户：

```bash
sudo adduser newuser
sudo usermod -aG sudo newuser
```
