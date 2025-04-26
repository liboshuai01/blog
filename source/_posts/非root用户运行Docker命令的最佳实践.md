---
title: 非root用户运行Docker命令的最佳实践
tags:
  - Linux
  - Centos
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232345973.png'
toc: true
abbrlink: 128954f4
date: 2022-12-07 23:43:18
---

在企业环境中，为了保证系统安全和权限的合理分配，通常不允许直接使用 root 用户执行 Docker 命令。本文将介绍如何配置普通用户，使其能够无须 root 权限便可执行 Docker 命令，从而提升安全性和操作便捷性。

<!-- more -->

---

## 背景与目的

默认情况下，Docker 进程的管理权限绑定在 `root` 用户上。直接使用 root 用户执行 Docker 命令存在安全隐患，不利于权限控制和审计。为解决此问题，推荐方式是将需要执行 Docker 命令的普通用户添加到 Docker 用户组（一般为 `docker` 组），使其拥有对 Docker.sock 的访问权限，从而无须提升权限即可操作 Docker。

---

## 操作步骤详解

下面详细说明如何配置非 root 用户执行 Docker 命令。

### 1. 确认或创建 `docker` 用户组

Docker 的套接字文件 `/var/run/docker.sock` 默认会被分配给 `docker` 用户组。如果该组不存在，需要手动创建。

```bash
# 查看系统是否存在docker用户组
getent group docker

# 如果没有输出，说明docker组不存在，执行下面命令创建docker组
sudo groupadd docker
```

> **说明：** 使用 `getent group docker` 代替 `cat /etc/group | grep docker`，更为高效且语言无关。

### 2. 检查并修改 Docker 套接字的用户组归属

由于 Docker 的套接字默认位于 `/var/run/docker.sock`，确认该文件所属用户组是否是 `docker`，如果不是，需要修改。

```bash
# 查看docker.sock的权限和所属用户组
ls -l /var/run/docker.sock

# 示例如下，输出类似：
# srw-rw---- 1 root docker 0 Jun  1 10:00 /var/run/docker.sock

# 若所属组不是docker，则执行：
sudo chgrp docker /var/run/docker.sock
```

> **建议：** Docker 服务重启后，套接字权限通常会自动恢复，若遇到权限变化问题，可考虑使用 `docker.service` 的配置覆盖。

### 3. 将目标用户添加到 `docker` 用户组

将需要执行 Docker 命令的普通用户添加到 `docker` 组。

```bash
# 以当前用户为例
sudo usermod -aG docker ${USER}
```

> **注意：** `-aG` 表示向附加组中添加用户，避免覆盖已有组。

### 4. 重新加载用户组权证（使配置即时生效）

为了立即使用户组权限生效，可以选择以下两种方式：

- 重新登录用户会话，注销并重新登录
- 或执行 `newgrp docker` 切换当前 shell 的组权限

```bash
newgrp docker
```

---

## 效果与验证

完成配置后，可以通过如下命令验证用户的组权限及 Docker 命令执行能力。

### 1. 查看用户所属组

```bash
groups ${USER}
# 输出中应包含docker组
```

### 2. 测试执行 docker 命令

```bash
docker ps
```

若没有产生权限错误，且正常显示容器列表，说明配置成功。

---

## 常见问题排查与建议

1. **权限不足报错**  
   确认用户是否添加到 `docker` 用户组，并重新登录 Session。

2. **Docker 服务未启动**  
   运行 `sudo systemctl status docker`，确认 Docker 正常运行。

3. **docker.sock 权限会重置**  
   某些系统中，重启 Docker 服务后 `/var/run/docker.sock` 的权限会被重置。可通过创建 systemd override 文件固定权限：

   ```bash
   sudo mkdir -p /etc/systemd/system/docker.service.d
   sudo tee /etc/systemd/system/docker.service.d/override.conf <<EOF
   [Service]
   ExecStartPost=/bin/chgrp docker /var/run/docker.sock
   ExecStartPost=/bin/chmod 660 /var/run/docker.sock
   EOF
   
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

4. **安全建议**  
   将用户添加至 `docker` 组等同于给予 root 权限，请仅对可信用户配置此权限，避免安全风险。

---

## 总结

通过将普通用户加入 `docker` 用户组，可以避免使用 root 身份执行 Docker 命令，提升系统安全性和运维便利性。掌握以上操作步骤和注意事项，有助于企业级 Docker 运行环境的规范管理。

欢迎大家留言反馈或分享您在权限管理上的经验和问题，推动团队踊跃交流和实践！