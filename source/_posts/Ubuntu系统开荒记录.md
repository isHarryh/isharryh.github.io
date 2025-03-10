---
title: Ubuntu系统开荒记录
toc: true
categories:
  - CS
tags: [运维, 操作系统, 数据库]
date: 2025-02-26 21:22:00
updated: 2025-03-10 22:56:00
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

> `sudo` 是“superuser do”的缩写，可以临时授权当前用户以其他用户的身份（默认 root）来执行命令，是一种提权操作。

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

最后，重启 SSH 服务。我们可以在本地新开一个终端来测试密钥对登录：

```bash
ssh -i <私钥路径> <用户名>@<服务器地址>
```

> 若私钥文件在本地保存到了默认路径 `~/.ssh/id_rsa` ，则可以省略 `-i <私钥路径>` 参数。

#### 延长会话存续时间

如需防止 SSH 会话在闲置一段时间后被自动关闭，我们可以延长会话的存续时间。打开 SSH 配置文件，修改以下参数：

```ini
ClientAliveInterval 60
ClientAliveCountMax 30
```

重启 SSH 服务，服务器所允许的客户端会话的存续时间就被设置为了 1800 秒

### 创建新用户

出于安全考虑，应该尽可能避免使用 root 用户进行常规操作。我们可以创建一个新的管理员用户：

```bash
sudo adduser newuser
sudo usermod -aG sudo newuser
```

### 安装运行环境

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

## 常见应用场景

### 数据库

数据库主要分为关系型数据（SQL）和非关系型数据库（NoSQL）。主流的数据库列表格如下：

| 数据库 | 类型 | 适用场景 | 主要特点 |
|---|---|---|---|
| **MySQL** | SQL | 传统网站 | 最流行的 SQL 数据库 |
| **SQLite** | SQL | 轻量级应用 | 文件数据库 |
| **PostgreSQL** | SQL | 企业级应用 | 支持复杂查询和高扩展性 |
| **MongoDB** | NoSQL | 实时应用，API 驱动 | 灵活的 JSON-like 文档存储 |
| **Redis** | NoSQL | 实时应用，缓存 | 内存数据库 |

下面详细讲解 PostgreSQL 的部署步骤。

#### PostgreSQL 安装

[PostgreSQL](https://www.postgresql.org/) 是比 MySQL 更强大的关系型数据库。


安装 PostgreSQL：

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib -y
```

调整 PostgreSQL 的性能配置：

```bash
sudo nano /etc/postgresql/版本号/main/postgresql.conf
```

修改共享内存大小 `shared_buffers` 为合理的数值（建议设为可用内存的 25%），修改最大连接数 `max_connections` 为合理的数值。

确保 PostgreSQL 已启动且已设置开机自启动：

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql
```

#### PostgreSQL Hello World

以下内容演示了最基本的 PostgreSQL 命令行交互操作。

首先，切换到 PostgreSQL 提供的默认用户 `postgres`：

```bash
sudo -i -u postgres
```

然后，进入 PSQL 命令行模式：

```bash
psql
```

> 这会默认连接到内建数据库 `postgres`。在 PSQL 模式下，命令提示符形如 `xxx=#`，其中 `xxx` 是当前所在的数据库名称。

运行 Hello World 示例：

1. 在 PSQL 模式下，创建新数据库：
   ```sql
   CREATE DATABASE hello_world_db;
   ```
2. 连接到该数据库：
   ```txt
   \c hello_world_db
   ```
3. 创建数据表：
   ```sql
   CREATE TABLE my_users (
       id SERIAL PRIMARY KEY,
       name VARCHAR(100) NOT NULL
   );
   ```
4. 插入数据：
   ```sql
   INSERT INTO my_users (name) VALUES ('Alice');
   INSERT INTO my_users (name) VALUES ('Bob');
   ```
5. 查询数据：
   ```sql
   SELECT * FROM my_users;
   ```
6. 列出当前数据库中的所有数据表：
   ```sql
   \dt
   ```
7. 删库跑路（需要先连接回默认数据库 `postgres`）：
   ```sql
   \c postgres
   ```
   ```sql
   DROP DATABASE hello_world_db;
   ```

> 如需退出 PSQL 模式，运行 `\q` 即可。如需从 PostgreSQL 用户切换回原来的用户，运行 `exit` 即可。
