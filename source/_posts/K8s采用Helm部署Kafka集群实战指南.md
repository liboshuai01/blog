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

Apache Kafka 作为一款高性能、分布式的发布订阅消息系统，广泛应用于大数据、实时计算、日志收集等场景。在 Kubernetes (K8s) 环境中部署和管理 Kafka 集群，Helm 无疑是一个强大且便捷的工具。本文将详细介绍如何使用 Helm 在 K8s 上部署 Bitnami 提供的 Kafka Helm Chart，并通过**命令行参数覆盖**的方式进行定制化配置，辅以脚本实现自动化部署与管理，并集成 Prometheus ServiceMonitor 进行监控。

**前提条件:**

1.  **已就绪的 Kubernetes 集群:** 确保你有一个正常运行的 K8s 集群。
2.  **Helm 已安装:** Helm V3 版本，并已配置好与 K8s 集群的连接。
3.  **kubectl 已安装:** 用于与 K8s 集群交互。
4.  **StorageClass 已配置:** 集群中需要有一个可用的 StorageClass 用于持久化存储。本文示例中将使用名为 `nfs-storage` 的 StorageClass。
5.  **(可选但推荐) Prometheus Stack 已安装:** 如果您希望使用 ServiceMonitor 自动发现 Kafka JMX 指标，确保您的集群中已安装 `kube-prometheus-stack` 或类似的 Prometheus Operator 解决方案。脚本中的 `PROM_STACK_RELEASE_NAME` 和 `PROM_STACK_NAMESPACE` 需要与您的 Prometheus 安装相匹配。

<!-- more -->

### 步骤一：准备 Helm 环境和确认 Chart 版本

在开始之前，我们需要确保 Helm 仓库已添加并更新，以便获取最新的 Chart 信息。我们的部署脚本 (`install.sh`) 会自动处理这一步。

脚本中，我们将指定 Bitnami Kafka Chart 的版本。在新的 `install.sh` 脚本中，我们使用变量 `KAFKA_CHART_VERSION` 来定义版本，默认为 `32.2.8`。你可以使用 `helm search repo bitnami/kafka --versions` 查看所有可用版本，并根据需要调整脚本中的版本号。

```shell
# install.sh 脚本中的相关命令 (在脚本执行时会自动运行)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

要确认您希望使用的 StorageClass (脚本中默认为 `nfs-storage`) 是否存在，可以执行：
```shell
kubectl get storageclass
```
输出应类似：
```
NAME                    PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   6d16h
```
如果你的 StorageClass 名称不同，请相应修改脚本中的 `STORAGE_CLASS` 变量值。

### 步骤二：理解并准备部署脚本 (`install.sh`)

之前我们通过修改 `values.yaml` 文件来定制 Kafka 集群。现在，我们将采用一种更直接和自动化的方式：使用 `helm install` 命令的 `--set` 或 `--set-string` 参数在部署时直接覆盖 Chart 的默认值，并通过脚本变量进行管理。

以下是我们的 `install.sh` 脚本，它包含了部署 Kafka 集群所需的所有配置，并集成了 Prometheus ServiceMonitor：

```shell
#!/usr/bin/env bash

set -x # 开启调试模式，打印执行的命令

# 添加 Bitnami Helm 仓库并更新
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# --- 配置变量 ---
# Kafka 安装相关
KAFKA_RELEASE_NAME="kafka-cluster"
KAFKA_NAMESPACE="kafka-cluster"
KAFKA_CHART_VERSION="32.2.8" # 请确认这是您希望使用的稳定版本
STORAGE_CLASS="nfs-storage"

# Prometheus Stack 的 Release 名称 (必须与你安装 kube-prometheus-stack 时使用的 RELEASE_NAME 一致)
PROM_STACK_RELEASE_NAME="kube-prom-stack"
PROM_STACK_NAMESPACE="monitoring" # ServiceMonitor将创建在这个命名空间

