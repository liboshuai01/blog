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

Apache Kafka 作为一款高性能、分布式的发布订阅消息系统，广泛应用于大数据、实时计算、日志收集等场景。在 Kubernetes (K8s) 环境中部署和管理 Kafka 集群，Helm 无疑是一个强大且便捷的工具。本文将详细介绍如何使用 Helm 在 K8s 上部署 Bitnami 提供的 Kafka Helm Chart，并根据实际需求进行定制化配置。

**前提条件:**

1.  **已就绪的 Kubernetes 集群:** 确保你有一个正常运行的 K8s 集群。
2.  **Helm 已安装:** Helm V3 版本，并已配置好与 K8s 集群的连接。
3.  **kubectl 已安装:** 用于与 K8s 集群交互。
4.  **StorageClass 已配置:** 集群中需要有一个可用的 StorageClass 用于持久化存储。本文示例中将使用名为 `nfs-storage` 的 StorageClass。

<!-- more -->

### 步骤一：搜索并选择 Kafka Helm Chart

首先，我们需要在 Helm 仓库中搜索可用的 Kafka Chart。Bitnami 维护了许多高质量的 Helm Chart，包括 Kafka。

```shell
[root@master helm]# helm search repo kafka
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
bitnami/kafka                   32.2.2          4.0.0           Apache Kafka is a distributed streaming platfor...
bitnami/dataplatform-bp2        12.0.5          1.0.1           DEPRECATED This Helm chart can be used for the ...
bitnami/schema-registry         25.1.2          7.9.0           Confluent Schema Registry provides a RESTful in...
```

我们可以看到 `bitnami/kafka` 是一个不错的选择。为了确保我们使用的是一个稳定且经过验证的版本，可以进一步查看其所有可用版本：

```shell
[root@master helm]# helm search repo bitnami/kafka --versions
NAME            CHART VERSION   APP VERSION     DESCRIPTION
bitnami/kafka   32.2.2          4.0.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   32.2.1          4.0.0           Apache Kafka is a distributed streaming platfor...
bitnami/kafka   32.2.0          4.0.0           Apache Kafka is a distributed streaming platfor...
......
......
......
```
本文选择 `32.2.2` 版本进行部署。

### 步骤二：拉取并解压 Helm Chart

选定版本后，使用 `helm pull` 命令将 Chart 包下载到本地：

```shell
[root@master helm]# helm pull bitnami/kafka --version 32.2.2
```

下载完成后，会得到一个 `kafka-32.2.2.tgz` 文件。解压并重命名文件夹，方便后续操作：

```shell
[root@master helm]# tar -zxvf kafka-32.2.2.tgz && rm -rf kafka-32.2.2.tgz && mv kafka kafka-cluster
kafka/
kafka/templates/
kafka/templates/controller-eligible/
......
......
......
```

### 步骤三：准备并修改 `values.yaml` 配置文件

在部署之前，我们需要根据集群环境和需求修改 Chart 的默认配置。首先，检查当前 K8s 集群可用的 StorageClass：

```shell
[root@master helm]# kubectl get storageclass
NAME                    PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)   cluster.local/nfs-subdir-external-provisioner   Delete          Immediate           true                   6d16h
```
可以看到，我们有一个名为 `nfs-storage` 的 StorageClass。

接下来，备份默认的 `values.yaml` 文件，然后编辑它：

```shell
[root@master helm]# cp kafka-cluster/values.yaml kafka-cluster/values.yaml.template
[root@master helm]# vim kafka-cluster/values.yaml
```

以下是关键的修改内容及说明 (基于您提供的 `values.yaml` 修改)：

