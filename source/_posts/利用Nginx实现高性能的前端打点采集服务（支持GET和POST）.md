---
title: 利用Nginx实现高性能的前端打点采集服务（支持GET和POST）
tags:
  - Linux
  - Nginx
  - 前端
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250424112920997.png'
toc: true
abbrlink: 91322bdf
date: 2025-04-15 11:28:12
---

在业务开发中，我们经常需要通过“数据驱动”做决策。前端页面中的各类打点事件产生大量请求，如何高效、稳定地进行数据采集，成为后端服务设计的重要课题。相比引入复杂的多语言服务，我们可以巧妙利用 Nginx 的轻量高性能特性，搭建一个具备 CORS 支持，既能处理 GET 请求，也能优雅接收 POST 请求体的打点采集服务，满足生产级的需求。

本文将结合实践，详细介绍如何通过配置 Nginx，解决 Nginx 默认不支持 POST 请求且无法记录 POST Body 的问题，设计支持跨域，日志格式友好，并能便于后续离线分析的打点收集服务。

<!-- more -->

---

## 问题背景与挑战

- **数据来源**：业务 Web 服务的请求日志是数据采集的重要来源，为了降低对业务的影响，通常会设计专门用于统计的打点服务器。
- **打点并发压力**：前端页面瞬间会发起大量打点请求，容易导致友军 DDoS，影响用户体验和服务器稳定。
- **Nginx的局限**：
    - 默认不支持 POST 请求访问指定路径，返回 405 错误。
    - 即使支持部分动态解析，默认 access_log 也不会记录 POST 的请求体。
    - 跨域（CORS）限制导致前端 POST 请求受限。
- **如何用原生 Nginx 实现完整流程**，成为值得探讨的重点。

---

## Nginx 默认 POST 请求的限制

通过实验在无任何配置的 Nginx 容器中使用 `curl -d '{...}' -X POST` 访问，会收到 HTTP 405 `Not Allowed` 错误。

这是因为诸如 `ngx_http_stub_status_module` 等模块只允许 GET/HEAD 方法。Nginx 默认作为静态文件服务器，不解析 POST 请求体，也不支持写入日志。

---

## 使 Nginx 支持原生 POST 请求及日志记录

### 1. 利用 `error_page 405 =200 $uri` 绕过默认限制

添加以下配置可使 Nginx 返回 200，而非 405：

```nginx
error_page 405 =200 $uri;
```

调试验证，客户端 POST 请求能拿到响应。但是，日志里仍然不记录 POST Body。

### 2. 自定义日志格式，记录请求体

默认 `access_log` 不包含请求体 `$request_body`，需要定义新的日志格式：

```nginx
log_format main escape=json '$remote_addr - $remote_user [$time_local] "$request" '
                            '$status $body_bytes_sent "$http_referer" '
                            '"$http_user_agent" "$http_x_forwarded_for" $request_body';
```
此处使用 `escape=json` 让请求体中的转义字符得到处理，方便后续日志解析。

### 3. 通过 `proxy_pass` 激活 Nginx POST Body 解析

为了让 Nginx 读取和记录请求体，必须将请求代理到内部路径：

```nginx
location / {
    proxy_pass http://127.0.0.1/internal-api-path;
}
location /internal-api-path {
    access_log off;
    default_type application/json;
    return 200 '{"code":0,"data":"success"}';
}
```

这里一个小技巧：通过代理触发 Nginx 解析 POST Body，且内部接口返回固定 JSON，方便前端调用及后端排查。

---

## 过滤非 POST 请求，避免日志污染及接口滥用

```nginx
map $request_method $loggable {
    default 0;
    POST 1;
}

server {
    location / {
        if ( $request_method !~ ^(POST|OPTIONS)$ ) { return 405; }
        access_log /var/log/nginx/access.log main if=$loggable;
        proxy_pass http://127.0.0.1/internal-api-path;
    }
}
```

- 利用 `map` 变量，只有 POST 请求被记录日志。
- 非 POST/OPTIONS 请求直接返回 405。
- 避免日志记录无用的 GET 请求，更节省资源。

---

## 解决跨域问题，支持前端跨站访问

现代浏览器发起跨域 POST 请求时，会首先做一个 `OPTIONS` 预检请求，如果服务器不响应，前端请求失败。

完整配置如下：

```nginx
map $http_origin $corsHost {
    default 0;
    "~(.*).soulteary.com"  1;
    "~(.*).baidu.com"       1;
}

server {
    location / {
        if ( $request_method !~ ^(POST|OPTIONS)$ ) { return 405; }
        if ( $corsHost = 0 ) { return 405; }

        if ( $corsHost = 1 ) {
            add_header 'Access-Control-Allow-Credentials'   'false';
            add_header 'Access-Control-Allow-Headers'       'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With,Date,Pragma';
            add_header 'Access-Control-Allow-Methods'       'POST,OPTIONS';
            add_header 'Access-Control-Allow-Origin'        '$http_origin';
        }

        if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Credentials'   'false';
            add_header 'Access-Control-Allow-Headers'       'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With,Date,Pragma';
            add_header 'Access-Control-Allow-Methods'       'POST,OPTIONS';
            add_header 'Access-Control-Allow-Origin'        '$http_origin';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
        }

        access_log /var/log/nginx/access.log main if=$loggable;
        proxy_pass http://127.0.0.1/internal-api-path;
    }
}
```

- 利用 `map` 定义白名单域名，拒绝非法跨域。
- 预检请求返回 HTTP 204，无响应体。
- 动态添加 `Access-Control-Allow-Origin`，更安全。

---

## 支持 GET 请求的“打点埋点GIF方案”

