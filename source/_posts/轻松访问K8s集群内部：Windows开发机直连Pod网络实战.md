---
title: 轻松访问K8s集群内部：Windows开发机直连Pod网络实战
tags:
  - Linux
  - K8s
  - Windows
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505120207311.png'
toc: true
abbrlink: 2b92c30b
date: 2025-05-12 02:05:52
---

作为后端开发者，我们经常需要在本地开发环境中与Kubernetes集群进行交互，特别是在调试或本地测试时，能够直接访问集群内部的Pod和服务网络会极大提高效率。本文将详细介绍如何在同一个局域网环境下，通过配置静态路由和DNS，使您的Windows开发机能够直接访问运行在Linux机器上K3s集群中的Pod网络（以访问`redis-operator`部署的`redis-cluster`为例）。

这种方法利用了网络第三层（网络层）的静态路由原理，直接告诉您的Windows系统或网络网关，发往特定IP段（K8s Pod/Service CIDR）的流量应该导向哪个K8s节点。

<!-- more -->

### 适用场景与优缺点

**适用场景：**

1.  Kubernetes集群（如K3s）和开发机器在同一个局域网内。
2.  您有权限在Windows开发机器上添加静态路由。
3.  主要用于内部开发或测试环境，对网络暴露有一定容忍度。

**优点：**

*   **配置简单：** 无需额外部署复杂组件，仅需在开发机上配置静态路由和DNS。
*   **透明高效：** 对开发者几乎透明，网络访问直达，性能较好。
*   **原生体验：** 可以像访问局域网内其他服务一样访问Pod IP和Service域名。

**缺点：**

*   **网络规划：** Pod/Service网段不能和本地局域网的其他网段冲突。
*   **单点路由：** 如果路由指向的K8s节点宕机，访问会中断（除非有更复杂的路由冗余方案，但超出了本文范围）。
*   **安全性：** 相当于将集群内部网络部分暴露给了配置了路由的机器，需评估安全风险。
*   **不可移植：** 此配置仅对当前Windows机器有效。若要在整个局域网内生效，需在网关路由器上配置，这通常需要网络管理员权限。

### 操作步骤

#### 前提条件

1.  Windows开发机与K3s集群所在的Linux机器在同一个局域网内。
2.  您拥有Windows开发机的管理员权限。
3.  K3s集群已正常运行，并且CNI网络插件（K3s默认为Flannel）已正确配置Pod网络。

#### 步骤一：确定K3s集群任意节点的局域网IP地址

首先，您需要获取K3s集群中任何一个节点的局域网IP地址。这个节点将作为流量转发到K8s内部网络的入口。通常选择Master节点或一个稳定的Worker节点。

登录到您的K3s节点（例如Master节点），执行以下命令查看IP地址：

```shell
ip addr show
```

假设您查到的`eth0`网卡（或其他主网卡）的IP地址是 `192.168.6.202`。记下这个IP，后文称之为 `<K3S_NODE_LAN_IP>`。

```
[root@master ~]# ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:06:f6:04 brd ff:ff:ff:ff:ff:ff
    inet 192.168.6.202/24 brd 192.168.6.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    ...
```

#### 步骤二：确定K3s集群的Pod CIDR和Service CIDR

Kubernetes集群使用不同的IP范围来分配给Pods和服务(Services)。

1.  **Pod CIDR (Pod Subnet):** K3s在启动时会配置Pod网络的CIDR。
    *   **默认值：** K3s使用Flannel作为默认CNI，其Pod CIDR通常是 `10.42.0.0/16`。
    *   **查看方法：**
        *   如果K3s server启动时指定了 `--cluster-cidr` 参数，该参数值即为Pod CIDR。
        *   可以检查K3s server进程的启动参数：
            ```shell
            ps aux | grep 'k3s server'
            ```
        *   或者，如果集群已经运行，可以尝试从节点信息中获取（注意：这依赖于CNI是否正确更新了Node Spec）：
            ```shell
            kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
            ```
        *   对于Flannel，也可以在K3s节点上查看 `flannel.1` 接口的IP，通常其网络部分是Pod子网的一部分，而`cni0`桥接器通常会分配该子网的`.1`地址。从您的`ip addr show`输出中：
            ```
            13: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
                inet 10.42.0.0/32 scope global flannel.1  # 这是该节点上的flannel网络端点
            14: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default qlen 1000
                inet 10.42.0.1/24 brd 10.42.0.255 scope global cni0 # 这是该节点上Pod的网关
            ```
            这表明此节点的Pod子网是 `10.42.0.0/24`，而整个集群的Pod CIDR通常是 `10.42.0.0/16`（允许更多节点和子网）。

