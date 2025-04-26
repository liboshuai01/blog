---
title: Manjaro Linux 下 UFW 防火墙实用指南
tags:
  - Linux
  - Manjaro
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261313450.png'
toc: true
abbrlink: 7671540c
date: 2024-11-10 13:12:24
---

在信息安全日益重要的当下，防火墙作为系统安全的第一道防线，发挥着关键作用。UFW（Uncomplicated Firewall）是一个简洁易用的防火墙管理工具，广泛应用于各种 Linux 发行版。本文将为你详细介绍如何在 Manjaro Linux 上安装、配置和管理 UFW，帮助你提升系统的安全防护能力。

<!-- more -->

## 安装与启动 UFW

Manjaro Linux 采用 Arch 体系，使用 `pacman` 作为包管理工具。你可以通过以下命令快速安装 UFW：

```bash
sudo pacman -S ufw
```

安装完成后，建议启用并启动 UFW 服务以确保防火墙功能正常运行：

```bash
sudo systemctl enable ufw
sudo systemctl start ufw
```

这样，在系统启动时 UFW 会自动激活，保障系统网络安全。

## 检查 UFW 当前状态

了解防火墙的运行状态和规则是维护安全的重要环节。你可以使用以下命令查看 UFW 的基本状态：

```bash
sudo ufw status
```

若需更详细的状态信息，包括规则、日志级别等，可以执行：

```bash
sudo ufw status verbose
```

该命令会帮助你全面掌握防火墙策略和实际生效的规则。

## 启用与禁用防火墙

若初次使用或完成规则配置后，启用 UFW 是确保规则生效的关键：

```bash
sudo ufw enable
```

启用后，UFW 会自动应用当前配置的规则来筛选网络流量。

在某些情况下，比如调试网络连接问题，临时关闭防火墙是必要的：

```bash
sudo ufw disable
```

禁用后，所有流量将不受限制，系统暴露于外部环境，使用时请谨慎。

## 配置防火墙规则

### 允许指定端口或服务

例如，要允许 SSH（默认端口 22）的流量通过：

```bash
sudo ufw allow ssh
```

你也可以直接指定端口号：

```bash
sudo ufw allow 22
```

同样地，若你在运行 Web 服务，允许 HTTP 和 HTTPS 端口流量：

```bash
sudo ufw allow http
sudo ufw allow https
```

### 阻止特定端口流量

如果你想禁止某个端口的流量，如阻止 80 端口：

```bash
sudo ufw deny 80
```

这对于关闭不必要或存在安全隐患的服务端口非常实用。

### 允许来自特定 IP 的访问

限制访问来源是防火墙策略的常见需求，比如仅允许某个 IP 访问服务器：

```bash
sudo ufw allow from 192.168.1.100
```

你也可以限定仅允许该 IP 访问特定端口：

```bash
sudo ufw allow from 192.168.1.100 to any port 22
```

### 查看与管理规则

查看所有规则时，推荐使用编号显示，方便后续管理：

```bash
sudo ufw status numbered
```

若需要删除某条规则，先通过上述命令找到对应编号，然后执行：

```bash
sudo ufw delete [编号]
```

例如删除第 2 条规则：

```bash
sudo ufw delete 2
```

这极大提升规则管理的灵活性和准确性。

## 高级配置与维护

### 设置默认策略

为了强化系统安全，建议设置合理的默认策略。一般情况下默认拒绝所有入站连接，允许所有出站连接：

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

这意味着除非明确允许，否则所有入站连接都会被阻止。

### 重新加载配置

修改配置文件或规则后，使用以下命令使更改立即生效：

```bash
sudo ufw reload
```

这避免了重启防火墙服务带来的连接中断。

### 查看日志信息

UFW 的日志帮助你了解被阻止的访问尝试和可能的安全事件：

```bash
sudo less /var/log/ufw.log
```

建议定期检查日志，及时发现异常流量。

## 结语

通过使用 UFW，Manjaro Linux 用户可以以简洁高效的方式管理防火墙，提升系统的网络安全水平。本文覆盖了从安装、基本操作，到高级配置的实用技巧。建议结合实际需求，合理规划防火墙规则，定期维护更新，保障系统免受潜在威胁。如需针对特定应用调整规则，更可结合 UFW 文档深入学习扩展用法。

安全无小事，防火墙是守护系统稳定运行的坚实屏障，期待你能借助 UFW 构建更安全的 Manjaro 环境。