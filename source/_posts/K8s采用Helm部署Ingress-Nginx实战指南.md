---
title: K8s采用Helm部署Ingress-Nginx实战指南
tags:
  - Linux
  - k8s
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505061122195.png'
toc: true
abbrlink: ffec2a5
date: 2025-05-06 11:20:16
---

在 Kubernetes (K8s) 世界中，Ingress 是管理集群外部访问内部服务（主要是 HTTP 和 HTTPS）的核心组件。它允许你定义路由规则，将外部请求转发到集群内的不同服务。Ingress-Nginx 是一个基于 Nginx 的 Ingress 控制器实现，因其高性能、稳定性和丰富的功能而广受欢迎。

Helm 作为 Kubernetes 的包管理器，极大地简化了复杂应用的部署和管理过程。本文将详细介绍如何使用 Helm 将 Ingress-Nginx 部署到你的 K8s 集群，重点采用 `DaemonSet + HostNetwork` 模式，这种模式在追求高性能和简化网络路径的场景下具有优势。

<!-- more -->

## Ingress-Nginx 部署模式简介

在开始之前，我们先简单回顾一下 Ingress-Nginx 常见的几种部署模式：

1.  **Deployment + LoadBalancer Service:**
    *   **原理:** Ingress Controller Pods 由 Deployment 管理。创建一个 `type: LoadBalancer` 的 Service 指向这些 Pods。云厂商（如 AWS, GCP, Azure）会自动创建并关联一个外部负载均衡器（ELB/GLB/ALB）及公网 IP。
    *   **优点:** 易于与公有云集成，自动获取公网 IP 和负载均衡。
    *   **缺点:** 依赖云厂商支持，可能产生额外费用，网络路径相对较长。
    *   **适用场景:** 公有云环境。

2.  **Deployment + NodePort Service:**
    *   **原理:** Ingress Controller Pods 由 Deployment 管理。创建一个 `type: NodePort` 的 Service 指向这些 Pods。Ingress Controller 会暴露在集群每个节点的一个静态高位端口上。
    *   **优点:** 不依赖特定云厂商，部署相对简单。
    *   **缺点:** NodePort 端口通常在高位范围 (30000-32767)，需要外部负载均衡器（如 HAProxy, F5）或 DNS 轮询将 80/443 端口的流量转发到节点的 NodePort。增加了一层转发，可能影响性能。
    *   **适用场景:** 自建机房或需要手动控制负载均衡器的环境。

3.  **DaemonSet + HostNetwork:**
    *   **原理:** Ingress Controller Pods 由 DaemonSet 管理，确保在指定的每个节点上都运行一个 Pod。Pod 配置 `hostNetwork: true`，直接使用宿主机的网络命名空间。这意味着 Pod 直接监听宿主机的 80/443 端口。
    *   **优点:** 网络路径最短（请求 -> Node IP:80/443 -> Pod），性能通常最优。无需额外的 Service 暴露端口（虽然可以保留 Service 用于内部发现）。
    *   **缺点:** Pod 直接占用宿主机端口，可能与宿主机上其他服务冲突。每个节点只能运行一个监听相同端口（如 80/443）的 Ingress Controller Pod。需要通过 `nodeSelector` 或 `affinity` 精确控制部署节点。
    *   **适用场景:** 对性能要求高、网络延迟敏感的生产环境，且有专用节点承载 Ingress 流量。

**本文重点实践 `DaemonSet + HostNetwork` 模式。**

## 准备工作

