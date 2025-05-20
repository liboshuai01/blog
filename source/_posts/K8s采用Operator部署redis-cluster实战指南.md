---
title: K8s采用Operator部署redis-cluster实战指南
tags:
  - Linux
  - K8s
  - Operator
  - Redis
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250520115854685.png'
toc: true
abbrlink: 6024f5c5
date: 2025-05-10 13:53:34
---

本文将指导您使用 Kubernetes Operator 在 K8s 集群中，高效部署一个高可用的 Redis Cluster。此方案利用 Redis Operator (来自 Opstree) 来自动化 Redis 集群的创建、配置和管理，实现声明式部署。我们将默认配置3主3从的集群，并同样依赖预配置的 StorageClass (如基于 NFS) 实现持久化存储。

<!-- more -->

---

## 🚀 引言：为何选择 Operator 模式部署 Redis 集群？

在 Kubernetes 生态中，Operator 模式通过引入自定义资源 (CRD) 和自定义控制器，将特定应用的运维知识编码到软件中，从而实现复杂有状态应用的自动化管理。相较于 Helm Chart（通常侧重于应用的初始部署和配置），Operator 提供了更深层次的生命周期管理能力，包括自动扩缩容、版本升级、故障恢复、备份等。

对于 Redis Cluster 这样的分布式有状态服务，Operator 能够：

1.  **简化部署与管理**: 通过一个简单的 YAML (Custom Resource) 即可定义和部署整个集群。
2.  **自动化运维**: Operator 会持续监控集群状态，自动处理节点故障、数据同步等问题。
3.  **高可用性**: 确保 Redis 服务在节点故障时能自动切换和恢复，保障业务连续性。
4.  **声明式配置**: 您只需声明期望的状态，Operator 负责将其变为现实。

本指南将使用 Opstree 的 Redis Operator 来部署一个生产级的 Redis Cluster。

---

## 🛠️ 环境准备 (Prerequisites)

在开始之前，请确保您的环境满足以下条件：

1.  **Kubernetes 集群**: 版本 1.18+ 推荐。确保 `kubectl` 已配置并能正常访问集群。
2.  **Helm 客户端**: 版本 3.x。Helm 将用于安装 Redis Operator 本身。
3.  **StorageClass**: 预先配置好的、可动态申请持久卷 (PV) 的 StorageClass。Redis 数据将持久化到此存储中。如果未指定，将使用集群的默认 StorageClass。

