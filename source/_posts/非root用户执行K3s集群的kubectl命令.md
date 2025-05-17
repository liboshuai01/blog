---
title: 非root用户执行K3s集群的kubectl命令
tags:
  - Linux
  - Centos
  - Docker
  - K8s
  - k3s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505180144334.png'
toc: true
abbrlink: 96dde92b
date: 2025-05-18 01:42:59
---

在部署 K3s 轻量级 Kubernetes 集群后，默认生成的集群配置文件（kubeconfig）通常位于 `/etc/rancher/k3s/k3s.yaml`，且其所有者和权限设定通常只允许 root 用户访问。为了遵循安全最佳实践并在日常管理中使用普通用户执行 `kubectl` 命令与集群进行交互，我们需要进行一些配置。本文将详细介绍如何让非 root 用户安全、便捷地使用 `kubectl` 管理 K3s 集群。

<!-- more -->

---

## 背景与目的

K3s 默认安装时，会在服务器上生成一个 kubeconfig 文件，这个文件包含了连接和认证到集群所需的所有信息（API Server 地址、证书、密钥等）。出于安全考虑，这个文件 `/etc/rancher/k3s/k3s.yaml` 通常被设置为只有 root 用户可读。

直接使用 root 用户执行 `kubectl` 命令是不推荐的，这会带来不必要的权限风险。我们的目标是配置一个普通用户，让他能够读取这个 kubeconfig 文件，并让 `kubectl` 命令知道去哪里找到它，从而以该普通用户的身份安全地管理 K3s 集群。

---

## 操作步骤详解

下面介绍如何配置普通用户来执行 K3s 集群的 `kubectl` 命令。假设你的 K3s 集群已经由 root 用户安装并正常运行，并且你已经有一个用于管理的普通用户账号（例如：`youruser`）。

### 1. 确认 K3s Kubeconfig 文件位置及权限

首先，以 root 用户或通过 `sudo` 确认 K3s 的 kubeconfig 文件是否存在以及其当前的权限设置。

```bash
# 确保你有sudo权限或切换到root用户
sudo ls -l /etc/rancher/k3s/k3s.yaml
```

典型输出可能类似这样：
```
-rw------- 1 root root 6789 Jan 15 10:00 /etc/rancher/k3s/k3s.yaml
```
这表明文件 `/etc/rancher/k3s/k3s.yaml` 属于 `root` 用户和 `root` 组，并且只有 `root` 用户拥有读写权限 (`-rw-------`)。

### 2. 为普通用户创建 `.kube` 目录

`kubectl` 默认会在用户主目录下的 `.kube` 目录中查找名为 `config` 的 kubeconfig 文件 (`~/.kube/config`)。我们需要确保目标普通用户有这个目录。

切换到你的普通用户（或者使用 `sudo -u youruser` 来执行命令）：

```bash
# 切换到目标普通用户，例如：
# su - youruser

# 或者直接使用sudo执行mkdir命令
mkdir -p ~/.kube
```

`mkdir -p` 会创建目录以及任何必需的父目录，`-p` 选项可以防止目录已存在时报错。

### 3. 复制 K3s Kubeconfig 到用户主目录并修改权限

这是关键步骤。我们将`/etc/rancher/k3s/k3s.yaml` 文件复制到普通用户的 `~/.kube/config` 位置。由于源文件需要 root 权限读取，复制操作需要使用 `sudo`。复制完成后，文件的新副本将属于普通用户自己。

```bash
# 以sudo权限执行复制命令
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

# 确保用户拥有此文件
sudo chown $(id -u):$(id -g) ~/.kube/config

# 设定安全的权限，只允许文件所有者读写 (可选但推荐)
chmod 600 ~/.kube/config
```

解释：
*   `sudo cp ... ~/.kube/config`: 使用 `sudo` 将 root 拥有的文件复制到主目录下。复制后的文件所有者将是执行 `sudo cp` 的用户（通常如果你没切换用户，还是 root），所以需要下一步的 `chown`。
*   `sudo chown $(id -u):$(id -g) ~/.kube/config`: 将复制后的文件所有者更改为当前用户（普通用户）。`$(id -u)` 和 `$(id -g)` 会分别获取当前用户的用户ID和组ID。
*   `chmod 600 ~/.kube/config`: 设置文件权限为 `600`，意味着只有文件所有者有读写权限，其他任何用户都没有权限。这是 kubeconfig 文件的标准安全权限设置。