出于兼容一些老设备或轻量采集考虑，仍然需要支持经典的通过 image 请求的 GET 打点：

```nginx
log_format slr_event_format '{"eventTime":"$msec","channel":"$arg_channel","targetField":"$arg_targetField","targetValue":"$arg_targetValue","eventField":"$arg_eventField","eventValue":"$arg_eventValue","eventAttrMap":"$arg_eventAttrMap"}';

map $time_iso8601 $log_date {
    '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
    default 'date-not-found';
}

location /slr_event_local.gif {
    log_subrequest on;
    access_log /var/log/nginx/slr_event_local_${log_date}.json slr_event_format;
    add_header Expires "Fri, 01 Jan 1980 00:00:00 GMT";
    add_header Pragma "no-cache";
    add_header Cache-Control "no-cache, max-age=0, must-revalidate";
    empty_gif;
}
```

- 日志格式以 JSON 字符串格式记录 URL 参数。
- 按日期分文件保存，方便后续切割和归档。
- 返回一个透明 1x1 像素 GIF，前端 img 标签发起请求即可。
- 关闭浏览器缓存，确保每次请求都到后端。

---

## 完整且实用的 Nginx 配置示例

以下是一份集成上述功能，满足生产使用的 Nginx 配置节选（略去冗余注释），供参考：

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events { worker_connections 1024; }

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main escape=json '$remote_addr - $remote_user [$time_local] "$request" '
                                '$status $body_bytes_sent "$http_referer" '
                                '"$http_user_agent" "$http_x_forwarded_for" $request_body';

    log_format slr_event_format '{"eventTime":"$msec","channel":"$arg_channel","targetField":"$arg_targetField","targetValue":"$arg_targetValue","eventField":"$arg_eventField","eventValue":"$arg_eventValue","eventAttrMap":"$arg_eventAttrMap"}';

    map $time_iso8601 $log_date {
        '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
        default 'date-not-found';
    }

    map $request_method $loggable {
        default 0;
        POST 1;
    }

    map $http_origin $corsHost {
        default 0;
        "~(.*).soulteary.com"  1;
        "~(.*).baidu.com"      1;
    }

    server {
        listen 48881;
        server_name localhost;
        charset utf-8;
        client_max_body_size 10m;

        # GET埋点接口(本地环境)
        location /slr_event_local.gif {
            log_subrequest on;
            access_log /var/log/nginx/slr_event_local_${log_date}.json slr_event_format;
            add_header Expires "Fri, 01 Jan 1980 00:00:00 GMT";
            add_header Pragma "no-cache";
            add_header Cache-Control "no-cache, max-age=0, must-revalidate";
            empty_gif;
        }

        # POST埋点接口(本地环境)
        location /batch_slr_event_local {
            if ( $request_method !~ ^(POST|OPTIONS)$ ) { return 405; }
            if ( $corsHost = 0 ) { return 405; }

            if ( $corsHost = 1 ) {
                add_header 'Access-Control-Allow-Credentials' 'false';
                add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With,Date,Pragma';
                add_header 'Access-Control-Allow-Methods' 'POST,OPTIONS';
                add_header 'Access-Control-Allow-Origin' $http_origin;
            }

            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Credentials' 'false';
                add_header 'Access-Control-Allow-Headers' 'Accept,Authorization,Cache-Control,Content-Type,DNT,If-Modified-Since,Keep-Alive,Origin,User-Agent,X-Mx-ReqToken,X-Requested-With,Date,Pragma';
                add_header 'Access-Control-Allow-Methods' 'POST,OPTIONS';
                add_header 'Access-Control-Allow-Origin' $http_origin;
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
            }

            access_log /var/log/nginx/slr_event_local_${log_date}.json main if=$loggable;
            proxy_pass http://127.0.0.1:48881/internal-api-path-local;
        }

        location /internal-api-path-local {
            access_log off;
            default_type application/json;
            return 200 '{"code": 0, "data": "success"}';
        }

        error_page 405 =200 $uri;
    }
}
```

> 注意：
> - 这里定义了两个入口：`/slr_event_local.gif` 负责 GET 打点日志（兼容旧方案），`/batch_slr_event_local` 允许 POST 请求体日志打点。
> - 采用动态日志文件切割：每天生成不同日期文件，便于后续集中处理。
> - 通过严格的 CORS 白名单保护接口不被滥用。
> - 预检 OPTIONS 请求配置完备，避免了前端跨域阻断。

---

## 实战经验与后续优化建议

- **日志接入和存储**：采集日志后，结合日志聚合平台（比如 ELK, ClickHouse）针对 `request_body` 进行分析和挖掘。
- **上游代理支持**：结合 Traefik 等反向代理，实现打点服务的水平扩展，提高并发和稳定性。
- **打点SDK设计优化**：前端通过自定义 SDK 做合并打包，减少请求量，降低服务器压力。
- **安全防护**：设置请求大小限制 (`client_max_body_size`) 和白名单限制，防止恶意请求。
- **日志格式**：自定义 JSON 友好格式，便于后续流式解析、在线统计。
- **健康检查**：添加 `/health` 接口，方便容器编排与自动化运维。

---

## 总结

通过本文示范的 Nginx 配置，我们有效解决了：

- Nginx 默认无法处理 POST 请求体和记录日志的问题
- 避免了跨域请求阻断，支持现代前端跨域采集
- 灵活支持 GET 图像请求和 POST JSON 批量上报
- 满足生产刚需的高性能、低运维成本和稳健可扩展

这种“轻量且强大”的打点采集服务足以支撑绝大多数中小业务线的需求，且极易集成入现有运维体系。