---
title: K8s采用Helm部署NFS-Client-Provisioner实现动态存储
tags:
  - Linux
  - K8s
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425104445050.png'
toc: true
abbrlink: e3673e0e
date: 2025-05-05 10:43:17                                                                                                                                                               
---

在自建 Kubernetes 集群时，为实现持久化存储，NFS 是一种常见且稳定的选择。为了方便管理和自动化动态存储，NFS Client Provisioner（即 nfs-client-provisioner）成为了社区推荐的解决方案。
本文将详细介绍如何在 Kubernetes 集群中通过 Helm 部署 nfs-client-provisioner（及其继任者 nfs-subdir-external-provisioner），并演示其基本使用方法。

<!-- more -->

## 环境信息

> 笔者系统环境为: RockyLinux 8.10

| 主机名        | IP     | 角色     |
|------------|--------|--------|
| k8s-master | master | master |
| k8s-node1  | node1  | node01 |
| k8s-node2  | node2  | node02 |

## 安装前准备工作

### 安装 NFS 客户端

所有 Kubernetes 节点上都需要安装 NFS 客户端工具以便挂载 NFS 共享目录：

```bash
yum install -y nfs-utils
```

### 搭建 NFS 服务器

本示例选择 `k8s-master` 作为 NFS 服务端节点，进行如下配置：

#### 安装 rpcbind 服务

NFS 服务器需要依赖 `rpcbind` 服务进行远程过程调用绑定，需特别安装并启动：

```bash
yum install -y rpcbind
systemctl enable rpcbind
systemctl start rpcbind
```

#### 创建共享目录

```bash
mkdir -p /data/nfs/k8s
```

#### 配置导出目录

编辑 `/etc/exports` 文件，添加共享目录与客户端权限：

```bash
/data/nfs/k8s *(insecure,rw,sync,no_root_squash)
```

#### 启动 NFS 服务并设置开机自启

```bash
systemctl enable nfs-server
systemctl start nfs-server
```

确保 `rpcbind` 和 `nfs-server` 两个服务均已启动并自启。

#### 重新导出共享目录

使配置生效：

```bash
exportfs -r
exportfs
```

### 验证 NFS 挂载

在任意工作节点（如 `k8s-node1`）测试挂载：

```bash
mkdir -p /data/nfs/k8s # 如果客户端上此目录不存在，则创建
mount -t nfs master:/data/nfs/k8s /data/nfs/k8s
df -h | grep /data/nfs/k8s
```

可在服务器端创建文件，客户端查看是否同步。完成验证后可根据需求卸载：

```bash
umount /data/nfs/k8s
```

## 使用 Helm 安装 NFS Client Provisioner（旧版）

> **注意：** `nfs-client-provisioner` 已经过时并被官方淘汰，现由 Kubernetes SIGs 维护的新项目 `nfs-subdir-external-provisioner`继任，提供更好的维护支持和功能。**建议使用新版 `nfs-subdir-external-provisioner`。**

### 添加 Helm 仓库

```bash
helm repo add azure http://mirror.azure.cn/kubernetes/charts/
helm repo update
```

### 查找 nfs-client-provisioner 版本

```bash
helm search repo nfs-client-provisioner
```

### 安装 nfs-client-provisioner

示例如下，注意替换 `nfs.server` 与 `nfs.path` 至自己的实际 NFS 服务地址和共享路径：

```bash
helm install nfs-storage azure/nfs-client-provisioner \
  --set nfs.server=master \
  --set nfs.path=/data/nfs/k8s \
  --set storageClass.name=nfs-storage \
  --set storageClass.defaultClass=true \
  --set rbac.create=true \
  --namespace kube-system
```

- `nfs.server`：NFS 服务器 IP 地址
- `nfs.path`：NFS 共享目录路径
- `storageClass.name`：定义创建的存储类名称
- `storageClass.defaultClass`：是否作为默认存储类
- `rbac.create`：自动创建 RBAC 资源，需设置为 `true`
- `namespace`：建议安装在 `kube-system` 命名空间，便于管理

### 设置默认 StorageClass 注意事项 (旧版)

在通过 `--set storageClass.defaultClass=true` 将 `nfs-storage` 设置为默认 StorageClass 后，需确保 Kubernetes 集群中不会有多个
StorageClass 被标记为默认，否则会引发 PVC 动态绑定冲突。

**例如，常见的本地存储插件 `local-path` 默认也通常是默认 StorageClass**，需要把它的默认标识取消：

