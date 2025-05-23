---
title: K8s采用Helm部署Kafka集群实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Kafka
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505130251467.png'
toc: true
abbrlink: 84c192a2
date: 2025-05-09 16:59:31
---

Apache Kafka 作为一款高性能、分布式的发布订阅消息系统，广泛应用于大数据、实时计算、日志收集等场景。在 Kubernetes (K8s) 环境中部署和管理 Kafka 集群，Helm 无疑是一个强大且便捷的工具。本文将详细介绍如何使用 Helm 在 K8s 上部署 Bitnami 提供的 Kafka Helm Chart，并通过**命令行参数覆盖**的方式进行定制化配置，辅以脚本实现自动化部署与管理。

**前提条件:**

1.  **已就绪的 Kubernetes 集群:** 确保你有一个正常运行的 K8s 集群。
2.  **Helm 已安装:** Helm V3 版本，并已配置好与 K8s 集群的连接。
3.  **kubectl 已安装:** 用于与 K8s 集群交互。
4.  **StorageClass 已配置:** 集群中需要有一个可用的 StorageClass 用于持久化存储。本文示例中将使用名为 `nfs-storage` 的 StorageClass。

<!-- more -->

### 步骤一：准备 Helm 环境和确认 Chart 版本

在开始之前，我们需要确保 Helm 仓库已添加并更新，以便获取最新的 Chart 信息。我们的部署脚本 (`install.sh`) 会自动处理这一步。

脚本中，我们指定了 Bitnami Kafka Chart 的版本为 `32.2.6`。你可以使用 `helm search repo bitnami/kafka --versions` 查看所有可用版本，并根据需要调整脚本中的版本号。

```shell
# install.sh 脚本中的相关命令
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

要确认 `nfs-storage` 这个 StorageClass 是否存在，可以执行：
```shell
kubectl get storageclass
```
输出应类似：
```
NAME                    PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   6d16h
```
如果你的 StorageClass 名称不同，请相应修改后续脚本中的 `global.defaultStorageClass` 值。

### 步骤二：理解并准备部署脚本 (`install.sh`)

之前我们通过修改 `values.yaml` 文件来定制 Kafka 集群。现在，我们将采用一种更直接和自动化的方式：使用 `helm install` 命令的 `--set` 或 `--set-string` 参数在部署时直接覆盖 Chart 的默认值。

以下是我们的 `install.sh` 脚本，它包含了部署 Kafka 集群所需的所有配置：

```shell
#!/usr/bin/env bash

set -x # 开启调试模式，打印执行的命令

# 添加 Bitnami Helm 仓库并更新
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 安装 Kafka 集群
helm install kafka-cluster bitnami/kafka --version 32.2.6 \
  --namespace kafka-cluster \
  --create-namespace \
  --set-string global.defaultStorageClass="nfs-storage" \
  --set listeners.client.protocol=PLAINTEXT \
  --set listeners.client.sslClientAuth=none \
  --set listeners.controller.protocol=PLAINTEXT \
  --set listeners.controller.sslClientAuth=none \
  --set listeners.interbroker.protocol=PLAINTEXT \
  --set listeners.interbroker.sslClientAuth=none \
  --set listeners.external.protocol=PLAINTEXT \
  --set listeners.external.sslClientAuth=none \
  --set controller.replicaCount=3 \
  --set controller.persistence.enabled=true \
  --set controller.persistence.size=16Gi \
  --set controller.logPersistence.enabled=true \
  --set controller.logPersistence.size=8Gi \
  --set metrics.jmx.enabled=true
  # 如果需要分离的 Broker 节点 (KRaft 提供的 Dedicated Broker Mode)，请取消注释并配置以下参数
  # --set broker.replicaCount=3 \
  # --set broker.persistence.enabled=true \
  # --set broker.persistence.size=16Gi \
  # --set broker.logPersistence.enabled=true \
  # --set broker.logPersistence.size=8Gi \

