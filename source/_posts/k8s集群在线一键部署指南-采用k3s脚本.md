---
title: k8s集群在线一键部署指南（采用k3s脚本）
tags:
  - 环境搭建
  - Linux
  - K8s
  - Rancher
  - K3s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250411142201678.png'
toc: true
abbrlink: 617fd45
date: 2025-04-11 14:24:00
---

K3s 是由 Rancher Labs（现为 SUSE 的一部分）精心打造的一款轻量级 Kubernetes 发行版。它专为在资源受限的环境中运行 Kubernetes 而设计，例如边缘计算、物联网（IoT）设备、CI/CD 系统以及开发测试场景。K3s 通过移除和替换一些非核心组件（如 etcd 替换为 SQLite，移除内置的云提供商插件等），并将其打包成一个小于 100MB 的二进制文件，极大地简化了 Kubernetes 的部署和运维复杂性。

尽管轻量，K3s 仍然是一个完全符合标准的、生产级别的 Kubernetes 发行版。它支持标准的 Kubernetes API，这意味着您可以无缝迁移现有的 Kubernetes 应用，并充分利用庞大的 Kubernetes 生态系统。本文将引导您完成 K3s 集群的一键式在线部署，并介绍如何进行卸载。

<!-- more -->

## 核心优势

*   **轻量高效**：极低的资源占用，二进制文件小，启动速度快。
*   **安装便捷**：一条命令即可完成单节点或多节点集群的部署。
*   **完全兼容**：遵循 Kubernetes API 标准，与 kubectl、Helm 等工具无缝协作。
*   **内置组件**：集成了 Traefik Ingress Controller、CoreDNS、Metrics Server 等常用组件，开箱即用。
*   **多种架构支持**：支持 x86_64, ARM64, 和 ARMv7 等多种体系架构。

## 一、环境与准备工作

在开始部署之前，请确保您的环境满足以下基本要求：

### 1. 操作系统推荐

*   **Linux 发行版**：
   *   红帽系列 (CentOS, RHEL, Fedora)
   *   Debian/Ubuntu 系列
   *   Raspberry Pi OS (树莓派)
*   **内核版本**：建议 Linux 内核版本 3.10 或更高。

### 2. 硬件推荐

| 组件 | 最低要求 | 推荐配置 |
| ---- | -------- | -------- |
| CPU  | 1 核心   | 2 核心+  |
| 内存 | 512MB    | 1GB+     |

### 3. 网络配置

*   **端口开放**：
   *   Master 节点：需要确保 `6443` (Kubernetes API Server) 端口可以被 Worker 节点访问。
   *   Flannel VXLAN：默认使用 UDP `8472` 端口。
   *   如果使用其他CNI，请参考其端口要求。
*   **主机名解析**：确保集群中所有节点之间可以通过主机名相互访问。如果您的环境没有配置 DNS 服务器，建议在所有节点的 `/etc/hosts` 文件中添加主机名与 IP 地址的映射关系。

### 4. 用户权限

所有安装操作建议使用 `root` 用户执行，或具有 `sudo` 权限的用户。

## 二、集群规划

本文将以一个 Master 节点和两个 Worker 节点的集群为例。

| 主机名    | IP 地址 (示例) | 角色                   | CPU | 内存  |
| :-------- | :------------- | :--------------------- | :-- | :---- |
| `k3s-master` | `192.168.1.100` | `control-plane`, `master` | 2   | 4GB   |
| `k3s-node1`  | `192.168.1.101` | `worker`               | 2   | 4GB   |
| `k3s-node2`  | `192.168.1.102` | `worker`               | 2   | 4GB   |

> **重要提示**：请将上述表格中的主机名和 IP 地址替换为您环境中的实际值。

## 三、部署步骤

> **注意**：以下所有操作步骤均以 `root` 用户执行为例。如果使用非 root 用户，请在命令前添加 `sudo`。

### 1. 系统预配置 (所有节点)

在 `k3s-master`、`k3s-node1`、`k3s-node2` 所有节点上执行以下操作：

#### a. 关闭防火墙（简化演示，生产环境请配置精确规则）

