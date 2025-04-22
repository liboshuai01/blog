---
title: Nginx部署前后端分离项目并实现负载均衡
tags:
  - Linux
  - Nginx
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504190111406.png'
toc: true
abbrlink: '87555684'
date: 2023-02-13 20:00:05
---

本文介绍了如何利用 Nginx 部署前后端分离的 Web 项目，并通过 Nginx 反向代理实现后端服务的负载均衡。

部署场景基于两台服务器，分别承担静态前端资源和后端 API 服务。Nginx 作为网关，负责监听统一端口，转发前端资源请求和后端接口请求。

前端静态资源直接由 Nginx 提供服务，后端 API 请求则根据一定的分发策略，实现多节点负载均衡，提高整体系统的可用性和并发能力。

通过调整 Nginx 配置文件，可以灵活指定静态资源目录、代理路径、负载均衡策略以及常见的安全转发头部设置。

<!-- more -->

> 注意：此种方式仅仅解决了前端访问后端负载均衡的问题，但是nginx本身的高可用并没有解决，需要通过keepalived或F5来实现

环境详情
---

| 服务器   | IP            | 前端路径                                      | 后端端口  | Nginx监听端口 | 项目访问地址               |
|-------|---------------|-------------------------------------------|-------|-----------|----------------------|
| node1 | 192.168.6.114 | /home/lbs/project/ruoyi-pro/frontend/dist | 48080 | 48000     | 192.168.6.114:48000/ |
| node2 | 192.168.6.115 | /home/lbs/project/ruoyi-pro/frontend/dist | 48080 | 48000     | 192.168.6.115:48000/ |

Nginx配置
---

> 存放在`conf.d`目录下，文件名随意，只要以`.conf`结尾即可

```txt
# 定义后端服务的上游服务器（负载均衡池）
upstream backend {
    server 192.168.6.114:48080;  # 后端服务1
    server 192.168.6.115:48080;  # 后端服务2
}

server {
    listen       48000;        # 监听 48000 端口
    server_name  _;            # 匹配所有主机名或 IP

    # 静态文件服务配置，`/` 应与前端配置的打包base路径一致
    location / {
        root   /home/lbs/project/ruoyi-pro/frontend/dist;    # 前端打包后的静态文件目录
        index  index.html index.htm;                         # 默认首页文件
        try_files $uri $uri/ /index.html;                    # 支持 history 路由，找不到文件时返回 index.html
    }

    # 后端接口代理配置，需要与前端配置的后端接口根路径保持一致
    location /admin-api {
        proxy_set_header Host $http_host;                           # 将请求的 Host 头转发给后端
        proxy_set_header X-Real-IP $remote_addr;                    # 设置客户端真实 IP，便于后端获取
        proxy_set_header REMOTE-HOST $remote_addr;                  # 自定义头，传递客户端 IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;# 转发整个链路的客户端 IP
        proxy_pass http://backend/;                                 # 通过 upstream backend 进行负载均衡转发
    }
}
```

热加载Nginx配置
---

```shell
./nginx -t
./nginx -s reload
```

验证测试
---

> 浏览器访问地址：http://192.168.6.114:48000/

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504190126313.png)

结语
---

通过本方案的实践，前后端分离项目可借助 Nginx 实现高效的资源分发与后端负载均衡，有效提升了系统的可维护性与扩展性。

虽然本示例尚未涉及 Nginx 本身的高可用配置，但已为生产环境打下坚实基础。

后续可以结合 Keepalived、F5 等高可用方案，保证网关层的服务连续性。

运维与后端开发人员可按照文中步骤逐步实操，也可根据实际需求进一步优化 Nginx 配置，持续提升系统性能与可靠性。