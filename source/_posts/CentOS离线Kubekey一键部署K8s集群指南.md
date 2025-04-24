---
title: CentOS离线Kubekey一键部署K8s集群指南
tags:
  - Linux
  - k8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242156648.png'
toc: true
abbrlink: 56f8b097
date: 2024-03-27 21:55:15
---

在国内网络环境复杂的情况下，直接在线安装 Kubernetes 集群常常受限、耗时且易出错。基于此，本文分享利用 [KubeKey](https://kubesphere.io/zh/docs/installing-on-linux/introduction/)，结合离线安装包，实现 CentOS 系统下一键离线部署 Kubernetes 集群的完整流程。该方案针对 Kubernetes v1.23.10、Docker 20.10.8 和 KubeSphere v3.3.2 版本进行了深度优化，支持单 Master 节点与多节点 Worker 节点部署，特别适合生产环境或网络隔离场景。

通过本文，您可以快速搭建稳定、高效的 K8s 集群，免除因网络限制造成的安装困扰，极大提升运维效率。

> 注：当前方案仅支持单 Master 节点配置，多 Master 高可用建议采用二进制手动部署或其他方案。

<!-- more -->

## 环境准备

### 硬件与网络要求

| 节点角色   | IP地址     | 主机名 | CPU/内存    | 硬盘大小  |
|------------|------------|--------|-------------|-----------|
| Master 主节点 | 10.0.0.87 | master | 4核 CPU / 8GB 内存 | 40GB    |
| 普通节点1   | 10.0.0.88 | node1  | 4核 CPU / 8GB 内存 | 40GB    |
| 普通节点2   | 10.0.0.89 | node2  | 4核 CPU / 8GB 内存 | 40GB    |
| 普通节点3   | 10.0.0.90 | node3  | 4核 CPU / 8GB 内存 | 40GB    |

### 基础设置

- 保证所有节点间网络互通，建议简化操作可以暂时关闭防火墙。
- 节点上尽量保持干净环境，确保未安装旧版本 Docker 或 Kubernetes 组件，避免冲突。
- 建议所有节点操作系统均为 CentOS（版本 7.x 以上），符合 Kubernetes 兼容性要求。

## 离线安装流程详解

### 准备离线安装包

1. 在联网环境中下载离线安装包压缩文件：[install-k8s.tar](https://pan.baidu.com/s/1ChJor7eaaKYyoKw46cNjYw?pwd=83n8)
2. 将压缩包上传至 Master 节点或安装主控机内，解压目录：
```bash
tar -mxvf install-k8s.tar && cd install-k8s
```

解压后目录结构包含 KubeKey 二进制文件、Kubernetes、Docker、KubeSphere 等相关离线镜像包，确保安装过程免受网络干扰。

### 集群节点配置文件调整

打开 `install-config.yaml` 文件，根据实际集群节点信息进行修改，确保 IP、账号密码准确无误：

```yaml
spec:
  hosts:
  - {name: master, address: 10.0.0.87, internalAddress: 10.0.0.87, user: root, password: "9ol.(OL>"}
  - {name: node1, address: 10.0.0.88, internalAddress: 10.0.0.88, user: root, password: "9ol.(OL>"}
  - {name: node2, address: 10.0.0.89, internalAddress: 10.0.0.89, user: root, password: "9ol.(OL>"}
  - {name: node3, address: 10.0.0.90, internalAddress: 10.0.0.90, user: root, password: "9ol.(OL>"}
  roleGroups:
    etcd:
    - master
    control-plane:
    - master
    worker:
    - node1
    - node2
    - node3
    registry:
    - master
```

> **提示**：`name` 字段是自定义节点标识，`address` 与 `internalAddress` 是节点内网 IP，`user` 和 `password` 是 ssh 登录凭证。

### 启动安装脚本

执行安装脚本开始集群搭建：

```bash
./install.sh
```

- 安装过程中大约3分钟后，控制台会提示输入 `yes` 以确认继续，这是为确保所有准备步骤已完成。
- 确认后安装会继续执行，预计耗时在10分钟左右，根据服务器性能差异略有波动。
- 安装完成后控制台会输出类似下方成功提示：

![安装完成界面截图](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242153267.png)

## 完成后安装验证

### 访问 Kubernetes Dashboard 及 KubeSphere 控制台

参考安装提示，打开浏览器访问：

```text
http://<主节点外网IP>:30880
```

账号密码如下：

- 账号：admin
- 密码：P@88w0rd

成功登录后，即可通过图形化界面管理集群和应用，极大提升运维效率。

![KubeSphere控制台截图](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242153123.png)

### 使用命令行验证集群状态

- 查询所有命名空间内 Pod 状态：

```bash
kubectl get pod --all-namespaces
```

- 详细查看某个 Pod 拉取镜像及运行状态：

```bash
kubectl describe pod <pod名称> -n <命名空间>
```

- 查看 KubeSphere 安装日志，便于排查安装运行时问题：

```bash
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

## 重要提示与参考资料

- 本方案支持单 Master 节点，若需生产环境下的高可用集群部署，建议采用多 Master 及 ETCD 独立集群的二进制安装方案。
- 在防火墙和网络策略配置严格的环境中，需保证相关端口的互通，具体端口参考官方 Kubernetes 及 KubeSphere 文档。
- 确保离线包和安装脚本来自可信渠道，避免安全风险。

### 相关资料链接

- [KubeKey官方介绍及文档](https://kubesphere.io/zh/docs/installing-on-linux/introduction/air-gapped-installation/)
- [KubeSphere安装视频教程（B站）](https://www.bilibili.com/video/BV1xxxx)（示例参考）
- 镜像及包下载链接：见本文开头百度网盘

---

通过以上步骤，您将收获一个功能完善、稳定可靠的 Kubernetes 集群，兼具 KubeSphere 强大管理能力的企业级平台。无论是线上办公应用还是研发环境，都能迅速上线并顺利运转。

希望本文能助您一臂之力，让 Kubernetes 离线安装变得轻松、便捷！祝您部署成功，如遇问题欢迎留言交流。