*   一个可用的 Kubernetes 集群。
*   `kubectl` 命令行工具已配置并连接到集群。
*   `helm` v3 命令行工具已安装，如果没有安装，可以参考[Kubernetes 包管理利器 helm 入门指南](https://liboshuai.icu/pages/2eda4ff/)来进行安装。
*   （可选）`docker` 命令行工具，用于处理镜像（如果需要手动拉取和加载）。

## 安装步骤

### 1. 添加 Ingress-Nginx Helm 仓库

首先，我们需要将官方的 Ingress-Nginx Helm chart 仓库添加到本地 Helm 配置中，并更新仓库信息。

```bash
# 添加仓库
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# 更新本地仓库缓存
helm repo update

# (可选) 搜索可用的 chart 版本
helm search repo ingress-nginx/ingress-nginx -l
```

### 2. 下载并解压 Chart 包

为了方便修改配置，我们将下载 Chart 包到本地。我们将使用 `v4.12.2` 版本作为示例（你可以根据 `helm search` 的结果选择需要的版本）。

```bash
# 指定版本下载 Chart 包
helm pull ingress-nginx/ingress-nginx --version 4.12.2
```

如果在执行 `helm pull` 时遇到网络问题，例如 `connection reset by peer` 错误，这通常是由于网络限制或 GFW 干扰导致的无法访问 GitHub Releases。你可以尝试直接使用 `wget` 或浏览器下载：

```bash
# 备选下载方式
wget https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-4.12.2/ingress-nginx-4.12.2.tgz
```

下载完成后，解压 Chart 包：

```bash
# 解压缩
tar -zxvf ingress-nginx-4.12.2.tgz

# 进入 chart 目录
cd ingress-nginx
```

接下来的修改都将在 `ingress-nginx` 目录下的 `values.yaml` 文件中进行。

### 3. 镜像准备（应对镜像拉取困难）

官方 Ingress-Nginx 使用的镜像托管在 `registry.k8s.io`。在中国大陆或其他网络受限的环境中，可能无法直接拉取。我们需要使用国内或其他可访问的镜像源进行替代，并重新打标签。

**重要提示：** 以下操作需要在**所有**你计划部署 Ingress Controller Pod 的节点上执行，或者在一个节点操作后将镜像文件拷贝到其他目标节点。

以 `controller:v1.11.3` 和 `kube-webhook-certgen:v1.4.4` (对应 Chart `v4.12.2` 中的默认版本)为例，假设你找到了华为云或其他镜像仓库的替代镜像：

```bash
# === 处理 controller 镜像 ===
# 1. 从替代源拉取镜像
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/controller:v1.11.3

# 2. 将镜像重新打标签，使其与官方名称一致
docker tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/controller:v1.11.3 registry.k8s.io/ingress-nginx/controller:v1.11.3

# 3. (可选) 保存镜像为 tar 文件，用于分发
docker save -o ncontroller.tar registry.k8s.io/ingress-nginx/controller:v1.11.3

# === 处理 kube-webhook-certgen 镜像 ===
# 1. 从替代源拉取镜像
docker pull swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.4.4

# 2. 重新打标签
docker tag swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.4.4 registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.4.4

# 3. (可选) 保存镜像为 tar 文件
docker save -o certgen.tar registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.4.4

# === 在其他目标节点上 ===
# 4. (如果使用了 .tar 文件) 将 tar 文件拷贝到其他节点
# scp ncontroller.tar root@<other-node-ip>:/root
# scp certgen.tar root@<other-node-ip>:/root

# 5. (如果使用了 .tar 文件) 在其他节点加载镜像
# docker load -i ncontroller.tar
# docker load -i certgen.tar
```

确保在执行 Helm 安装之前，所有目标节点上都存在 `registry.k8s.io/ingress-nginx/controller:v1.11.3` 和 `registry.k8s.io/ingress-nginx/kube-webhook-certgen:v1.4.4` 这两个本地镜像。

### 4. 修改 `values.yaml` 文件

现在，打开 `ingress-nginx` 目录下的 `values.yaml` 文件，进行以下修改以适配 `DaemonSet + HostNetwork` 模式：

*   **注释掉 Controller 镜像的 `digest`:**
    这样 K8s 会根据 `tag` 来拉取镜像，避免因我们手动 re-tag 导致的 sha256 摘要不匹配问题。

    ```yaml
    controller:
      image:
        registry: registry.k8s.io
        image: ingress-nginx/controller
        tag: "v1.11.3"
        # digest: sha256:d56f135b6462cfc476447cfe564b83a45e8bb7da2774963b00d12161112270b7 # 注释掉或删除此行
        # ... 其他 image 配置 ...
    ```

*   **启用 `hostNetwork`:**
    这是实现 HostNetwork 模式的关键。

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      hostNetwork: true # 将 false 修改为 true
      # ... 省略其他 controller 配置 ...
    ```

*   **修改 `dnsPolicy`:**
    使用 `hostNetwork: true` 时，需要将 DNS 策略设置为 `ClusterFirstWithHostNet`，以便 Pod 能正确解析集群内部服务名，同时也能继承宿主机的 DNS 配置。

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      dnsPolicy: ClusterFirstWithHostNet # 将 ClusterFirst 修改为 ClusterFirstWithHostNet
      # ... 省略其他 controller 配置 ...
    ```

*   **添加 `nodeSelector`:**
    我们希望 Ingress Controller 只运行在特定的节点上（例如，专门用于处理外部流量的边缘节点）。这里我们添加一个标签选择器 `ingress: "true"`。稍后需要给目标节点打上这个标签。

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      nodeSelector:
        kubernetes.io/os: linux
        ingress: "true" # 添加此行
      # ... 省略其他 controller 配置 ...
    ```

*   **修改部署类型 `kind` 为 `DaemonSet`:**
    确保 Controller 以 DaemonSet 方式部署。

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      kind: DaemonSet # 将 Deployment 修改为 DaemonSet
      # ... 省略其他 controller 配置 ...
    ```

*   **修改 Admission Webhooks 的 `image` 配置 (注释 `digest`)**
    Admission Webhooks 用于验证 Ingress 资源的配置。同样，如果手动处理了 `kube-webhook-certgen` 镜像，需要注释掉其 `digest`。

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      admissionWebhooks:
        # ... 省略其他 admissionWebhooks 配置 ...
        patch:
          image:
            registry: registry.k8s.io
            image: ingress-nginx/kube-webhook-certgen
            tag: v1.4.4
            # digest: sha256:a9f03b34a3cbfbb26d103a14046ab2c5130a80c3d69d526ff8063d2b37b9fd3f # 注释掉或删除此行
            # ... 
    ```
    *注意：在较新版本的 Chart 中，`kube-webhook-certgen` 的配置路径可能在 `defaultBackend` 或单独的 `admissionWebhooks` 顶级键下，请根据你的 `values.yaml` 结构调整。此处路径基于原文档描述。*

*   **修改 Service 类型:**

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      service:
        enabled: true # 确保 service 是启用的
        # ... 省略其他 service 配置 ...
        type: NodePort # 将 LoadBalancer 或 ClusterIP 修改为 NodePort
        nodePorts:
          http: "30080" # (可选) 指定 HTTP 的 NodePort
          https: "30443" # (可选) 指定 HTTPS 的 NodePort
          tcp: {}
          udp: {}
      # ... 省略其他 controller 配置 ...
    ```
    **效果:** 这将创建一个 `NodePort` 类型的 Service，除了可以通过 `NodeIP:80/443` (HostNetwork)访问 Ingress 外，还可以通过 `NodeIP:30080` 和 `NodeIP:30443` 访问。

### 5. 处理 `helm install` 错误 ( `ne` comparison error)

在某些 Helm 或 Go 模板版本组合下，直接安装可能会遇到类似以下的错误：
`Error: INSTALLATION FAILED: template: ingress-nginx/templates/controller-role.yaml:48:9: executing "ingress-nginx/templates/controller-role.yaml" at <ne (index .Values.controller.extraArgs "update-status") "false">: error calling ne: incompatible types for comparison`

这是因为模板中试图比较 `extraArgs` 字典中的 `update-status` 值（可能不存在，即 `nil`）与字符串 `"false"`。我们需要显式地在 `values.yaml` 中设置这个参数。

*   **修改 `controller.extraArgs`:**

    ```yaml
    controller:
      # ... 省略其他 controller 配置 ...
      extraArgs:
        update-status: "false" # 添加此行或修改{}为这个
      # ... 省略其他 controller 配置 ...
    ```
    将 `extraArgs: {}` 修改为此内容。注意 `update-status` 的值是**字符串** `"false"`。

### 6. 安装 Ingress-Nginx

现在，所有配置都已修改完毕，可以执行安装了。

```bash
# 1. (如果不存在) 创建目标命名空间
kubectl create ns ingress-nginx

# 2. 使用修改后的 values.yaml 文件执行 Helm 安装
#    确保当前目录在 ingress-nginx chart 目录下 (包含 Chart.yaml 和 values.yaml)
helm install ingress-nginx -n ingress-nginx .
```

### 7. 处理 `helm install` 错误 (`Request entity too large`)

如果遇到 `Error: INSTALLATION FAILED: create: failed to create: Request entity too large: limit is 3145728` 错误，这通常发生在 Helm 将 release 信息存储为 K8s Secret 或 ConfigMap 时，内容超出了 K8s API Server 的请求体大小限制（默认为 3MB）。
这可能是因为 `values.yaml` 文件过大，或者 Chart 本身包含大量模板文件。

**解决方法：**

1.  **清理之前的失败尝试 (如果适用):**
    ```bash
    helm uninstall ingress-nginx -n ingress-nginx
    # 可能需要手动删除 Helm 的 Secret/ConfigMap 记录，请谨慎操作
    # kubectl delete secret -n ingress-nginx -l owner=helm,name=ingress-nginx
    ```
2.  **检查 `values.yaml` 大小:** 确保没有包含异常大的内容（例如，嵌入的大型证书或配置片段）。
3.  **考虑 K8s API Server 配置:** 在某些情况下，可能需要调整 K8s API Server 的 `--max-request-bytes` 参数，但这通常不是首选，需要集群管理员权限。
4.  **尝试分步安装或减小 Chart 复杂度 (高级):** 对于非常复杂的 Chart，有时会考虑将 Chart 拆分。
5.  **重新尝试：** 有时简单地清理并重试即可。原文档建议删除本地 Chart 目录后重新解压修改，这主要确保本地文件没有损坏，但通常不直接解决 "Request entity too large" 问题。

### 8. 处理端口冲突（例如 K3s 内置 Traefik）

如果你使用的是 K3s 等自带 Ingress Controller（如 Traefik）的 K8s 发行版，并且配置了 `hostNetwork: true`，Ingress-Nginx Pod 可能会因为 80/443 端口已被占用而无法启动，状态卡在 `Pending`。

使用 `kubectl describe pod <ingress-nginx-pod-name> -n ingress-nginx` 查看 Pod 事件，可能会看到类似 `FailedScheduling ... node(s) didn't have free ports for the requested pod ports` 的错误。

**解决方法：** 卸载或禁用内置的 Ingress Controller。以 K3s 的 Traefik 为例：

```bash
# 1. 查找 Traefik相关的 Helm release (通常在 kube-system 命名空间)
helm list -n kube-system | grep traefik

# 2. 卸载 Traefik (release 名称可能不同，例如 traefik 或 traefik-crd)
helm uninstall traefik -n kube-system
helm uninstall traefik-crd -n kube-system # 如果有单独的 CRD release

# 3. (或者) 如果是通过 K3s 配置文件禁用的，请参考 K3s 文档，在启动参数中添加 --disable traefik 并重启 K3s 服务。
```

卸载冲突的服务后，Ingress-Nginx Pod 应该能够被正确调度并启动。

## Post-Installation: 部署后配置

### 1. 验证部署

检查 Ingress-Nginx 相关资源的状态：

```bash
kubectl get pods,svc,ds -n ingress-nginx -o wide
```

你应该能看到 `ingress-nginx-controller` 的 DaemonSet 管理的 Pods 处于 `Running` 状态，并且它们运行在你期望打标签的节点上。同时，也能看到对应的 Service (如果启用了)。

### 2. 为节点打标签

根据之前在 `values.yaml` 中设置的 `nodeSelector` (`ingress: "true"`)，需要为你希望运行 Ingress Controller Pod 的节点添加标签。

```bash
# 1. 查看节点列表
kubectl get nodes

# 2. 为目标节点打标签 (替换 k8s-worker01, k8s-worker02 为你的实际节点名)
kubectl label node k8s-worker01 ingress=true
kubectl label node k8s-worker02 ingress=true
# ... 为所有需要的节点打标签 ...

# (可选) 查看节点标签确认
kubectl get node --show-labels
```

如果标签打好后 Pod 仍未调度，请检查 `kubectl describe pod ...` 的事件。

### 3. 处理 Master 节点 Taints (测试环境)

默认情况下，Kubernetes Master 节点通常带有 `NoSchedule` 或 `NoExecute` 污点 (Taints)，阻止普通工作负载调度上去。如果你希望在 Master 节点上也运行 Ingress Controller（通常**不推荐**在生产环境这样做，但在资源有限的测试环境中可能需要），你需要移除相应的污点。

```bash
# 查看 Master 节点的 Taints
kubectl describe node <your-master-node-name> | grep Taints

# 假设 Master 节点名为 k8s-master01，且有污点 node-role.kubernetes.io/master:NoSchedule
# 移除污点 (注意最后的 -)
kubectl taint node k8s-master01 node-role.kubernetes.io/master-

# 如果污点是 node-role.kubernetes.io/control-plane:NoSchedule (或者类似), 命令相应调整
# kubectl taint node k8s-master01 node-role.kubernetes.io/control-plane-
```

移除污点后，如果 Master 节点也打了 `ingress=true` 标签，DaemonSet 就会在该节点上创建一个 Pod。

## 进阶配置 (可选)

### 1. 配置 TCP/UDP 代理

Ingress 资源主要处理 HTTP/HTTPS，但 Ingress-Nginx Controller 也可以通过 ConfigMap 配置来代理 TCP 或 UDP 流量。

*   **修改 `values.yaml`:**
    在 `controller.tcp` 或 `controller.udp` 下添加端口和服务映射。

    ```yaml
    controller:
      # ...
      tcp:
        # <external-port>: "<namespace>/<service-name>:<service-port>"
        "3306": "default/mysql-service:3306"
        "5432": "infra/postgres-service:5432"
      udp:
        # <external-port>: "<namespace>/<service-name>:<service-port>"
        "53": "kube-system/kube-dns:53"
      # ...
    ```

*   **更新 Helm Release:**
    修改 `values.yaml` 后，使用 `helm upgrade` 应用更改。

    ```bash
    helm upgrade ingress-nginx -n ingress-nginx .
    ```

### 2. 修改 Controller 监听端口 (Host Ports)

默认情况下，`hostNetwork: true` 使 Controller 直接监听宿主机的 80 (HTTP) 和 443 (HTTPS) 端口。如果需要修改这些端口（例如，避免与宿主机其他服务冲突），**最佳实践是通过 `values.yaml` 配置并使用 `helm upgrade`，而不是直接 `kubectl edit ds`** (因为 Helm upgrade 会覆盖手动修改)。

*   **修改 `values.yaml`:**
    使用 `controller.extraArgs` 来传递 Nginx Ingress Controller 的命令行参数 `--http-port` 和 `--https-port`。

    ```yaml
    controller:
      # ...
      # 确保 extraArgs 中有这几项
      extraArgs:
        update-status: "false" # 保留之前的修改
        http-port: "8880"      # 设置新的 HTTP 端口
        https-port: "8881"     # 设置新的 HTTPS 端口
      # ...
    ```

*   **更新 Helm Release:**

    ```bash
    helm upgrade ingress-nginx -n ingress-nginx .
    ```

    更新后，Ingress Controller Pod 将在宿主机上监听 8880 (HTTP) 和 8881 (HTTPS) 端口。

## 验证测试：确保 Ingress-Nginx 正常工作

部署完 Ingress-Nginx Controller 后，我们需要验证它是否能正确地将外部流量路由到集群内部的示例应用。

**核心验证思路：**
1.  在集群中部署一个简单的 Web 应用（例如 Nginx）。
2.  为该应用创建一个 Service，使其在集群内部可访问。
3.  创建一个 Ingress 资源，定义路由规则，将特定域名或路径的请求指向该 Service。
4.  通过配置本地 DNS 解析（如修改 `hosts` 文件）或使用 `curl` 的 `--resolve` 选项，模拟外部域名访问。
5.  发送 HTTP 请求，验证是否能成功访问到示例应用。

### 1. 准备示例应用清单 (`demo-app.yaml`)

创建一个名为 `demo-app.yaml` 的文件，包含一个 Nginx Deployment、一个 ClusterIP Service 和一个 Ingress 资源。

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
          image: nginx:latest # 使用一个常见的 nginx 镜像
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
  type: ClusterIP # Ingress 通常指向 ClusterIP 类型的 Service
  selector:
    app: demo-nginx
  ports:
    - name: http
      protocol: TCP
      port: 80 # Service 监听的端口
      targetPort: 80 # Pod 内容器的端口
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-nginx-ingress
  annotations:
    # 如果你的集群有多个 Ingress Controller，或者默认 Ingress Class 不是 nginx，
    # 建议显式指定 Ingress Class。
    # kubernetes.io/ingress.class: nginx # 较旧版本
    # nginx.ingress.kubernetes.io/rewrite-target: / # 如果需要 URL 重写
  labels:
    app: demo-nginx # 自定义标签，便于管理
spec:
  ingressClassName: nginx # 重要：确保与你的 Ingress-Nginx Controller 匹配
                          # Helm 安装时默认的 IngressClass 名称通常是 "nginx"
  rules:
    - host: "nginx-demo.local.show" # 定义一个用于测试的域名
      http:
        paths:
          - path: / # 匹配根路径
            pathType: Prefix # 路径匹配类型
            backend:
              service:
                name: demo-nginx-service # 指向上面创建的 Service
                port:
                  name: http # 指向 Service 定义的端口名 (或 number: 80)
```

**说明：**
*   **`metadata.name`**: 为资源定义了清晰的名称，方便区分。
*   **`spec.ingressClassName: nginx`**: 这是**至关重要**的一步。它告诉 Kubernetes 这个 Ingress 资源应该由名为 `nginx` 的 Ingress Controller 来处理。如果你在 Helm 安装 `ingress-nginx` 时指定了不同的 `controller.ingressClassResource.name`，这里也需要相应修改。
*   **`host: "nginx-demo.local.show"`**: 我们使用一个自定义的本地域名进行测试。你可以替换为你喜欢的任何域名，但确保后续步骤中使用的域名与此处一致。
*   **`pathType: Prefix`**: 表示以 `/` 开头的所有路径都会被匹配。

### 2. 应用清单并检查资源状态

在你的 Kubernetes 集群中应用这个清单文件。假设你将上述内容保存在了 `demo-app.yaml` 中。

```bash
# 部署示例应用 (通常部署在 default 命名空间，或指定你自己的命名空间)
kubectl apply -f demo-app.yaml

# 等待 Pod 启动完成
kubectl get pods -l app=demo-nginx -w
```

检查所有相关资源是否已成功创建并处于健康状态：

```bash
# 查看 Pods, Service, 和 Ingress 资源
kubectl get pods,svc,ingress -l app=demo-nginx
```

你应该看到类似如下的输出：

```
NAME                                  READY   STATUS    RESTARTS   AGE
pod/demo-nginx-app-xxxxxxxxxx-yyyyy   1/1     Running   0          60s
pod/demo-nginx-app-xxxxxxxxxx-zzzzz   1/1     Running   0          60s

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/demo-nginx-service     ClusterIP   10.43.xxx.xxx   <none>        80/TCP    60s

NAME                                     CLASS   HOSTS                   ADDRESS         PORTS   AGE
ingress.networking.k8s.io/demo-nginx-ingress   nginx   nginx-demo.local.show   <NodeIP_or_LB_IP>   80      60s
```
*   **`ADDRESS` 字段**：对于 `DaemonSet + HostNetwork` 模式，Ingress 资源的 `ADDRESS` 字段可能会显示所有 Ingress Controller 节点 IP，或具体 Controller Service 的 ClusterIP (如果 helm chart 创建了 service)，或者在某些情况下可能为空。 **重要的是，实际访问将通过 Ingress Controller Pod 所在节点的 IP 进行。**

### 3. 配置本地 DNS 解析或使用 `curl --resolve`

为了让你的本地机器能够将 `nginx-demo.local.show` 解析到 Ingress Controller 节点，你有两种主要方法：

**方法一：修改本地 `hosts` 文件**

*   **获取一个运行 Ingress-Nginx Controller Pod 的节点 IP 地址。**
    由于我们使用的是 `DaemonSet + HostNetwork`，Ingress-Nginx Pod 会在所有（或通过 `nodeSelector` 指定的）节点上运行，并直接监听节点的 80/443 端口。
    ```bash
    # 查看 Ingress-Nginx Controller Pod 运行在哪些节点及其 IP
    kubectl get pods -n ingress-nginx -o wide -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller
    ```
    从输出中选择任意一个 `NODE` 的 `INTERNAL-IP` 或 `EXTERNAL-IP` (如果外部可达)。

*   **编辑你的 `hosts` 文件：**
    *   Linux/macOS: `/etc/hosts`
    *   Windows: `C:\Windows\System32\drivers\etc\hosts`

    添加一行 (将 `<NODE_IP>` 替换为你选择的节点 IP)：
    ```
    <NODE_IP>   nginx-demo.local.show
    ```
    例如：
    ```
    192.168.1.101   nginx-demo.local.show
    ```

**方法二：使用 `curl` 的 `--resolve` 选项 (无需修改 `hosts` 文件)**

这种方法更灵活，因为它只对当前的 `curl` 命令生效。

```bash
# 将 <NODE_IP> 替换为 Ingress Controller 节点 IP
# 将 <PORT> 替换为 Ingress Controller 监听的 HTTP 端口 (默认为 80)
curl --resolve "nginx-demo.local.show:<PORT>:<NODE_IP>" http://nginx-demo.local.show/
```
例如，如果节点 IP 是 `192.168.1.101`，HTTP 端口是 `80`：
```bash
curl --resolve "nginx-demo.local.show:80:192.168.1.101" http://nginx-demo.local.show/
```

### 4. 发送 HTTP 请求进行访问测试

现在，你可以通过配置的域名访问你的 Nginx 示例应用了。

```bash
# 如果你修改了 hosts 文件
curl http://nginx-demo.local.show

# 或者直接使用上面带 --resolve 的 curl 命令
curl --resolve "nginx-demo.local.show:80:<NODE_IP>" http://nginx-demo.local.show
```

**预期输出：**
如果一切配置正确，你应该能看到 Nginx 的欢迎页面：

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 5. 故障排查提示

如果访问失败，可以尝试以下步骤进行排查：

1.  **检查 Ingress-Nginx Controller Pods 日志：**
    ```bash
    kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller --tail=100
    ```
    查看是否有错误信息，例如无法连接到后端 Service、配置重载失败等。

2.  **检查示例应用 Pods 日志：**
    ```bash
    kubectl logs -l app=demo-nginx --tail=100
    ```
    确保 Nginx 应用本身运行正常。

3.  **检查 Ingress 资源事件：**
    ```bash
    kubectl describe ingress demo-nginx-ingress
    kubectl get events --field-selector involvedObject.kind=Ingress,involvedObject.name=demo-nginx-ingress
    ```
    查看是否有与 Ingress 配置相关的错误或警告。

4.  **验证 Ingress Controller 是否正确处理了 Ingress 规则：**
    你可以进入 Ingress Controller Pod 内部，检查生成的 Nginx 配置文件 (`nginx.conf`)。
    ```bash
    # 获取一个 Ingress Controller Pod 的名称
    NGINX_CONTROLLER_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')

    # 查看 Nginx 配置 (注意：配置文件路径可能因版本而异)
    kubectl exec -n ingress-nginx -it $NGINX_CONTROLLER_POD -- cat /etc/nginx/nginx.conf
    ```
    在配置文件中搜索你的域名 `nginx-demo.local.show`，确认是否有对应的 `server` 和 `location` 块，以及 `proxy_pass` 是否指向了正确的 Service IP 和端口。

5.  **网络连通性：**
    *   确保你的本地机器可以访问到 Ingress Controller 节点 IP 的 80/443 端口（检查防火墙规则）。
    *   在 Ingress Controller Pod 内部尝试 `curl` 你的 `demo-nginx-service` 的 ClusterIP 和端口，验证 Pod 到 Service 的连通性。

通过以上步骤，你应该能够有效地验证 Ingress-Nginx 的部署，并对常见的测试问题进行排查。

## 总结

通过本文的步骤，你应该能够使用 Helm 成功地将 Ingress-Nginx 以 `DaemonSet + HostNetwork` 模式部署到你的 Kubernetes 集群中。这种模式提供了优异的网络性能，特别适合对延迟敏感的应用。我们还涵盖了镜像处理、常见错误排查、节点标签管理以及一些进阶配置。

请记住，选择哪种部署模式取决于你的具体环境（公有云 vs 私有云）、性能需求、运维复杂度和成本考虑。希望这篇博文能帮助你更好地理解和应用 Ingress-Nginx。

---