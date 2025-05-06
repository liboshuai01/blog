---
title: k3s集群一键部署
tags:
  - 环境搭建
  - Linux
  - k8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250411142201678.png'
toc: true
abbrlink: 617fd45
date: 2025-04-11 14:24:00
---

K3s 是一个由 Rancher Labs 推出的轻量级 Kubernetes 发行版，专为边缘计算、物联网以及资源有限的环境而设计。
它对 Kubernetes 核心组件进行了精简和优化，通过集成常用插件、内置默认配置以及简化安装流程，大大降低了部署和维护的复杂性，帮助开发者快速建立容器编排环境。

此外，K3s 占用系统资源较低，既适合在生产环境中运行，也可应用于开发测试与小规模集群场景。
它完全兼容标准 Kubernetes API，使得用户可以无缝迁移已有应用，并充分利用 Kubernetes 丰富的生态系统，进一步推动云原生技术的普及。

<!-- more -->

环境推荐
---

- 红帽系列、Ubuntu/Debian、Raspberry PI 树莓派
- 内存至少512MB，推荐1GB
- CPU至少1核心，推荐2核心

集群规则
---

| 主机名    | 角色                   | cpu | 内存  |
|--------|----------------------|-----|-----|
| master | control-plane,master | 2   | 4GB |
| node1  | worker               | 2   | 4GB |
| node2  | worker               | 2   | 4GB |

部署步骤
---

> 注意：下面所有操作步骤均使用`root`用户执行。

1. 分别在`master`、`node1`、`node2`节点执行下面命令，关闭防火墙。

    ```shell
    systemctl stop firewalld && systemctl disable firewalld
    ```

2. 在`master`节点执行下面命令，安装k3s。

    ```shell
    curl -sfL https://get.k3s.io | sh -
    
    # 中国用户，可以使用以下方法加速安装：
    curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -
    ```
   
3. 查看`master`节点生成的token值，用于下面其他节点加入到master中。

    ```shell
    [root@master ~]# cat /var/lib/rancher/k3s/server/token
    K100e11390c5138875992a67d7cceb3c3a8ca032c0454533d39e7d91ee752f155e4::server:d20edf3e50f42e9480177f24dc73d3ae
    ```

4. 分别在`node1`、`node2`节点执行下面命令，安装k3s并加入到master节点。

    ```shell
    curl -sfL https://get.k3s.io | K3S_URL=https://master:6443 K3S_TOKEN=K100e11390c5138875992a67d7cceb3c3a8ca032c0454533d39e7d91ee752f155e4::server:d20edf3e50f42e9480177f24dc73d3ae sh -
    
    # 中国用户，可以使用以下方法加速安装：
    curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://master:6443 K3S_TOKEN=K100e11390c5138875992a67d7cceb3c3a8ca032c0454533d39e7d91ee752f155e4::server:d20edf3e50f42e9480177f24dc73d3ae sh -
    ```

5. 在`master`节点执行下面命令，验证是否安装成功。

    ```shell
    [root@master ~]# kubectl get no
    NAME     STATUS   ROLES                  AGE   VERSION
    master   Ready    control-plane,master   53m   v1.32.3+k3s1
    node1    Ready    <none>                 45m   v1.32.3+k3s1
    node2    Ready    <none>                 46m   v1.32.3+k3s1
    ```
   
6. 添加`kubeconfig`文件路径环境变量到`~/.bashrc`文件最后一行（防止helm等工具无法识别`kubeconfig`文件路径）。

   ```shell
   export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
   ```
   
7. 给`node1`、`node2`节点设置worker角色标签（可选）

   ```shell
   [root@master ~]# kubectl label nodes node1 node-role.kubernetes.io/worker=true
   node/node1 labeled
   [root@master ~]# kubectl label nodes node2 node-role.kubernetes.io/worker=true
   node/node2 labeled
   [root@master ~]# kubectl get no
   NAME     STATUS   ROLES                  AGE    VERSION
   master   Ready    control-plane,master   109m   v1.32.3+k3s1
   node1    Ready    worker                 102m   v1.32.3+k3s1
   node2    Ready    worker                 102m   v1.32.3+k3s1
   ```

结语
---

通过本文的步骤，我们成功完成了 k3s 集群的一键部署，不仅大幅简化了安装流程，也有效降低了硬件资源要求。

构建完成后，你可以进一步探索集群的扩展、安全防护以及自动化维护策略，充分发挥 Kubernetes 的优势，为业务系统的高效运行提供坚实基础。