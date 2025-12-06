---
title: 开始上手Nginx服务器
toc: true
categories:
  - CS
tags: [Linux, 服务器]
date: 2025-03-11 22:30:00
updated: 2025-03-28 22:29:00
---

本文是作者在 Ubuntu 系统上部署 Nginx 服务器的记录。

## Nginx 基础

### 简介

[Nginx](https://nginx.org) 是一个高性能的服务器应用程序，由俄罗斯工程师 Igor Sysoev 于 2004 年开发。它主要被用于搭建**HTTP 服务器**、**反向代理服务器**、**负载均衡服务器**等各类 Web 服务器。

Nginx 的主要优势在于**高并发和高能效**。它采用异步、非阻塞的事件驱动架构，能处理大量并发请求（10K 量级）。同时，在相同的硬件条件下，Nginx 的 CPU 和内存消耗要低于同类产品 Apache。

### 安装方法

安装 Nginx：

```bash
sudo apt update
sudo apt install nginx -y
```

确保 Nginx 已启动且已设置开机自启动：

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

访问 `http://<服务器地址>`，若能显示出默认的 Nginx 欢迎页面（Welcome to Nginx），则表示安装成功。若无法访问，请检查服务器端的防火墙配置。

### Hello World

以下内容演示了如何利用 Nginx 在本地的 80 端口部署一个简易的**静态网站**。

<!-- more -->

首先，创建一个静态 HTML 文件：

```bash
sudo mkdir -p /var/www/hello_world
echo "<h1>Hello World</h1>" | sudo tee /var/www/hello_world/index.html
```

通过以下步骤来初始化网站配置：

1. 在 Nginx 的 `sites-available` 目录中，创建一个 Nginx 配置文件：
   ```bash
   sudo nano /etc/nginx/sites-available/hello_world
   ```
2. 添加以下内容到配置文件中：
   ```nginx
   server {
       listen 80;
       server_name localhost;

       root /var/www/hello_world;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }
   }
   ```
3. 将该配置文件通过符号链接，链接到 `sites-enabled` 目录中，从而启用该配置：
   ```bash
   sudo ln -s /etc/nginx/sites-available/hello_world /etc/nginx/sites-enabled/
   ```
4. 移除 `sites-enabled` 中的默认配置（如有）：
   ```bash
   sudo rm /etc/nginx/sites-enabled/default
   ```
5. 测试配置是否正确（应该回显 `syntax is ok`）：
   ```bash
   sudo nginx -t
   ```
6. 重启 Nginx 使得配置生效：
   ```bash
   sudo systemctl restart nginx
   ```

最后，使用 `curl` 或者直接访问服务器的公网 IP 来浏览 Hello World 页面。

## Nginx 典型用例

### 反向代理

一般情况下，为了提供互联网服务，我们会开放服务器的 80/443 端口。但是，一个服务器（或者说一个公网 IP）只能提供一个 80 端口和一个 443 端口，因此只够承载一个程序。

为了在同一服务器上运行多个互联网服务，我们通常会在服务器的其他端口上运行这些服务，然后通过反向代理来与 80/443 端口建立联系。例如，我可以：

- 在 3100 端口运行一个应用网站 `app.example.com`；
- 在 3200 端口运行一个 API 服务 `api.example.com`；
- 在 3300 端口运行一个博客 `blog.example.com`。

将上述域名都通过 DNS 解析到同一服务器 IP，然后配置 Nginx 如下：

```nginx
server {
    listen 80;
    server_name app.example.com;
    
    location / {
        proxy_pass http://localhost:3100;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://localhost:3200;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 80;
    server_name blog.example.com;
    
    location / {
        proxy_pass http://localhost:3300;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

这样一来，Nginx 就可以根据请求的域名来向具体的服务端口递交数据了。

> 事实上，也可以不在“域名”的层级做反向代理，而是在“路由”的层级做反向代理。例如，我们可以设置 `example.com/api/` 指向 API 服务，设置 `example.com/app/` 指向应用程序服务。这时只需在同一 `server` 字段下面配置多个 `location` 字段即可。

### 配置 SSL 加密

> 基本常识：
> - 安全套接字层（Secure Sockets Layer，SSL）是一种安全协议，用于验证通信双方的身份，并保护数据在互联网中传输的安全性和完整性。
> - 启用了 SSL 的 HTTP 协议被称为 HTTPS 协议。
> - SSL 基于密钥交换技术。一套完整的 SSL 证书包括证书（内含公钥）和私钥两部分。
> - SSL 证书具有域名限制。不能在 B 域名下使用 A 域名的 SSL 证书。
> - 证书认证机构（Certificate Authority，CA）是负责颁发、验证和管理 SSL 证书的第三方可信机构。未经 CA 颁发的 SSL 证书通常被认为是不可信的证书。

在启用 HTTPS 前，首先先准备一套 SSL 证书（证书文件 `.crt` 和私钥文件 `.key`）。然后，在配置文件中修改端口号为 443，并添加证书配置，如下所示：

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;
}
```

以上方法是手工配置 SSL 的方法，不仅 SSL 证书需要自己申请，配置流程也较为繁琐。这里推荐使用 [Certbot](https://certbot.eff.org) 来为 Nginx 服务器自动申请 [Let’s Encrypt](https://letsencrypt.org) 颁发的免费证书，并且实现证书的自动配置和自动更新。

下面演示 Certbot + Nginx 的配置步骤：

1. 确保需要申请证书的域名所对应的 Nginx 站点可被访问。
   > 由于 CA 需要验证域名的合法性，所以如果 Nginx 站点不能访问或 DNS 的指向不正确，就会发生如下错误：
   > ```txt
   > Certbot failed to authenticate some domains (authenticator: nginx). The Certificate Authority reported these problems:
   > Domain: example.com
   > Type:   unauthorized
   > Detail: xxxx
   > ```
1. 安装 Certbot 及其插件：
   ```bash
   sudo apt install certbot python3-certbot-nginx
   ```
2. 通过 Certbot 申请证书，并自动安装证书到 Nginx：
   ```bash
   sudo certbot --nginx -d <域名>
   ```
   首次申请时，会要求输入你的邮箱地址，以便向你发送安全通知。
   > 配置成功后的回显示例：
   > ```txt
   > Successfully deployed certificate for example.com to /etc/nginx/sites-enabled/hello_world
   > Congratulations! You have successfully enabled HTTPS on https://example.com
   > ```
3. 测试 Certbot 的自动更新功能：
   ```bash
   sudo certbot renew --dry-run
   ```
4. 将 `http://` 改为 `https://`，在浏览器中访问站点，以检查 HTTPS 协议是否正常运行。

配置结束后，打开站点的 Nginx 配置文件：

```nginx
server {
    server_name example.com;

    location / {
        # ...
    }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/harryh.cn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/harryh.cn/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = example.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    server_name harryh.cn;
    return 404; # managed by Certbot
}
```

可以发现，原站点的配置文件被 Certbot 自动修改了，而且还开启了“强制 HTTPS 重定向”的功能。
