---
title: 使用Docker-Compose快速部署Nginx
tags:
  - Linux
  - Centos
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504240035780.png'
toc: true
abbrlink: afa7cb7f
date: 2023-12-14 00:34:17
---

在现代软件开发与部署流程中，容器化技术已成主流，Docker 和 Docker Compose 简化了应用的管理与交付。Nginx 作为轻量级高性能的 Web 服务器和反向代理，广泛应用于生产环境。本文详细介绍如何利用 `docker-compose` 快速搭建一个完整的 Nginx 容器环境，包含配置挂载、日志管理及静态资源托管，帮助您轻松实现高效部署。

<!-- more -->

---

## 准备工作

确保宿主机上已安装并正确配置以下环境：

- Docker （建议 20.x 版本以上）
- Docker Compose （V2 推荐）
- 基本命令行操作能力

---

## 编写 `docker-compose.yml`

在项目根目录新建 `docker-compose.yml` 文件，定义 Nginx 服务。示例如下：

```yaml
version: '3'

services:
  nginx:
    image: nginx:1.26.3
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./conf/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./html:/usr/share/nginx/html:ro
      - ./logs:/var/log/nginx
```

**说明：**

- 使用官方稳定版本 `nginx:1.26.3` 镜像。
- 挂载配置、静态文件、日志目录，实现配置与数据的持久化和灵活管理。
- `restart: always` 保证容器随 Docker 启动自动启动。
- 映射宿主机端口 80 到容器端口 80，方便本机访问。

---

## 创建项目目录与配置文件

为保证容器运行完整，需创建对应目录和基础文件：

### 1. 创建目录结构

```bash
mkdir -p conf conf.d logs html
```

### 2. Nginx 主配置文件 `conf/nginx.conf`

```nginx
user  root;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
```

用命令快速创建：

```bash
tee ./conf/nginx.conf <<'EOF'
user  root;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/conf.d/*.conf;
}
EOF
```

### 3. 默认站点配置 `conf.d/default.conf`

```nginx
server {
    listen       80;
    listen       [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```

创建命令：

```bash
tee ./conf.d/default.conf <<'EOF'
server {
    listen       80;
    listen       [::]:80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
EOF
```

### 4. 静态网页文件

首页 `html/index.html`：

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto; font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
<p>For more information, visit <a href="http://nginx.org/">nginx.org</a>.</p>
</body>
</html>
```

错误页 `html/50x.html`：

```html
<!DOCTYPE html>
<html>
<head>
<title>Error</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto; font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>An error occurred.</h1>
<p>The page you requested is currently unavailable. Please try again later.</p>
<p>If you are the administrator, check the error logs for details.</p>
</body>
</html>
```

命令快速创建：

```bash
tee ./html/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto; font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
<p>For more information, visit <a href="http://nginx.org/">nginx.org</a>.</p>
</body>
</html>
EOF

tee ./html/50x.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
<title>Error</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto; font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>An error occurred.</h1>
<p>The page you requested is currently unavailable. Please try again later.</p>
<p>If you are the administrator, check the error logs for details.</p>
</body>
</html>
EOF
```

---

## 启动并验证容器

执行以下命令启动 Nginx 容器：

```bash
docker-compose up -d
```

容器启动后，访问 `http://localhost` 应可看到默认欢迎页面。如果无法访问：

- 检查容器状态：`docker ps`
- 查看日志：`docker logs nginx`
- 确认端口 80 是否被其他进程占用

---

## 扩展与优化建议

- **安全加固**
    - 非生产环境中可适当设置 `user root`，生产建议使用非 root 用户。
    - 通过添加 HTTPS 支持，部署 SSL 证书，保障数据安全。
- **性能调优**
    - 根据服务器硬件调整 `worker_processes` 和 `worker_connections`。
    - 配置 gzip 压缩，缓存策略提升响应速度。
- **集群与负载均衡**
    - 利用 Docker Swarm 或 Kubernetes 实现 Nginx 高可用集群。
    - 结合负载均衡器分摊流量。
- **日志管理**
    - 使用 ELK（Elasticsearch, Logstash, Kibana）或 Prometheus + Grafana 监控日志和指标。
- **配置版本管理**
    - 使用 Git 管理配置文件，实现版本控制和协作。

---

## 总结

本文从零开始，通过编写 `docker-compose.yml` 和搭建目录结构，为您讲解了如何快速使用 Docker Compose 部署一个功能完善的 Nginx 服务。此方案不仅简化了部署流程，还保证了配置的灵活性与可维护性。您可以基于此基础，结合企业需求，进一步定制与优化 Nginx 设置。

祝您顺利搭建高效、稳定的 Nginx 服务器，开启高效的容器化开发旅程！

---

如果您有任何疑问或想了解更多关于 Docker、Nginx 及微服务架构内容，欢迎留言交流！
