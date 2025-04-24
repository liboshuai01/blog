---
title: CentOS7服务器一键初始化配置脚本
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242143123.png'
toc: true
abbrlink: c8df43e1
date: 2024-03-23 21:42:31
---

在部署全新的 CentOS 7 服务器时，系统管理员往往需要进行一系列繁琐的基础配置工作，包括设置主机名、关闭安全策略、调整系统资源限制、安装常用工具及配置 Docker 环境等。这些手动操作不仅耗时，而且容易因疏忽导致配置错误。为此，本文提供了一份一键初始化脚本，帮助您快速完成基础环境搭建，保障服务器具备稳定、高效的运行基础。

<!-- more -->

## 脚本设计理念

- **自动化执行**：只需复制并运行脚本，减少重复劳动力
- **安全考虑**：需以 root 身份运行，脚本中包含权限检测
- **灵活变量配置**：关键参数如主机名和用户名可自定义修改
- **实用工具集成**：安装常用命令行工具，加快后续操作效率
- **Docker 环境完整部署**：引入阿里云镜像源，提高安装速度与稳定性
- **系统性能优化**：通过调整虚拟内存、文件描述符等系统参数提升性能

## 一键初始化脚本详解

将以下脚本复制到 CentOS 7 服务器终端，并以 root 账户执行即可。请根据您的实际需求，调整脚本开头的变量以适配不同环境。

```bash
cat > init-centos.sh << "EOF" && chmod +x init-centos.sh && ./init-centos.sh
#!/bin/bash

# 确保脚本以 root 身份执行
if [ "$(id -u)" -ne 0 ]; then
  echo "错误：该脚本需要以 root 身份运行。"
  exit 1
fi

#####################################
#           变量定义                 #
#####################################
# 设置新的主机名，请根据需求修改
NEW_HOSTNAME="docker"
# 指定需要配置免密 sudo 及 docker 命令权限的用户名
USERNAME=lbs

echo "开始设置主机名为 '$NEW_HOSTNAME'..."
hostnamectl set-hostname "$NEW_HOSTNAME"

echo "关闭 SELinux（切换为disabled）..."
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config || \
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo "禁用 Swap 以提升性能..."
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

echo "停止并禁用 firewalld 防火墙..."
systemctl disable firewalld.service
systemctl stop firewalld.service

echo "配置系统资源限制（文件描述符、进程数）..."
cat >> /etc/security/limits.conf <<EOL
* soft nofile 65536
* hard nofile 65536
* soft nproc  65536
* hard nproc  65536
EOL

cat > /etc/security/limits.d/20-nproc.conf <<EOL
* soft nproc  65536
* hard nproc  65536
EOL

echo "调整虚拟内存映射上限 (vm.max_map_count=2000000)..."
if ! grep -q "vm.max_map_count" /etc/sysctl.conf; then
  echo "vm.max_map_count=2000000" >> /etc/sysctl.conf
fi
sysctl -p

echo "安装常用基础工具包..."
yum install -y vim git curl wget unzip zip lrzsz net-tools epel-release tree gcc automake autoconf libtool make openssl yum-utils device-mapper-persistent-data lvm2

echo "配置并安装 Docker CE..."
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce-25.0.5 docker-ce-cli-25.0.5 containerd.io

echo "启动并开机自启 Docker 服务..."
systemctl enable docker
systemctl start docker

echo "创建 Docker 配置目录并生成 daemon.json 配置文件..."
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<-'EOT'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "info",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "data-root": "/var/lib/docker"
}
EOT

echo "设置用户 '$USERNAME' 免密 sudo 权限..."
if [ ! -f /etc/sudoers.d/${USERNAME} ]; then
  echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME}
  chmod 440 /etc/sudoers.d/${USERNAME}
fi

echo "将用户 '${USERNAME}' 添加到 docker 用户组，允许不使用 sudo 执行 docker 命令..."
usermod -aG docker ${USERNAME}

echo "初始化完成！为了确保所有设置生效，请重启系统。"
EOF
```

## 脚本使用指南

1. **变量调整**
    * `NEW_HOSTNAME`：修改为您服务器的目标主机名
    * `USERNAME`：输入需要配置权限的系统用户名（须已经存在）

2. **运行脚本**  
   以 root 用户登录服务器，将上面完整脚本复制粘贴回车执行。脚本将自动完成所有配置。

3. **系统重启**  
   脚本执行完成后，请务必重启服务器以应用所有内核和系统级参数变更。

## 脚本亮点说明

- **SELinux关闭**：大多数生产环境会因兼容性需求关闭 SELinux，避免权限问题。
- **Swap关闭**：容器化环境推荐禁用 Swap，优化内存分配。
- **资源限制提升**：大幅增加文件句柄和进程数，避免高负载时系统瓶颈。
- **Docker 官方仓库替换为阿里云镜像**：提升软件包下载速度和稳定性。
- **Docker配置优化**：采用 systemd cgroup 驱动，多线程下载限制，日志文件大小控制，保障 Docker 持续稳定运行。
- **权限管理**：为指定用户提供免密 sudo 及 docker 组权限，便于日常运维操作。

## 总结

通过执行这份自动化初始化脚本，您可以快速搭建一套既合规又高效的 CentOS 7 基础服务器环境，省去重复配置时间，并减少人为操作失误风险。欢迎根据自身环境对脚本进行个性化定制，打造符合团队标准的服务器模板。

祝您服务器管理顺利，开发、部署更高效！