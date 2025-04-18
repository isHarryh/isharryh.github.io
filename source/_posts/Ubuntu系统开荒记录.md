---
title: Ubuntu系统开荒记录
toc: true
categories:
  - CS
tags: [Linux, 服务器]
date: 2025-02-26 21:22:00
updated: 2025-03-13 22:35:00
---

本文是作者自用的 Ubuntu 操作系统（版本 24.04）服务器的一次开荒记录。

## 初始化

### 使用 SSH 接入系统

在本地计算机上使用安全终端（Secure Shell，SSH）连接到远程服务器。命令行运行：

```bash
ssh <用户名>@<服务器地址>
```

上述命令是使用用户名和普通密码进行身份认证的，运行命令后需要输入密码。

> 事实上，采用此身份验证方式具有一定的暴力破解风险，因此推荐采用密钥对进行身份验证（后文会提到）。

<!-- more -->

### 更新系统软件

更新所有系统包，确保拥有最新的安全补丁和功能。

```bash
sudo apt update && sudo apt upgrade -y
sudo apt dist-upgrade -y
```

> `sudo` 指的是“superuser do”，即临时授权当前用户以其他用户的身份（默认 root）来执行命令。

> `apt` 指的是 Advanced Package Tool，用于自动化管理软件包。

### 安装常用工具

常用的工具类软件包括：

- 文本编辑器：
  - `vim`（经典预装）
  - `nano`（用户友好）
- 任务管理器：
  - `top`（经典预装）
  - `htop`（用户友好）
- 资源管理器：
  - `ls`（经典预装）
  - `eza`（用户友好）
- 网络工具：
  - `curl`（多用于处理网络请求）
  - `wget`（多用于批量下载文件）
- 版本控制系统：
  - `git`

运行以下命令即可安装软件：

```bash
sudo apt install <软件名> <软件名...> -y
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

> 如果开放防火墙后，网络流量仍无法流入服务器，此时需要考虑服务器提供商是否有额外的安全规则。

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

#### 延长会话存续时间

如需防止 SSH 会话在闲置一段时间后被自动关闭，我们可以延长会话的存续时间。打开 SSH 配置文件，修改以下参数：

```ini
ClientAliveInterval 60
ClientAliveCountMax 30
```

重启 SSH 服务，服务器所允许的客户端会话的存续时间就被设置为了 1800 秒（当然可以改为其他时间，但出于安全考虑，时间不易过长）。

### 配置命令别名

有时为了方便，会给某些命令设置别名（Alias），以便快捷调用。

#### 临时别名

临时别名只在当前终端会话中有效，终端关闭后会失效。可直接在命令行中使用 `alias` 设置临时别名（例如将 `gs` 设置为 `git status` 的一个别名）：

```bash
alias gs='git status'
```

#### 永久别名

永久别名会生效于所有终端会话中，直到手动删除。这就需要将别名添加到用户的配置文件中。对于 `bash` 用户，编辑 `.bashrc` 文件：

```bash
nano ~/.bashrc
```

> 如果是 `zsh` 用户，所有针对 `.bashrc` 文件的操作都替换成 `.zshrc` 的即可。

在配置文件中新增一行，添加你需要的别名，例如：

```bash
alias gs='git status'
```

保存并退出文件后，执行以下命令使配置生效：

```bash
source ~/.bashrc
```

> 可以直接使用不带参数的 `alias` 命令查看当前会话中已设置的所有别名。

## 用户管理

### 密钥对认证

使用 SSH 密钥对认证来进行无密码登录，可以显著提高安全性（防止暴力破解）。

#### 原理

密钥对认证基于非对称加密技术，其认证流程如下：

1. 客户端向服务器发起 SSH 连接；
2. 服务器生成一个随机字符串，发送给客户端；
3. 客户端用私钥加密这个字符串，并将密文发送回服务器；
4. 服务器尝试用已授权的公钥来解密此密文，如果解密结果与原始内容匹配，则认证成功。

一个公钥（锁）和它对应的私钥（钥匙）组成一个密钥对。服务器的一个用户可以拥有多个已授权的公钥，它们存储于 `~/.ssh/authorized_keys` 文件中（`~` 指的是用户的主目录，例如 root 是 `/root`，其他用户是 `/home/用户名`）。

#### 注册密钥对

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
> 
> 务必确保 `authorized_keys` 文件的所有者是当前用户（可使用 `chown` 命令修改所有者），且文件权限是 `600` 即 `-rw-------`。文件权限和所有者可以使用 `ls -l` 命令查看。

打开 SSH 配置文件，修改以下参数（开启密钥对登录，并禁止密码登录）：

```ini
PubkeyAuthentication yes
PasswordAuthentication no
```

最后，重启 SSH 服务。

#### 使用密钥对登录

我们可以在本地新开一个终端来测试密钥对登录：

```bash
ssh -i <私钥路径> <用户名>@<服务器地址>
```

> 若私钥文件在本地保存到了默认路径，例如 `~/.ssh/id_rsa`，则可以省略 `-i <私钥路径>` 参数。

### 创建用户

出于安全考虑，应该尽可能避免使用 root 用户进行常规操作。我们可以创建一个新的管理员用户：

```bash
sudo adduser newuser
sudo usermod -aG sudo newuser
```

为新用户重复一遍密钥对配置流程，即可使用密钥对登录。

要想查看当前登录的用户名，运行：

```bash
whoami
```

### 用户组操作

对用户权限的管理，主要是对用户组的管理。

创建用户组：

```bash
sudo groupadd 用户组名
```

将用户加入到用户组（不影响用户的主组）：

```bash
sudo usermod -aG 用户组名 用户名
```

修改用户的主组：

```bash
sudo usermod -g 用户组名 用户名
```

查看指定用户所属的用户组：

```bash
groups 用户名
```

锁定用户（禁止登录）：

```bash
sudo usermod -L 用户名
```

解锁用户：

```bash
sudo usermod -U 用户名
```

### 借取其他用户权限

在当前用户拥有 `sudo` 权限的情况下，可以用其他用户的身份来运行命令：

```bash
sudo -u 用户名 命令
```

> 未提供 `-u` 参数时，就默认是以 root 身份运行命令。

如果需要执行较多命令，可以直接登录到其他用户：

```bash
su - 用户名
```

> `-` 是 `-l` 即 `--login` 的缩写。

使用 `exit` 即可返回原先的用户。

## 应用程序

### 运行环境

以下列举了一些运行环境的安装步骤。

#### Java

安装指定版本（以 v17.x 为例）的 Java JDK：

```bash
sudo apt install -y openjdk-17-jdk
```

```bash
java --version
```

#### Node.js

安装指定版本（以 v22.x 为例）的 Node.js 运行时：

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

```bash
node --version
npm --version
```

#### Python

自行编译指定版本（以 v3.8.20 为例）的 Python 运行时：

```bash
sudo apt install -y build-essential libssl-dev zlib1g-dev libbz2-dev \
                    libreadline-dev libsqlite3-dev wget curl llvm \
                    libncurses5-dev libncursesw5-dev xz-utils tk-dev \
                    libffi-dev liblzma-dev python3-openssl git
```

```bash
mkdir ~/python38
cd ~/python38
wget https://www.python.org/ftp/python/3.8.20/Python-3.8.20.tgz
tar -xf Python-3.8.20.tgz
```

```bash
cd Python-3.8.20
./configure --enable-optimizations
make -j$(nproc)
sudo make install
```

```bash
python3.8 --version
```
