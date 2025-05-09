---
title: K8s采用Helm部署高可用Redis集群实战指南
tags:
  - Linux
  - k8s
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250509140551459.png'
toc: true
abbrlink: 5c36b781
date: 2025-05-09 13:59:31
---


本文将引导您使用 Helm 在 Kubernetes (K8s) 集群中，快速部署一个基于 Redis 3主3从架构的高可用分布式缓存集群。此部署方案依赖于现有的 Kubernetes 集群、Helm 客户端，并预设已配置基于 NFS 的 StorageClass 以实现持久化存储。

<!-- more -->

---

### 🚀 引言：为何选择高可用 Redis 集群？

在现代 Web 应用中，缓存是提升性能、降低数据库负载的关键组件。Redis 以其高性能和丰富的数据结构成为缓存首选。然而，单点 Redis 存在可用性风险。通过部署 Redis 主从集群，可以实现数据冗余和故障自动切换，确保服务的高可用性，为您的应用提供稳定可靠的缓存服务。

---

### 🛠️ 环境准备 (Prerequisites)

在开始之前，请确保您的环境满足以下条件：

1.  **Kubernetes 集群**: 版本 1.16+ 推荐。确保 `kubectl` 已配置并能正常访问集群。
2.  **Helm 客户端**: 版本 3.x。Helm 是 Kubernetes 的包管理器，极大简化了应用部署。
3.  **NFS StorageClass**: 预先配置好的、可动态申请持久卷 (PV) 的 StorageClass。Redis 数据将持久化到 NFS 存储中。