2.  **Service CIDR (Service Subnet):** K3s在启动时也会配置Service网络的CIDR。
    *   **默认值：** K3s的Service CIDR通常是 `10.43.0.0/16`。
    *   **查看方法：**
        *   如果K3s server启动时指定了 `--service-cidr` 参数，该参数值即为Service CIDR。
        *   同样可以检查K3s server进程的启动参数。
        *   也可以通过 `kubectl cluster-info dump`（输出较多，需要筛选）：
            ```shell
            # kubectl cluster-info dump | grep service-cluster-ip-range
            # 或者更简单地查看API Server的Service IP，它一定在Service CIDR内
            kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}'
            ```
            如果kubernetes service的IP是 `10.43.0.1`，那么Service CIDR很可能是 `10.43.0.0/16`。

根据上述方法，我们假设最终确定：
*   **Pod CIDR:** `10.42.0.0/16` (子网掩码 `255.255.0.0`)
*   **Service CIDR:** `10.43.0.0/16` (子网掩码 `255.255.0.0`)

#### 步骤三：在Windows开发机上添加静态路由

现在，我们需要告诉Windows系统，所有发往Pod CIDR和Service CIDR的IP包都应该通过我们在步骤一中确定的K3s节点 (`<K3S_NODE_LAN_IP>`)。

1.  以 **管理员权限** 打开命令提示符 (CMD) 或 PowerShell。
2.  执行以下命令（请将 `<K3S_NODE_LAN_IP>` 替换为您的实际K3s节点IP，例如 `192.168.6.202`）：

    ```cmd
    REM 添加到Pod网络的路由
    route -p ADD 10.42.0.0 MASK 255.255.0.0 <K3S_NODE_LAN_IP> METRIC 1

    REM 添加到Service网络的路由
    route -p ADD 10.43.0.0 MASK 255.255.0.0 <K3S_NODE_LAN_IP> METRIC 1
    ```
    例如：
    ```cmd
    route -p ADD 10.42.0.0 MASK 255.255.0.0 192.168.6.202 METRIC 1
    route -p ADD 10.43.0.0 MASK 255.255.0.0 192.168.6.202 METRIC 1
    ```
    *   `-p`: 使路由持久化，系统重启后依然有效。
    *   `METRIC 1`: 指定路由的跃点数（成本），较低的metric值表示较高的优先级。

3.  验证路由是否添加成功：
    ```cmd
    route print
    ```
    在输出的 "IPv4 路由表" -> "永久路由" (Persistent Routes) 或 "活动路由" (Active Routes) 部分，您应该能看到新添加的这两条路由。

#### 步骤四：配置Windows DNS以解析K8s内部域名

为了能够通过服务名（例如 `my-service.default.svc.cluster.local`）访问K8s服务，您的Windows机器需要将K8s集群内部的DNS服务器（通常是CoreDNS）作为其DNS解析器。

1.  **获取K3s CoreDNS Service IP：**
    在您的K3s Master节点上执行：
    ```shell
    # kubectl get svc -n kube-system kube-dns
    # 或（更常见的名称）:
    kubectl get svc -n kube-system coredns
    ```
    您会看到类似输出：
    ```
    NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
    kube-dns  ClusterIP   10.43.0.10   <none>        53/UDP,53/TCP,9153/TCP   5d
    ```
    记下这个 `CLUSTER-IP`，例如 `10.43.0.10`。这是您K3s集群的DNS服务器地址。**注意：** 这个IP属于Service CIDR，上一步添加的静态路由使得它可以被访问。

