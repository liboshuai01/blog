---
title: Kubernetes使用Helm部署NFS-Client-Provisioner实现动态存储
tags:
  - Linux
  - k8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425104445050.png'
toc: true
abbrlink: e3673e0e
date: 2024-04-15 10:43:17                                                                                                                                                               
---

在自建 Kubernetes 集群时，为实现持久化存储，NFS 是一种常见且稳定的选择。为了方便管理和自动化动态存储，NFS Client Provisioner（即 nfs-client-provisioner）成为了社区推荐的解决方案。本文将详细介绍如何在 Kubernetes 集群中通过 Helm 部署 nfs-client-provisioner，并演示其基本使用方法。

<!-- more -->

## 环境信息

本示例基于三台华为软开云 ECS 服务器，操作系统为 CentOS Linux 7.6。集群组成如下：

| 主机名   | IP             | 角色   |
| -------- | -------------- | ------ |
| mldong01 | 192.168.0.245  | master |
| mldong02 | 192.168.0.54   | node01 |
| mldong03 | 192.168.0.22   | node02 |

## 安装前准备工作

### 安装 NFS 客户端

所有 Kubernetes 节点上都需要安装 NFS 客户端工具以便挂载 NFS 共享目录：

```bash
yum install -y nfs-utils
```

### 搭建 NFS 服务器

本示例选择 `mldong01` 作为 NFS 服务端节点，进行如下配置：

#### 安装 rpcbind 服务

NFS 服务器需要依赖 `rpcbind` 服务进行远程过程调用绑定，需特别安装并启动：

```bash
yum install -y rpcbind
systemctl enable rpcbind
systemctl start rpcbind
```

#### 创建共享目录

```bash
mkdir -p /mnt/nfs
```

#### 配置导出目录

编辑 `/etc/exports` 文件，添加共享目录与客户端权限：

```bash
/mnt/nfs 192.168.0.0/24(rw,sync,fsid=0)
```

- `192.168.0.0/24` 表示允许同网段内的客户端访问
- `rw` 允许读写访问
- `sync` 表示同步写入，确保数据一致性
- `fsid=0` 将此目录作为 NFS 根目录

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

在任意工作节点（如 `mldong02`）测试挂载：

```bash
mkdir -p /mnt/nfs
mount -t nfs 192.168.0.245:/mnt/nfs /mnt/nfs
df -h | grep /mnt/nfs
```

可在服务器端创建文件，客户端查看是否同步。完成验证后可根据需求卸载：

```bash
umount /mnt/nfs
```

## 使用 Helm 安装 NFS Client Provisioner

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
  --set nfs.server=192.168.0.245 \
  --set nfs.path=/mnt/nfs \
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

### 验证 StorageClass

安装成功后，查看当前存储类：

```bash
kubectl get storageclass
```

您将看到 `nfs-storage` 已成为默认存储类。

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
  chmod 777 -R /mnt/nfs/k8s
  ```

  此权限设置使 Kubernetes 集群中运行的 Pod 能顺利访问 NFS 目录。

### 检查 NFS 服务器存储目录

NFS 服务器 `/mnt/nfs/k8s` 目录将会自动创建对应 PVC 的存储文件夹，文件夹名称格式为：

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

通过上述步骤，您能够在 Kubernetes 集群中使用 Helm 快速部署 nfs-client-provisioner，实现动态、共享的持久化存储。需要特别注意的是，NFS 服务器端除了安装 `nfs-utils`，还必须安装并启动 `rpcbind` 服务，否则 NFS 服务无法正常工作。此方案适合多节点访问共享文件系统场景。后续若有更复杂 NFS 存储策略管理或权限控制，需要结合具体业务需求做进一步配置和优化。

本文提供了部署的基础思路及示例配置，希望对 Kubernetes 持久化存储实践有所帮助。欢迎在实际环境里根据自己的需求进行扩展和深入学习。