```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

执行后可以确认：

```bash
kubectl get storageclass
```

此时 `nfs-storage` 应该是唯一带有 `(default)` 标识的存储类。

这样做确保：

- 新建 PVC 如果没有特别指定 `storageClassName`，会默认使用 `nfs-storage`。
- 避免多个存储类同时为默认，导致 PVC 动态供给异常。

## 使用 Helm 安装 nfs-subdir-external-provisioner（新版）

**推荐方案：**
`nfs-client-provisioner` 已经过时并被官方淘汰，现由 Kubernetes SIGs 维护的新项目 `nfs-subdir-external-provisioner`
继任，提供更好的维护支持和功能。

### 添加 Helm 仓库并安装

您可以手动执行以下步骤，或者使用提供的 `install.sh` 脚本来自动化此过程。

1.  **手动添加 Helm 仓库：**
    首先，添加 `nfs-subdir-external-provisioner` 的 Helm 仓库并更新本地 Helm 仓库列表。

    ```bash
    helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    helm repo update
    ```

2.  **手动安装 nfs-subdir-external-provisioner：**
    使用 Helm 安装 chart。请务必将 `nfs.server` 和 `nfs.path` 替换为您的 NFS 服务器地址和共享目录。以下命令将安装 `4.0.18` 版本，并将其部署在 `kube-system` 命名空间中（如果不存在则创建），同时创建必要的 RBAC 资源，并将创建的 StorageClass 设置为默认。

    ```bash
    helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --version 4.0.18 \
      --namespace kube-system \
      --create-namespace \
      --set nfs.server=master \
      --set nfs.path=/data/nfs/k8s \
      --set storageClass.name=nfs-storage \
      --set storageClass.defaultClass=true \
      --set rbac.create=true
    ```

    参数说明：
    *   `--version 4.0.18`: 指定安装的 chart 版本，确保部署的确定性。
    *   `--namespace kube-system`: 指定安装的命名空间。
    *   `--create-namespace`: 如果指定的命名空间不存在，则自动创建。
    *   `nfs.server`: NFS 服务器地址（IP 或主机名）。在此示例中为 `master`。
    *   `nfs.path`: NFS 共享目录路径。在此示例中为 `/data/nfs/k8s`。
    *   `storageClass.name`: 自定义存储类名称。在此示例中为 `nfs-storage`。
    *   `storageClass.defaultClass`: 是否设置为默认存储类 (`true`)。
    *   `rbac.create`: 是否创建所需的 RBAC 权限 (`true`)。

#### 使用 `install.sh` 脚本自动化安装

为了简化安装过程，您可以将以下内容保存为 `install.sh` 文件，赋予执行权限 (`chmod +x install.sh`)，然后运行 (`./install.sh`)：

```shell
#!/usr/bin/env bash

set -x # 打印执行的命令

# 添加 nfs-subdir-external-provisioner 的 Helm 仓库
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
# 更新本地 Helm 仓库列表
helm repo update

# 使用 Helm 安装 nfs-subdir-external-provisioner
# 注意：这里的 release 名称是 nfs-subdir-external-provisioner
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --version 4.0.18 \
  --namespace kube-system \
  --create-namespace \
  --set nfs.server=master \
  --set nfs.path=/data/nfs/k8s \
  --set storageClass.name=nfs-storage \
  --set storageClass.defaultClass=true \
  --set rbac.create=true

echo "NFS Subdir External Provisioner installation initiated."
echo "Run 'kubectl get pods -n kube-system -l app.kubernetes.io/name=nfs-subdir-external-provisioner' to check pod status."
echo "Run 'kubectl get storageclass' to check the StorageClass."
```

### 验证安装与 StorageClass

安装完成后，您可以检查 `nfs-storage` StorageClass 是否已成功创建并被标记为默认。

手动检查命令：
```bash
kubectl get storageclass
```
您应该能看到类似如下的输出，其中 `nfs-storage` 带有 `(default)` 标记：
```
NAME                   PROVISIONER                                                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-storage (default)  cluster.local/nfs-subdir-external-provisioner-nfs-subdir...  Delete          Immediate           true                   2m
local-path             rancher.io/local-path                                      Delete          WaitForFirstConsumer   false                  10d
```
*(PROVISIONER 名称可能会因 chart 版本和 release 名称而略有不同，关键是 `nfs-storage` 存在且为 `(default)`)*

#### 使用 `status.sh` 脚本验证

您也可以将以下内容保存为 `status.sh` 文件，赋予执行权限 (`chmod +x status.sh`)，然后运行 (`./status.sh`) 快速检查 `nfs-storage` StorageClass 的状态：

```shell
#!/usr/bin/env bash

set -x # 打印执行的命令