```shell
systemctl stop firewalld
systemctl disable firewalld
# 对于 Ubuntu/Debian 系统
# ufw disable
```

#### b. 配置主机名解析（如果无DNS）

编辑 `/etc/hosts` 文件，添加如下内容（请根据您的实际 IP 和主机名修改）：

```shell
# 例如:
# nano /etc/hosts

192.168.1.100 k3s-master
192.168.1.101 k3s-node1
192.168.1.102 k3s-node2
```

#### c. 确保所有节点时间同步

可以使用 `ntpdate` 或 `chrony` 进行时间同步，这对于 Kubernetes 集群的稳定性至关重要。

### 2. 安装 K3s Master 节点

在 `k3s-master` 节点上执行以下命令安装 K3s 服务：

```shell
# 官方安装脚本
curl -sfL https://get.k3s.io | sh -

# 中国大陆用户，可以使用以下镜像加速安装：
# curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
```

安装完成后，K3s 服务会自动启动。您可以检查服务状态：

```shell
systemctl status k3s
```

### 3. 获取 Master 节点的 Token

Master 节点安装成功后，会生成一个 Token，用于 Worker 节点加入集群。在 `k3s-master` 节点上执行以下命令查看 Token：

```shell
[root@k3s-master ~]# cat /var/lib/rancher/k3s/server/node-token
K10xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx::server:yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
```
记下这个 Token 值，后续 Worker 节点加入时会用到。

### 4. 安装 K3s Worker 节点并加入集群

分别在 `k3s-node1` 和 `k3s-node2` 节点上执行以下命令。请将 `YOUR_MASTER_IP_OR_HOSTNAME` 替换为 Master 节点的实际 IP 地址或主机名 (例如 `k3s-master` 或 `192.168.1.100`)，并将 `YOUR_NODE_TOKEN` 替换为上一步获取到的 Token。

```shell
# 将 YOUR_MASTER_IP_OR_HOSTNAME 和 YOUR_NODE_TOKEN 替换为实际值
curl -sfL https://get.k3s.io | K3S_URL=https://YOUR_MASTER_IP_OR_HOSTNAME:6443 K3S_TOKEN=YOUR_NODE_TOKEN sh -

# 中国大陆用户，可以使用以下镜像加速安装：
# curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://YOUR_MASTER_IP_OR_HOSTNAME:6443 K3S_TOKEN=YOUR_NODE_TOKEN sh -
```

例如，对于 `k3s-node1`，如果 Master IP 是 `192.168.1.100`，Token 是 `K10...::server:yyy...`：
```shell
# 示例:
# curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.100:6443 K3S_TOKEN=K10xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx::server:yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy sh -
```

安装完成后，K3s Agent 服务会自动启动。您可以检查服务状态：

```shell
systemctl status k3s-agent
```

### 5. 验证集群状态

在 `k3s-master` 节点上执行以下命令，查看集群节点状态：

```shell
[root@k3s-master ~]# kubectl get nodes -o wide
NAME         STATUS   ROLES                  AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
k3s-master   Ready    control-plane,master   10m   v1.28.7+k3s1  192.168.1.100   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.7.11-k3s2
k3s-node1    Ready    <none>                 5m    v1.28.7+k3s1  192.168.1.101   <none>        Ubuntu 20.04.3 LTS      5.4.0-150-generic       containerd://1.7.11-k3s2
k3s-node2    Ready    <none>                 5m    v1.28.7+k3s1  192.168.1.102   <none>        Ubuntu 20.04.3 LTS      5.4.0-150-generic       containerd://1.7.11-k3s2
```
如果所有节点都显示 `Ready` 状态，说明集群已成功部署。
> 注意：`VERSION`、`OS-IMAGE`、`KERNEL-VERSION`等信息会根据您的实际环境有所不同。

### 6. 配置 `kubectl` (Master 节点)

K3s 会自动在 Master 节点的 `/etc/rancher/k3s/k3s.yaml` 生成 kubeconfig 文件。`kubectl` 命令也会自动配置好，可以直接使用。
为了方便 Helm 等其他工具识别，或者在非 root 用户下使用 `kubectl`，可以将 kubeconfig 路径添加到环境变量：