# --- 安装 Kafka 集群 ---
helm install ${KAFKA_RELEASE_NAME} bitnami/kafka --version ${KAFKA_CHART_VERSION} \
  --namespace ${KAFKA_NAMESPACE} \
  --create-namespace \
  \
  --set-string global.defaultStorageClass="${STORAGE_CLASS}" \
  \
  --set listeners.client.protocol=PLAINTEXT \
  --set listeners.client.sslClientAuth=none \
  --set listeners.controller.protocol=PLAINTEXT \
  --set listeners.controller.sslClientAuth=none \
  --set listeners.interbroker.protocol=PLAINTEXT \
  --set listeners.interbroker.sslClientAuth=none \
  --set listeners.external.protocol=PLAINTEXT \
  --set listeners.external.sslClientAuth=none \
  \
  --set controller.replicaCount=3 \
  --set controller.persistence.enabled=true \
  --set controller.persistence.size=16Gi \
  --set controller.logPersistence.enabled=true \
  --set controller.logPersistence.size=8Gi \
  \
  --set metrics.jmx.enabled=true \
  --set metrics.serviceMonitor.enabled=true \
  --set metrics.serviceMonitor.namespace=${PROM_STACK_NAMESPACE} \
  --set metrics.serviceMonitor.labels.release=${PROM_STACK_RELEASE_NAME}
  # 如果需要分离的 Broker 节点 (KRaft 提供的 Dedicated Broker Mode)，请取消注释并配置以下参数
  # --set broker.replicaCount=3 \
  # --set broker.persistence.enabled=true \
  # --set broker.persistence.size=16Gi \
  # --set broker.logPersistence.enabled=true \
  # --set broker.logPersistence.size=8Gi \

