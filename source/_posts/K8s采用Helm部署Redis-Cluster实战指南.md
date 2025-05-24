---
title: K8s采用Helm部署Redis-Cluster实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Redis
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250509140551459.png'
toc: true
abbrlink: 5c36b781
date: 2025-05-09 13:59:31
---

本文将引导您使用 Helm 在 Kubernetes (K8s) 集群中，快速部署一个基于 Redis (默认3主3从架构) 的高可用分布式缓存集群。我们将展示如何使用便捷的Shell脚本进行一键部署，同时也会介绍通过自定义`values.yaml`文件进行更灵活配置的方法。此部署方案依赖于现有的 Kubernetes 集群、Helm 客户端，并预设已配置基于 NFS 的 StorageClass 以实现持久化存储。

<!-- more -->

---

## 🚀 引言：为何选择高可用 Redis 集群？

在现代 Web 应用中，缓存是提升性能、降低数据库负载的关键组件。Redis 以其高性能和丰富的数据结构成为缓存首选。然而，单点 Redis 存在可用性风险。通过部署 Redis 主从集群，可以实现数据冗余和故障自动切换，确保服务的高可用性，为您的应用提供稳定可靠的缓存服务。

---

## 🛠️ 环境准备 (Prerequisites)

在开始之前，请确保您的环境满足以下条件：

1.  **Kubernetes 集群**: 版本 1.16+ 推荐。确保 `kubectl` 已配置并能正常访问集群。
2.  **Helm 客户端**: 版本 3.x。Helm 是 Kubernetes 的包管理器，极大简化了应用部署。
3.  **NFS StorageClass**: 预先配置好的、可动态申请持久卷 (PV) 的 StorageClass。Redis 数据将持久化到 NFS 存储中。