```shell
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bashrc
source ~/.bashrc
```
这样，您就可以在 Master 节点上直接使用 `kubectl` 命令管理集群了。

如果您想从其他机器远程管理 K3s 集群，可以将 `/etc/rancher/k3s/k3s.yaml` 文件复制到远程机器的 `~/.kube/config`，并修改其中的 `server` 地址为 Master 节点的外部可访问 IP 或域名。

### 7. 为 Worker 节点打上角色标签 (可选)

默认情况下，Worker 节点的 `ROLES` 列可能显示为 `<none>`。您可以为它们打上 `worker` 标签，以更清晰地标识其角色：

在 `k3s-master` 节点上执行：
```shell
[root@k3s-master ~]# kubectl label node k3s-node1 node-role.kubernetes.io/worker=worker
node/k3s-node1 labeled

[root@k3s-master ~]# kubectl label node k3s-node2 node-role.kubernetes.io/worker=worker
node/k3s-node2 labeled
```

再次查看节点状态：
```shell
[root@k3s-master ~]# kubectl get nodes -o wide
NAME         STATUS   ROLES                  AGE   VERSION        INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION          CONTAINER-RUNTIME
k3s-master   Ready    control-plane,master   15m   v1.28.7+k3s1  192.168.1.100   <none>        CentOS Linux 7 (Core)   3.10.0-1160.el7.x86_64   containerd://1.7.11-k3s2
k3s-node1    Ready    worker                 10m   v1.28.7+k3s1  192.168.1.101   <none>        Ubuntu 20.04.3 LTS      5.4.0-150-generic       containerd://1.7.11-k3s2
k3s-node2    Ready    worker                 10m   v1.28.7+k3s1  192.168.1.102   <none>        Ubuntu 20.04.3 LTS      5.4.0-150-generic       containerd://1.7.11-k3s2
```
现在 `ROLES` 列已经正确显示为 `worker`。

## 四、卸载 K3s 集群

如果您需要卸载 K3s 集群，K3s 提供了便捷的卸载脚本。

### 1. 卸载 Worker 节点

在每个 Worker 节点（例如 `k3s-node1`、`k3s-node2`）上执行以下命令：

```shell
/usr/local/bin/k3s-agent-uninstall.sh
```
该脚本会停止 K3s Agent 服务，并清理相关文件和目录。

### 2. 卸载 Master 节点

在 Master 节点（例如 `k3s-master`）上执行以下命令：

```shell
/usr/local/bin/k3s-uninstall.sh
```
该脚本会停止 K3s Server 服务，并清理相关文件、目录和数据（包括 `/var/lib/rancher/k3s` 目录）。

> **警告**：卸载 Master 节点将删除所有集群数据，包括 etcd (或 SQLite) 中的状态、证书等。请务必在操作前备份重要数据。

### 3. 清理残留 (可选)

卸载脚本通常能很好地清理 K3s 相关内容。但您也可以手动检查并清理任何可能的残留，例如：
*   iptables 规则 (如果 K3s 未完全清除)
*   CNI 创建的网络接口
*   `/run/k3s`、`/run/flannel` 等临时目录

## 五、总结与展望

通过本文的指导，您已经成功部署了一个轻量级的 K3s Kubernetes 集群，并了解了如何进行卸载。K3s 的简洁性和高效性使其成为各种场景下运行 Kubernetes 的绝佳选择。

接下来，您可以开始探索：
*   使用 `kubectl` 部署和管理应用程序。
*   通过 Helm Chart 部署复杂应用。
*   配置持久化存储 (PersistentVolume, PersistentVolumeClaim)。
*   探索 K3s 内置的 Traefik Ingress Controller 进行服务暴露。
*   集成监控和日志系统，如 Prometheus, Grafana, ELK/EFK Stack。

K3s 大大降低了 Kubernetes 的入门门槛，让开发者和运维人员能够更专注于应用本身，而非繁琐的集群搭建与维护。希望这篇指南能帮助您快速上手 K3s，开启您的云原生之旅！

---