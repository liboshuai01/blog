---
title: iptables限制CentOS互联网访问，仅允许局域网通信
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250423112928764.png'
toc: true
abbrlink: 15c8f145
date: 2023-12-05 11:28:39
---

在某些情况下，我们希望临时阻止 CentOS 服务器访问互联网，且防止互联网访问该服务器，但仍然允许与局域网内设备的通信。这样做可以用于安全隔离、测试或维护场景。本文将介绍如何通过 `iptables` 实现这一目标，包括具体命令和保存规则的方法。

<!-- more -->

---

## 环境准备

- 操作系统：CentOS（假设版本 7 及以上）
- 工具：`iptables`（防火墙工具）

---

## 安装并启用 iptables 服务

CentOS 7 之后默认使用 `firewalld`，不过我们这里使用 `iptables` 来进行更细粒度控制。首先安装并启用 `iptables-services`：

```bash
sudo yum install iptables-services -y
sudo systemctl enable iptables
sudo systemctl start iptables
```

这确保了 `iptables` 服务可用，且可以在系统启动时自动应用规则。

---

## 使用 iptables 阻止所有互联网访问

### 1. 阻止所有出站流量

默认策略设置为拒绝所有出站流量，防止服务器访问互联网：

```bash
sudo iptables -P OUTPUT DROP
```

### 2. 阻止所有入站流量

同理，将所有入站流量默认拒绝，避免互联网主机访问服务器：

```bash
sudo iptables -P INPUT DROP
```

---

## 允许访问局域网通信

通常，局域网地址段为私有IP地址，如 `192.168.0.0/16`、`10.0.0.0/8` 或 `172.16.0.0/12`。以 `192.168.0.0/16` 为例，添加允许规则：

```bash
# 允许服务器向局域网主机发送数据
sudo iptables -A OUTPUT -d 192.168.0.0/16 -j ACCEPT

# 允许局域网主机访问服务器
sudo iptables -A INPUT -s 192.168.0.0/16 -j ACCEPT
```

> **注意**：请根据实际局域网地址段替换上述 IP 范围，确保规则有效。

---

## 额外允许 localhost 通信（推荐）

为了保证服务器自身服务的正常运行，应该允许环回接口的流量：

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
```

---

## 保存规则，使其重启后生效

`iptables` 规则默认为临时，重启服务器会丢失。通过以下命令保存当前规则：

```bash
sudo service iptables save
```

或者 CentOS 7+ 系统使用：

```bash
sudo iptables-save | sudo tee /etc/sysconfig/iptables
sudo systemctl restart iptables
```

之后，系统重启时会自动加载你保存的规则。

---

## 恢复默认允许互联网访问规则

当需要恢复正常的互联网访问权限时，请执行：

```bash
# 设置默认策略为接受所有流量
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT

# 删除之前新增的局域网规则
sudo iptables -L --line-numbers
```

使用 `iptables -L --line-numbers` 列出规则，记下对应规则行号，逐一删除：

```bash
sudo iptables -D OUTPUT <line_number>
sudo iptables -D INPUT <line_number>
```

---

## 总结

通过设置 `iptables` 默认策略为拒绝，并显式允许局域网地址段，可以有效做到：

- 阻止服务器访问互联网
- 阻止互联网访问服务器
- 同时允许局域网内正常通信

这在服务器维护、隔离测试或安全环境中非常实用。请根据实际网络环境调整 IP 地址段，并结合系统服务确保服务器核心功能正常。

---

如需进一步自动化管理防火墙规则，也可以考虑引入 `firewalld`、`nftables` 或配置网络策略管理工具，提升规则维护和集成效率。