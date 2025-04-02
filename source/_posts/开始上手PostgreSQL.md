---
title: 开始上手PostgreSQL
toc: true
categories:
  - CS
tags: [Linux, 数据库]
date: 2025-03-10 22:56:00
updated: 2025-04-02 16:29:00
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

## PostgresSQL 基础

### 简介

[PostgreSQL](https://www.postgresql.org)（简称 Postgres）是一个开源的、功能强大的、企业级的关系型数据库管理系统，以稳定性、扩展性和标准兼容性著称。

### 安装方法

下面详细讲解 PostgreSQL 的部署步骤。

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