# 获取 StorageClass 列表并筛选出 nfs-storage
kubectl get storageclass | grep nfs-storage
```

### 设置默认 StorageClass 注意事项 (新版)

当您通过 Helm 安装 `nfs-subdir-external-provisioner` 并使用 `--set storageClass.defaultClass=true` 将您创建的 `nfs-storage` 设置为默认 StorageClass 时，必须确保 Kubernetes 集群中这是**唯一**被标记为默认的 StorageClass。否则，PVC 在尝试动态绑定时可能会遇到冲突或选择非预期的 StorageClass。

**处理 K3s 自带的 `local-path` StorageClass：**

K3s 为了提供开箱即用的本地存储解决方案，会默认安装 `local-path-provisioner`，它会自动创建一个名为 `local-path` 的 StorageClass，并且这个 `local-path` 通常也会被设置为默认。

如果您仅执行 `kubectl delete storageclass local-path`，K3s 在后续的某些操作中（如重启服务或内部组件同步）可能会检测到这个它认为的 "核心组件" 丢失并重新创建它。为了彻底阻止 K3s 自动创建 `local-path` StorageClass，并确保您的 `nfs-storage` 是唯一的默认 StorageClass，请遵循以下步骤：

1.  **禁用 K3s 的 `local-storage` 组件：**
    这是最关键的一步，可以从根本上阻止 K3s 创建 `local-path`。您需要修改 K3s 的 systemd 服务单元文件。
    打开 K3s 服务文件 (通常是 `/etc/systemd/system/k3s.service`):
    ```bash
    sudo vim /etc/systemd/system/k3s.service
    ```
    找到 `ExecStart` 行，并在 `k3s server` 命令后添加 `--disable local-storage` 参数。例如：
    ```diff
    ExecStart=/usr/local/bin/k3s \
        server \
    +   --disable local-storage \
        --some-other-existing-options
    ```
    保存文件后，重新加载 systemd 配置并重启 K3s 服务：
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart k3s
    ```

2.  **清理现有的 `local-path` StorageClass (如果K3s重启后仍然存在)：**
    在 K3s 重启并禁用了 `local-storage` 组件后，它应该不会再主动创建 `local-path`。但如果之前它已经存在，或者在您操作期间被短暂重建，您可能需要手动清理它。
*   首先，如果 `local-path` 仍然存在并且是默认的，先取消其默认标记（这一步在禁用组件后可能不再严格必要，但作为保险措施是好的）：
    ```bash
    kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
    ```
*   然后，删除 `local-path` StorageClass：
    ```bash
    kubectl delete storageclass local-path
    ```

3.  **验证 StorageClass 配置：**
    执行以下命令查看当前的 StorageClass 状态：
    ```bash
    kubectl get storageclass
    ```
    您应该能看到 `nfs-storage` 是唯一带有 `(default)` 标识的存储类，并且 `local-path` 不再存在或至少不再是默认。

通过以上步骤，您可以确保：

*   K3s 不会再自动创建或恢复 `local-path` StorageClass。
*   您通过 Helm 安装的 `nfs-storage` 成为集群中唯一的默认 StorageClass。
*   新建的 PersistentVolumeClaim (PVC) 如果没有显式指定 `storageClassName`，将会默认使用 `nfs-storage` 进行动态供给。
*   避免了因多个默认 StorageClass 导致的潜在冲突和 PVC 绑定问题。

## 创建与使用 PersistentVolumeClaim

### 编写 PVC 资源配置

以下示例定义一个名为 `pvc-test` 的 PVC，使用前面创建的 `nfs-storage` 存储类：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-test
spec:
  storageClassName: "nfs-storage" # 确保这里是 nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
```

保存为 `pvc-test.yaml`。

### 创建 PVC

```bash
kubectl apply -f pvc-test.yaml
```

### 查看 PVC 状态

通过命令查看 PVC 状态：

```bash
kubectl get pvc pvc-test
```

理想状态为 `Bound`，表示 PVC 已成功绑定底层 PV。

### 常见错误及处理

- 若出现 `Pending` 状态，说明没有合适的 PV，或者 StorageClass 配置有问题（例如 provisioner pod 未运行，NFS 服务器无法访问等）。检查 `nfs-subdir-external-provisioner` 的 pod 日志 (`kubectl logs -n kube-system -l app.kubernetes.io/name=nfs-subdir-external-provisioner`)。
- 若日志显示 `permission denied`，请确认 NFS 共享目录权限，推荐使用：

  ```bash
  sudo chmod 777 -R /data/nfs/k8s
  sudo chown nfsnobody:nfsnobody -R /data/nfs/k8s # 有时也需要调整属主
  ```

  此权限设置使 Kubernetes 集群中运行的 Pod 能顺利访问 NFS 目录。`no_root_squash` 在 `/etc/exports` 中通常允许 root 写入，但 Pod 内的用户可能不是 root。

### 检查 NFS 服务器存储目录

NFS 服务器 `/data/nfs/k8s` 目录将会自动创建对应 PVC 的存储文件夹，文件夹名称格式通常为：

```
<namespace>-<pvc名称>-<pv名称的一部分>
```
例如： `default-pvc-test-pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

