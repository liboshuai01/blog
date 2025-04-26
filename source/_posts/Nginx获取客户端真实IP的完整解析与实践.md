---
title: Nginx获取客户端真实IP的完整解析与实践
tags:
  - Linux
  - Nginx
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261131562.png'
toc: true
abbrlink: 6e7192ea
date: 2024-09-10 11:29:46
---

在现代生产环境中，Nginx 作为反向代理服务器被广泛应用于服务的暴露与负载均衡。尤其是在面对公网访问时，通常会在 Nginx 前端部署 CDN 或云厂商负载均衡器，以提升访问性能与安全性。如何准确获取并传递客户端的真实 IP，成为开发与运维的重要课题。

本文将详细剖析 Nginx 获取客户端真实 IP 的原理、配置方法以及常用变量的使用场景，帮助您在不同协议与架构下正确处理客户端 IP。

<!-- more -->

## 反向代理与客户端 IP 获取的基本问题

默认情况下，Nginx 并不会自动修改或添加请求头中的 `X-Forwarded-For`，这意味着：

- 如果前端的负载均衡器或 CDN 不传递 `X-Forwarded-For`，Nginx 获取到的客户端 IP 实际上是负载均衡器的 IP。
- 如果前端已经传递了 `X-Forwarded-For`，Nginx 会原样保留并转发该请求头。

因此，为确保后端服务能正确识别客户端真实 IP，通常需要根据实际情况对请求头进行配置和调整。

## 常用变量及其含义

理解 Nginx 的内置变量对于正确配置至关重要，以下为关键变量解析：

### `$remote_addr`

- 表示与 Nginx 服务器建立连接的上一层代理或客户端的 IP 地址。
- 在 TCP 四层负载均衡场景下（开启获取客户端真实 IP 功能），`$remote_addr` 通常是客户端的真实 IP。
- 在 HTTP 七层负载均衡场景下，`$remote_addr` 通常是负载均衡器或 CDN 的 IP，因为客户端真实 IP 通过 `X-Forwarded-For` 传递。

### `$http_x_forwarded_for`

- 获取请求头中 `X-Forwarded-For` 的原始值，通常是前端负载均衡器或 CDN 添加的客户端 IP 列表。
- 该值可以包含一个或多个 IP 地址，多个 IP 用逗号分隔，从左到右依次表示客户端真实 IP 及经过的代理 IP。

### `$proxy_add_x_forwarded_for`

- Nginx 自带的复合变量，用于追加当前连接的客户端 IP（`$remote_addr`）到已有的 `X-Forwarded-For` 列表中。
- 若请求中没有原始的 `X-Forwarded-For`，该变量值即为 `$remote_addr`。
- 例如，如果前端已经传递了 `X-Forwarded-For: 1.1.1.1`，且 `$remote_addr` 是 `2.2.2.2`，设置后为：`X-Forwarded-For: 1.1.1.1, 2.2.2.2`。

## 典型配置方法及应用场景

### 在 Nginx 配置中传递客户端真实 IP

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

此配置可以在保持原有 `X-Forwarded-For` 值的基础上，追加当前连接 IP，适用于多级代理链场景，方便后端服务追踪来源链路。

### 直接覆盖 `X-Forwarded-For` 头

```nginx
proxy_set_header X-Forwarded-For $remote_addr;
```

此配置会覆盖前面传递的所有 `X-Forwarded-For` 信息，直接将当前连接的 IP 作为唯一值。

适用场景：

- 您确信当前连接的 `$remote_addr` 是客户端的真实 IP（如 TCP 四层负载均衡开启真实 IP穿透时）。
- 或者想避免链路传递过长或含有无用代理 IP 时重置 IP 信息。

### 保持负载均衡器传递的真实 IP

当前端负载均衡器已经正确注入客户端真实 IP 到 `X-Forwarded-For`，一般无需在 Nginx 反代层修改该头部，直接转发即可：

```nginx
proxy_set_header X-Forwarded-For $http_x_forwarded_for;
```

或者默认不设置，Nginx 会自动将接收到的请求头透传。

## 不同协议和监听层级对 IP 的影响

- **TCP 四层监听**（四层负载均衡）：

    - 负载均衡器通过网络层直接转发流量，若开启“获取客户端真实 IP”，则 `$remote_addr` 即真实客户端 IP，`X-Forwarded-For` 通常不存在。

- **HTTP 七层监听**（七层负载均衡）：

    - 负载均衡器通过 HTTP 协议代理请求，将客户端真实 IP 放进 `X-Forwarded-For` 头部，`$remote_addr` 是负载均衡器实例 IP。

综上，确定您的架构和协议环境，是选择合适真实 IP 获取策略的前提。

## Nginx 访问日志中打印真实 IP

为了在日志中更准确地记录客户端的真实 IP，配置访问日志时可使用：

```nginx
log_format main '$remote_addr - $remote_user [$time_local] '
                '"$request" $status $body_bytes_sent '
                '"$http_referer" "$http_user_agent" '
                '"$http_x_forwarded_for"';

access_log /var/log/nginx/access.log main;
```

其中，`$http_x_forwarded_for` 可显示真实的客户端 IP 链路，方便排查和分析。

## 安全防护建议

由于 `X-Forwarded-For` 头容易被客户端伪造，建议：

- 只信任来自可信负载均衡/代理服务器的 IP 和 `X-Forwarded-For`。
- 使用 `real_ip` 模块（`ngx_http_realip_module`）替换 `$remote_addr` 为真实客户端 IP，配置如下：

```nginx
set_real_ip_from 192.168.0.0/16;  # 代理服务器 IP 段
real_ip_header X-Forwarded-For;
real_ip_recursive on;
```

此配置确保后端应用能够获得正确的 `$remote_addr`，并防止伪造。

## 总结

- 理解负载均衡器的协议层级和“获取客户端真实 IP”功能对于正确配置 Nginx 与后端应用至关重要。
- `$proxy_add_x_forwarded_for` 是追加客户端 IP 的最佳实践，适合多级代理环境。
- 通过 `real_ip` 模块配置，可将真实客户端 IP 映射到 `$remote_addr`，方便应用处理。
- 日志中打印 `$http_x_forwarded_for` 有助于追踪多跳代理链路。

合理配置与使用这些变量，能帮助您在复杂的负载均衡与代理环境中准确获取客户端真实 IP，确保业务的稳定运行和安全防护。