2.  **在Windows上配置DNS服务器：**
    *   打开 "控制面板" -> "网络和 Internet" -> "网络和共享中心"。
    *   点击您当前连接的网络（例如 "以太网"）。
    *   点击 "属性"。
    *   选择 "Internet 协议版本 4 (TCP/IPv4)"，然后点击 "属性"。
    *   选择 "使用下面的 DNS 服务器地址"。
    *   在 "首选 DNS 服务器" 中填入您上一步获取的K3s CoreDNS Service IP (例如 `10.43.0.10`)。
    *   **重要：** **不要在 "备用 DNS 服务器" 中填写任何公共DNS服务器**（如 `8.8.8.8` 或您的路由器IP）。如果填写了备用DNS，Windows可能会在首选DNS（CoreDNS）解析内部域名超时或失败时（这对于不存在的内部域名是正常的），转而查询备用DNS，导致内部域名解析失败。如果确实需要备用DNS来解析公网域名，您应该配置CoreDNS使其能够正确转发公网域名的解析请求（通常CoreDNS默认会使用宿主机的 `/etc/resolv.conf` 进行上游转发）。
    *   点击 "确定" 保存设置。

3.  **刷新Windows DNS缓存：**
    以管理员权限打开命令提示符，执行：
    ```cmd
    ipconfig /flushdns
    ```

#### 步骤五：理解目标服务和Pod的FQDN（以Redis Cluster为例）

假设您使用 `redis-operator` 部署了一个Redis集群，它可能会创建多个Services，包括Headless Services用于直接发现Pod。

查看Service资源信息：

```shell
[root@master ~]# kubectl get svc -n default # 假设部署在default命名空间
NAME                                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
kubernetes                          ClusterIP   10.43.0.1       <none>        443/TCP             5d16h
redis-cluster-follower              ClusterIP   10.43.229.199   <none>        6379/TCP,9121/TCP   36h
redis-cluster-follower-additional   ClusterIP   10.43.61.127    <none>        6379/TCP            36h
redis-cluster-follower-headless     ClusterIP   None            <none>        6379/TCP            36h
redis-cluster-leader                ClusterIP   10.43.15.7      <none>        6379/TCP,9121/TCP   36h
redis-cluster-leader-additional     ClusterIP   10.43.80.36     <none>        6379/TCP            36h
redis-cluster-leader-headless       ClusterIP   None            <none>        6379/TCP            36h
```

对于Headless Service (如 `redis-cluster-leader-headless`)，它的DNS记录会直接解析到支持该服务的所有Pod的IP地址。Pod的完全限定域名 (FQDN) 格式通常是：
`<pod-name>.<headless-service-name>.<namespace>.svc.cluster.local`

基于此，您的Redis Pods FQDN可能如下（假设Pod名称为 `redis-cluster-leader-0/1/2` 和 `redis-cluster-follower-0/1/2`，且在 `default` 命名空间）：

*   **Leader Pods:**
    *   `redis-cluster-leader-0.redis-cluster-leader-headless.default.svc.cluster.local`
    *   `redis-cluster-leader-1.redis-cluster-leader-headless.default.svc.cluster.local`
    *   `redis-cluster-leader-2.redis-cluster-leader-headless.default.svc.cluster.local`
*   **Follower Pods:**
    *   `redis-cluster-follower-0.redis-cluster-follower-headless.default.svc.cluster.local`
    *   ...以此类推。

您也可以直接连接到非Headless Service的`CLUSTER-IP`，例如 `redis-cluster-leader` 的 `10.43.15.7`。

#### 步骤六：测试连接

1.  **测试DNS解析：**
    在Windows命令提示符中，尝试解析一个Pod的FQDN或Service的名称：
    ```cmd
    nslookup redis-cluster-leader-0.redis-cluster-leader-headless.default.svc.cluster.local
    nslookup redis-cluster-leader.default.svc.cluster.local
    ```
    您应该能看到它们分别解析为对应的Pod IP地址（例如 `10.42.x.y`）和Service IP地址（例如 `10.43.x.y`）。

