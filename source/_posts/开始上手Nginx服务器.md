---
title: 开始上手Nginx服务器
toc: true
categories:
  - CS
tags: [Linux, 服务器]
date: 2025-03-11 22:30:00
updated: 2025-03-28 20:01:00
---

本文是作者在 Ubuntu 系统上部署 Nginx 服务器的记录。

## Nginx 基础

### 简介

通常使用 [Nginx](https://nginx.org) 来部署 Web 服务器或反向代理服务器。

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

### Hello World

以下内容演示了如何利用 Nginx 在本地的 80 端口部署一个简易的静态网页。

首先，创建一个静态网页：

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
