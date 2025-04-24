---
title: Helm仓库与Chart快速管理指南
tags:
  - Linux
  - k8s
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242201173.png'
toc: true
abbrlink: 423ed5b3
date: 2024-03-27 22:00:28
---

在 Kubernetes 应用部署中，Helm 作为包管理工具，极大简化了复杂应用的安装与升级流程。本文将详细讲解 Helm 仓库的管理和 Chart 的基本操作，帮助你更高效、更稳定地使用 Helm。

<!-- more -->

## 查看已配置的 Helm 仓库列表

要查看当前系统中已经配置的所有 Helm 仓库，可以使用以下命令：

```bash
helm repo list
```

该命令会列出所有 Helm 仓库的名称及对应的地址，便于你了解当前可用的仓库资源。

## 删除默认仓库，加速国内访问

Helm 默认仓库通常位于国外，访问速度较慢。若不需要默认仓库，可以通过以下命令将其删除，防止安装或更新时出现超时问题：

```bash
helm repo remove stable
```

## 添加常用的国内 Helm 仓库

为了提升访问速度和稳定性，建议添加国内的镜像仓库。以下是几个常用且可靠的国内 Helm 仓库示例：

```bash
# 阿里云 Kubernetes 仓库
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

# Azure 中国区镜像仓库
helm repo add azure http://mirror.azure.cn/kubernetes/charts

# Bitnami 官方仓库（访问快，镜像优质）
helm repo add bitnami https://charts.bitnami.com/bitnami
```

你可以根据实际需求自定义仓库名称和地址，也可以添加更多符合企业网络环境的镜像源。

## 更新仓库索引数据

添加或删除仓库后，别忘了更新本地缓存，获取最新的 Chart 数据：

```bash
helm repo update
```

## 搜索特定的 Chart

要查找感兴趣的应用或服务的 Helm Chart，使用 `helm search repo` 命令，示例如下：

```bash
helm search repo redis
```

这将检索所有已配置仓库中包含 “redis” 关键字的 Chart 包，方便你挑选合适的版本。

## 下载 Chart 包到本地

如果希望先下载 Chart 包进行离线部署或二次修改，可以使用 `helm pull` 命令：

```bash
helm pull <chart-name> --version <version> --untar
```

- `<chart-name>`：Chart 全名，例如 bitnami/redis
- `--version`（可选）：指定下载的 Chart 版本号
- `--untar`（可选）：下载后自动解压

这一步有助于理解 Chart 结构，或在安全受控环境中进行安装。

## 安装 Helm Chart 到 Kubernetes 集群

完成准备后，可以使用如下命令安装 Chart 到集群：

```bash
helm install <release-name> <chart-name> --namespace <namespace> --create-namespace
```

- `<release-name>`：此次安装的实例名称，便于后续管理
- `<chart-name>`：待安装的 Chart，如 bitnami/redis
- `--namespace`：指定安装的命名空间，若没有则新建

此外，还可通过 `--set` 参数覆盖默认值，实现灵活配置。

## 卸载已安装的 Helm Release

当某个应用不再需要时，可以通过命令卸载对应的 Helm Release：

```bash
helm uninstall <release-name> --namespace <namespace>
```

这一操作会清理资源，保持环境整洁。

---

## 总结

掌握 Helm 仓库的增删改查和 Chart 安装卸载，是 Kubernetes 运维和开发人员必备的技能。使用国内镜像替换默认仓库，可以显著提升操作速度和稳定性。希望本篇操作指南能助你顺利开展 Helm 相关工作，实现高效的应用管理。