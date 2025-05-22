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
本文将详细介绍如何在 Kubernetes 集群中通过 Helm 部署 nfs-client-provisioner，并演示其基本使用方法。

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
mkdir -p /data/nfs/k8s
mount -t nfs master:/data/nfs/k8s /data/nfs/k8s
df -h | grep /data/nfs/k8s
```

可在服务器端创建文件，客户端查看是否同步。完成验证后可根据需求卸载：

```bash
umount /data/nfs/k8s
```

## 使用 Helm 安装 NFS Client Provisioner（旧版）

> `nfs-client-provisioner` 已经过时并被官方淘汰，现由 Kubernetes SIGs 维护的新项目 `nfs-subdir-external-provisioner`继任，提供更好的维护支持和功能。

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

### 设置默认 StorageClass 注意事项

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

**注意：**
`nfs-client-provisioner` 已经过时并被官方淘汰，现由 Kubernetes SIGs 维护的新项目 `nfs-subdir-external-provisioner`
继任，提供更好的维护支持和功能。

### 添加 Helm 仓库

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
```

### 安装 nfs-subdir-external-provisioner

示例如下，请将 `nfs.server` 和 `nfs.path` 替换为自己的 NFS 服务器地址和共享目录：

```bash
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=master \
  --set nfs.path=/data/nfs/k8s \
  --set storageClass.name=nfs-storage \
  --set storageClass.defaultClass=true \
  --set rbac.create=true \
  --namespace kube-system
```

参数说明与之前类似：

- `nfs.server`：NFS 服务器地址（IP 或主机名）
- `nfs.path`：NFS 共享目录路径
- `storageClass.name`：自定义存储类名称
- `storageClass.defaultClass`：是否设置为默认存储类
- `rbac.create`：是否创建所需的 RBAC 权限
- `namespace`：推荐安装在 `kube-system` 命名空间

### 设置默认 StorageClass 注意事项

当您通过 Helm 安装 `nfs-subdir-external-provisioner` (或旧版的 `nfs-client-provisioner`) 并使用 `--set storageClass.defaultClass=true` 将您创建的 `nfs-storage` 设置为默认 StorageClass 时，必须确保 Kubernetes 集群中这是**唯一**被标记为默认的 StorageClass。否则，PVC 在尝试动态绑定时可能会遇到冲突或选择非预期的 StorageClass。

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
  storageClassName: "nfs-storage"
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

- 若出现 `Pending` 状态，说明没有合适的 PV 或存储类配置有问题。
- 若日志显示 `permission denied`，请确认 NFS 共享目录权限，推荐使用：

  ```bash
  chmod 777 -R /data/nfs/k8s
  ```

  此权限设置使 Kubernetes 集群中运行的 Pod 能顺利访问 NFS 目录。

### 检查 NFS 服务器存储目录

NFS 服务器 `/data/nfs/k8s` 目录将会自动创建对应 PVC 的存储文件夹，文件夹名称格式为：

```
<namespace>-<pvc名称>-<pvc UID>-<随机字符串>
```

在 PVC 删除后，目录不会立刻被删除，而是会被重命名为带有 `archived` 前缀的目录，方便数据保留和备份。

## 使用 PVC 挂载到 Pod 中

以下是一个示例 Deployment 配置，展示如何将创建的 PVC 挂载到 Pod 中：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: mldong-test
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
            claimName: pvc-test
```

部署后，Pod 会使用指定 PVC 挂载对应的 NFS 存储，数据实现持久化。

## 资源卸载

### 删除 PVC

```bash
kubectl delete -f pvc-test.yaml
```

### 卸载 nfs-client-provisioner

若不再使用，也可以通过 Helm 删除：

```bash
helm uninstall nfs-storage -n kube-system
```

## 总结

通过上述步骤，您能够在 Kubernetes 集群中使用 Helm 快速部署 nfs-client-provisioner，实现动态、共享的持久化存储。需要特别注意的是，NFS
服务器端除了安装 `nfs-utils`，还必须安装并启动 `rpcbind` 服务，否则 NFS 服务无法正常工作。此方案适合多节点访问共享文件系统场景。后续若有更复杂
NFS 存储策略管理或权限控制，需要结合具体业务需求做进一步配置和优化。

本文提供了部署的基础思路及示例配置，希望对 Kubernetes 持久化存储实践有所帮助。欢迎在实际环境里根据自己的需求进行扩展和深入学习。