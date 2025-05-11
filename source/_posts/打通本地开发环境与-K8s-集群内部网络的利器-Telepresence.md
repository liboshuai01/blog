---
title: 打通本地开发环境与 K8s 集群内部网络的利器 Telepresence
tags:
  - Linux
  - K8s
  - Telepresence
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505111801562.png'
toc: true
abbrlink: b34d7b14
date: 2025-05-11 17:59:18
---

作为后端开发人员，我们经常面临这样的挑战：如何在与生产环境高度一致的环境中测试和调试我们的应用程序。Kubernetes 虽然功能强大，但在尝试从本地开发机器与集群内部运行的服务进行交互时，可能会引入复杂性。为每个开发迭代都通过 NodePort 或 Ingress 暴露服务不仅繁琐，而且并非总是安全的。

**Telepresence** 应运而生：这是一个 CNCF（云原生计算基金会）的项目，它能将您的本地开发环境“传送”到您的 Kubernetes 集群中。它在您的本地机器和集群之间代理网络流量，使您本地运行的代码能够像直接在 Pod内部运行一样访问集群服务（如数据库、消息队列或其他微服务）。同样，它也可以将发往集群中某个服务的流量重定向到您的本地机器。

本指南将引导您在 Windows 计算机上设置 Telepresence，以连接到您的 Kubernetes 集群，从而使您能够（例如）使用本地工具直接访问内部的 Redis 集群。

**目标读者：** 希望简化其 Kubernetes 开发工作流程的 Windows 用户。
**技术栈重点：** Kubernetes (任何发行版，如原生 K8s、K3s、EKS、GKE、AKS)、Windows、PowerShell。

<!-- more -->

### 前提条件

在开始之前，请确保您已具备以下条件：

1.  **一个可用的 Kubernetes 集群：** 这可以是一个远程集群，也可以是本地集群，如 K3s、Minikube 或 Docker Desktop 自带的 Kubernetes。
2.  **`kubectl` 已配置：** 您的 `kubectl` 命令行工具必须已配置为与您的 Kubernetes 集群通信。您可以通过运行 `kubectl cluster-info` 来验证。
3.  **Windows 计算机：** 具有管理员权限，用于安装 Telepresence 及其依赖项。
4.  **PowerShell：**作为安装过程中的命令行界面。

### 第 1 部分：在 Windows 上准备 Kubernetes 配置文件

Telepresence 需要访问您集群的配置文件 (`kubeconfig`) 以了解如何连接。

1.  **在服务器上找到您的 kubeconfig 文件：**
    *   对于标准 Kubernetes 安装 (例如，使用 kubeadm)：文件通常位于您的 Linux 主节点或 `kubectl` 配置的任何位置的 `~/.kube/config`。
    *   对于 K3s 安装：文件通常位于您的 K3s 服务器上的 `/etc/rancher/k3s/k3s.yaml`。

2.  **将 kubeconfig 复制到您的 Windows 计算机：**
    *   如果 `C:\Users\YourUserName\.kube` 目录尚不存在，请创建它 (将 `YourUserName` 替换为您的实际 Windows 用户名，例如 `Administrator`)。
    *   将服务器上 kubeconfig 文件*的内*容复制到此目录下一个名为 `config` 的新文件中：`C:\Users\YourUserName\.kube\config`。
    *   **重要提示：** 如果您复制的是 K3s 的 `k3s.yaml`，请确保 `config` 文件中的 `server` 地址可以从您的 Windows 计算机访问。如果它是 `https://127.0.0.1:6443`，那么只有当 kubectl/Telepresence 与 K3s 服务器在同一台机器上，或者您有适当的端口转发时，它才能工作。对于远程K3s集群，此IP将是K3s服务器的实际IP地址。

### 第 2 部分：在 Windows 上安装 Telepresence 客户端

我们将使用具有管理员权限的 PowerShell 来完成此操作。