echo ""
echo "Kafka 集群 (${KAFKA_RELEASE_NAME}) 安装/升级过程已启动到命名空间 '${KAFKA_NAMESPACE}'。"
echo "JMX metrics 已启用。"
echo "ServiceMonitor 将创建在命名空间 '${PROM_STACK_NAMESPACE}' 中，并带有标签 'release: ${PROM_STACK_RELEASE_NAME}'。"
echo "---------------------------------------------------------------------"
echo "监控 Pod 状态: kubectl get pods -n ${KAFKA_NAMESPACE} -w"
echo "检查 Service (JMX metrics): kubectl get svc -n ${KAFKA_NAMESPACE} | grep jmx"
echo "检查 ServiceMonitor: kubectl get servicemonitor -n ${PROM_STACK_NAMESPACE} ${KAFKA_RELEASE_NAME}-jmx-metrics"
echo "检查 Prometheus Targets: 访问 Prometheus UI 的 Targets 页面 (例如 http://<prometheus-host>/targets)"
echo "---------------------------------------------------------------------"
echo "如果遇到问题, 请检查 Prometheus Operator 日志: kubectl logs -n ${PROM_STACK_NAMESPACE} -l app.kubernetes.io/name=prometheus-operator -c prometheus-operator"
```

**脚本变量和关键参数说明：**

*   **脚本变量:**
    *   `KAFKA_RELEASE_NAME`: Helm Release 的名称，默认为 `kafka-cluster`。
    *   `KAFKA_NAMESPACE`: Kafka 集群部署的命名空间，默认为 `kafka-cluster`。
    *   `KAFKA_CHART_VERSION`: Bitnami Kafka Chart 的版本，默认为 `32.2.8`。
    *   `STORAGE_CLASS`: 用于持久化存储的 StorageClass 名称，默认为 `nfs-storage`。
    *   `PROM_STACK_RELEASE_NAME`: 您环境中 `kube-prometheus-stack` 的 Helm Release 名称。**请务必修改此变量以匹配您的实际环境**，否则 ServiceMonitor 可能不会被 Prometheus Operator 正确识别。
    *   `PROM_STACK_NAMESPACE`: `kube-prometheus-stack` 部署的命名空间，ServiceMonitor CRD 将创建于此，默认为 `monitoring`。

*   `--set-string global.defaultStorageClass="${STORAGE_CLASS}"`: 全局指定默认的 StorageClass。使用 `--set-string` 确保其被视为字符串。
*   `listeners.*.protocol=PLAINTEXT` 和 `listeners.*.sslClientAuth=none`: 将所有监听器的通信协议设置为 `PLAINTEXT` 并禁用 SSL。**这在测试或内部环境中可以简化部署，但在生产环境中强烈建议启用 SSL/SASL 以保证通信安全。**
*   `controller.replicaCount=3`: 设置 Controller 节点数量为3 (KRaft 模式下，若不分离 Broker，则这些节点同时扮演 Controller 和 Broker 角色)。
*   `controller.persistence.*` 和 `controller.logPersistence.*`: 为 Controller (及 Combined Mode下的 Broker) 节点启用并配置数据和日志的持久化。
*   **Broker 节点配置 (注释部分)**: 用于 KRaft "Dedicated Mode"，即 Controller 和 Broker 节点分离。
*   `metrics.jmx.enabled=true`: 启用 JMX Exporter，暴露 Kafka 指标。
*   `metrics.serviceMonitor.enabled=true`: 启用 ServiceMonitor 资源的创建。
*   `metrics.serviceMonitor.namespace=${PROM_STACK_NAMESPACE}`: 指定 ServiceMonitor 资源创建在 Prometheus Operator 所在的命名空间（通常是 `monitoring`）。
*   `metrics.serviceMonitor.labels.release=${PROM_STACK_RELEASE_NAME}`: 为 ServiceMonitor 添加标签，通常 Prometheus Operator 会使用这个标签 (例如 `release=<prometheus-stack-release-name>`) 来发现和匹配 ServiceMonitor。**确保 `${PROM_STACK_RELEASE_NAME}` 与您安装 `kube-prometheus-stack` 时使用的 `release` 名称一致。**

你可以将上述内容保存为 `install.sh` 文件。

### 步骤三：部署 Kafka 集群

在执行脚本之前，确保它有执行权限：

```shell
chmod +x install.sh
```

然后，**根据您的环境修改 `install.sh` 脚本中的 `PROM_STACK_RELEASE_NAME` 和 `PROM_STACK_NAMESPACE` (如果需要)**，然后运行脚本来部署 Kafka 集群：

```shell
./install.sh
```

部署命令执行后，Helm 会输出部署状态和一些有用的信息。脚本末尾也会打印出检查 Kafka 和 ServiceMonitor 状态的常用命令。
Helm 输出的 `NOTES` 部分也会包含 Kafka 客户端如何连接和测试集群的信息，例如：

```
NAME: kafka-cluster
LAST DEPLOYED: Wed May 10 10:05:00 2025 # 时间会是你执行的时间
NAMESPACE: kafka-cluster
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 32.2.8
APP VERSION: 4.0.0 # App Version 可能会根据Chart版本变动

Did you know there are enterprise versions of the Bitnami catalog? For enhanced secure software supply chain features, unlimited pulls from Docker, LTS support, or application customization, see Bitnami Premium or Tanzu Application Catalog. See https://www.arrow.com/globalecs/na/vendors/bitnami for more information.

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka-cluster.kafka-cluster.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-cluster.kafka-cluster.svc.cluster.local:9092
    kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092
    # ... (其他 controller/broker headless service 地址)

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

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production...
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

### 步骤四：验证和测试 Kafka 集群

部署完成后，等待所有 Pod 变为 `Running` 状态。你可以使用脚本输出中提示的命令或我们准备的 `status.sh` 脚本查看所有相关资源：

