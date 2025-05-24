---
title: K8s采用Helm部署Ingress-Nginx实战指南
tags:
  - Linux
  - K8s
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505061122195.png'
toc: true
abbrlink: ffec2a5
date: 2025-05-06 11:20:16
---

在 Kubernetes (K8s) 世界中，Ingress 是管理集群外部访问内部服务（主要是 HTTP 和 HTTPS）的核心组件。它允许你定义路由规则，将外部请求转发到集群内的不同服务。Ingress-Nginx 是一个基于 Nginx 的 Ingress 控制器实现，因其高性能、稳定性和丰富的功能而广受欢迎。

Helm 作为 Kubernetes 的包管理器，极大地简化了复杂应用的部署和管理过程。本文将详细介绍如何使用 Helm 将 Ingress-Nginx 部署到你的 K8s 集群，重点采用 `DaemonSet + HostNetwork` 模式，并通过脚本化方式进行安装与配置。这种模式在追求高性能和简化网络路径的场景下具有优势。

<!-- more -->

## Ingress-Nginx 部署模式简介

在开始之前，我们先简单回顾一下 Ingress-Nginx 常见的几种部署模式：

1.  **Deployment + LoadBalancer Service:**
    *   原理: Ingress Controller Pods 由 Deployment 管理。创建一个 `type: LoadBalancer` 的 Service 指向这些 Pods。云厂商会自动创建并关联一个外部负载均衡器及公网 IP。
    *   优点: 易于与公有云集成，自动获取公网 IP 和负载均衡。
    *   缺点: 依赖云厂商支持，可能产生额外费用，网络路径相对较长。
    *   适用场景: 公有云环境。

2.  **Deployment + NodePort Service:**
    *   原理: Ingress Controller Pods 由 Deployment 管理。创建一个 `type: NodePort` 的 Service 指向这些 Pods。Ingress Controller 会暴露在集群每个节点的一个静态高位端口上。
    *   优点: 不依赖特定云厂商，部署相对简单。
    *   缺点: NodePort 端口通常在高位范围 (30000-32767)，需要外部负载均衡器将 80/443 端口的流量转发到节点的 NodePort。增加了一层转发。
    *   适用场景: 自建机房或需要手动控制负载均衡器的环境。

3.  **DaemonSet + HostNetwork:**
    *   原理: Ingress Controller Pods 由 DaemonSet 管理，确保在指定的每个节点上都运行一个 Pod。Pod 配置 `hostNetwork: true`，直接使用宿主机的网络命名空间，监听宿主机的 80/443 端口。
    *   优点: 网络路径最短，性能通常最优。
    *   缺点: Pod 直接占用宿主机端口，可能冲突。每个节点只能运行一个监听相同端口的 Ingress Controller Pod。需通过 `nodeSelector` 或 `affinity` 精确控制部署节点。
    *   适用场景: 对性能要求高、网络延迟敏感的生产环境，且有专用节点承载 Ingress 流量。

**本文重点实践 `DaemonSet + HostNetwork` 模式，并通过脚本进行部署。**

## 准备工作

