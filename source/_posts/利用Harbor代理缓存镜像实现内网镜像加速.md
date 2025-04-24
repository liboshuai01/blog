---
title: 利用Harbor代理缓存镜像实现内网镜像加速
abbrlink: 4758ac99
date: 2023-12-06 11:38:41
tags:
  - Linux
  - Harbor
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250423114058678.png'
toc: true
---

在企业内网环境中，开发者经常需要使用外网公共镜像资源。但由于网络受限或者带宽有限，直接拉取外网镜像往往效率低下甚至无法访问。为解决这一瓶颈，Harbor v2.1 引入了强大的「代理缓存」功能，帮助内网Harbor仓库智能代理并缓存外部镜像，极大提升镜像拉取效率，节省带宽，规避 Docker Hub 访问速率限制。

本文将详细讲解如何在内网环境中配置并使用 Harbor 代理缓存功能，轻松实现外网镜像资源的同步与加速。

<!-- more -->

---

## 什么是 Harbor 代理缓存？

Harbor 代理缓存（Proxy Cache）是一种中间镜像仓库，类似 Nexus 的代理模式，允许 Harbor 代理来自 Docker Hub 或其他私有仓库的镜像请求。它的工作原理是：

- 当内网客户端第一次拉取某个镜像时，代理缓存仓库会向外网仓库请求镜像并下载至本地缓存。
- 随后其他客户端拉取同一镜像时，直接从本地缓存仓库获取，避免重复访问外网。
- 有效节约内外网带宽，减少 Docker Hub 的访问频率，避免触发速率限制。

从 Harbor v2.1.1 起，代理缓存功能已优化支持 Docker Hub 的速率限制策略，保证流畅使用。

---

## 环境假设

- 已搭建内网 Harbor v2.1.1 或以上版本
- 内网 Harbor 服务器 IP：10.0.0.38，监听端口为 8004
- 目标缓存外网镜像仓库为官方 Docker Hub

---

## 配置步骤

### 1. 新建代理缓存仓库

1. 登录 Harbor 管理控制台，进入【项目管理】。
2. 创建新项目，命名为 `dockerhub`（或者其他合适名称）。
3. 进入该项目，选择【仓库】 - 【代理缓存】 - 【+ 新建代理缓存仓库】。
4. 填写代理源仓库地址，例如 `https://registry-1.docker.io`（Docker Hub 官方地址），并设置代理仓库名称为 `dockerhub`。
5. 保存完成代理缓存仓库创建。

> ![新建代理缓存仓库示例](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250423113721326.png)  
> ![填写代理源信息](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250423113721373.png)

### 2. 新建代理缓存项目

为了管理代理仓库，我们通常单独创建一个项目用于代理缓存，方便权限和镜像分类。

1. 在 Harbor 上创建名为 `dockerhub` 的项目（或配合代理缓存仓库名一致）。
2. 确认该项目已经启用了代理缓存仓库支持。

> ![创建代理缓存项目示例](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250423113721381.png)

### 3. 通过代理缓存项目拉取镜像

配置完成后，客户端只需修改镜像源地址前缀，指向代理缓存项目即可使用缓存功能。

```bash
docker login 10.0.0.38:8004
docker pull 10.0.0.38:8004/dockerhub/mysql:latest
```

运行以上命令后，如果本地缓存库没有 `mysql:latest`，Harbor 会自动从 Docker Hub 拉取并缓存镜像。后续拉取同样的镜像时，则直接从本地缓存获取，速度更快且不消耗外网带宽。

示例拉取日志：

```
latest: Pulling from dockerhub/mysql
0e03bdcc26d7: Pull complete
Digest: sha256:8c5aeeb6a5f3ba4883347d3747a7249f491766ca1caa47e5da5dfcf6b9b717c0
Status: Downloaded newer image for 10.0.0.38:8004/dockerhub/mysql:latest
```

---

## 验证缓存是否成功

登录 Harbor Web 控制台，切换到 `dockerhub` 项目，您可以看到已缓存的镜像 `mysql:latest` 列表。与内网仓库镜像对应，说明代理缓存仓库工作正常。

---

## 总结

通过 Harbor 的代理缓存功能，内网用户可以：

- 无感知访问外网镜像资源
- 避免重复拉取同一镜像节约带宽
- 缓存镜像有效管理版本变更
- 解决 Docker Hub 速率限制带来的访问瓶颈

这对内网环境搭建私有镜像仓库、保证开发效率、降低运维压力具有非常大帮助。建议近期升级 Harbor 至 v2.1.1 及以上版本，充分利用代理缓存优势。

---

如果您正面临内网镜像访问缓慢或受限问题，不妨试试 Harbor 的代理缓存功能，助力您的容器镜像管理更高效、更稳定！