在 PVC 删除后，根据 `ReclaimPolicy`（`nfs-subdir-external-provisioner` 默认为 `Delete`），目录及其内容通常会被删除。如果希望保留，可以将 `ReclaimPolicy` 修改为 `Retain`，此时删除 PVC 后，PV 会进入 `Released` 状态，数据保留在 NFS 上，但目录可能会被 provisioner 重命名为带有 `archived` 前缀的目录（如 `archived-<namespace>-<pvc名称>-<pv名称的一部分>`），方便数据保留和备份。

## 使用 PVC 挂载到 Pod 中

以下是一个示例 Deployment 配置，展示如何将创建的 PVC 挂载到 Pod 中：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  # namespace: mldong-test # 如果需要在特定命名空间，请取消注释并确保PVC也在该命名空间或可被访问
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-pvc
              mountPath: /usr/share/nginx/html
      volumes:
        - name: nginx-pvc
          persistentVolumeClaim:
            claimName: pvc-test # PVC名称
```

部署后，Pod 会使用指定 PVC 挂载对应的 NFS 存储，数据实现持久化。

## 资源卸载

### 删除 PVC

如果不再需要某个 PVC，可以先删除使用该 PVC 的 Pod，然后删除 PVC 定义。例如，删除上面示例中创建的 `pvc-test`：
(假设 `pvc-test.yaml` 是之前创建 PVC 的文件)

```bash
kubectl delete deployment nginx # 如果部署了上面的nginx示例
kubectl delete -f pvc-test.yaml
```

### 卸载 nfs-subdir-external-provisioner

若不再需要 `nfs-subdir-external-provisioner` 服务，可以通过 Helm 将其卸载。
以下命令将卸载之前通过 Helm 安装的、名为 `nfs-subdir-external-provisioner` 的 release，该 release 部署在 `kube-system` 命名空间中。

手动卸载命令：
```bash
helm uninstall nfs-subdir-external-provisioner -n kube-system
```

#### 使用 `uninstall.sh` 脚本自动化卸载

您也可以将以下内容保存为 `uninstall.sh` 文件，赋予执行权限 (`chmod +x uninstall.sh`)，然后运行 (`./uninstall.sh`) 来简化卸载过程：

```shell
#!/usr/bin/env bash

set -x # 打印执行的命令

# 使用 Helm 卸载 nfs-subdir-external-provisioner
# 假设 release 名称为 nfs-subdir-external-provisioner (与 install.sh 中使用的名称一致)
# 且位于 kube-system 命名空间
helm uninstall nfs-subdir-external-provisioner -n kube-system

echo "NFS Subdir External Provisioner uninstallation initiated."
echo "You may also want to remove the Helm repository if no longer needed:"
echo "helm repo remove nfs-subdir-external-provisioner"
# 并且手动清理 StorageClass (如果helm uninstall 未自动删除)
# kubectl delete storageclass nfs-storage
```
卸载 Helm release 会移除相关的 Deployment、ServiceAccount、RBAC 规则。StorageClass `nfs-storage` 通常也会被一并删除，但最好检查一下。NFS 服务器上的数据目录不会被自动删除，需要您根据需求手动清理。

### 卸载 nfs-client-provisioner (旧版)

若不再使用旧版 `nfs-client-provisioner`，也可以通过 Helm 删除：

```bash
helm uninstall nfs-storage -n kube-system # 假设release名为nfs-storage
```

## 总结

通过上述步骤，您能够在 Kubernetes 集群中使用 Helm 快速部署 `nfs-subdir-external-provisioner`（或旧版的 `nfs-client-provisioner`），实现动态、共享的持久化存储。需要特别注意的是，NFS
服务器端除了安装 `nfs-utils`，还必须安装并启动 `rpcbind` 服务，否则 NFS 服务无法正常工作。此方案适合多节点访问共享文件系统场景。后续若有更复杂
NFS 存储策略管理或权限控制，需要结合具体业务需求做进一步配置和优化。

本文提供了部署的基础思路及示例配置，建议优先采用新版的 `nfs-subdir-external-provisioner`。希望对 Kubernetes 持久化存储实践有所帮助。欢迎在实际环境里根据自己的需求进行扩展和深入学习。