### 4. 配置 `kubectl` 使用复制的 kubeconfig 文件

`kubectl` 找到 kubeconfig 文件的方法有几种：

*   **默认位置:** 如果你将文件复制到了 `~/.kube/config`，`kubectl` 会自动检测到并使用它，无需额外配置。这是最推荐和最简洁的方式。
*   **`KUBECONFIG` 环境变量:** 你也可以设置 `KUBECONFIG` 环境变量来指定 kubeconfig 文件的路径。这在使用多个 kubeconfig 文件或文件不在默认位置时非常有用。

如果你已经将文件复制到 `~/.kube/config`，通常这一步可以跳过。但如果出于某种原因文件在其他位置，或者你想明确指定，可以在用户 shell 配置文件（如 `~/.bashrc`, `~/.zshrc` 等）中添加以下行：

```bash
# 编辑你的shell配置文件，比如 .bashrc
# vi ~/.bashrc

# 在文件末尾添加一行：
export KUBECONFIG=~/.kube/config

# 保存并退出编辑器

# 使配置立即生效
source ~/.bashrc
```

> **提示:** 如果你的shell不是 Bash，请修改对应的配置文件，如 `.zshrc` 或 `.profile`。

### 5. 验证配置是否成功

现在，以普通用户的身份执行 `kubectl` 命令来验证是否可以正常与 K3s 集群通信。

```bash
# 确保当前用户是非root用户
whoami

# 执行kubectl命令，例如获取节点列表
kubectl get nodes

# 执行另一个命令，例如查看集群信息
kubectl cluster-info
```

如果命令执行成功，显示集群节点信息或集群Endpoint信息，说明非 root 用户已经可以正常使用 `kubectl` 管理 K3s 集群了。

---

## 常见问题排查与建议

1.  **`kubectl` 命令未找到:** 确保 `kubectl` 命令已经安装并且在你的用户 PATH 环境变量中。K3s 通常会自动安装一个 `kubectl` 二进制文件到 `/usr/local/bin/kubectl`。验证下这个路径是否在你的用户 PATH 里，或者直接使用完整路径执行 `/usr/local/bin/kubectl ...`。
2.  **`Unable to connect to the server: dial tcp ... connect: connection refused`:** 这通常意味着 `kubectl` 找到了配置文件，但无法连接到 K3s API Server Endpoint。检查：
    *   K3s 服务是否正在运行 (`sudo systemctl status k3s`)。
    *   防火墙是否阻止了到 API Server 端口 (默认 6443) 的连接。
    *   `/etc/rancher/k3s/k3s.yaml` 中的 `server` 地址是否可达。
3.  **`error: error loading config file "~/.kube/config": open /home/youruser/.kube/config: permission denied`:** 这种错误表明 `kubectl` 尝试读取 `~/.kube/config` 文件时遇到权限问题。请回到步骤 3，确保 `~/.kube/config` 文件所有者是当前用户，并且权限设置允许用户读取 (如 `chmod 600` 或 `chmod 644`)。
4.  **`ls -l ~/.kube/config` 显示文件属于 root:** 回到步骤 3，确认 `sudo chown $(id -u):$(id -g) ~/.kube/config` 命令是否正确执行，将文件所有权赋给了当前用户。
5.  **复制的 kubeconfig 文件内容过期或不正确:** 如果 K3s 集群配置发生变化（例如证书轮换），你可能需要重新同步或更新 `~/.kube/config` 文件。最简单的方法是再次执行步骤 3 的复制命令。

---

## 总结

通过将 K3s 生成的 kubeconfig 文件安全地复制到普通用户主目录下的标准位置 (`~/.kube/config`)，并确保文件的所有权和权限正确设置，我们可以让非 root 用户无须特殊权限即可执行 `kubectl` 命令，从而实现集群的安全管理。这是一个在生产环境中管理 K3s 或其他 Kubernetes 发行版时非常重要的最佳实践。

希望这篇教程对你有所帮助！欢迎在评论区分享你的经验或遇到的问题。