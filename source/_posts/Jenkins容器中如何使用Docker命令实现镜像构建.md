---
title: Jenkins容器中如何使用Docker命令实现镜像构建
tags:
  - Centos
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105440842.png'
toc: true
abbrlink: aaba5ea7
date: 2024-04-12 10:54:14
---

在基于 Jenkins 和 Docker 构建的 DevOps 流水线中，常见的需求是需要在 Jenkins 容器内部执行 Docker 命令，用于构建、打包镜像等操作。然而，由于 Jenkins 本身运行在容器中，Docker 环境也被隔离，直接调用 Docker 命令通常会失败。本文将重点介绍如何高效、安全地解决这一问题，实现容器内调用宿主机 Docker 引擎的最佳实践。

<!-- more -->

## 核心问题

Jenkins 容器内的 Docker 命令执行失败，主要源于如下原因：

- 容器内缺少 Docker 客户端二进制文件
- 容器无法访问宿主机的 Docker 守护进程（docker daemon）
- 访问 docker.sock 权限受限

所以，解决方案实际上是让 Jenkins 容器通过共享宿主机环境的方式，**调用宿主机的 Docker 引擎**，而不是在容器内独立运行 Docker。

## 完整的 Docker Compose 配置示例

以下是一个完整的示例`docker-compose.yaml`，展示如何部署可调用 Docker 的 Jenkins 容器：

```yaml
version: "3"
services:
  jenkins:
    image: jenkins/jenkins:2.452
    container_name: jenkins
    restart: on-failure:3
    ports:
      - 8020:8080            # Jenkins Web UI 端口映射
      - 50000:50000          # Jenkins agent 通信端口映射
    volumes:
      - jenkins_data:/var/jenkins_home       # Jenkins持久化数据卷
      - /etc/localtime:/etc/localtime:ro     # 保持时区同步
      - /var/run/docker.sock:/var/run/docker.sock   # 共享 Docker 守护进程套接字
      - /etc/docker:/etc/docker               # 共享 Docker CLI 配置（镜像加速等）
      - /usr/bin/docker:/usr/bin/docker       # 共享 Docker 客户端二进制文件
    group_add:
      - 128                                  # 赋予docker组权限，解决权限问题
    environment:
      TZ: Asia/Shanghai                       # 设置时区

volumes:
  jenkins_data:
    driver: local
```

## 配置细节解读

- **共享 Docker 套接字** `- /var/run/docker.sock:/var/run/docker.sock`

  容器通过挂载宿主机的 Docker 套接字（unix socket），能够和宿主机 Docker 守护进程直接通信。这样，容器只是一个客户端，Docker 实际运行在宿主机，避免了在容器内嵌套运行 Docker 的复杂问题。

- **共享 Docker 客户端** `- /usr/bin/docker:/usr/bin/docker`

  Jenkins 容器直接使用宿主机的 Docker 客户端二进制，无需额外安装，提高环境一致性。

- **共享 Docker 配置文件** `- /etc/docker:/etc/docker`

  挂载宿主机 Docker 配置目录，使容器内的 Docker 客户端能够读取镜像加速器、认证信息等配置，保证命令执行环境一致。

- **权限管理** `group_add: - 128`

  Docker 守护进程的 socket 文件默认属于 docker 组（UID一般为 128 或 999，因系统而异），通过在容器内为 Jenkins 用户添加相应组，解决访问`/var/run/docker.sock`的权限问题，避免“permission denied”异常。

- **时间同步** `- /etc/localtime:/etc/localtime:ro`

  保证 Jenkins 容器时间与宿主机一致，避免时间差导致的证书等问题。

## 权限问题的注意事项

- `group_add` 的值需要与宿主机 docker 组的 GID 一致。可以通过 `getent group docker` 命令查询。例如：

  ```bash
  getent group docker
  ```

  输出类似：

  ```
  docker:x:128:ubuntu
  ```

  其中 `128` 就是 docker 组的 GID。请根据实际情况调整。

- 另外，也可以直接将 Jenkins 容器用户 UID/GID 设置为与宿主机 docker 组匹配，或调整 socket 权限，但挂载 docker.sock 并添加组是最简洁且推荐的方法。

## 扩展说明

- 在流水线脚本中使用 Docker 命令时，无需任何特殊额外配置。因为 Jenkins 容器本身已经拥有对宿主机 Docker 引擎的访问权限。

- 示例流水线代码片段：

  ```groovy
  pipeline {
      agent any
      stages {
          stage('Build Docker Image') {
              steps {
                  sh 'docker build -t myapp:${BUILD_NUMBER} .'
                  sh 'docker push myapp:${BUILD_NUMBER}'
              }
          }
      }
  }
  ```

- 由于通过套接字共享的是宿主机 Docker 守护进程，所有操作都会在宿主机上执行，具有相同的权限范围，切记做好权限控制和安全管理。

## 总结

通过在 Jenkins 容器中挂载宿主机的 `docker.sock`，共享 Docker 客户端二进制及配置文件，并添加容器组权限，可以使 Jenkins 容器内顺利调用 Docker 命令，实现镜像的构建和管理。这种方式高效、简洁，已被广泛采纳，是使用 Jenkins + Docker 进行 CI/CD 自动化部署的最佳实践之一。

希望本文能帮助你快速解决在 Docker 部署 Jenkins 容器中使用 Docker 命令的难题，打造稳定便捷的 DevOps 流水线环境。