---
title: K8s采用Helm部署kube-prometheus-stack实战指南
tags:
  - Linux
  - K8s
  - Helm
  - Prometheus
  - Grafana
  - Alertmanager
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505241803479.png'
toc: true
abbrlink: 9958a6cd
date: 2025-05-24 18:01:30
---

在Kubernetes (K8s) 生态中，监控是确保应用稳定性和性能的关键一环。`kube-prometheus-stack` 集合了 Prometheus、Grafana、Alertmanager 以及众多 Exporter，提供了一套全面且强大的云原生监控解决方案。通过 Helm，我们可以极大地简化其部署和管理过程。本文将详细介绍如何使用 Helm 在 K8s 集群中部署 `kube-prometheus-stack`。

**为什么选择 kube-prometheus-stack?**

*   **一站式解决方案**: 集成了 Prometheus (指标收集与存储)、Alertmanager (告警处理)、Grafana (数据可视化) 以及常用的 K8s 监控组件。
*   **社区活跃**: 由 Prometheus 社区维护，更新及时，文档丰富。
*   **高度可配置**: 通过 Helm values 文件可以灵活定制各个组件。
*   **CRD驱动**: 利用 Prometheus Operator 的 CRD (如 `ServiceMonitor`, `PodMonitor`) 自动发现和配置监控目标。

<!-- more -->

### 前提条件

在开始之前，请确保您已具备以下环境和工具：

