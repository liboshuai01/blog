---
title: K8s修改NodePort端口范围详细指南
tags:
  - Linux
  - K8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425102631839.png'
toc: true
abbrlink: a70eb909
date: 2024-03-28 10:24:22
---

在 Kubernetes 集群中，NodePort 是一种常用的 Service 类型，默认分配的端口范围是 30000-32767。这一范围在某些企业网络环境下可能存在限制，导致无法使用默认端口。本文将详细介绍如何修改 Kubernetes 默认 NodePort 端口范围，以满足不同网络策略和安全需求。

<!-- more -->

## 为什么需要修改 NodePort 端口范围

默认的 NodePort 端口范围较大且固定，但某些场景下：

- 企业防火墙策略限制了开放端口范围；
- 端口冲突需要调整可用端口区间；
- 集群资源隔离与管理需要定制端口范围。

因此，调整 NodePort 端口范围成为保障集群服务正常访问的重要操作。

## 修改 kube-apiserver 启动参数来调整端口范围

Kubernetes API Server 通过参数 `--service-node-port-range` 来定义 NodePort 的端口范围。通过调整该参数即可灵活设置所需端口区间。

### 定位配置文件

若您的 Kubernetes 集群是基于 kubeadm 安装，配置文件位于 Master 节点：

```
/etc/kubernetes/manifests/kube-apiserver.yaml
```

此文件为静态 Pod 配置文件，kubelet 会自动根据此文件启动 API Server。

### 修改配置参数

编辑 `kube-apiserver.yaml` 文件，找到 `containers.command` 下的启动参数列表，添加或修改如下参数：

```yaml
- --service-node-port-range=20000-22767
```

这里示例使用的端口范围是 20000-22767，您可根据实际需求调整为其他非冲突且允许开放的端口范围。

示例修改段落如下：

```yaml
        - --service-cluster-ip-range=10.96.0.0/12
        - --service-node-port-range=20000-22767  # 自定义 NodePort 端口范围
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

请确保该参数前无注释符号 `#`，且与其他参数同级缩进。

### 保存更改后自动重启

`/etc/kubernetes/manifests/kube-apiserver.yaml` 是 kubelet 管理的静态 Pod 配置，修改后 kubelet 会自动检测变更并重启 API Server，无需手动重启服务。

## 手动重启 API Server（适用于需要快速刷新配置时）

若环境未能自动重启，可以执行以下命令强制删除 API Server Pod，kubelet 将自动重建：

```bash
kubectl -n kube-system get pods -l component=kube-apiserver -o name | xargs kubectl -n kube-system delete
```

这条命令会删除所有 Master 节点上的 kube-apiserver Pod，确保新参数生效。

## 多 Master 节点环境的注意事项

如您的集群拥有多个 Master 节点，需在每个节点上都重复修改 `kube-apiserver.yaml` 文件，并完成重启，确保全局端口配置一致。

## 验证修改结果

确认修改是否生效，可以通过查看 API Server Pod 启动参数：

```bash
kubectl -n kube-system get pods -l component=kube-apiserver -o jsonpath='{.items[0].spec.containers[0].command}' | tr ',' '\n' | grep service-node-port-range
```

输出类似如下内容即表示配置生效：

```
--service-node-port-range=20000-22767
```

或者使用 `kubectl describe pod` 命令详细查看：

```bash
kubectl -n kube-system describe pod <kube-apiserver-pod-name>
```

在 `Command` 部分查找 `--service-node-port-range` 参数。

## 修改后对已有 NodePort 服务的影响

修改 NodePort 端口范围只影响新创建的 NodePort 服务，已有的服务端口不会自动迁移。若已有 NodePort 服务的端口不在新范围内，将无法正常使用。

因此，您需要针对已部署的 NodePort 服务：

- 删除旧服务
- 重新创建服务，确保端口分配在新的范围内

谨慎操作以避免服务中断。

## 总结与建议

- 通过修改 `kube-apiserver.yaml` 中 `--service-node-port-range` 参数，实现 NodePort 端口范围调整。
- 多节点 Master 集群需逐节点更新配置并重启 API Server。
- 修改生效后需要重新创建 NodePort 服务，保证端口在新范围内。
- 该操作不影响 ClusterIP 或 LoadBalancer 类型的服务。
- 建议在变更前备份相关配置文件，并在测试环境验证。

通过以上步骤，您可以灵活控制 Kubernetes NodePort 服务使用的端口范围，更好地适配企业网络环境和安全策略。

---

希望本指南能帮助您顺利调整 Kubernetes NodePort 配置，提升集群的灵活性和安全性。