> **💡 提示**
> * 本文示例中的 StorageClass 名称为 `nfs-storage`。请根据您的实际环境替换。
> * 如果您尚未配置 NFS StorageClass，可以参考官方文档或相关教程进行部署。例如，[Kubernetes使用Helm部署NFS-Client-Provisioner实现动态存储](https://lbs.wiki/pages/e3673e0e/) 这篇文章提供了很好的指导。动态存储配置是实现数据持久化的关键。

---

## ⚙️ 核心部署步骤

我们将介绍两种部署方式：通过提供的脚本快速部署，以及通过下载和修改 Helm Chart进行自定义部署。

### 方式一：🚀 使用脚本快速部署 (推荐入门)

这种方式通过预设的脚本和参数，可以快速完成 Redis Cluster 的部署。

**1. 准备脚本文件**

创建以下三个脚本文件：

*   `install-redis-cluster.sh`: 用于安装 Redis Cluster。
*   `status-redis-cluster.sh`: 用于检查部署状态。
*   `uninstall-redis-cluster.sh`: 用于卸载 Redis Cluster。

**`install-redis-cluster.sh`**
```shell
#!/usr/bin/env bash

# --- 可配置变量 ---
HELM_RELEASE_NAME="redis-cluster"
NAMESPACE="redis-cluster"
CHART_VERSION="12.0.4" # Bitnami Redis Cluster Chart 版本，请按需选择
STORAGE_CLASS="nfs-storage" # 替换为您的 StorageClass 名称
REDIS_PASSWORD="YOUR_PASSWORD" # 替换为您的强密码
METRICS_ENABLED="true"
# ------------------

# 添加 Bitnami Helm 仓库 (如果已添加，此步骤会提示已存在)
helm repo add bitnami https://charts.bitnami.com/bitnami

# 更新本地 Helm 仓库索引
helm repo update

# 使用 Helm 安装 Redis Cluster
helm install "$HELM_RELEASE_NAME" bitnami/redis-cluster --version "$CHART_VERSION" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  --set-string global.storageClass="$STORAGE_CLASS" \
  --set-string global.redis.password="$REDIS_PASSWORD" \
  --set metrics.enabled="$METRICS_ENABLED" \
  # --set cluster.nodes=6 # 默认3主3从，可以调整节点数
  # --set cluster.replicas=1 # 每个主节点的从节点数量

echo "Redis Cluster 安装命令已执行。请使用 status-redis-cluster.sh 检查状态。"
```
> **🛡️ 安全提示**
> 执行前，务必将 `install-redis-cluster.sh` 脚本中的 `YOUR_STRONG_PASSWORD_HERE` 替换为一个复杂且唯一的强密码。

**`status-redis-cluster.sh`**
```shell
#!/usr/bin/env bash

NAMESPACE="redis-cluster" # 与安装脚本中的 NAMESPACE 一致

echo "--- Helm Release 状态 ---"
helm list -n "$NAMESPACE"

echo ""
echo "--- Pods 状态 (等待所有 Pods Running 且 Ready) ---"
kubectl get pods -n "$NAMESPACE" -l app.kubernetes.io/name=redis-cluster -w

# 如果想查看所有资源，可以使用:
# kubectl get all -n "$NAMESPACE"
```

**`uninstall-redis-cluster.sh`**
```shell
#!/usr/bin/env bash

HELM_RELEASE_NAME="redis-cluster"
NAMESPACE="redis-cluster" # 与安装脚本中的 NAMESPACE 一致

helm uninstall "$HELM_RELEASE_NAME" -n "$NAMESPACE"

echo "Redis Cluster 已卸载。"
echo "注意：持久卷声明 (PVCs) 可能仍然存在。如果需要删除数据，请手动删除它们："
echo "kubectl get pvc -n $NAMESPACE -l app.kubernetes.io/instance=$HELM_RELEASE_NAME"
echo "kubectl delete pvc -n $NAMESPACE -l app.kubernetes.io/instance=$HELM_RELEASE_NAME"
```

**2. 执行脚本**

首先，赋予脚本执行权限：
```bash
chmod +x install-redis-cluster.sh status-redis-cluster.sh uninstall-redis-cluster.sh
```

然后，运行安装脚本：
```bash
./install-redis-cluster.sh
```
脚本会自动添加 Bitnami 仓库、更新索引、创建命名空间 `redis-cluster`（如果不存在），并部署 Redis Cluster。

部署命令执行后，运行状态检查脚本：
```bash
./status-redis-cluster.sh
```
观察 Pods 状态，等待所有 Pods (通常是 6 个，3 主 3 从) 变为 `Running` 且 `READY` 状态为 `2/2` (Redis 实例 + metrics exporter)。

### 方式二：🛠️ 自定义 Helm Chart 部署

如果您需要更细致的配置，例如修改副本数、资源限制、亲和性等，建议下载 Chart 并修改 `values.yaml`。

**1. 添加 Bitnami Helm 仓库 (同上)**

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

**2. 查看并选择 Chart 版本**

```bash
helm search repo bitnami/redis-cluster
```
您会看到可用的 Chart 版本。本文假设我们使用 `12.0.4` 版本，其对应的 Redis 应用版本通常是 `7.0.x`。（请根据 `helm search` 的结果选择最新稳定版或特定版本）。

**3. 下载并自定义 Helm Chart**

```bash
# 创建工作目录 (如果不存在)
CHART_VERSION_TO_DOWNLOAD="12.0.4" # 与脚本中版本对应
mkdir -p ~/kube-deploy/redis-cluster-custom && cd ~/kube-deploy/redis-cluster-custom

# 下载指定版本的 Chart
helm pull bitnami/redis-cluster --version "$CHART_VERSION_TO_DOWNLOAD"

# 解压 Chart 包
tar -xvf redis-cluster-${CHART_VERSION_TO_DOWNLOAD}.tgz

# 复制默认的 values.yaml 文件，用于自定义配置
cp ./redis-cluster/values.yaml ./values-custom.yaml
```
此时，您的目录结构应如下所示：
```text
.
├── redis-cluster                 # 解压后的 Chart 目录
│   ├── Chart.yaml
│   ├── templates
│   ├── values.yaml               # 默认配置文件
│   └── ...
├── redis-cluster-${CHART_VERSION_TO_DOWNLOAD}.tgz # 下载的 Chart 压缩包
└── values-custom.yaml            # 我们将修改此文件
```

**4. 定制 `values-custom.yaml`**

首先，确认集群中可用的 StorageClass 名称：
```bash
kubectl get storageclasses
```
编辑 `values-custom.yaml` 文件。以下是一些关键配置：

```yaml
# values-custom.yaml (示例片段)

# 全局配置
global:
  storageClass: "nfs-storage"     # 确保 "nfs-storage" 是您环境中可用的 StorageClass
  redis:
    password: "YourStrongPassword123_ForCustom!"  # 设置 Redis 集群访问密码

# 集群节点配置 (默认3主，每个主带1个从，共6个节点)
# cluster:
#   nodes: 6 # Pod 数量
#   replicas: 1 # 每个 Master 的 Slave 数量

# 如果您想调整主节点数量，例如改为2主，则 nodes 应为 4 (2主 + 2从)
# cluster:
#   masterGroupName: "master" # 此参数在较新版本chart中可能已移除或整合
#   updatePartition: 0
#   # Number of nodes in the cluster
#   # For a cluster with N masters and M replicas per master, cluster.nodes must be N * (M+1)
#   nodes: 6
#   # Number of replicas per master
#   replicas: 1

# 资源请求与限制 (根据需要调整)
# master:
#   resources:
#     requests:
#       cpu: 100m
#       memory: 256Mi
#     limits:
#       cpu: 500m
#       memory: 512Mi
# slave: # 在较新版本 chart 中可能叫做 replica
#   replica:
#     resources:
#       requests:
#         cpu: 100m
#         memory: 256Mi
#       limits:
#         cpu: 500m
#         memory: 512Mi

# Redis Exporter 指标监控
metrics:
  enabled: true
  # resources:
  #   requests:
  #     cpu: 50m
  #     memory: 64Mi
  #   limits:
  #     cpu: 100m
  #     memory: 128Mi

# 持久化配置
persistence:
  enabled: true
  # storageClass: "" # 如果全局已设置，此处可留空或注释掉
  size: 8Gi # 根据您的数据量调整
```
> **🛡️ 安全提示**
> 请务必将 `YourStrongPassword123_ForCustom!` 替换为一个复杂且唯一的强密码。查阅 `redis-cluster/values.yaml` 获取所有可配置项。

**5. 创建 Kubernetes 命名空间 (如果尚未创建)**
为 Redis 集群创建一个专用的命名空间，有助于资源隔离和管理。

```bash
kubectl create namespace redis-cluster # 与脚本部署的命名空间保持一致，或选择新的
```

**6. 部署 Redis 集群**
使用 Helm install 命令，并指定自定义的 `values-custom.yaml` 文件进行部署。

```bash
HELM_RELEASE_NAME="redis-cluster-custom" # 可以使用不同的 Release 名称以免与脚本部署冲突
NAMESPACE_FOR_CUSTOM_DEPLOY="redis-cluster" # 或您选择的其他命名空间

helm install "$HELM_RELEASE_NAME" ./redis-cluster \
  --namespace "$NAMESPACE_FOR_CUSTOM_DEPLOY" \
  -f ./values-custom.yaml
```

---

## 🔎 验证与访问 (通用)

无论使用脚本部署还是自定义部署，验证步骤类似。以下命令假设命名空间为 `redis-cluster`，Release 名称为 `redis-cluster` (如果使用自定义部署且Release名称不同，请相应修改)。

### 1. 检查 Helm 部署状态

```bash
helm list --namespace redis-cluster
```
您应该能看到类似输出，`STATUS` 应为 `deployed`。如果使用 Chart `12.0.4`，`APP VERSION` 可能是 `7.0.15` 或类似版本。
```text
NAME            NAMESPACE       REVISION    UPDATED                                 STATUS      CHART                       APP VERSION
redis-cluster   redis-cluster   1           2025-05-10 10:15:00 CST                 deployed    redis-cluster-12.0.4        7.0.15
```

### 2. 检查 Pod 运行状态

```bash
# 使用 status-redis-cluster.sh 脚本，或者手动执行：
kubectl get pods --namespace redis-cluster -l app.kubernetes.io/name=redis-cluster -w
```
等待所有 Pod 的 `STATUS` 变为 `Running`，并且 `READY` 状态为 `2/2`（Redis 进程 + Exporter）。
```text
NAME                      READY   STATUS    RESTARTS   AGE
redis-cluster-nodes-0     2/2     Running   0          5m
redis-cluster-nodes-1     2/2     Running   0          5m
redis-cluster-nodes-2     2/2     Running   0          5m
redis-cluster-nodes-3     2/2     Running   0          5m
redis-cluster-nodes-4     2/2     Running   0          5m
redis-cluster-nodes-5     2/2     Running   0          5m
```
**注意**: Pod 名称可能从 `redis-cluster-0` 变为 `redis-cluster-nodes-0`，具体取决于 Chart 版本和配置。
按 `Ctrl+C` 退出 `-w` (watch) 模式。

### 3. 连接及测试集群

首先，获取 Redis 密码 (假设 Release 名称为 `redis-cluster`，密码 Key 为 `redis-password`)：

```bash
export REDIS_PASSWORD=$(kubectl get secret --namespace redis-cluster redis-cluster -o jsonpath="{.data.redis-password}" | base64 --decode)
echo "Redis Password: $REDIS_PASSWORD"
```

然后，启动一个临时的 Redis 客户端 Pod 来连接集群 (确保镜像版本与您部署的 Redis APP VERSION 匹配)：
```bash
# APP_VERSION 通常可以在 `helm list` 输出中看到，或在 Chart 信息中找到
# 对于 bitnami/redis-cluster chart 12.0.4, APP VERSION 可能是 7.0.15
# 对于 bitnami/redis-cluster chart 7.5.0, APP VERSION 可能是 6.2.6
# 请根据您的实际部署版本调整下面的镜像标签
REDIS_CLI_IMAGE_TAG="7.0.15" # 替换为与您部署的 Redis 版本匹配的客户端镜像

kubectl run redis-client --namespace redis-cluster --rm --tty -i \
  --env REDIS_PASSWORD_ENV="$REDIS_PASSWORD" \
  --image docker.io/bitnami/redis-cluster:$REDIS_CLI_IMAGE_TAG \
  -- bash
```

在临时 Pod 的 shell 中，使用 `redis-cli` 连接到集群：
(Service 名称通常是 Helm Release 名称，即 `redis-cluster`)
```bash
# 在 redis-client Pod 内部执行
redis-cli -c -h redis-cluster-headless -a "$REDIS_PASSWORD_ENV"
# 或者，如果连接主服务 redis-cluster (它是一个 LoadBalancer 或 ClusterIP，取决于 chart 配置)
# redis-cli -c -h redis-cluster -a "$REDIS_PASSWORD_ENV"
```
> **注意**：Bitnami chart 通常会创建一个名为 `redis-cluster-headless` 的 Headless Service 用于内部节点发现，以及一个名为 `redis-cluster` 的常规 Service 用于外部客户端连接。优先尝试连接 `redis-cluster-headless`，如果不行再尝试 `redis-cluster`。对于集群模式，`redis-cli -c` 可以处理重定向。

连接成功后，您可以执行 Redis 命令来验证集群状态：
```
# 在 redis-cli 提示符下执行
> cluster info
# 期望看到: cluster_state:ok, cluster_slots_assigned:16384, cluster_size:3 (主节点数), ...

> cluster nodes
# 期望看到: 列出所有节点的信息，及其角色和连接状态。

> set mykey "Hello K8s Redis Cluster"
# > GET mykey
# "Hello K8s Redis Cluster"

> exit
```
测试完毕后，在 `redis-client` Pod 的 `bash` 提示符下输入 `exit` 退出临时 Pod。

### 4. 集群内部访问

在 Kubernetes 集群内部，其他应用可以通过以下 Service FQDN (完全限定域名) 访问 Redis 集群：

*   **主服务 (用于客户端连接)**: `redis-cluster.redis-cluster.svc.cluster.local` ( `<release-name>.<namespace>.svc.cluster.local` )。
    大多数 Redis Cluster 客户端库只需要这个地址和密码即可自动发现所有节点。

*   **Headless 服务 (用于节点发现和直连Pod)**: `redis-cluster-headless.redis-cluster.svc.cluster.local`
    各 Pod 的 DNS (通过 Headless 服务，`<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local` ):
    ```
    redis-cluster-nodes-0.redis-cluster-headless.redis-cluster.svc.cluster.local
    redis-cluster-nodes-1.redis-cluster-headless.redis-cluster.svc.cluster.local
    ...
    ```

---

## 📈 运维与进阶

### 1. 常见问题与排查 (Troubleshooting)

*   **Pod 状态 Pending / PVC 状态 Pending**:
    *   检查 `kubectl describe pod <pod-name> -n redis-cluster` 和 `kubectl describe pvc <pvc-name> -n redis-cluster`。
    *   确认 `storageClass` 名称在脚本或 `values-custom.yaml` 中正确配置，且该 StorageClass 存在并工作正常。
    *   检查 NFS Provisioner 日志 (如果使用)。
*   **连接 Redis 超时/拒绝**:
    *   确认 Redis Pods (`redis-cluster-nodes-*`) 都处于 `Running` 且 `2/2 Ready`。
    *   检查网络策略 (NetworkPolicies)，确保客户端 Pod 与 Redis Pods 之间的网络是通的 (端口 6379)。
    *   确认 Service `redis-cluster.redis-cluster` 和 `redis-cluster-headless.redis-cluster` 正确指向健康的 Pods。
*   **认证失败 (AUTH failed)**:
    *   确认客户端使用的密码与部署时设置的密码一致。
*   **Pod CrashLoopBackOff**:
    *   查看 Pod 日志: `kubectl logs <pod-name> -n redis-cluster` 和 `kubectl logs --previous <pod-name> -n redis-cluster`。
    *   可能是配置错误 (如错误的 `values.yaml` 设置)、持久化卷权限问题或资源不足。

### 2. Helm Release 管理

*   **升级**: 如果您修改了 `values-custom.yaml` 或想升级 Chart 版本：
    ```bash
    # 针对自定义部署
    helm upgrade redis-cluster-custom ./redis-cluster -n redis-cluster -f values-custom.yaml
    # 或针对脚本部署 (如果想修改参数并升级)
    helm upgrade redis-cluster bitnami/redis-cluster --version <new-chart-version> \
      -n redis-cluster \
      --set-string global.storageClass="nfs-storage" \
      --set-string global.redis.password="NEW_PASSWORD" \
      # ... 其他参数
    ```
*   **回滚**:
    ```bash
    helm history redis-cluster -n redis-cluster # 查看历史版本
    helm rollback redis-cluster <REVISION_NUMBER> -n redis-cluster
    ```
*   **卸载**: 使用前面提供的 `uninstall-redis-cluster.sh` 脚本，或者手动执行：
    ```bash
    helm uninstall redis-cluster -n redis-cluster
    ```
    > **⚠️ 重要**: 默认情况下，卸载 Helm Chart *不会* 删除 PVC。如果需要彻底删除数据，请在卸载后手动删除相关的 PVCs：
    > `kubectl delete pvc -n redis-cluster -l app.kubernetes.io/instance=redis-cluster`
    > (将 `redis-cluster` 替换为您的 Helm Release 名称)。

### 3. 最佳实践与后续优化

*   **资源管理**: 在 `values-custom.yaml` (对于自定义部署) 或通过 `--set` 参数 (对于脚本化或升级) 为 Redis Pods 配置合理的 CPU 和内存 `requests` 与 `limits`。
*   **监控告警**: `metrics.enabled=true` (已在脚本和 `values.yaml` 示例中启用) 会部署 Redis Exporter。配合 Prometheus 和 Grafana 收集和展示 Redis 指标。
*   **网络策略**: 如果您的 K8s 集群启用了网络策略，请配置允许应用 Pod 访问 Redis Pods 的特定端口 (TCP 6379 及集群总线端口 16379)。
*   **数据备份与恢复**: 利用 Redis 的 RDB/AOF 持久化机制，并定期备份 NFS 上的持久化文件。考虑使用 Velero 等 K8s 备份工具进行更全面的集群资源备份。

---

## 🏁 总结

通过本文的指导，您可以根据自己的需求选择使用便捷脚本或自定义 `values.yaml` 文件，在 Kubernetes 上成功部署一个高可用的 Redis 集群。这两种方式都利用了 Bitnami 提供的成熟 Chart 和 NFS 持久化存储，确保了数据持久性和服务可靠性。

灵活运用提供的脚本和自定义选项，结合持续的监控和维护，将使这个 Redis 集群成为您应用架构中坚实可靠的一部分。

---

## 📚 相关资源推荐

*   **Kubernetes Persistent Volumes**: [PV 和 PVC 详解](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)
*   **Helm 官方文档**: [Helm Docs](https://helm.sh/docs/)
*   **Redis Cluster 教程**: [Redis Cluster Tutorial](https://redis.io/docs/manual/scaling/)
*   **Bitnami Redis Cluster Chart (Artifact Hub)**: [搜索 "redis-cluster" 并找到 bitnami 的 chart](https://artifacthub.io/) 以获取最新的 Chart 版本和详细参数。例如，`12.0.4` 的文档。

---