echo "Kafka cluster installation process initiated."
echo "Use 'kubectl get pods -n kafka-cluster -w' to monitor pod status."
echo "Or use './status.sh' to check all resources in the namespace."
```

**关键参数说明 (`--set` 和 `--set-string`)：**

*   `--set-string global.defaultStorageClass="nfs-storage"`: 全局指定默认的 StorageClass。使用 `--set-string` 确保 `"nfs-storage"` 被视为字符串。
*   `listeners.*.protocol=PLAINTEXT`: 将所有监听器的通信协议设置为 `PLAINTEXT`，禁用 SSL 加密。
*   `listeners.*.sslClientAuth=none`: 配合 `PLAINTEXT`，禁用 SSL 客户端认证。**这在测试或内部环境中可以简化部署，但在生产环境中强烈建议启用 SSL/SASL 以保证通信安全。**
*   `controller.replicaCount=3`: 设置 Controller 节点数量为3。在 KRaft 模式下，这些节点同时承担 Controller 和 Broker 的角色（如果 `broker.replicaCount` 未设置或为0，则为 Combined Mode）。
*   `controller.persistence.enabled=true`: 为 Controller 节点启用数据持久化。
*   `controller.persistence.size=16Gi`: 为每个 Controller 节点的数据卷分配 16Gi 存储。
*   `controller.logPersistence.enabled=true`: 为 Controller 节点启用 Kafka 自身运行日志的持久化。
*   `controller.logPersistence.size=8Gi`: 为每个 Controller 节点的日志卷分配 8Gi 存储。
*   **Broker 节点配置 (注释部分)**:
    *   `broker.replicaCount=3`: 如果你希望运行 KRaft 的 "Dedicated Mode"，即 Controller 节点和 Broker 节点分离，你需要取消注释并设置此参数。此时 `controller.replicaCount` 将只管理 Controller 角色的节点。
    *   `broker.persistence.*`: 相应地为 Broker 节点配置持久化。
    *   在当前脚本配置下（仅设置 `controller.replicaCount`），Kafka 将以 "Combined Mode" 运行，即 Controller 节点同时也是 Broker 节点。
*   `metrics.jmx.enabled=true`: 启用 Prometheus JMX exporter，用于收集 Kafka 集群的指标。

你可以将上述内容保存为 `install.sh` 文件。

### 步骤三：部署 Kafka 集群

在执行脚本之前，确保它有执行权限：

```shell
chmod +x install.sh
```

然后，运行脚本来部署 Kafka 集群：

```shell
./install.sh
```

部署命令执行后，Helm 会输出部署状态和一些有用的信息，类似于：

```
NAME: kafka-cluster
LAST DEPLOYED: Tue May 10 10:05:00 2025 # 时间会是你执行的时间
NAMESPACE: kafka-cluster
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 32.2.6
APP VERSION: 4.0.0 # App Version 可能会根据Chart版本变动

Did you know there are enterprise versions of the Bitnami catalog? For enhanced secure software supply chain features, unlimited pulls from Docker, LTS support, or application customization, see Bitnami Premium or Tanzu Application Catalog. See https://www.arrow.com/globalecs/na/vendors/bitnami for more information.

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka-cluster.kafka-cluster.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092
    kafka-cluster-controller-1.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092
    kafka-cluster-controller-2.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092
    # 如果你配置了独立的 broker 节点，这里会显示 broker 的 headless service 地址

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-cluster-client --restart='Never' --image docker.io/bitnami/kafka:4.0.0-debian-12-r3 --namespace kafka-cluster --command -- sleep infinity # Image tag 可能随 App Version 变化
    kubectl exec --tty -i kafka-cluster-client --namespace kafka-cluster -- bash

    PRODUCER:
        kafka-console-producer.sh \
            --bootstrap-server kafka-cluster.kafka-cluster.svc.cluster.local:9092 \
            --topic test

    CONSUMER:
        kafka-console-consumer.sh \
            --bootstrap-server kafka-cluster.kafka-cluster.svc.cluster.local:9092 \
            --topic test \
            --from-beginning

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - controller.resources
  # - broker.resources (if broker.replicaCount > 0)
  - defaultInitContainers.prepareConfig.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

### 步骤四：验证和测试 Kafka 集群

部署完成后，等待所有 Pod 变为 `Running` 状态。你可以使用以下命令监控 Pod 状态：

```shell
kubectl get pods -n kafka-cluster -w
```
或者使用我们准备的 `status.sh` 脚本查看所有相关资源：

**`status.sh` 脚本内容：**
```shell
#!/usr/bin/env bash

set -x

kubectl get all -n kafka-cluster
kubectl get pvc -n kafka-cluster # 同时查看持久卷声明的状态
```
保存为 `status.sh`，添加执行权限 (`chmod +x status.sh`)，然后运行 `./status.sh`。

当看到 `kafka-cluster-controller-0`, `kafka-cluster-controller-1`, `kafka-cluster-controller-2` (以及可能的 `kafka-cluster-broker-*` Pods，如果你配置了它们) 都处于 `Running` 状态时，集群基本就绪。

根据 Helm 输出的 `NOTES` 部分，我们可以启动一个临时的 Kafka 客户端 Pod 来进行测试：

1.  **启动客户端 Pod：** (请根据 Helm NOTES 中的镜像版本调整，如果不同)
    ```shell
    kubectl run kafka-cluster-client --restart='Never' --image docker.io/bitnami/kafka:4.0.0-debian-12-r3 --namespace kafka-cluster --command -- sleep infinity
    ```
