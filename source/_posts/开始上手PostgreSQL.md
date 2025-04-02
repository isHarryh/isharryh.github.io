---
title: 开始上手PostgreSQL
toc: true
categories:
  - CS
tags: [Linux, 数据库]
date: 2025-03-10 22:56:00
updated: 2025-04-02 21:16:00
---

本文是作者在 Ubuntu 系统上部署 PostgreSQL 数据库的记录。

## 数据库简介

数据库（Database，DB）是用于**存储、管理和检索数据**的信息系统，旨在保证数据的一致性、完整性和安全性。

### 主流分类

数据库主要分为关系型数据（SQL）和非关系型数据库（NoSQL）。主流的数据库列表格如下：

| 数据库 | 类型 | 适用场景 | 主要特点 |
|---|---|---|---|
| **MySQL** | SQL | 传统网站 | 最流行的 SQL 数据库 |
| **SQLite** | SQL | 轻量级应用 | 文件数据库 |
| **PostgreSQL** | SQL | 企业级应用 | 支持复杂查询和高扩展性 |
| **MongoDB** | NoSQL | 实时应用，API 驱动 | 灵活的 JSON-like 文档存储 |
| **Redis** | NoSQL | 实时应用，缓存 | 内存数据库 |

> SQL 指的是结构化查询语言（Structured Query Language），即数据库操作所使用的语言。

### 核心术语

- 数据表（Table）简称表，是数据库中用于存储数据的基本结构，由若干行和列组成。
- 行（Row）又称记录（Record）。
- 列（Column）又称字段（Field）。
- 主键（Primary Key）缩写为 PK，是用于唯一标识一条记录的字段。
- 外键（Foreign Key）缩写为 FK，是用于与其他数据表的主键建立关系的字段。
- 索引（Index）是一类用于加快查询速度的数据结构。
- 事务（Transaction）是一组数据库操作的集合，遵循原子性、一致性、隔离性和持久性原则（ACID 原则）。

## PostgreSQL 基础

### 简介

[PostgreSQL](https://www.postgresql.org)（简称 Postgres）是一个开源的、功能强大的、企业级的关系型数据库管理系统，以稳定性、扩展性和标准兼容性著称。

### 安装方法

下面详细讲解 PostgreSQL 的部署步骤。

<!-- more -->

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

### Hello World

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

## PostgreSQL 权限管理

### 用户

PostgreSQL 自带一个超级用户 `postgres`。但通常，我们会自行创建新的用户来细化权限分配。

#### 创建用户

创建一个名为 `username` 的普通用户：

```sql
CREATE ROLE username;
```

创建一个允许登录（`LOGIN`）的普通用户，并初始化其密码为 `password`：

```sql
CREATE ROLE username WITH LOGIN PASSWORD 'password';
```

创建一个超级用户（慎用）：

```sql
CREATE ROLE username WITH LOGIN SUPERUSER PASSWORD 'password';
```

#### 查询用户

查询所有用户的名称和他们的用户级权限：

```sql
\du
```

#### 删除用户

删除一个名为 `username` 的用户：

```sql
DROP ROLE username;
```

如果用户拥有数据库对象的所有权，建议先转移这些所有权，再清理残留权限，最后删除该用户。三个步骤示例如下，需要谨慎执行：

```sql
REASSIGN OWNED BY username TO new_owner;
DROP OWNED BY username;
DROP ROLE username;
```

### 权限

#### 授予权限

数据库访问权限（连接权）：

```sql
GRANT CONNECT ON DATABASE database_name TO username;
```

模式访问权限（分为建表权和使用权），以 `public` 模式为例：

```sql
GRANT CREATE ON SCHEMA public TO username;
GRANT USAGE ON SCHEMA public TO username;
```

数据表的基本访问权限（增查删改）：

```sql
GRANT INSERT, SELECT, DELETE, UPDATE ON table_name TO username;
```

> 如需对 `public` 模式下的所有数据表进行授权操作，只需将 `table_name` 替换为 `ALL TABLES IN SCHEMA public`。

> 可以使用 `ALL PRIVILEGES` 来指代“所有权限”。特别地，`ALL PRIVILEGES` 不会授予超级用户权限，也不会授予 `DROP SCHEMA` 权限。

#### 继承权限

如果多个用户需要相同的权限，可以创建一个新角色承载这些权限，再让前述用户继承这个新用户的权限。示例如下：

```sql
CREATE ROLE developers;
-- Some GRANT operations here...
GRANT developers TO username1, username2;
```

#### 撤销权限

要想撤销权限，只需将授权语句中的 `GRANT` 改为 `REVOKE`、将 `TO` 改成 `FROM` 即可。示例如下：

```sql
REVOKE CREATE ON SCHEMA public FROM username;
```

## PostgreSQL 连接方法

要想让应用程序连接到数据库，至少要得知数据库的名称、网络地址、用户名和密码。

有的应用程序要求提供 URL 才能连接到数据库服务端。PostgreSQL 的 URL 格式如下所示：

```txt
postgresql://username:password@hostname/database_name
```

### 本机连接

PostgreSQL 默认对本机（`localhost` 或 `127.0.0.1`）开放连接权限，默认端口为 5432 端口。

### 非本机连接

要想在其他设备（主机）上连接 PostgreSQL，需要修改两处配置。

1. 在全局设置中，开放 PostgreSQL 的监听。
   ```bash
   sudo nano /etc/postgresql/版本号/main/postgresql.conf
   ```
   将 `listen_addresses` 字段的值从 `localhost` 修改为 `*`：
   ```ini
   listen_addresses = '*'
   ```
   如果已知确切的监听地址，可以直接在此指定具体的地址（使用逗号分隔）。
2. 在主机认证设置中，添加主机到白名单（以 IPv4 为例）。
   ```bash
   sudo nano /etc/postgresql/版本号/main/pg_hba.conf
   ```
   将这一行添加到文件中，并将 `xx.xx.xx.xx/xx` 改为主机的 IP 地址和子网掩码：
   ```ini
   host all all xx.xx.xx.xx/xx scram-sha-256
   ```
   从左到右，值的含义分别是连接类型、准入数据库范围、准入用户范围、主机地址、认证方法。

> 若将数据库开放到公网中，请注意潜在的安全风险（例如暴力破解）。
