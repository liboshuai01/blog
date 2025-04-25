---
title: Kubernetes上通过Helm部署高可用Redis集群
tags:
  - 环境搭建
  - Linux
  - k8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425103751734.png'
toc: true
abbrlink: ae73bd3a
date: 2025-04-25 10:37:02
---

本文详细介绍如何利用 Helm 在 Kubernetes（K8s）集群中部署一个基于 Redis 3 主 3 从架构的高可用分布式缓存集群。部署过程依赖 Kubernetes 集群和 Helm 工具，且预设已配置基于 NFS 的 StorageClass 用于持久化存储。

<!-- more -->

---

## 环境准备

- Kubernetes 集群（版本 1.16+ 推荐）
- Helm 客户端（版本 3.x）
- 预先配置好的 NFS StorageClass，用于动态申请持久卷

> **提示**  
> 若尚未配置 NFS StorageClass，可参考相关文章快速部署 [Helm 安装 nfs-client-provisioner](https://juejin.cn/spost/7353547774174806031)。

---

## 添加 Bitnami Helm 仓库

Bitnami 提供了稳定且社区认可的 Redis 集群 Helm Chart，首先添加该仓库：

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

可用 Helm Chart 版本及描述可通过如下命令查看：

```bash
helm search repo redis
```

---

## 下载并准备 Helm Chart

为了自定义配置，建议将 Helm Chart 下载到本地进行修改：

```bash
mkdir -p ~/redis && cd ~/redis

helm pull bitnami/redis-cluster --version 7.5.0
tar -xvf redis-cluster-7.5.0.tgz

cp redis-cluster/values.yaml ./values-custom.yaml
```

目录结构示例：

```text
.
├── redis-cluster
│   ├── Chart.yaml
│   ├── templates
│   └── values.yaml
├── redis-cluster-7.5.0.tgz
└── values-custom.yaml
```

---

## 配置持久化存储和密码

确认集群中可用的 StorageClass：

```bash
kubectl get storageclasses
```

输出示例：

```text
NAME                          PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs (default)                 nfs-provisioner-01   Delete          Immediate           false                  14d
```

将配置文件 `values-custom.yaml` 修改为使用 `nfs` 作为默认 StorageClass，并设置 Redis 访问密码：

```yaml
global:
  storageClass: "nfs"     # 指定持久卷存储Class为nfs
  redis:
    password: "redis123"  # Redis集群访问密码，自定义密码建议设置强密码
```

---

## 创建 Kubernetes 命名空间

建议将 Redis 集群部署在专门的命名空间中，便于运维管理：

```bash
kubectl create ns mynamespace
```

---

## 使用 Helm 安装 Redis 集群

以自定义参数部署 Redis 3 主 3 从集群：

```bash
helm -n mynamespace install redis-cluster ./redis-cluster -f values-custom.yaml
```

- `-n mynamespace` 指定安装命名空间
- `install redis-cluster` 表示安装命名为 redis-cluster 的实例
- `-f values-custom.yaml` 使用自定义配置覆盖默认配置

---

## 验证集群部署状态

查看 Helm 部署状态：

```bash
helm -n mynamespace list
```

示例输出：

```text
NAME          	NAMESPACE     	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
redis-cluster 	mynamespace	1       	2022-05-06 20:46:11 +0800 CST          	deployed	redis-cluster-7.5.0 	6.2.6 
```

查看 Pod 运行状态，确认 6 个 Redis 实例（3 主 3 从）均已运行：

```bash
kubectl -n mynamespace get pods -l app.kubernetes.io/name=redis-cluster
```

示例输出：

```text
NAME              READY   STATUS    RESTARTS   AGE
redis-cluster-0   1/1     Running   0          10m
redis-cluster-1   1/1     Running   0          10m
redis-cluster-2   1/1     Running   0          10m
redis-cluster-3   1/1     Running   0          10m
redis-cluster-4   1/1     Running   0          10m
redis-cluster-5   1/1     Running   0          10m
```

Redis 集群节点分布说明：

- 主节点：redis-cluster-0，redis-cluster-2，redis-cluster-4
- 从节点：redis-cluster-1，redis-cluster-3，redis-cluster-5

---

## 访问和验证 Redis 集群

1. 获取 Redis 密码，存入环境变量：

```bash
export REDIS_PASSWORD=$(kubectl get secret --namespace mynamespace redis-cluster -o jsonpath="{.data.redis-password}" | base64 --decode)
```

2. 启动临时 Pod 连接 Redis 集群：

```bash
kubectl run -n mynamespace redis-client --rm -i --tty \
--env REDIS_PASSWORD=$REDIS_PASSWORD \
--image docker.io/bitnami/redis-cluster:6.2.6-debian-10-r193 \
-- bash
```

3. 使用 redis-cli 连接集群：

```bash
redis-cli -c -h redis-cluster -a $REDIS_PASSWORD
```

4. 执行集群状态命令验证：

```bash
> cluster info
> cluster nodes
```

---

## 集群访问示例

在 Kubernetes 集群内，使用 DNS 名称可访问集群各节点：

```bash
ping redis-cluster-0.redis-cluster.mynamespace
ping redis-cluster-1.redis-cluster.mynamespace
...
ping redis-cluster-5.redis-cluster.mynamespace
```

客户端访问时，如使用 Redis Cluster 模式客户端，使用服务名 `redis-cluster.mynamespace.svc.cluster.local` 即可访问集群主服务。

---

## 常见问题及建议

- **持久化存储失败**：确认 StorageClass 名称正确无误，同时 PVC 状态为 Bound。
- **Redis 密码错误**：通过 Secret 管理密码，避免硬编码风险，确保客户端密码一致。
- **网络策略和防火墙**：确保 Pod 间通信和服务访问权限正确开通。
- **版本兼容性**：使用的 Redis 和 Helm Chart 版本应相互兼容，避免因版本差异导致异常。
- **监控与日志**：部署监控工具（如 Prometheus）和日志收集方案，有助于及时发现问题。

---

## 总结

本文示范了基于 Helm 的 Bitnami Redis Cluster Chart，在 Kubernetes 集群上快速部署一个 3 主 3 从的 Redis 高可用集群，结合 NFS 持久化存储，保障数据持久性及系统的稳定性。通过合理的配置和验证步骤，助力运维人员高效搭建 Redis 缓存服务，提高业务可用性和性能。

欢迎根据业务需求，灵活调整副本数、资源配置和安全设置，实现个性化 Redis 集群方案。

---

**相关阅读推荐**

- [Kubernetes Persistent Volume (PV) 和 Persistent Volume Claim (PVC) 详解](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)
- [Helm 入门与高级用法指南](https://helm.sh/docs/)
- [Redis Cluster 原理与最佳实践](https://redis.io/docs/manual/scaling/)

---

> 以上内容，欢迎点赞和分享，助力更多开发者无痛上手 Kubernetes 及 Redis 集群部署！