**`status.sh` 脚本内容：**
```shell
#!/usr/bin/env bash

set -x

# 使用 install.sh 中定义的变量 (如果 status.sh 与 install.sh 在同一目录或已 source install.sh)
# 如果独立运行，请直接替换变量或在此处定义它们
KAFKA_NAMESPACE="kafka-cluster"
KAFKA_RELEASE_NAME="kafka-cluster" # 与 install.sh 中的 KAFKA_RELEASE_NAME 保持一致
PROM_STACK_NAMESPACE="monitoring" # 与 install.sh 中的 PROM_STACK_NAMESPACE 保持一致

echo "--- Pods in ${KAFKA_NAMESPACE} ---"
kubectl get pods -n ${KAFKA_NAMESPACE} -w # -w 用于持续监控，按Ctrl+C退出

echo "--- All resources in ${KAFKA_NAMESPACE} ---"
kubectl get all -n ${KAFKA_NAMESPACE}

echo "--- PVCs in ${KAFKA_NAMESPACE} ---"
kubectl get pvc -n ${KAFKA_NAMESPACE}

echo "--- JMX Service in ${KAFKA_NAMESPACE} ---"
kubectl get svc -n ${KAFKA_NAMESPACE} | grep jmx

echo "--- ServiceMonitor in ${PROM_STACK_NAMESPACE} for ${KAFKA_RELEASE_NAME} ---"
kubectl get servicemonitor -n ${PROM_STACK_NAMESPACE} ${KAFKA_RELEASE_NAME}-jmx-metrics -o yaml
# (ServiceMonitor 的名称可能为 ${KAFKA_RELEASE_NAME}-metrics 或其他，具体取决于 Chart)
# 针对 Bitnami Kafka Chart 32.2.8, ServiceMonitor 名称默认为 ${KAFKA_RELEASE_NAME}-jmx-metrics

echo "--- Check Prometheus Targets page for Kafka endpoints ---"
echo "Visit your Prometheus UI (e.g., via port-forward or Ingress) and check the 'Targets' page."

echo "--- If ServiceMonitor issues, check Prometheus Operator logs in ${PROM_STACK_NAMESPACE} ---"
kubectl logs -n ${PROM_STACK_NAMESPACE} -l app.kubernetes.io/name=prometheus-operator -c prometheus-operator --tail=100
```
保存为 `status.sh`，添加执行权限 (`chmod +x status.sh`)，然后运行 `./status.sh`。

当看到 `kafka-cluster-controller-0`, `kafka-cluster-controller-1`, `kafka-cluster-controller-2` 等 Pods 都处于 `Running` 状态时，集群基本就绪。

**测试 Kafka 功能：** (根据 Helm NOTES 中的镜像版本调整，如果不同)
1.  **启动客户端 Pod：**
    ```shell
    # 请根据您helm install输出的NOTES部分提供的镜像名和标签进行调整
    kubectl run kafka-cluster-client --restart='Never' --image docker.io/bitnami/kafka:4.0.0-debian-12-r3 --namespace kafka-cluster --command -- sleep infinity
    ```
2.  **进入客户端 Pod 的 shell：**
    ```shell
    kubectl exec --tty -i kafka-cluster-client --namespace kafka-cluster -- bash
    ```
3.  **在客户端 Pod 内，创建一个测试topic：**
    ```shell
    kafka-topics.sh \
        --create \
        --bootstrap-server kafka-cluster.kafka-cluster.svc.cluster.local:9092 \
        --topic test \
        --partitions 6 \
        --replication-factor 3 # 对于3节点的combined mode集群，副本因子最大为3
    ```
4.  **在客户端 Pod 内，启动生产者发送消息：**
    ```shell
    kafka-console-producer.sh \
        --bootstrap-server kafka-cluster.kafka-cluster.svc.cluster.local:9092 \
        --topic test
    ```
    输入消息: `>Hello Kafka from New Script`
5.  **在客户端 Pod 内，启动消费者接收消息 (新开一个终端执行 kubectl exec...)：**
    ```shell
    kafka-console-consumer.sh \
        --bootstrap-server kafka-cluster.kafka-cluster.svc.cluster.local:9092 \
        --topic test \
        --from-beginning
    ```
    你应该能看到发送的消息。