```yaml
# Copyright Broadcom, Inc. All Rights Reserved.
# SPDX-License-Identifier: APACHE-2.0

## @section Global parameters
global:
  imageRegistry: ""
  imagePullSecrets: []
  defaultStorageClass: "nfs-storage" # 修复设置存储类: 全局指定默认的StorageClass

## Kafka listeners configuration
listeners:
  client:
    containerPort: 9092
    protocol: PLAINTEXT # 修改设置为PLAINTEXT: 客户端连接协议，取消认证简化测试
    name: CLIENT
    sslClientAuth: none # 修改设置为none: 取消SSL客户端认证
  controller:
    name: CONTROLLER
    containerPort: 9093
    protocol: PLAINTEXT # 修改设置为PLAINTEXT: 控制器间通信协议，取消认证
    sslClientAuth: none # 修改设置为none: 取消SSL客户端认证
  interbroker:
    containerPort: 9094
    protocol: PLAINTEXT # 修改设置为PLAINTEXT: Broker间通信协议，取消认证
    name: INTERNAL
    sslClientAuth: none # 修改设置为none: 取消SSL客户端认证
  external: # 如果需要外部访问，请确保配置正确并暴露服务
    containerPort: 9095
    protocol: PLAINTEXT # 修改设置为PLAINTEXT: 外部访问协议，取消认证
    name: EXTERNAL
    sslClientAuth: none # 修改设置为none: 取消SSL客户端认证

## @section Controller-eligible statefulset parameters
controller:
  replicaCount: 3 # Kafka Controller 节点数量，推荐至少3个以保证高可用
  # ... 其他controller配置 ...
  persistence:
    enabled: true # 启动数据持久化
    existingClaim: ""
    storageClass: "nfs-storage" # 修复设置存储类: 为Controller节点数据指定StorageClass
    accessModes:
      - ReadWriteOnce
    size: 16Gi # 修复设置存储大小: 每个节点的数据存储大小
    mountPath: /bitnami/kafka
  logPersistence: # Kafka 日志持久化配置 (可选)
    enabled: false # 示例中未启用日志持久化，可根据需要开启
    existingClaim: ""
    storageClass: "nfs-storage" # 修复设置存储类: 如果启用，为日志指定StorageClass
    accessModes:
      - ReadWriteOnce
    size: 8Gi # 修复设置存储大小: 如果启用，日志存储大小
    mountPath: /opt/bitnami/kafka/logs

# 生产环境可以额外修改如下配置，启动“Dedicated Mode - 控制器和代理分离模式”（测试环境可以忽略，仅启动controller，则为“Combined Mode - 控制器和代理合并模式”）
## @section Broker-only statefulset parameters
##
broker:
  ## @param broker.replicaCount Number of Kafka broker-only nodes
  ##
  replicaCount: 3 # Kafka Broker 节点数量
  # ... 其他broker配置 ...
  persistence:
    enabled: true
    existingClaim: ""
    storageClass: "nfs-storage" # 修复设置存储类
    accessModes:
      - ReadWriteOnce
    size: 16Gi # 修改设置存储大小
    annotations: {}
    labels: {}
    selector: {}
    mountPath: /bitnami/kafka
  logPersistence:
    enabled: true
    existingClaim: ""
    storageClass: "nfs-storage" # 修复设置存储类
    accessModes:
      - ReadWriteOnce
    size: 8Gi  # 修改设置存储大小
    annotations: {}
    selector: {}
    mountPath: /opt/bitnami/kafka/logs
  
## Prometheus Exporters / Metrics
##
metrics:
  ## Prometheus JMX exporter: exposes the majority of Kafka metrics
  ##
  jmx:
    ## @param metrics.jmx.enabled Whether or not to expose JMX metrics to Prometheus
    ##
    enabled: true # 启用 Prometheus JMX exporter
    
# ... 其他配置 ...
```

**关键修改点说明：**

1.  **`global.defaultStorageClass`**: 设置为 `nfs-storage`，确保所有未显式指定 StorageClass 的 PVC都将使用此值。
2.  **`listeners.*.protocol`**: 全部修改为 `PLAINTEXT`。这意味着 Kafka 集群内部以及客户端连接将不使用 SSL 加密和 SASL 认证。**这在测试或内部环境中可以简化部署，但在生产环境中强烈建议启用 SSL/SASL 以保证通信安全。**
3.  **`listeners.*.sslClientAuth`**: 全部修改为 `none`，配合 `PLAINTEXT` 协议。
4.  **`controller.replicaCount`**: 设置为 `3`，表示部署3个 Controller 节点（在 KRaft 模式下，Controller 和 Broker 角色可以合并）。
5.  **`controller.persistence.enabled`**: 设置为 `true`，启用数据持久化。
6.  **`controller.persistence.storageClass`**: 设置为 `nfs-storage`，为 Kafka 数据指定使用的 StorageClass。
7.  **`controller.persistence.size`**: 设置为 `8Gi`，为每个 Kafka 节点的数据卷大小。
8.  **`controller.logPersistence`**: 示例中 `enabled` 为 `false`，但我们仍为其配置了 `storageClass` 和 `size`。如果需要持久化 Kafka 自身运行日志，可以将其 `enabled` 设置为 `true`。
9.  **`metrics.jmx.enabled`**: 设置为`true`，启用 Prometheus JMX exporter，用于收集 Kafka 集群的指标。

### 步骤四：部署 Kafka 集群

配置文件修改完毕后，使用 `helm install` 命令部署 Kafka 集群。我们将在名为 `kafka-cluster` 的 namespace 中创建它（如果 namespace 不存在，`--create-namespace` 会自动创建）。

```shell
[root@master helm]# helm install kafka-cluster ./kafka-cluster -n kafka-cluster --create-namespace
```

部署命令执行后，Helm 会输出部署状态和一些有用的信息：