```powershell
# 以管理员身份打开 PowerShell

# 1. 下载最新的 Telepresence Windows (amd64) 版本。
#    $ProgressPreference = 'SilentlyContinue' 会隐藏下载进度条，使输出更简洁。
$ProgressPreference = 'SilentlyContinue'
Write-Host "正在下载 Telepresence..."
Invoke-WebRequest https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-windows-amd64.zip -OutFile telepresence.zip
Write-Host "下载完成。"

# 2. 创建一个临时目录并解压缩 zip 文件。
Write-Host "正在解压 Telepresence..."
New-Item -ItemType Directory -Force -Path ".\telepresenceInstaller\telepresence" | Out-Null
Expand-Archive -Path telepresence.zip -DestinationPath ".\telepresenceInstaller\telepresence"
Remove-Item 'telepresence.zip'
cd telepresenceInstaller/telepresence
Write-Host "解压完成。"

# 3. 运行安装脚本。
#    这将安装 Telepresence 的依赖项 (例如用于网络代理的 WinFSP)。
#    默认情况下，它会安装到 C:\telepresence。您可以使用 -Path C:\my\custom\path 指定自定义路径。
#    -ExecutionPolicy bypass 用于仅为此命令允许脚本运行。
Write-Host "正在安装 Telepresence 依赖项..."
powershell.exe -ExecutionPolicy bypass -c " . '.\install-telepresence.ps1';"
Write-Host "依赖项安装完成。"

# 4. 清理临时安装文件。
cd ../..
Remove-Item telepresenceInstaller -Recurse -Confirm:$false -Force
Write-Host "清理完成。"

# 5. (可选但推荐) 将 Telepresence 添加到您的 PATH 环境变量。
#    安装脚本应该会自动执行此操作，但我们在此验证并展示如何手动执行。
#    如果 'C:\telepresence' (或您的自定义路径) 不在您的 PATH 中，请添加它：
#    [System.Environment]::SetEnvironmentVariable("PATH", $env:Path + ";C:\telepresence", "Machine")
#    您需要重新启动 PowerShell 以使 PATH 更改生效。
#    或者，您可以使用其完整路径运行 Telepresence，例如：C:\telepresence\telepresence.exe

Write-Host "Telepresence 安装完成。请重启您的 PowerShell 会话以使 PATH 更改生效。"
Write-Host "您可以在新的 PowerShell 终端中运行 'telepresence version' 进行验证。"
```

> 但是我的电脑执行后，`Telepresence`安装到了`C:\Program Files\telepresence`，并且没有自动配置环境变量，所以需要手动添加环境变量，只能进入到对应目录打开终端，才能使用。

安装完成后，打开一个*新的* PowerShell 终端并验证：

```powershell
telepresence version
```

您应该能看到客户端和 root 守护进程的版本信息。

### 第 3 部分：在您的 Kubernetes 集群中安装 Traffic Manager

Telepresence 使用一个名为 "Traffic Manager" 的组件，它在您的 Kubernetes 集群内部运行以处理代理。

```powershell
# 此命令使用 Helm (与 Telepresence 捆绑) 来安装 Traffic Manager。
telepresence helm install
```

这将在 `ambassador` 命名空间中安装必要的资源 (通常是一个 `traffic-manager` 的 Deployment 和 Service)。您可以验证其状态：

```powershell
kubectl get pods -n ambassador
```

您应该能看到 `traffic-manager` Pod 处于运行状态。

### 第 4 部分：将您的本地计算机连接到集群

现在，让我们建立连接：

```powershell
telepresence connect
```

此命令通过 Traffic Manager 将您的工作站连接到 Kubernetes 集群。它会建立一个类似 VPN 的连接，使您的本地机器成为集群网络的一部分。

检查状态：

```powershell
telepresence status
```

输出应类似于：

```
Root Daemon: Running
  Version   : v2.x.y (api v3)
  DNS       : ...
  Proxy     : ON (networking to the cluster is enabled)
User Daemon: Running
  Version           : v2.x.y (api v3)
  Ambassador Cloud: Logged out
  Kube context      : your-kube-context
  Telepresence proxy: ON (suggest an intercept)
```

一个快速测试，看看您是否可以解析 Kubernetes 内部 DNS：

```powershell
copied
$ curl -ik https://kubernetes.default
```

您应该会收到来自 Kubernetes API 服务器的响应。`HTTP/1.1 401 Unauthorized` 是预期的，因为 `curl` 没有提供身份验证凭据，但它确认了*从您的本地计算机*到 API 服务器的 DNS 解析和网络连接性，就好像它在集群内部一样。

### 第 5 部分：实践案例 – 访问内部 Redis 集群

让我们使用您提供的 Redis 集群示例。您的 `kubectl get all` 输出显示了各种 Redis 服务：