2.  **测试网络连通性 (Ping Pod IP或Service IP)：**
    首先获取一个Pod的实际IP地址：
    ```shell
    [root@master ~]# kubectl get pods -o wide
    # 找到例如 redis-cluster-leader-0 的 IP，比如是 10.42.0.5
    ```
    在Windows命令提示符中Ping该Pod IP和Service IP：
    ```cmd
    ping 10.42.0.5     REM 替换为实际Pod IP
    ping 10.43.15.7    REM 替换为实际Service IP (e.g., redis-cluster-leader)
    ```
    如果Ping通了，说明路由和网络层面基本通畅。注意：某些CNI或Pod配置可能会禁止ICMP (ping)。

3.  **使用Redis客户端连接：**
    您可以使用`redis-cli` (如果已安装) 或其他Redis GUI工具。
    *   **连接到Headless Service的特定Pod FQDN：**
        ```cmd
        redis-cli -h redis-cluster-leader-0.redis-cluster-leader-headless.default.svc.cluster.local -p 6379
        ```
    *   **连接到普通Service的FQDN或ClusterIP：**
        ```cmd
        redis-cli -h redis-cluster-leader.default.svc.cluster.local -p 6379
        REM 或者直接用Service IP
        redis-cli -h 10.43.15.7 -p 6379
        ```
    *   **如果您的Redis客户端支持集群模式（`-c` 选项），连接任意一个节点即可让客户端自动发现其他节点：**
        ```cmd
        redis-cli -c -h redis-cluster-leader-0.redis-cluster-leader-headless.default.svc.cluster.local -p 6379
        ```

### 故障排除

*   **无法ping通Pod/Service IP：**
    1.  **检查Windows路由表：** `route print` 确认静态路由是否存在且正确（目标网络、掩码、网关、接口、Metric）。
    2.  **检查K3s节点IP：** 确保路由中指定的 `<K3S_NODE_LAN_IP>` 是正确的且可从Windows访问。
    3.  **检查K3s节点防火墙：** 确保K3s节点（如 `192.168.6.202`）的防火墙（如`firewalld`, `ufw`, `iptables`）允许来自您Windows开发机IP的转发流量，或者允许发往Pod/Service CIDR的流量。特别是`FORWARD`链。
        *   确保IP转发已在K3s节点上启用: `sysctl net.ipv4.ip_forward` (应为1)。
        *   K3s通常（例如使用flannel）会配置必要的iptables规则。但外部防火墙可能会干扰。
    4.  **检查CNI插件：** 确保K3s集群的CNI插件运行正常，Pod之间网络互通。

*   **DNS解析失败 (`nslookup` 找不到域名)：**
    1.  **检查Windows DNS配置：** 确保首选DNS服务器是CoreDNS的ClusterIP，且没有配置备用DNS。
    2.  **尝试 `nslookup <domain> <coredns_ip>`：** 例如 `nslookup kubernetes.default.svc.cluster.local 10.43.0.10`。如果这个成功，但 `nslookup <domain>` 失败，说明Windows系统没有正确使用您配置的DNS。
    3.  **检查CoreDNS运行状态：** 在K3s集群中查看CoreDNS Pod的日志：`kubectl logs -n kube-system -l k8s-app=kube-dns`。
    4.  **刷新DNS缓存：** `ipconfig /flushdns`。

*   **DNS解析成功，但无法连接服务：**
    这通常指向网络路由问题（见上文）或服务本身的问题（例如Pod未运行，端口不正确）。

### 总结

通过在Windows开发机上配置静态路由并将其DNS指向K3s集群的CoreDNS，您可以方便地从本地直接访问K3s集群内部的Pod和服务。这对于开发和调试非常有用。但请务必注意潜在的IP冲突和安全影响。如果此方案不适用（例如，无法修改本地路由或DNS，或对安全性有更高要求），可以考虑`kubectl port-forward`、VPN接入集群网络，或使用Telepresence、kubefwd等更高级的本地开发工具。

希望这篇指南能帮助您更顺畅地进行Kubernetes相关的开发工作！

---