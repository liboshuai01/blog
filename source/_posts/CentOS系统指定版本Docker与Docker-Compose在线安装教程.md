---
title: CentOS系统指定版本Docker与Docker Compose在线安装教程
tags:
  - Linux
  - Centos
  - Docker
  - Docker-Compose
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425145545072.png'
toc: true
abbrlink: 987e17e9
date: 2024-05-15 14:51:33
---

本文将详细介绍如何在 CentOS 系统中，安装指定版本的 Docker 和 Docker Compose，保证环境的稳定与兼容。文中推荐的 Docker 版本为 **25.0.5**，Docker Compose 版本为 **2.22.0**，适合生产环境使用。

<!-- more -->

---

## 系统及环境要求

在安装之前，请确认以下基础要求：

- **CentOS 内核版本需高于 3.10**，通过执行以下命令查看：

  ```bash
  uname -r
  ```

  Docker 在较低内核版本系统上可能无法正常工作，请务必确保符合要求。

- 安装过程需要网络通畅，尤其是访问外网的 GitHub 用于下载 Docker Compose。

---

## 清理旧版本 Docker（如有）

如果系统中存在旧版本 Docker，为避免冲突建议先卸载：

```bash
sudo yum remove -y docker docker-common docker-selinux docker-engine docker-ce docker-ce-cli containerd.io
```

---

## 安装必备依赖包

为了保证 Docker 正常安装与运行，需先安装以下工具：

- `yum-utils`：提供 `yum-config-manager` 工具，用于管理 yum 源。
- `device-mapper-persistent-data` 和 `lvm2`：Docker 默认使用 devicemapper 存储驱动，依赖此两个包。

执行：

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

---

## 配置 Docker 官方阿里云镜像源

为了加速 Docker 及其组件的下载速度，推荐使用阿里云镜像源替代默认源：

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

配置完成后，可以通过以下命令验证可用的 Docker 版本列表：

```bash
sudo yum list docker-ce --showduplicates | sort -r
```

---

## 安装指定版本的 Docker

本文推荐安装稳定版本 Docker **25.0.5** ，包含核心组件 Docker Engine 和 Docker CLI，以及依赖的 containerd：

```bash
sudo yum install -y docker-ce-25.0.5 docker-ce-cli-25.0.5 containerd.io
```

安装完成后，启动 Docker 服务并设置开机自启：

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

查看 Docker 服务状态确保其正常运行：

```bash
sudo systemctl status docker
```

---

## 验证 Docker 安装结果

使用以下命令检查 Docker 信息，确认版本及各项配置均正常：

```bash
docker info
docker version
```

---

## 安装指定版本 Docker Compose

Docker Compose 是 Docker 应用容器编排的利器。这里安装推荐的 **v2.22.0** 版本。

> 说明：Docker Compose v2 由 Go 语言编写，提供独立的二进制文件，兼容性更强。

执行以下步骤：

1. 下载对应的二进制文件（Linux x86_64 架构）

   ```bash
   wget https://github.com/docker/compose/releases/download/v2.22.0/docker-compose-linux-x86_64
   ```

2. 移动文件到系统可执行路径，并赋予执行权限：

   ```bash
   sudo mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   ```

3. 验证安装：

   ```bash
   docker-compose --version
   ```

---

## 常见问题及优化建议

- **防火墙和 SELinux**

  如果遇到 Docker 启动异常，建议检查防火墙和 SELinux 配置，必要时临时关闭调试：

  ```bash
  sudo systemctl stop firewalld
  sudo setenforce 0
  ```

  生产环境请根据安全需求合理配置。

- **允许非 root 用户执行 Docker**

  为方便操作，可以将用户添加到 `docker` 组：

  ```bash
  sudo usermod -aG docker ${USER}
  ```

  添加后，需重新登录终端或使用 `newgrp docker` 生效。

- **使用镜像加速**

  除镜像源外，可以配置 Docker 守护进程使用镜像加速器，例如阿里云加速器，有助于大幅度提升拉取镜像速度。

---

## 总结

通过本文步骤，您可以在 CentOS 系统上顺利安装指定版本的 Docker 和 Docker Compose，并确保基础环境稳定。保持 Docker 版本及组件的适时升级，有助于提升容器服务的安全性与性能。

祝您部署顺利，容器化时代畅行无阻！