*   一个可用的 Kubernetes 集群。
*   `kubectl` 命令行工具已配置并连接到集群。
*   `helm` v3 命令行工具已安装。如果未安装，可参考[Kubernetes 包管理利器 helm 入门指南](https://lbs.wiki/pages/2eda4ff/)。

## 安装步骤：使用 `install.sh` 脚本

我们将使用一个 `install.sh` 脚本来自动化 Ingress-Nginx 的部署过程。这种方式避免了手动修改 `values.yaml` 文件，所有配置通过 `helm install` 命令的 `--set` 参数传入。

### `install.sh` 脚本内容

```shell
#!/usr/bin/env bash

set -x # 开启命令执行追踪，方便调试

# 1. 添加 Ingress-Nginx Helm 仓库并更新
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# 2. (可选) 卸载可能冲突的预置 Ingress Controller (如 K3s 中的 Traefik)
# 如果你的集群（例如 K3s）默认安装了其他 Ingress Controller 并占用了 80/443 端口，
# 需要先卸载它们，以避免端口冲突。
helm uninstall traefik -n kube-system
helm uninstall traefik-crd -n kube-system # Traefik CRDs 可能也需要单独卸载

# 3. 使用 Helm 安装 Ingress-Nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --version 4.12.2 \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.hostNetwork=true \
  --set controller.dnsPolicy=ClusterFirstWithHostNet \
  --set-string controller.nodeSelector.ingress="true" \
  --set controller.kind=DaemonSet \
  --set controller.service.enabled=true \
  --set controller.service.type=NodePort \
  --set-string controller.extraArgs.update-status="false" # 解决 'ne' comparison error

# 4. 为指定节点打上标签，以便 Ingress Controller Pod 调度到这些节点
kubectl label node master ingress=true --overwrite # 如果 master 节点也作为 ingress 节点
kubectl label node node1 ingress=true --overwrite
kubectl label node node2 ingress=true --overwrite
# 请根据你的实际节点名称和规划调整以上命令
# --overwrite 选项允许覆盖已存在的同名标签值
```

### 脚本解析与配置说明

1.  **`helm repo add ingress-nginx ...` 和 `helm repo update`**:
    添加官方 Ingress-Nginx Helm chart 仓库并更新本地缓存。这是获取最新 chart 的标准步骤。

2.  **`helm uninstall traefik ...`**:
    这一步是可选的，主要针对像 K3s 这样默认安装了 Traefik Ingress Controller 的环境。如果 Traefik 占用了 80/443 端口，`HostNetwork` 模式的 Ingress-Nginx Pod 将无法启动。如果你的集群没有预装或已禁用其他 Ingress Controller，可以注释或删除这两行。

3.  **`helm install ingress-nginx ...`**:
    这是核心安装命令。
    *   `ingress-nginx/ingress-nginx`: 指定要安装的 chart。
    *   `--version 4.12.2`: 指定 chart 版本，确保部署的一致性。你可以通过 `helm search repo ingress-nginx/ingress-nginx -l` 查看可用版本。
    *   `--namespace ingress-nginx`: 指定 Ingress-Nginx 安装到的命名空间。
    *   `--create-namespace`: 如果 `ingress-nginx` 命名空间不存在，则自动创建。
    *   `--set controller.hostNetwork=true`: **关键配置**。启用 Controller Pod 的 `hostNetwork` 模式，使其直接使用宿主机的网络栈，监听宿主机端口。
    *   `--set controller.dnsPolicy=ClusterFirstWithHostNet`: 当 `hostNetwork` 为 `true` 时，将 Pod 的 DNS 策略设置为 `ClusterFirstWithHostNet`。这允许 Pod 解析集群内部服务，同时也能继承宿主机的 DNS 配置。
    *   `--set-string controller.nodeSelector.ingress="true"`: 设置节点选择器。Ingress-Nginx Controller Pod 将只会被调度到带有 `ingress: "true"` 标签的节点上。使用 `--set-string` 确保 `"true"`  被视为字符串。
    *   `--set controller.kind=DaemonSet`: **关键配置**。将 Controller 的部署类型设置为 `DaemonSet`，确保在每个被选中的节点上都运行一个 Controller Pod。
    *   `--set controller.service.enabled=true`: 确保为 Controller 创建一个 Service。虽然在 `HostNetwork` 模式下，流量直接通过节点 IP 访问 Pod，但 Service 仍可用于集群内部发现或某些特定场景（如与某些监控系统集成）。
    *   `--set controller.service.type=NodePort`: 将 Controller Service 类型设置为 `NodePort`。这结合 `HostNetwork` 意味着你可以通过 `NodeIP:HostPort` (如 `NodeIP:80/443`) 和 `NodeIP:NodePort` (如 `NodeIP:30080/30443`，如果显式设置了 `nodePorts`) 两种方式访问。默认情况下，Helm chart 会为 HTTP 和 HTTPS 分配 NodePort。
    *   `--set-string controller.extraArgs.update-status="false"`: 解决某些 Helm/Go template 版本组合下可能出现的 `Error: ... <ne (index .Values.controller.extraArgs "update-status") "false">: error calling ne: incompatible types for comparison` 错误。通过显式设置此参数为字符串 `"false"` 来避免此问题。

4.  **`kubectl label node ... ingress=true --overwrite`**:
    为计划运行 Ingress-Nginx Controller Pod 的节点打上 `ingress=true` 标签。脚本中示例了 `master`, `node1`, `node2`。你需要根据你的集群节点名称和规划进行修改。`--overwrite` 选项允许在标签已存在时更新其值。

### 执行安装

1.  将上述脚本内容保存为 `install.sh`。
2.  赋予执行权限：`chmod +x install.sh`。
3.  运行脚本：`./install.sh`。

### 处理 `helm install` 错误 (`Request entity too large`)

尽管我们现在使用 `--set` 参数，而不是一个可能非常大的 `values.yaml` 文件，但在极少数情况下，如果 Chart 本身非常复杂或传递的参数组合导致最终生成的 Kubernetes 对象描述过大，仍然可能遇到 `Error: INSTALLATION FAILED: create: failed to create: Request entity too large: limit is 3145728` 错误。

**解决方法：**
1.  **清理之前的失败尝试：**
    ```bash
    helm uninstall ingress-nginx -n ingress-nginx
    # 可能需要手动删除 Helm 的 Secret/ConfigMap 记录，请谨慎操作
    # kubectl delete secret -n ingress-nginx -l owner=helm,name=ingress-nginx
    ```
2.  **检查 K8s API Server 配置：** 在某些情况下，可能需要调整 K8s API Server 的 `--max-request-bytes` 参数，但这通常不是首选，需要集群管理员权限。
3.  **检查 Helm 版本：** 确保使用的是较新的 Helm 3 版本。

## Post-Installation: 部署后配置与验证

### 1. 验证部署状态 (使用 `status.sh` 脚本)

你可以使用一个简单的 `status.sh` 脚本来快速查看 Ingress-Nginx 相关资源的状态。

**`status.sh` 脚本内容:**
```shell
#!/usr/bin/env bash

set -x

kubectl get all -n ingress-nginx # 获取 ingress-nginx 命名空间下的所有资源
echo "--- Ingress Controller Pods Details ---"
kubectl get pods -n ingress-nginx -o wide -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller
echo "--- Ingress Classes ---"
kubectl get ingressclass
```

**执行脚本:**
1.  保存脚本为 `status.sh`。
2.  赋予执行权限: `chmod +x status.sh`。
3.  运行脚本: `./status.sh`。

**预期输出:**
你应该能看到 `ingress-nginx-controller` 的 DaemonSet 管理的 Pods 处于 `Running` 状态，并且它们运行在你期望打标签的节点上 (通过 `NODE` 列和 `kubectl get pods -o wide` 查看)。同时，Service (通常名为 `ingress-nginx-controller`) 也应存在。还会列出 `ingressclass` 资源，其中应该有一个名为 `nginx` 的 IngressClass (由 Helm chart 默认创建)。

### 2. 节点标签回顾

`install.sh` 脚本已经包含了为节点打标签的步骤。请确保你已根据实际情况修改了脚本中的节点名，并且这些节点确实是你希望承载入口流量的节点。

你可以通过以下命令再次确认节点标签：
```bash
kubectl get nodes --show-labels | grep "ingress=true"
```

### 3. 处理 Master 节点 Taints (测试环境)

默认情况下，Kubernetes Master 节点通常带有 `NoSchedule` 或 `NoExecute` 污点 (Taints)，阻止普通工作负载调度上去。如果你在 `install.sh` 中为 Master 节点打了 `ingress=true` 标签，并希望 Ingress Controller Pod 也运行在 Master 节点上（通常**不推荐**在生产环境这样做，但在资源有限的测试环境中可能需要），你需要移除相应的污点。

```bash
# 查看 Master 节点的 Taints
kubectl describe node <your-master-node-name> | grep Taints

# 假设 Master 节点名为 k8s-master01 (请替换为你的实际 Master 节点名)
# 且有污点 node-role.kubernetes.io/master:NoSchedule (常见的 Master 污点)
# 移除污点 (注意最后的 -)
kubectl taint node k8s-master01 node-role.kubernetes.io/master-

# 如果污点是 node-role.kubernetes.io/control-plane:NoSchedule (或者类似), 命令相应调整
# kubectl taint node k8s-master01 node-role.kubernetes.io/control-plane-
```
移除污点后，如果 Master 节点也打了 `ingress=true` 标签，DaemonSet 就会在该节点上创建一个 Pod。

## 进阶配置 (可选) - 使用 `helm upgrade`

如果需要在初始安装后修改配置，例如添加 TCP/UDP 代理或修改 Controller 监听端口，应该使用 `helm upgrade` 命令并配合 `--set` 或 `--set-string` 参数。

### 1. 配置 TCP/UDP 代理

Ingress 资源主要处理 HTTP/HTTPS，但 Ingress-Nginx Controller 也可以通过 ConfigMap 配置来代理 TCP 或 UDP 流量。通过 Helm，这可以通过设置 `controller.tcp` 或 `controller.udp` 值来实现。

```bash
# 示例：暴露 MySQL (TCP 3306) 和 DNS (UDP 53)
helm upgrade ingress-nginx ingress-nginx/ingress-nginx --version 4.12.2 \
  -n ingress-nginx \
  --reuse-values \ # 重用上次安装时的值
  --set-string controller.tcp."3306"="default/mysql-service:3306" \
  --set-string controller.udp."53"="kube-system/kube-dns:53"
```
*   `--reuse-values`: 重要的是包含此参数，否则未在 `upgrade` 命令中明确设置的参数将恢复为 chart 的默认值。
*   注意 TCP/UDP 端口号需要用引号括起来，因为它们是字典的键。

### 2. 修改 Controller 监听端口 (Host Ports)

默认情况下，`hostNetwork: true` 使 Controller 直接监听宿主机的 80 (HTTP) 和 443 (HTTPS) 端口。如果需要修改这些端口，可以通过设置 `controller.extraArgs` 中的 `--http-port` 和 `--https-port` 来实现。

```bash
# 示例：将 HTTP 端口改为 8880，HTTPS 端口改为 8881
helm upgrade ingress-nginx ingress-nginx/ingress-nginx --version 4.12.2 \
  -n ingress-nginx \
  --reuse-values \
  --set controller.extraArgs.http-port=8880 \
  --set controller.extraArgs.https-port=8881
```
更新后，Ingress Controller Pod 将在宿主机上监听新的端口。

## 验证测试：确保 Ingress-Nginx 正常工作

部署完 Ingress-Nginx Controller 后，我们需要验证它是否能正确地将外部流量路由到集群内部的示例应用。

**核心验证思路：**
1.  在集群中部署一个简单的 Web 应用（例如 Nginx）。
2.  为该应用创建一个 Service。
3.  创建一个 Ingress 资源，定义路由规则。
4.  通过配置本地 DNS 解析（如修改 `hosts` 文件）或使用 `curl` 的 `--resolve` 选项，模拟外部域名访问。
5.  发送 HTTP 请求，验证是否能成功访问。

### 1. 准备示例应用清单 (`demo-app.yaml`)

创建一个名为 `demo-app.yaml` 的文件：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-nginx-app
  labels:
    app: demo-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-nginx
  template:
    metadata:
      labels:
        app: demo-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo-nginx-service
  labels:
    app: demo-nginx
spec:
  type: ClusterIP
  selector:
    app: demo-nginx
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-nginx-ingress
  labels:
    app: demo-nginx
spec:
  ingressClassName: nginx # 重要：确保与你的 Ingress-Nginx Controller 匹配
                          # Helm 安装时默认创建的 IngressClass 名称是 "nginx"
  rules:
    - host: "nginx-demo.local.show"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-nginx-service
                port:
                  name: http
```

**说明：**
*   **`spec.ingressClassName: nginx`**: 告诉 Kubernetes 这个 Ingress 资源应该由名为 `nginx` 的 Ingress Controller 来处理。这是 Helm chart 默认创建的 IngressClass 名称。如果你的 `install.sh` 通过 `--set controller.ingressClassResource.name=my-custom-nginx` 等方式自定义了 IngressClass 名称，此处需保持一致。

### 2. 应用清单并检查资源状态

```bash
kubectl apply -f demo-app.yaml
kubectl get pods,svc,ingress -l app=demo-nginx -w
```
你应该看到 Pods 运行，Service 创建，并且 Ingress 资源的 `ADDRESS` 字段可能显示你标记为 `ingress=true` 的节点的 IP 地址（或其中之一）。

### 3. 配置本地 DNS 解析或使用 `curl --resolve`

*   **获取一个运行 Ingress-Nginx Controller Pod 的节点 IP 地址。**
    选择一个你通过 `kubectl label node <nodename> ingress=true` 标记的节点的 IP。
    ```bash
    # 假设 node1 是你的 Ingress 节点
    NODE_IP=$(kubectl get node node1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
    # 或者 ExternalIP，如果外部可达且你希望通过它访问
    # NODE_IP=$(kubectl get node node1 -o jsonpath='{.status.addresses[?(@.type=="ExternalIP")].address}')
    echo "Using Node IP: $NODE_IP"
    ```
*   **方法一：修改本地 `hosts` 文件**
    添加一行 (将 `<NODE_IP>` 替换为实际节点 IP)：
    ```
    <NODE_IP>   nginx-demo.local.show
    ```
*   **方法二：使用 `curl` 的 `--resolve` 选项**
    ```bash
    # 将 <NODE_IP> 替换为 Ingress Controller 节点 IP
    # 端口仍然是 80 (或你在 Ingress 中定义的端口，或 Ingress Controller 修改后的监听端口)
    curl --resolve "nginx-demo.local.show:80:<NODE_IP>" http://nginx-demo.local.show/
    ```

### 4. 发送 HTTP 请求进行访问测试

```bash
# 如果你修改了 hosts 文件
curl http://nginx-demo.local.show

# 或者直接使用上面带 --resolve 的 curl 命令 (替换 <NODE_IP>)
curl --resolve "nginx-demo.local.show:80:<NODE_IP>" http://nginx-demo.local.show
```
**预期输出：** Nginx 的欢迎页面。

### 5. 故障排查提示
(与之前版本相同，此处省略详细步骤，主要包括检查 Controller Pod 日志、示例应用 Pod 日志、Ingress 资源事件、Ingress Controller 内部 Nginx 配置、网络连通性等。)

## 卸载 Ingress-Nginx (使用 `uninstall.sh` 脚本)

如果你需要卸载 Ingress-Nginx，可以使用以下脚本：

**`uninstall.sh` 脚本内容:**
```shell
#!/usr/bin/env bash

set -x

helm uninstall ingress-nginx -n ingress-nginx

# (可选) 移除之前添加的节点标签
# kubectl label node master ingress-
# kubectl label node node1 ingress-
# kubectl label node node2 ingress-
# 请根据你的实际节点名称调整
```

**执行脚本:**
1.  保存脚本为 `uninstall.sh`。
2.  赋予执行权限: `chmod +x uninstall.sh`。
3.  运行脚本: `./uninstall.sh`。

这将卸载 Helm release，并删除由该 release 创建的所有 Kubernetes 资源。你也可以选择手动移除之前为节点添加的 `ingress=true` 标签。

## 总结

通过本文的指导和提供的脚本，你应该能够使用 Helm 成功地将 Ingress-Nginx 以 `DaemonSet + HostNetwork` 模式部署到你的 Kubernetes 集群中。这种脚本化的部署方式简化了配置过程，并通过 `--set` 参数清晰地定义了部署选项。`DaemonSet + HostNetwork` 模式为对延迟敏感的应用提供了优异的网络性能。

记住，在生产环境中，务必仔细规划哪些节点将承载 Ingress流量，并确保这些节点的资源充足和网络配置得当。希望这篇更新后的博文能帮助你更高效地部署和管理 Ingress-Nginx。

---