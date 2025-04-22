---
title: CentOS7更改默认SSH端口与配置指南
abbrlink: 53a37b46
date: 2023-11-23 17:19:41
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250422172025098.png'
toc: true
---

SSH（Secure Shell）是 Linux 服务器远程管理的核心工具，其默认监听端口为 22。由于端口 22 众所周知，这也使得服务器容易受到自动化扫描和暴力破解攻击。虽然更改 SSH 默认端口只是安全加固的一种辅助手段，但它能有效地降低被自动化攻击工具扫描的风险。

本文将系统性地介绍如何在 CentOS 7 系统中安全地更改 SSH 端口，解决 SELinux 和防火墙相关配置，确保远程访问不中断。同时，我们还会探讨提升 SSH 安全性的更有效搭配方案。

<!-- more -->

---

## 为什么要更改 SSH 默认端口？

- **减少自动扫描攻击**：大量恶意机器人默认扫描 22 端口，通过更改端口可以减少被扫描的概率。
- **防止大规模暴力破解**：将 SSH 端口调整到非标准端口可以有效防止针对 22 端口的暴力破解攻击。
- **配合其他安全措施提升防御层次**：虽然更改端口并不能阻止有决心的攻击者扫描，但这是安全“深度防御”策略的一部分。

> **提示：**  
> 更安全的 SSH 防护手段是使用基于密钥的无密码登录、限制 IP 白名单、以及开启两步验证等措施。

---

## 步骤详解：如何更改 CentOS 7 的 SSH 默认端口

### 1. 备份 SSH 配置文件

在动手修改前，请务必备份当前配置，防止误修改导致无法登录救援。

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

### 2. 修改 SSH 配置文件，增加或更改端口

编辑 SSH 服务配置文件：

```bash
sudo vi /etc/ssh/sshd_config
```

找到 `Port` 行，默认是：

```bash
Port 22
```

您可以采用如下两种策略：

- **增加新端口，保留 22 端口**（安全切换期）

```bash
Port 22
Port 2244
```

- **直接修改成新端口，禁用 22**

```bash
Port 2244
```

修改后，保存并退出编辑器。

> **安全建议**：初次更改建议先添加新端口，确保能够成功通过新端口登录，再关闭 22，以免被锁定。

### 3. 配置 SELinux 允许新 SSH 端口

CentOS 7 默认开启 SELinux，且默认只开放 22 端口用于 SSH 服务，需添加新端口：

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2244
```

- 如果出现 `semanage: command not found` 错误，安装必需包：

```bash
sudo yum -y install policycoreutils-python
```

- 重复执行添加端口命令。

> **提示**：  
> 也可以临时关闭 SELinux，但不推荐，因为会降低系统整体安全等级：

```bash
sudo setenforce 0
```

---

### 4. 配置防火墙（Firewalld）放行新端口

确保新端口能通过防火墙访问：

```bash
sudo firewall-cmd --permanent --zone=public --add-port=2244/tcp
```

若出现 `FirewallD is not running`，先启动防火墙：

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

确认防火墙运行状态：

```bash
sudo firewall-cmd --state
```

加载新配置：

```bash
sudo firewall-cmd --reload
```

### 5. 重启 SSH 服务生效

生效更改：

```bash
sudo systemctl restart sshd.service
```

验证 SSH 服务状态：

```bash
sudo systemctl status sshd.service
```

---

## 测试 SSH 新端口连接

在本地尝试使用新端口连接服务器：

```bash
ssh root@服务器IP地址 -p 2244
```

确认可以成功登录后，如果之前是双端口开启，可以将 22 端口从配置以及防火墙规则中移除，以完成端口切换。

---

## 关闭旧端口 22（可选）

- 修改 `/etc/ssh/sshd_config` 注释掉或删除 `Port 22`，只保留新端口。

- 删除防火墙上的 22 端口放行规则：

```bash
sudo firewall-cmd --permanent --zone=public --remove-port=22/tcp
sudo firewall-cmd --reload
```

- 重启 SSH 服务。

---

## 阿里云服务器特殊说明：安全组配置

如果服务器部署于阿里云 ECS，除了系统防火墙，云平台安全组规则也需允许新 SSH 端口：

1. 登录阿里云控制台。
2. 找到对应实例的安全组配置。
3. 在“入方向”规则中添加 TCP 新端口（如 2244）。
4. 保存后生效。

该配置确保公网访问能够连接新端口。

---

## 更高级的 SSH 安全建议

仅仅更改端口，是“安全通过模糊”策略的开始，建议配合以下措施：

- **基于密钥的无密码登录**：生成 SSH 密钥对，只允许拥有密钥的主机登录，关闭密码登录。

  参考命令：

  ```bash
  ssh-keygen -t rsa -b 4096
  ssh-copy-id -p 2244 user@server
  ```

- **限制 IP 白名单**：通过防火墙只允许信任 IP 地址访问 SSH 端口。

- **禁用 root 直接登录**：修改 `/etc/ssh/sshd_config`，设置

  ```bash
  PermitRootLogin no
  ```

- **开启 Fail2Ban 等暴力破解防护工具**：自动禁止多次失败登录的 IP。

- **使用双因子认证（2FA）**：增加登录认证层级。

---

## 结语

更改 SSH 默认端口到非标准端口，是提高服务器安全的有效手段之一，但千万不要以此为唯一安全策略。结合 SSH 密钥认证、IP 限制、防火墙硬化和系统增强更能打造坚固的服务器安全防线。希望本文的步骤对您顺利完成 SSH 端口更改和安全强化有所帮助！

---

## 参考资料

- [OpenSSH 官方文档](https://www.openssh.com/manual.html)
- [CentOS 7 SELinux 基础](https://wiki.centos.org/HowTos/SELinux)
- [Firewalld 使用指南](https://firewalld.org/documentation/)