2.  **进入客户端 Pod 的 shell：**
    ```shell
    kubectl exec --tty -i kafka-cluster-client --namespace kafka-cluster -- bash
    ```
3.  **在客户端 Pod 内，创建一个测试topic：**
    打开一个终端执行 `kubectl exec ...` 进入客户端 Pod 后，运行：
    ```shell
    # bootstrap-server 地址优先使用 headless service 的具体 pod 地址进行测试
    kafka-topics.sh \
    --create \
    --bootstrap-server kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092 \
    --topic lbs \
    --partitions 6 \
    --replication-factor 3
    ```
4.  **在客户端 Pod 内，启动生产者发送消息：**
    打开一个终端执行 `kubectl exec ...` 进入客户端 Pod 后，运行：
    ```shell
    # bootstrap-server 地址优先使用 headless service 的具体 pod 地址进行测试
    kafka-console-producer.sh \
        --bootstrap-server kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092 \
        --topic test
    ```
    然后输入一些消息，例如：
    ```
    >Hello Kafka from Script
    >This is a test message via Helm set
    ```
5.  **在客户端 Pod 内，启动消费者接收消息：**
    再打开一个新的终端，同样执行 `kubectl exec ...` 进入同一个客户端 Pod 后，运行：
    ```shell
    kafka-console-consumer.sh \
        --bootstrap-server kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092 \
        --topic test \
        --from-beginning
    ```
    如果一切正常，你应该能看到之前生产者发送的消息。

### 步骤五：卸载 Kafka 集群

如果需要卸载 Kafka 集群，可以使用提供的 `uninstall.sh` 脚本：

**`uninstall.sh` 脚本内容：**
```shell
#!/usr/bin/env bash

set -x

helm uninstall kafka-cluster -n kafka-cluster

echo "Kafka cluster uninstallation initiated."
echo "Persistent Volume Claims (PVCs) might need to be manually deleted if reclaimPolicy is not Delete."
echo "Check with: kubectl get pvc -n kafka-cluster"
```
保存为 `uninstall.sh`，添加执行权限 (`chmod +x uninstall.sh`)，然后运行：
```shell
./uninstall.sh
```
这将删除由 Helm Chart创建的所有 Kubernetes 资源。注意：根据 StorageClass 的 `reclaimPolicy`，PVCs 可能不会被自动删除。如果 `reclaimPolicy` 是 `Retain`，你需要手动删除 PVCs 以释放存储空间。

### 重要提示和后续步骤

*   **资源限制 (Resources Warning):** Helm 的输出提示中有一个 `WARNING` 指出 `"resources" sections in the chart not set`。这意味着 Pod 的资源请求 (requests) 和限制 (limits) 没有明确设置。**在生产环境中，强烈建议根据您的工作负载需求，在 `install.sh` 脚本中通过 `--set controller.resources.requests.cpu=...` 等参数为相关组件配置明确的CPU和内存资源。** 例如：
    ```shell
    # 示例：为Controller设置资源请求和限制
    --set controller.resources.requests.cpu=500m \
    --set controller.resources.requests.memory=1Gi \
    --set controller.resources.limits.cpu=1 \
    --set controller.resources.limits.memory=2Gi \
    ```
*   **安全性:** 本指南为了简化部署，禁用了 SSL/SASL。在生产环境中，务必启用这些安全特性，并配置 ACLs (Access Control Lists) 来控制对 Topic 的访问。这需要修改 `listeners.*.protocol` 为 `SASL_SSL` 或 `SSL`，并配置相关的证书和认证参数。
*   **外部访问:** 如果需要从 K8s集群外部访问 Kafka，你需要配置 `listeners.external` 相关的服务类型（如 `LoadBalancer` 或 `NodePort`）和参数。Bitnami Kafka Chart 提供了这些配置选项，可以通过 `--set listeners.external.service.type=LoadBalancer` 等方式设置。
*   **KRaft 模式:** Bitnami Kafka Chart 的较新版本默认使用 KRaft 模式 (Kafka Raft Metadata mode)，不再依赖 ZooKeeper。`install.sh` 的配置是基于 KRaft 模式的。如果你的 Chart 版本或需求不同，请查阅相应 Chart 文档。
*   **脚本化管理的优势:** 使用脚本进行部署和卸载，可以确保操作的一致性和可重复性，便于版本控制和自动化集成。

### 总结

通过 Helm 结合命令行参数覆盖，并将其封装在shell脚本中，我们可以实现对 Kubernetes 上 Kafka 集群的快速、规范且自动化的部署和管理。通过调整脚本中的 `--set` 参数，可以灵活地定制 Kafka 集群的各项配置，以满足不同环境和业务的需求。记住，在生产部署前，务必仔细评估安全配置、资源分配和高可用性策略。

---