1.  **可用的 Kubernetes 集群**: 并且 `kubectl` 已配置好对集群的访问权限。
2.  **Helm v3**: Helm 是 Kubernetes 的包管理器，如果尚未安装，请参考 [Helm 官方文档](https://helm.sh/docs/intro/install/)进行安装。
3.  **StorageClass**: 集群中需要一个可用的 StorageClass 用于持久化存储 Prometheus、Alertmanager 和 Grafana 的数据。本指南使用名为 `nfs-storage` 的 StorageClass，请根据您的实际情况修改。
4.  **Ingress Controller**: 集群中需要安装并配置好 Ingress Controller (如 Nginx Ingress Controller) 以便通过域名访问监控组件。本指南使用名为 `nginx` 的 Ingress Class，请根据您的实际情况修改。
5.  **DNS 配置**: 确保您计划用于访问 Alertmanager, Prometheus, Grafana 的域名能够解析到您的 Ingress Controller 的外部 IP 地址。

### 步骤 1: 准备安装脚本与配置

我们将使用一个 `install.sh` 脚本来自动化部署过程。首先，创建 `install.sh` 文件：

```shell
#!/usr/bin/env bash

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# --- 配置变量 ---
# Helm 安装相关
RELEASE_NAME="kube-prom-stack"
CHART_VERSION="72.6.2" # 请确认这是您希望使用的稳定版本，或注释掉以使用最新版
NAMESPACE="monitoring"

# 存储类和Ingress类
STORAGE_CLASS_NAME="nfs-storage" # 根据您的环境修改
INGRESS_CLASS_NAME="nginx"      # 确保您的集群中已安装并配置了Nginx Ingress Controller

# Ingress 主机名 (请根据您的环境修改这些占位符)
# 确保这些域名可以解析到您的 Ingress Controller 的外部 IP
ALERTMANAGER_HOST="alertmanager.yourdomain.com" # 例如: alertmanager.k8s.example.com
PROMETHEUS_HOST="prometheus.yourdomain.com"   # 例如: prometheus.k8s.example.com
GRAFANA_HOST="grafana.yourdomain.com"       # 例如: grafana.k8s.example.com

# 持久化存储大小
ALERTMANAGER_STORAGE_SIZE="8Gi"
PROMETHEUS_STORAGE_SIZE="32Gi" # Prometheus 需要的存储通常比 Alertmanager 和 Grafana 多
GRAFANA_STORAGE_SIZE="8Gi"

# Grafana 管理员密码 (生产环境建议修改或使用 Secret)
GRAFANA_ADMIN_PASSWORD="YourSecurePassword" # 这是 chart 的默认密码是 'prom-operator', 请务必修改

# --- 安装 kube-prometheus-stack ---
helm install ${RELEASE_NAME} prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} \
  --namespace ${NAMESPACE} \
  --create-namespace \
  \
  --set alertmanager.enabled=true \
  --set alertmanager.ingress.enabled=true \
  --set alertmanager.ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set alertmanager.ingress.hosts[0]=${ALERTMANAGER_HOST} \
  --set alertmanager.ingress.paths[0]="/" \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=${STORAGE_CLASS_NAME} \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.accessModes[0]="ReadWriteOnce" \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=${ALERTMANAGER_STORAGE_SIZE} \
  \
  --set prometheus.enabled=true \
  --set prometheus.ingress.enabled=true \
  --set prometheus.ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set prometheus.ingress.hosts[0]=${PROMETHEUS_HOST} \
  --set prometheus.ingress.paths[0]="/" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=${STORAGE_CLASS_NAME} \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.accessModes[0]="ReadWriteOnce" \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=${PROMETHEUS_STORAGE_SIZE} \
  \
  --set grafana.enabled=true \
  --set grafana.adminPassword=${GRAFANA_ADMIN_PASSWORD} \
  --set grafana.ingress.enabled=true \
  --set grafana.ingress.ingressClassName=${INGRESS_CLASS_NAME} \
  --set grafana.ingress.hosts[0]=${GRAFANA_HOST} \
  --set grafana.ingress.path="/" \
  --set grafana.persistence.enabled=true \
  --set grafana.persistence.storageClassName=${STORAGE_CLASS_NAME} \
  --set grafana.persistence.accessModes[0]="ReadWriteOnce" \
  --set grafana.persistence.size=${GRAFANA_STORAGE_SIZE} \
  \
  --set prometheusOperator.enabled=true

echo ""
echo "kube-prometheus-stack (${RELEASE_NAME}) 安装过程已启动到命名空间 '${NAMESPACE}'。"
echo "---------------------------------------------------------------------"
echo "监控 Pod 状态: kubectl get pods -n ${NAMESPACE} -w"
echo ""
echo "访问服务 (请确保您的 DNS 配置正确或使用 Ingress Controller 的外部 IP):"
echo "  Alertmanager: http://${ALERTMANAGER_HOST}"
echo "  Prometheus:   http://${PROMETHEUS_HOST}"
echo "  Grafana:      http://${GRAFANA_HOST}"
echo "                (默认用户: admin, 密码: ${GRAFANA_ADMIN_PASSWORD})"
echo "---------------------------------------------------------------------"
echo "如果遇到问题, 使用 'helm status ${RELEASE_NAME} -n ${NAMESPACE}' 和 'kubectl logs -n ${NAMESPACE} -l app.kubernetes.io/name=prometheus-operator -c prometheus-operator' 进行排查。"
```

**重要配置说明:**

*   **`CHART_VERSION`**: 建议指定一个明确的、经过测试的稳定版本号。如果注释掉 `--version` 参数，Helm 会尝试安装最新版本。
*   **`NAMESPACE`**: 指定安装 `kube-prometheus-stack` 的命名空间，脚本中会使用 `--create-namespace` 自动创建。
*   **`STORAGE_CLASS_NAME`**: **务必修改**为您集群中实际可用的 StorageClass 名称。
*   **`INGRESS_CLASS_NAME`**: **务必修改**为您集群中 Ingress Controller 使用的 IngressClass 名称。
*   **`ALERTMANAGER_HOST`, `PROMETHEUS_HOST`, `GRAFANA_HOST`**: **务必修改**为您规划的域名，并确保这些域名已正确配置 DNS 解析。
*   **`GRAFANA_ADMIN_PASSWORD`**: **务必修改**为您自定义的安全密码。生产环境中更推荐使用 Kubernetes Secret 来管理敏感信息，或者在首次登录后立即修改。

### 步骤 2: 执行安装

1.  **赋予脚本执行权限**:
    ```bash
    chmod +x install.sh
    ```

2.  **运行安装脚本**:
    ```bash
    ./install.sh
    ```

脚本会首先添加 Prometheus 社区的 Helm 仓库并更新，然后根据配置变量执行 `helm install` 命令。

### 步骤 3: 验证安装状态

安装过程可能需要几分钟，等待所有 Pod 启动并运行。

1.  **监控 Pod 状态** (如脚本末尾提示):
    ```bash
    kubectl get pods -n monitoring -w
    ```
    等待所有 Pod 状态变为 `Running` 且 `READY` 列的容器数量正确 (例如 `1/1`, `2/2`)。

2.  **查看所有资源和持久化卷声明 (PVC)**:
    为了方便检查，我们创建一个 `status.sh` 脚本：
    ```shell
    #!/usr/bin/env bash

    kubectl get all -n monitoring
    kubectl get pvc -n monitoring
    ```
    赋予执行权限并运行：
    ```bash
    chmod +x status.sh
    ./status.sh
    ```
    您应该能看到相关的 Service, Deployment, StatefulSet, PVC 等资源。特别注意 PVC 的状态是否为 `Bound`。

3.  **访问Web UI**:
    在浏览器中打开您配置的域名：
    *   **Alertmanager**: `http://alertmanager.yourdomain.com`
    *   **Prometheus**: `http://prometheus.yourdomain.com`
    *   **Grafana**: `http://grafana.yourdomain.com`
        *   使用用户名 `admin` 和您在 `install.sh` 中设置的 `GRAFANA_ADMIN_PASSWORD` 密码登录。

### 步骤 4: (可选) 自定义与扩展

`kube-prometheus-stack` 安装完成后，您可以进行更多自定义操作：

*   **Grafana**:
    *   导入或创建新的 Dashboard。
    *   配置数据源 (默认已配置好 Prometheus)。
    *   配置用户和权限。
*   **Prometheus**:
    *   通过创建 `ServiceMonitor` 或 `PodMonitor` CRD 来自动发现和抓取新的应用指标。
    *   配置告警规则 (`PrometheusRule` CRD)。
*   **Alertmanager**:
    *   配置告警接收器 (如 Email, Slack, Webhook) 和路由规则。

### 步骤 5: 卸载

如果您需要卸载 `kube-prometheus-stack`，可以使用以下 `uninstall.sh` 脚本：

```shell
#!/usr/bin/env bash

RELEASE_NAME="kube-prom-stack" # 与安装时使用的 RELEASE_NAME 一致
NAMESPACE="monitoring"       # 与安装时使用的 NAMESPACE 一致

helm uninstall ${RELEASE_NAME} -n ${NAMESPACE}

echo "${RELEASE_NAME} 卸载过程已启动。"
echo "如果 StorageClass 的 reclaimPolicy 不是 Delete，Persistent Volume Claims (PVCs) 可能需要手动删除。"
echo "请使用以下命令检查：kubectl get pvc -n ${NAMESPACE}"
echo "如果需要，手动删除PVC：kubectl delete pvc <pvc-name> -n ${NAMESPACE}"
# 例如: kubectl delete pvc storage-alertmanager-kube-prom-stack-alertmanager-0 -n monitoring
#       kubectl delete pvc prometheus-kube-prom-stack-prometheus-db-prometheus-kube-prom-stack-prometheus-0 -n monitoring
#       kubectl delete pvc grafana-kube-prom-stack -n monitoring (这个PVC名字可能因chart版本而异，请以实际get pvc为准)
# 如果您也想删除命名空间，可以执行：
# kubectl delete namespace ${NAMESPACE}
```

1.  **赋予脚本执行权限**:
    ```bash
    chmod +x uninstall.sh
    ```
2.  **运行卸载脚本**:
    ```bash
    ./uninstall.sh
    ```

**注意**:
卸载操作默认不会删除持久化卷声明 (PVCs) 和它们对应的持久化卷 (PVs)，除非您的 StorageClass 的 `reclaimPolicy` 设置为 `Delete`。如果需要彻底清除数据，请在卸载后手动删除相关的 PVCs。您可以使用 `kubectl get pvc -n monitoring` 查看，并使用 `kubectl delete pvc <pvc-name> -n monitoring` 删除。如果命名空间也不再需要，可以执行 `kubectl delete namespace monitoring`。

### 故障排查

如果在安装或运行过程中遇到问题，可以尝试以下命令进行排查：

*   **查看 Helm Release 状态**:
    ```bash
    helm status kube-prom-stack -n monitoring
    ```
*   **查看 Prometheus Operator 日志**:
    ```bash
    kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator -c prometheus-operator -f
    ```
*   **查看特定 Pod 日志**:
    ```bash
    kubectl logs -n monitoring <pod-name> -c <container-name> -f
    ```
    例如，查看 Grafana Pod 日志：先用 `kubectl get pods -n monitoring` 找到 Grafana Pod 的名字，然后执行（假设Pod名为 `kube-prom-stack-grafana-xxxx`）：
    ```bash
    kubectl logs -n monitoring kube-prom-stack-grafana-xxxx -c grafana -f
    ```
*   **检查 Ingress 配置**:
    ```bash
    kubectl describe ingress -n monitoring
    ```
    确保 Ingress 规则已正确生成，并且 Ingress Controller 的日志没有报错。
*   **检查 PVC 和 PV 状态**:
    ```bash
    kubectl get pvc -n monitoring
    kubectl describe pvc <pvc-name> -n monitoring
    kubectl get pv
    ```
    查看 PVC 是否成功绑定到 PV，存储是否成功分配。

### 总结

通过 Helm 和 `kube-prometheus-stack`，我们可以快速、便捷地在 Kubernetes 集群中部署一套功能完善的监控系统。本文提供的脚本和步骤旨在帮助您快速上手，并根据实际需求进行调整。熟悉 Prometheus、Grafana 和 Alertmanager 的配置，将使您能够更好地利用这套强大的监控工具，保障应用服务的稳定运行。

---