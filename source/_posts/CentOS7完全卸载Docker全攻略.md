---
title: CentOS7完全卸载Docker全攻略
tags:
  - Linux
  - Centos
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250423112212150.png'
toc: true
abbrlink: b612e990
date: 2023-12-02 11:21:12
---

Docker 已成为现代云原生架构的核心组件，但在某些情况下，比如需要重新安装 Docker、升级版本或切换到其他容器管理工具时，彻底卸载 Docker 并清理相关数据成为必要步骤。本文将针对 CentOS 7 操作系统，详细梳理并扩展 Docker 卸载的完整流程，确保系统环境干净无残留。

<!-- more -->

---

## 卸载前的准备工作：确保容器和镜像不再使用

卸载 Docker 前，必须先确认没有任何运行中的容器和正在使用的 Docker 镜像，否则可能导致后续操作失败或数据残留。

### 1. 停止并杀死所有运行中的容器

终止所有正在运行的 Docker 容器：

```bash
docker ps -q | xargs -r docker kill
```

> **说明：** `docker ps -q` 会列出所有运行中容器的 ID，`xargs -r` 确保只有在有容器 ID 时才执行命令，避免报错。

### 2. 删除所有容器（包括已停止状态）

清理所有容器：

```bash
docker ps -a -q | xargs -r docker rm
```

### 3. 删除所有镜像

清除所有 Docker 镜像，释放磁盘空间：

```bash
docker images -q | xargs -r docker rmi -f
```

> **注意：** 使用 `-f` 参数强制删除，避免因镜像被依赖而删除失败

---

## 停止 Docker 服务及其相关套接字

卸载之前，要确保 Docker 服务完全停止，避免服务正在使用使卸载失败。

```bash
sudo systemctl stop docker.service
sudo systemctl stop docker.socket
```

执行后，可验证 Docker 服务状态，确认已停止：

```bash
sudo systemctl status docker.service
sudo systemctl status docker.socket
```

---

## 清理系统中的 Docker 目录和数据

Docker 存储数据的目录非常多，包括配置文件、运行时数据、日志以及容器镜像等。彻底清理这些目录，才能避免残留数据影响后续环境。

### 1. 删除 Docker 配置目录

```bash
sudo rm -rf /etc/docker
```

### 2. 删除运行时数据目录

```bash
sudo rm -rf /run/docker
```

### 3. 删除容器存储和镜像数据

```bash
sudo rm -rf /var/lib/docker
```

### 4. 删除可能存在的 Dockershim 目录（一般用于 Kubernetes 集成）

```bash
sudo rm -rf /var/lib/dockershim
```

### 5. 解除挂载占用（如有报错无法删除直到先卸载挂载点）

```bash
sudo umount /var/lib/docker/devicemapper
```

> **提示：** 使用 `df -h` 或 `mount | grep docker` 来查找被挂载的目录

---

## 四、卸载 Docker 相关软件包

CentOS 7 使用 `yum` 作为包管理工具，可以方便地查询和卸载已安装的 Docker 相关软件。

### 1. 查询已安装的 Docker 相关软件包

```bash
sudo yum list installed | grep -i docker
```

### 2. 卸载 Docker 相关软件包

执行卸载命令，一次清理所有相关的 Docker 组件及依赖软件：

```bash
sudo yum remove -y 'docker*'
sudo yum remove -y containerd.io.x86_64
```

> **说明：** 使用通配符 `'docker*'` 可以一并卸载所有以 docker 开头的软件包，`containerd.io` 是 Docker 的底层容器运行时，也需卸载。

---

## 验证卸载结果

卸载完成后，检查 Docker 是否已完全移除：

```bash
docker --version
```

如果输出类似 `command not found`，说明 Docker 已成功卸载。

此外，确认 Docker 服务是否仍在运行：

```bash
sudo systemctl status docker.service
```

正常情况下应显示未找到或停止状态。

---

## 小结

- **确保容器停止并全部删除**
- **删除所有镜像释放空间**
- **停用 Docker 服务，避免卸载冲突**
- **清理所有存储目录，防止残留数据**
- **卸载系统中所有 Docker 软件包**
- **验证卸载彻底，保持系统干净**

## 总结与建议

通过以上步骤，您已成功在 CentOS 7 系统上彻底卸载 Docker，并清理了所有容器、镜像和相关数据。这种深度清理确保后续重新安装或更换容器管理工具时，不会受到原有 Docker 环境的干扰。