> **💡 提示**
> 如果您尚未配置动态存储，可以参考类似 [Kubernetes使用Helm部署NFS-Client-Provisioner实现动态存储](https://lbs.wiki/pages/e3673e0e/) 的教程进行部署。Operator 部署的 Redis Cluster 同样需要可靠的持久化存储来保存数据。

---

## ⚙️核心部署步骤

### 1. 安装 Redis Operator

我们将使用 Helm 来安装 Opstree 的 Redis Operator。

```bash
# 添加 Opstree Helm 仓库
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/

# 更新本地 Helm 仓库索引
helm repo update

# 安装 Redis Operator 到 'operator' 命名空间 (如果不存在则创建)
helm install redis-operator ot-helm/redis-operator -n ot-operator --create-namespace
```

安装完成后，验证 Operator Pod 是否正常运行：

```bash
kubectl get pods -n ot-operator
```

您应该看到类似如下的输出，Operator Pod 处于 `Running` 状态：

```text
NAME                              READY   STATUS    RESTARTS   AGE
redis-operator-5fbc6c788f-xxxxx   1/1     Running   0          2m
```

### 2. 创建 Redis 访问凭证 (Secret)

Redis 集群需要密码进行保护。我们将创建一个 Kubernetes Secret 来存储这个密码。

首先，准备 `redis-secret.yaml` 文件：

```yaml
# redis-secret.yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: redis-secret # 此名称将在 RedisCluster CR 中引用
  # namespace: redis-ns # 建议为 Redis 集群创建一个专用命名空间
data:
  password: T3BzdHJlZUAxMjM0Cg== # "Opstree@1234" 的 Base64 编码
type: Opaque
```

> **🛡️ 安全提示与说明**
> *   `password` 字段的值必须是您选择的密码经过 Base64 编码后的结果。例如，要使用密码 `MyStrongP@ssw0rd!`，可以执行 `echo -n "MyStrongP@ssw0rd!" | base64` 来获取编码后的字符串。
> *   强烈建议为您的 Redis 集群创建一个专用的命名空间（例如 `redis-ns`），并在 Secret 和 RedisCluster CRD 中都指定它。如果未指定 namespace, 资源将创建在 `default` 命名空间。为清晰起见，后续步骤假设您已创建并使用特定命名空间，如`redis-ns`。
      >     ```bash
      >     kubectl create namespace redis-ns
      >     ```
      >      如果您使用自定义命名空间，请在 `metadata.namespace` 字段中指定，并在 `kubectl apply` 时使用 `-n redis-ns`。

应用 Secret 配置：

```bash
# 如果使用了自定义命名空间 redis-ns
kubectl apply -f redis-secret.yaml -n redis-ns

# 如果在 default 命名空间
# kubectl apply -f redis-secret.yaml
```

### 3. 定义并创建 Redis Cluster (Custom Resource)

现在，我们将定义 Redis 集群的配置。创建一个名为 `redis-cluster.yaml` 的文件：

```yaml
# redis-cluster.yaml
---
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisCluster
metadata:
  name: my-redis-cluster # Redis 集群的名称
  # namespace: redis-ns # 确保与 Secret 在同一命名空间
spec:
  clusterSize: 3 # 指定主节点数量，Operator 会自动创建 N 主 N 从 (总共 2N 个 Pod)
  clusterVersion: v7 # 可以选择 v6 或 v7，确保与 image 兼容
  podSecurityContext: # Pod 级别的安全上下文
    runAsUser: 1000
    fsGroup: 1000
  persistenceEnabled: true # 启用持久化
  kubernetesConfig:
    image: quay.io/opstree/redis:v7.0.15 # Redis 镜像
    imagePullPolicy: IfNotPresent
    redisSecret: # 引用上面创建的 Secret
      name: redis-secret # Secret 名称
      key: password    # Secret 中密码对应的键
  redisExporter: # 内置 Redis Exporter 配置，用于监控
    enabled: true
    image: quay.io/opstree/redis-exporter:v1.44.0
  storage: # 持久化存储配置
    volumeClaimTemplate: # 用于 Redis 数据
      spec:
        # storageClassName: "nfs-storage" # 明确指定 StorageClass，如果为空则使用默认SC
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi # 每个 Redis 实例的数据卷大小
    nodeConfVolumeClaimTemplate: # 用于 Redis 节点配置文件
      spec:
        # storageClassName: "nfs-storage" # 明确指定 StorageClass
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi # 每个 Redis 实例的配置卷大小
```

> **💡 配置说明**
> *   `metadata.name`: 您的 Redis 集群的名称。
> *   `clusterSize: 3`: 这将创建 3 个主节点和 3 个从节点（每个主节点一个从节点）。
> *   `persistenceEnabled: true`: 强烈建议生产环境启用持久化。
> *   `kubernetesConfig.redisSecret`: 必须与您之前创建的 Secret 名称匹配。
> *   `storage.volumeClaimTemplate.spec.storageClassName`: 如果您有多个 StorageClass，或者不想使用默认的，请在此处明确指定。如果留空或注释掉，系统将使用标记为 `(default)` 的 StorageClass。
> *   `redisExporter.enabled: true`: 默认启用 Redis Exporter，方便集成 Prometheus 进行监控。

部署 Redis Cluster 实例：

```bash
# 如果使用了自定义命名空间 redis-ns
kubectl apply -f redis-cluster.yaml -n redis-ns

# 如果在 default 命名空间
# kubectl apply -f redis-cluster.yaml
```
Operator 现在会检测到这个新的 `RedisCluster` 资源，并开始在指定的命名空间中创建所需的 StatefulSets、Services、PVCs 等组件。

---

## 🔎 验证与访问

### 1. 检查 Redis Cluster Pod 状态

等待一段时间，让 Operator 完成部署。然后检查 Pod 状态。假设您使用了`redis-ns`命名空间，并且`RedisCluster`的`metadata.name`为`my-redis-cluster`。

```bash
kubectl get pods -n redis-ns -l redis.redis.opstreelabs.in/cluster-name=my-redis-cluster -w
```

您会看到 6 个 Pod (3 leader/master, 3 follower/slave) 逐渐进入 `Running` 状态，且 `READY` 计数为 `2/2` (一个 Redis 容器，一个 Exporter 容器)。

```text
NAME                        READY   STATUS    RESTARTS   AGE
my-redis-cluster-follower-0   2/2     Running   0          3m31s
my-redis-cluster-follower-1   2/2     Running   0          3m27s
my-redis-cluster-follower-2   2/2     Running   0          3m25s
my-redis-cluster-leader-0     2/2     Running   0          3m40s
my-redis-cluster-leader-1     2/2     Running   0          3m36s
my-redis-cluster-leader-2     2/2     Running   0          3m34s
```
按 `Ctrl+C` 退出 `-w` (watch) 模式。

### 2. 检查 Service 状态

Operator 会为 Redis 集群创建多个 Service。

```bash
kubectl get svc -n redis-ns -l redis.redis.opstreelabs.in/cluster-name=my-redis-cluster
```

输出示例：
```text
NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
my-redis-cluster-follower           ClusterIP   10.43.229.199   <none>        6379/TCP,9121/TCP   6m34s
my-redis-cluster-follower-additional ClusterIP   10.43.61.127    <none>        6379/TCP            6m34s
my-redis-cluster-follower-headless  ClusterIP   None            <none>        6379/TCP            6m34s
my-redis-cluster-leader             ClusterIP   10.43.15.7      <none>        6379/TCP,9121/TCP   6m43s
my-redis-cluster-leader-additional  ClusterIP   10.43.80.36     <none>        6379/TCP            6m43s
my-redis-cluster-leader-headless    ClusterIP   None            <none>        6379/TCP            6m43s
```
关键 Service：
*   `my-redis-cluster-leader`: 主要的连接入口，指向当前的 leader/master 节点。客户端应连接此 Service。
*   `my-redis-cluster-follower`: 指向 follower/slave 节点，用于读密集型场景（需要客户端配置）。

### 3. 检查 Secret 是否可被 Operator 读取

（这一步在 Operator 部署时已间接验证，若 Pod 无法启动且日志提示认证失败，则需检查 Secret 配置）。
你可以确认 Secret 存在：
```bash
kubectl get secret redis-secret -n redis-ns
```
输出示例：
```text
NAME           TYPE     DATA   AGE
redis-secret   Opaque   1      10m
```

### 4. 连接及测试集群

首先，获取之前设置的 Redis 密码：

```bash
# 确保使用正确的 Secret 名称和命名空间
export REDIS_PASSWORD=$(kubectl get secret redis-secret -n redis-ns -o jsonpath="{.data.password}" | base64 --decode)
echo "Redis Password: $REDIS_PASSWORD"
```

然后，启动一个临时的 Redis 客户端 Pod 来连接集群 (使用 Bitnami 的 redis-cluster 镜像，它包含 `redis-cli`)：

```bash
# 确保 -n 与您的 Redis 集群部署的命名空间一致
# --image 版本应与您的 Redis 集群版本兼容或一致
kubectl run redis-client -n redis-ns --rm --tty -i \
  --env REDIS_PASSWORD_ENV="$REDIS_PASSWORD" \
  --image docker.io/bitnami/redis-cluster:7.0.15 \
  -- bash
```

在临时 Pod 的 shell 中，使用 `redis-cli` 连接到集群 Leader Service：

```bash
# 在 redis-client Pod 内部执行
# -h 参数使用 <redis-cluster-name>-leader (例如 my-redis-cluster-leader)
# 如果在 default 命名空间，可以直接使用 my-redis-cluster-leader
# 如果在 redis-ns 命名空间，也可以使用 FQDN: my-redis-cluster-leader.redis-ns.svc.cluster.local
redis-cli -c -h my-redis-cluster-leader -a "$REDIS_PASSWORD_ENV"
```
参数说明：
*   `-c`: 启用集群模式，允许自动重定向。
*   `-h my-redis-cluster-leader`: Redis 集群的主 Service 名称。
*   `-a "$REDIS_PASSWORD_ENV"`: 使用密码进行认证。

连接成功后，您可以执行 Redis 命令来验证集群状态：

```
# 在 redis-cli 提示符下执行
> cluster info
# 期望看到: cluster_state:ok, cluster_slots_assigned:16384, cluster_size:3 (主节点数), ...

> cluster nodes
# 期望看到: 列出所有 6 个节点的信息，3 个 master 和 3 个 slave，及其角色和连接状态。

> set mykey "Hello Kubernetes Operator Redis"
# > GET mykey
# "Hello Kubernetes Operator Redis"

> exit
```
测试完毕后，在 `redis-client` Pod 的 `bash` 提示符下输入 `exit` 退出临时 Pod。

### 5. 集群内部访问

在 Kubernetes 集群内部，其他应用可以通过以下 Service FQDN (完全限定域名) 访问 Redis 集群 (假设部署在 `redis-ns` 命名空间，集群名为 `my-redis-cluster`)：

*   **主服务 (用于客户端写入和读取)**: `my-redis-cluster-leader.redis-ns.svc.cluster.local:6379`
    大多数 Redis Cluster 客户端库只需要这个地址和密码即可自动发现所有节点。

---

## 📈 运维与进阶

### 1. 常见问题与排查 (Troubleshooting)

*   **Operator Pod 未运行**: 检查 `kubectl get pods -n operator` 和 `kubectl logs -n operator <operator-pod-name>`。
*   **RedisCluster CR 状态**: 使用 `kubectl describe rediscluster my-redis-cluster -n redis-ns` 查看 CR 的状态和事件，Operator 会更新这些信息。
*   **Redis Pod 状态 Pending**:
    *   检查 `kubectl describe pod <redis-pod-name> -n redis-ns`。
    *   可能是 PVC 无法绑定 (检查 StorageClass 配置，特别是 `storage.volumeClaimTemplate.spec.storageClassName` 是否正确或默认SC是否可用)、资源不足。
*   **Redis Pod CrashLoopBackOff**:
    *   查看 Pod 日志: `kubectl logs <redis-pod-name> -c redis -n redis-ns` (或 `-c redis-exporter`)。
    *   可能是密码 Secret 配置错误 (`redisSecret.name` 或 `redisSecret.key` 不匹配，或密码本身错误)、权限问题 (检查 `podSecurityContext`)。
*   **认证失败 (AUTH failed)**: 确认客户端使用的密码与 `redis-secret` 中存储的一致。
*   **连接超时/拒绝**:
    *   确认 Service `my-redis-cluster-leader.redis-ns` 正确指向健康的 Leader Pods。
    *   检查网络策略 (NetworkPolicies)。

### 2. Operator 特性与后续优化

Operator 模式使得许多运维任务更加简单：

*   **资源管理**: Opstree Redis Operator 允许在其 CRD 中配置资源请求和限制（通常在 `spec.kubernetesConfig.resources` 下，具体需查阅 Operator 文档最新版 CRD 定义）。为 Redis Pod 和 Exporter 配置合理的资源是保证稳定性的关键。
    ```yaml
    # 示例: 在 redis-cluster.yaml 中 (需确认 Operator CRD 支持)
    # spec:
    #   kubernetesConfig:
    #     resources:
    #       requests:
    #         cpu: "500m"
    #         memory: "1Gi"
    #       limits:
    #         cpu: "1"
    #         memory: "2Gi"
    #   redisExporter:
    #     resources:
    #       requests:
    #         cpu: "100m"
    #         memory: "128Mi"
    #       limits:
    #         cpu: "200m"
    #         memory: "256Mi"
    ```
*   **集群伸缩**: 修改 `RedisCluster` CRD 中的 `clusterSize` 字段，然后 `kubectl apply -f redis-cluster.yaml -n redis-ns`。Operator 会自动处理节点的增减和数据重分片。
*   **版本升级**: 修改 `RedisCluster` CRD 中的 `kubernetesConfig.image` (或 `clusterVersion`，取决于 Operator 设计) 为新版本，然后 `kubectl apply`。Operator 会执行滚动升级。
*   **监控告警**:
    *   由于 `redisExporter.enabled: true`，Redis 指标已通过端口 `9121` 暴露。
    *   配置 Prometheus 抓取这些指标 (通常通过 ServiceMonitor CRD，如果您的集群有 Prometheus Operator)。
    *   在 Grafana 中创建仪表盘展示 Redis 状态。
*   **备份与恢复**: 查阅 Opstree Redis Operator 的官方文档，看其是否提供了通过 CRD 进行备份和恢复的功能。若无，则需结合 Redis 的 RDB/AOF 持久化和底层存储的快照能力。
*   **卸载集群**:
    1.  删除 RedisCluster CR: `kubectl delete rediscluster my-redis-cluster -n redis-ns`
    2.  删除 Secret: `kubectl delete secret redis-secret -n redis-ns`
    3.  Operator 一般不会自动删除 PVC。如果需要彻底清除数据，请手动删除 PVC: `kubectl delete pvc -n redis-ns -l redis.redis.opstreelabs.in/cluster-name=my-redis-cluster`
*   **卸载 Operator**: `helm uninstall redis-operator -n operator`

---

## 🏁 总结

通过本指南，您学习了如何利用 Kubernetes Operator (以 Opstree Redis Operator 为例) 来部署和管理一个高可用的 Redis Cluster。Operator 模式极大地简化了有状态应用的运维复杂性，使得在 Kubernetes 上运行分布式缓存服务更为高效和可靠。

声明式的配置、自动化的生命周期管理以及与 Kubernetes 生态的深度集成，使得 Operator 成为在云原生环境中部署 Redis 等关键服务的理想选择。鼓励您深入研究 Operator 的 CRD 选项和官方文档，以充分发掘其潜力，满足更高级的定制化需求。

---

## 📚 相关资源推荐

*   **Opstree Redis Operator 文档**: [在其 GitHub 仓库查找](https://github.com/OT-CONTAINER-KIT/redis-operator) (具体链接可能随时间变化)
*   **Kubernetes Operator 概念**: [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
*   **Redis Cluster 教程**: [Redis Cluster Tutorial](https://redis.io/docs/manual/scaling/)
*   **OperatorHub.io**: [发现更多 Kubernetes Operators](https://operatorhub.io/)

---