**验证 JMX 指标和 ServiceMonitor：**
*   执行 `./status.sh` 或 `install.sh` 脚本输出的检查命令。
*   确保 `kubectl get servicemonitor -n ${PROM_STACK_NAMESPACE} ${KAFKA_RELEASE_NAME}-jmx-metrics` 能够找到 ServiceMonitor。
*   在 Prometheus UI 的 "Status" -> "Targets" 页面，你应该能看到 Kafka 相关的 Endpoints，并且状态是 UP。

### 步骤五：卸载 Kafka 集群

如果需要卸载 Kafka 集群，可以使用提供的 `uninstall.sh` 脚本。我们对其进行修改以使用与 `install.sh` 中相同的变量：

**`uninstall.sh` 脚本内容：**
```shell
#!/usr/bin/env bash

set -x

# --- 配置变量 (与 install.sh 保持一致) ---
KAFKA_RELEASE_NAME="kafka-cluster"
KAFKA_NAMESPACE="kafka-cluster"
# PROM_STACK_NAMESPACE="monitoring" # ServiceMonitor 会随 Helm release 一起删除

helm uninstall ${KAFKA_RELEASE_NAME} -n ${KAFKA_NAMESPACE}

echo ""
echo "Kafka cluster (${KAFKA_RELEASE_NAME}) uninstallation initiated from namespace '${KAFKA_NAMESPACE}'."
echo "The ServiceMonitor '${KAFKA_RELEASE_NAME}-jmx-metrics' in namespace '${PROM_STACK_NAMESPACE}' (if it existed and was managed by this chart) should also be deleted."
echo "Persistent Volume Claims (PVCs) might need to be manually deleted if reclaimPolicy is not Delete."
echo "Check with: kubectl get pvc -n ${KAFKA_NAMESPACE}"
echo "Also verify ServiceMonitor deletion: kubectl get servicemonitor -n ${PROM_STACK_NAMESPACE} ${KAFKA_RELEASE_NAME}-jmx-metrics"
```
保存为 `uninstall.sh`，添加执行权限 (`chmod +x uninstall.sh`)，然后运行：
```shell
./uninstall.sh
```
这将删除由 Helm Chart创建的所有 Kubernetes 资源，包括 ServiceMonitor。注意：根据 StorageClass 的 `reclaimPolicy`，PVCs 可能不会被自动删除。

### 重要提示和后续步骤

*   **资源限制 (Resources Warning):** Helm 的输出提示中有一个 `WARNING` 指出 `"resources" sections in the chart not set`。在生产环境中，强烈建议根据您的工作负载需求，在 `install.sh` 脚本中通过 `--set controller.resources.requests.cpu=...` 等参数为相关组件配置明确的CPU和内存资源。
*   **Prometheus 配置:** 确保 `install.sh`中的 `PROM_STACK_RELEASE_NAME` 和 `PROM_STACK_NAMESPACE` 与您环境中 `kube-prometheus-stack` (或其他 Prometheus Operator 实现) 的配置完全匹配，否则 ServiceMonitor 不会被 Prometheus 自动发现。
*   **安全性:** 本指南为了简化部署，禁用了 SSL/SASL。在生产环境中，务必启用这些安全特性。
*   **外部访问:** 如需从 K8s 集群外部访问 Kafka，需配置 `listeners.external`。
*   **KRaft 模式:** Bitnami Kafka Chart 较新版本默认使用 KRaft 模式。
*   **脚本化管理的优势:** 使用脚本进行部署和卸载，可以确保操作的一致性、可重复性，便于版本控制和自动化集成。

### 总结

通过 Helm 结合命令行参数覆盖，并将其封装在shell脚本中，我们可以实现对 Kubernetes 上 Kafka 集群的快速、规范且自动化的部署和管理，同时还能轻松集成 Prometheus 监控。通过调整脚本中的变量和 `--set` 参数，可以灵活地定制 Kafka 集群的各项配置。记住，在生产部署前，务必仔细评估安全配置、资源分配、监控集成和高可用性策略。

---