> **💡 提示**
> 如果您尚未配置 NFS StorageClass，可以参考官方文档或相关教程进行部署。例如，[Kubernetes使用Helm部署NFS-Client-Provisioner实现动态存储](https://liboshuai.icu/pages/e3673e0e/) 这篇文章提供了很好的指导。动态存储配置是实现数据持久化的关键。

---

### ⚙️核心部署步骤

#### 1. 添加 Bitnami Helm 仓库

Bitnami 提供了大量维护良好且社区广泛认可的 Helm Chart。我们将使用其 Redis Cluster Chart。

```bash
# 添加 Bitnami 仓库
helm repo add bitnami https://charts.bitnami.com/bitnami

# 更新本地 Helm 仓库索引
helm repo update

# 查看可用的 redis-cluster chart 版本
helm search repo bitnami/redis-cluster
```
您会看到类似如下的输出，选择一个合适的版本（本文以 `7.5.0` 为例）：
```text
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/redis-cluster   7.5.0           6.2.6           Redis(R) Cluster is a data structure server. It ...
...
```

#### 2. 下载并自定义 Helm Chart

为了更灵活地配置 Redis 集群，建议将 Chart 下载到本地进行修改。

```bash
# 创建工作目录
mkdir -p ~/kube-deploy/redis-cluster && cd ~/kube-deploy/redis-cluster

# 下载指定版本的 Chart (请替换为您选择的版本)
helm pull bitnami/redis-cluster --version 7.5.0

# 解压 Chart 包
tar -xvf redis-cluster-7.5.0.tgz

# 复制默认的 values.yaml 文件，用于自定义配置
cp ./redis-cluster/values.yaml ./values-custom.yaml
```

此时，您的目录结构应如下所示：

```text
.
├── redis-cluster          # 解压后的 Chart 目录
│   ├── Chart.yaml
│   ├── templates
│   ├── values.yaml        # 默认配置文件
│   └── ...
├── redis-cluster-7.5.0.tgz # 下载的 Chart 压缩包
└── values-custom.yaml     # 我们将修改此文件
```

#### 3. 定制 `values-custom.yaml`

首先，确认集群中可用的 StorageClass 名称：

```bash
kubectl get storageclasses
```

输出示例：
```text
NAME                    PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path              rancher.io/local-path                           Delete          WaitForFirstConsumer   false                  3d4h
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate              true                   3d3h
managed-premium         disk.csi.azure.com                              Delete          WaitForFirstConsumer   true                   120d
```
编辑 `values-custom.yaml` 文件，指定持久化存储类并设置 Redis 访问密码。

```yaml
# values-custom.yaml

# 全局配置
global:
  storageClass: "nfs-storage"     # 确保 "nfs-storage" 是您环境中可用的 StorageClass
  redis:
    password: "YourStrongPassword123!"  # 设置 Redis 集群访问密码，务必使用强密码

# (可选) 资源配置示例，根据实际需求调整
# master:
#   persistence:
#     size: 10Gi
#   resources:
#     requests:
#       cpu: "500m"
#       memory: "512Mi"
#     limits:
#       cpu: "1"
#       memory: "1Gi"
# slave:
#   persistence:
#     size: 10Gi
#   resources:
#     requests:
#       cpu: "250m"
#       memory: "256Mi"
#     limits:
#       cpu: "500m"
#       memory: "512Mi"

# 默认会部署 3 主 3 从。如果需要调整节点数，可以修改以下配置：
# cluster:
#   nodes: 6
#   replicas: 1 # 每个主节点的从节点数量
```
> **🛡️ 安全提示**
> 请务必将 `YourStrongPassword123!` 替换为一个复杂且唯一的强密码。避免使用弱密码。

#### 4. 创建 Kubernetes 命名空间

为 Redis 集群创建一个专用的命名空间，有助于资源隔离和管理。

```bash
kubectl create namespace redis-ha
```
(本文后续命令将使用 `redis-ha` 作为命名空间)

#### 5. 部署 Redis 集群

使用 Helm install 命令，并指定自定义的 `values-custom.yaml` 文件进行部署。

```bash
helm install redis-cluster ./redis-cluster \
  --namespace redis-ha \
  -f ./values-custom.yaml
```
参数说明：
*   `redis-cluster`: Helm Release 的名称，可以自定义。
*   `./redis-cluster`: 本地 Chart 的路径。
*   `--namespace redis-ha`: 指定部署到的命名空间。
*   `-f ./values-custom.yaml`: 使用我们自定义的配置文件。

---

### 🔎 验证与访问

#### 1. 检查 Helm 部署状态

```bash
helm list --namespace redis-ha
```
您应该能看到类似输出，`STATUS` 应为 `deployed`：
```text
NAME            NAMESPACE   REVISION    UPDATED                                 STATUS      CHART                   APP VERSION
redis-cluster   redis-ha    1           2023-10-27 10:30:00 EST                 deployed    redis-cluster-7.5.0     6.2.6
```

#### 2. 检查 Pod 运行状态

默认配置下，会创建 6 个 Redis Pod（3 主 3 从）。

```bash
kubectl get pods --namespace redis-ha -l app.kubernetes.io/name=redis-cluster -w
```
等待所有 Pod 的 `STATUS` 变为 `Running`，并且 `READY` 状态为 `1/1`。
```text
NAME              READY   STATUS    RESTARTS   AGE
redis-cluster-0   1/1     Running   0          5m
redis-cluster-1   1/1     Running   0          5m
redis-cluster-2   1/1     Running   0          5m
redis-cluster-3   1/1     Running   0          5m
redis-cluster-4   1/1     Running   0          5m
redis-cluster-5   1/1     Running   0          5m
```
按 `Ctrl+C` 退出 `-w` (watch) 模式。

*   **主节点 (Masters)**: 通常是 `redis-cluster-0`, `redis-cluster-2`, `redis-cluster-4`
*   **从节点 (Slaves/Replicas)**: 通常是 `redis-cluster-1`, `redis-cluster-3`, `redis-cluster-5` (它们分别对应一个主节点)

#### 3. 连接及测试集群

首先，获取之前设置的 Redis 密码：

```bash
export REDIS_PASSWORD=$(kubectl get secret --namespace redis-ha redis-cluster -o jsonpath="{.data.redis-password}" | base64 --decode)
echo "Redis Password: $REDIS_PASSWORD"
```

然后，启动一个临时的 Redis 客户端 Pod 来连接集群：

```bash
kubectl run redis-client --namespace redis-ha --rm --tty -i \
  --env REDIS_PASSWORD_ENV="$REDIS_PASSWORD" \
  --image docker.io/bitnami/redis-cluster:6.2.6 \
  -- bash
```
> **注意**: 如果上面命令中的 `bitnami/redis-cluster:6.2.6` 镜像版本与您部署的 `APP VERSION` 不一致，请替换为对应版本以确保 `redis-cli` 工具兼容。

在临时 Pod 的 shell 中，使用 `redis-cli` 连接到集群：

```bash
# 在 redis-client Pod 内部执行
redis-cli -c -h redis-cluster -a "$REDIS_PASSWORD_ENV"
```
参数说明：
*   `-c`: 启用集群模式，允许自动重定向。
*   `-h redis-cluster`: Redis 集群的 Service 名称 (由 Helm Chart 创建，通常是 Release 名称)。
*   `-a "$REDIS_PASSWORD_ENV"`: 使用密码进行认证。

连接成功后，您可以执行 Redis 命令来验证集群状态：

```
# 在 redis-cli 提示符下执行
> cluster info
# 期望看到: cluster_state:ok, cluster_slots_assigned:16384, cluster_size:3, ...

> cluster nodes
# 期望看到: 列出所有 6 个节点的信息，3 个 master 和 3 个 slave，及其角色和连接状态。

> set mykey "Hello Kubernetes Redis"
# > GET mykey
# "Hello Kubernetes Redis"

> exit
```
测试完毕后，在 `redis-client` Pod 的 `bash` 提示符下输入 `exit` 退出临时 Pod。

#### 4. 集群内部访问

在 Kubernetes 集群内部，其他应用可以通过以下 Service FQDN (完全限定域名) 访问 Redis 集群：

*   **主服务 (用于客户端连接)**: `redis-cluster.redis-ha.svc.cluster.local` (其中 `redis-cluster` 是 Helm Release 名称，`redis-ha` 是命名空间)。
    大多数 Redis Cluster 客户端库只需要这个地址和密码即可自动发现所有节点。

*   **各 Pod 的 DNS (通常用于调试或特定场景)**:
    ```
    redis-cluster-0.redis-cluster-headless.redis-ha.svc.cluster.local
    redis-cluster-1.redis-cluster-headless.redis-ha.svc.cluster.local
    ...
    redis-cluster-5.redis-cluster-headless.redis-ha.svc.cluster.local
    ```
    (注意，这里用的是 `redis-cluster-headless` 服务)

---

### 📈 运维与进阶

#### 1. 常见问题与排查 (Troubleshooting)

*   **Pod 状态 Pending**:
    *   检查 `kubectl describe pod <pod-name> -n redis-ha`。
    *   可能是资源不足 (CPU/Memory)、节点标签选择器不匹配、或者 PVC 无法绑定 (检查 StorageClass 和 NFS 服务状态)。
*   **PVC 状态 Pending**:
    *   检查 StorageClass 名称是否在 `values-custom.yaml` 中正确配置。
    *   确保存储供应者 (如 NFS Provisioner) 工作正常。
    *   查看 Provisioner 的日志。
*   **连接 Redis 超时/拒绝**:
    *   确认 Redis Pod 是否都处于 `Running` 状态。
    *   检查网络策略 (NetworkPolicies)，确保客户端 Pod 与 Redis Pod 之间的网络是通的。
    *   确认 Service `redis-cluster.redis-ha` 是否正确指向了健康的 Pods (`kubectl describe svc redis-cluster -n redis-ha`)。
*   **认证失败 (AUTH failed)**:
    *   确认客户端使用的密码与 `values-custom.yaml` 中配置并通过 Secret 存储的密码一致。
*   **Pod CrashLoopBackOff**:
    *   查看 Pod 日志: `kubectl logs <pod-name> -n redis-ha` 和 `kubectl logs --previous <pod-name> -n redis-ha`。
    *   可能是配置错误、权限问题或资源不足导致 Redis 进程启动失败。

#### 2. 最佳实践与后续优化

*   **资源管理**: 在 `values-custom.yaml` 中为 Redis Master 和 Slave Pods 配置合理的 CPU 和内存 `requests` 与 `limits`，以保证服务质量和集群稳定性。
*   **监控告警**:
    *   部署 Prometheus 和 Grafana，并使用 Redis Exporter (如 `bitnami/redis-exporter` Helm chart) 收集 Redis 指标。
    *   设置关键指标告警，如内存使用率、连接数、命令延迟、集群状态等。
*   **网络策略**: 如果您的 Kubernetes 集群启用了网络策略，请配置允许应用 Pod 访问 Redis Pods 的特定端口 (默认为 6379)。
*   **Helm Release 管理**:
    *   **升级**: `helm upgrade redis-cluster ./redis-cluster -n redis-ha -f values-custom.yaml`
    *   **回滚**: `helm rollback redis-cluster <REVISION_NUMBER> -n redis-ha`
    *   **卸载**: `helm uninstall redis-cluster -n redis-ha`
        > **注意**: 默认情况下，卸载 Helm Chart *不会* 删除 PVC。如果需要删除数据，请手动删除 PVC：`kubectl delete pvc -l app.kubernetes.io/instance=redis-cluster -n redis-ha`
*   **数据备份与恢复**: 针对 Redis Cluster 的备份恢复相对复杂。可以考虑使用 Redis 的 RDB/AOF 持久化机制，并定期备份 NFS 上的持久化文件。对于更高级的方案，可调研 Velero 等 K8s 备份工具。

---

### 🏁 总结

通过本文的指导，您成功地使用 Helm 在 Kubernetes 上部署了一个 3主3从的高可用 Redis 集群。此方案利用 Bitnami 提供的成熟 Chart，结合 NFS 持久化存储，不仅保障了数据的持久性，也大大提升了缓存服务的可靠性和系统的整体稳定性。

合理的配置、持续的监控和及时的维护，将使这个 Redis集群成为您应用架构中坚实可靠的一部分。鼓励您根据实际业务需求，进一步探索 `values.yaml` 中的其他配置选项，如节点亲和性、安全上下文等，打造更定制化的 Redis 服务。

---

### 📚 相关资源推荐

*   **Kubernetes Persistent Volumes**: [PV 和 PVC 详解](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)
*   **Helm 官方文档**: [Helm Docs](https://helm.sh/docs/)
*   **Redis Cluster 教程**: [Redis Cluster Tutorial](https://redis.io/docs/manual/scaling/)
*   **Bitnami Redis Cluster Chart**: 在 Artifact Hub 或 Bitnami官网查找 `redis-cluster` Chart 的详细参数。

---