```
NAME: kafka-cluster
LAST DEPLOYED: Tue May 13 02:40:10 2025
NAMESPACE: kafka-cluster
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: kafka
CHART VERSION: 32.2.2
APP VERSION: 4.0.0

Did you know there are enterprise versions of the Bitnami catalog? For enhanced secure software supply chain features, unlimited pulls from Docker, LTS support, or application customization, see Bitnami Premium or Tanzu Application Catalog. See https://www.arrow.com/globalecs/na/vendors/bitnami for more information.

** Please be patient while the chart is being deployed **

Kafka can be accessed by consumers via port 9092 on the following DNS name from within your cluster:

    kafka-cluster.kafka-cluster.svc.cluster.local

Each Kafka broker can be accessed by producers via port 9092 on the following DNS name(s) from within your cluster:

    kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092
    kafka-cluster-controller-1.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092
    kafka-cluster-controller-2.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092

To create a pod that you can use as a Kafka client run the following commands:

    kubectl run kafka-cluster-client --restart='Never' --image docker.io/bitnami/kafka:4.0.0-debian-12-r3 --namespace kafka-cluster --command -- sleep infinity
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
  - defaultInitContainers.prepareConfig.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```

### 步骤五：验证和测试 Kafka 集群

部署完成后，等待所有 Pod 变为 `Running` 状态：

```shell
kubectl get pods -n kafka-cluster -w
```
当看到 `kafka-cluster-controller-0`, `kafka-cluster-controller-1`, `kafka-cluster-controller-2` 都处于 `Running` 状态时，集群基本就绪。

根据 Helm 输出的 `NOTES` 部分，我们可以启动一个临时的 Kafka 客户端 Pod 来进行测试：

1.  **启动客户端 Pod：**
    ```shell
    kubectl run kafka-cluster-client --restart='Never' --image docker.io/bitnami/kafka:4.0.0-debian-12-r3 --namespace kafka-cluster --command -- sleep infinity
    ```
2.  **进入客户端 Pod 的 shell：**
    ```shell
    kubectl exec --tty -i kafka-cluster-client --namespace kafka-cluster -- bash
    ```
3.  **在客户端 Pod 内，启动生产者发送消息：**
    打开一个终端执行 `kubectl exec ...` 进入客户端 Pod 后，运行：
    ```shell
    kafka-console-producer.sh \
        --bootstrap-server kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092 \
        --topic test
    ```
    然后输入一些消息，例如：
    ```
    >Hello Kafka
    >This is a test message
    ```
4.  **在客户端 Pod 内，启动消费者接收消息：**
    再打开一个新的终端，同样执行 `kubectl exec ...` 进入同一个客户端 Pod 后，运行：
    ```shell
    kafka-console-consumer.sh \
        --bootstrap-server kafka-cluster-controller-0.kafka-cluster-controller-headless.kafka-cluster.svc.cluster.local:9092 \
        --topic test \
        --from-beginning
    ```
    如果一切正常，你应该能看到之前生产者发送的消息：
    ```
    Hello Kafka
    This is a test message
    ```

### 重要提示和后续步骤

*   **资源限制 (Resources Warning):** Helm 的输出提示中有一个 `WARNING` 指出 `"resources" sections in the chart not set`。这意味着 Pod 的资源请求 (requests) 和限制 (limits) 没有明确设置，使用的是 Chart 的预设值或 Kubernetes 的默认值。**在生产环境中，强烈建议根据您的工作负载需求，在 `values.yaml` 中为 `controller.resources` 和 `defaultInitContainers.prepareConfig.resources` 等配置明确的CPU和内存资源。**
*   **安全性:** 本指南为了简化部署，禁用了 SSL/SASL。在生产环境中，务必启用这些安全特性，并配置 ACLs (Access Control Lists) 来控制对 Topic 的访问。
*   **外部访问:** 如果需要从 K8s集群外部访问 Kafka，你需要配置 `listeners.external` 并通过 `LoadBalancer`、`NodePort` 或 `Ingress` 类型的 Service 暴露 Kafka 服务。Bitnami Kafka Chart 提供了这些配置选项。
*   **监控:** 集成 Prometheus 和 Grafana 等监控工具，对 Kafka 集群的性能指标进行监控。
*   **Zookeeper vs KRaft:** Bitnami Kafka Chart 的较新版本默认使用 KRaft 模式 (Kafka Raft Metadata mode)，不再依赖 ZooKeeper。这简化了架构。如果你使用的是需要 ZooKeeper 的旧版本 Chart，部署和配置会有所不同。

### 总结

通过 Helm，我们可以快速、规范地在 Kubernetes 上部署和管理 Kafka 集群。通过定制 `values.yaml` 文件，可以灵活地调整 Kafka 集群的各项配置，以满足不同环境和业务的需求。记住，在生产部署前，务必仔细评估安全配置、资源分配和高可用性策略。

---