```
NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/kubernetes                          ClusterIP   10.43.0.1       <none>        443/TCP             5d7h
service/redis-cluster-follower              ClusterIP   10.43.229.199   <none>        6379/TCP,9121/TCP   27h
service/redis-cluster-follower-additional   ClusterIP   10.43.61.127    <none>        6379/TCP            27h
service/redis-cluster-follower-headless     ClusterIP   None            <none>        6379/TCP            27h
service/redis-cluster-leader                ClusterIP   10.43.15.7      <none>        6379/TCP,9121/TCP   27h
service/redis-cluster-leader-additional     ClusterIP   10.43.80.36     <none>        6379/TCP            27h
service/redis-cluster-leader-headless       ClusterIP   None            <none>        6379/TCP            27h
```

这些都是 `ClusterIP` 类型的服务，意味着它们通常只能从 Kubernetes 集群内部访问。它们没有配置 `NodePort` 或 `Ingress` 规则来对外暴露。

然而，在 `telepresence connect` 激活的情况下，您的 Windows 计算机现在可以直接使用它们的 Kubernetes 服务 DNS 名称来解析和连接到这些服务。

**使用 Redis Insight (或任何 Redis 客户端) 连接：**

1.  打开 Redis Insight (或您偏好的 Redis GUI/CLI 工具)。
2.  添加新的 Redis 连接时：
    *   **主机 (Host)：** 使用 Kubernetes 服务名称。例如，要连接到主节点实例，您可以使用 `redis-cluster-leader`。如果您的 Redis 集群位于 `default` 以外的命名空间，请使用完全限定名称：`redis-cluster-leader.<namespace>`，例如 `redis-cluster-leader.my-apps`。如果 Telepresence 提示不明确或找不到，尝试使用包含 `.svc.cluster.local` 的 FQDN，例如 `redis-cluster-leader.default.svc.cluster.local`。
    *   **端口 (Port)：** `6379` (根据您的服务定义)。
    *   **密码 (Password)：** 如果您的 Redis 集群受密码保护，请在此处输入。

现在，您应该能够从本地的 Redis Insight 直接连接到在 Kubernetes 内部运行的 Redis 集群了，正如您在截图中展示的那样！

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505111748771.png) *(图片显示通过 Redis Insight 成功连接到 Redis)*

这展示了 Telepresence 的强大之处：无需复杂的端口转发或服务暴露即可访问内部集群资源。

### 使用 Telepresence 的主要优势

*   **简化本地开发：** 在本地运行您的微服务，并让它与开发/测试集群中的其他服务无缝通信。
*   **快速迭代周期 (Inner Loop)：** 快速迭代。在本地更改代码，重新运行，并针对集群中实际的依赖服务进行测试。
*   **真实环境测试：** 针对集群内的实际云资源 (数据库、队列、其他 API) 测试您的本地更改。
*   **强大的调试能力：** 使用您喜欢的本地 IDE 和调试器来单步调试与集群资源交互的代码。
*   **减少依赖模拟 (有时)：** 通过连接到真实实例，减少模拟依赖服务的需求。

### 断开连接与清理

完成工作后：

```powershell
telepresence quit
```

这将断开您的本地计算机与集群网络的连接，并从您的本地系统中移除 Telepresence 守护进程。集群中的 Traffic Manager 将保持不变，除非您明确卸载它 (`telepresence helm uninstall`)。

### 故障排除技巧

*   **防火墙：** 确保您的 Windows 防火墙 (或任何第三方防火墙) 没有阻止 Telepresence 的网络操作。您可能需要添加例外。
*   **VPN 冲突：** 如果您正在使用其他 VPN，它可能会与 Telepresence 冲突。在 Telepresence 激活时，最好断开其他 VPN。
*   **DNS 问题：** 如果无法解析服务名称，请检查 `telepresence status` 中的 DNS 服务器信息。您也可以尝试使用服务的完全限定域名 (FQDN) (例如, `myservice.mynamespace.svc.cluster.local`)。
*   **日志：** 要获取更详细的诊断信息，您可以增加日志级别：
    `telepresence loglevel debug`
    然后运行 `telepresence connect` 并检查 Telepresence 日志目录中的日志 (通常在 `C:\Users\YourUserName\AppData\Local\telepresence\logs`)。

---

这份扩展指南应该为希望在 Windows 上利用 Telepresence 以获得更高效 Kubernetes 开发体验的任何人提供坚实的基础。祝您编码愉快！