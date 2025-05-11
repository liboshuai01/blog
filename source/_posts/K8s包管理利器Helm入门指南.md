---
title: K8s包管理利器Helm入门指南
tags:
  - Linux
  - K8s
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050317333.png'
toc: true
abbrlink: 2eda4ff
date: 2025-05-05 03:15:02
---

大家好！作为一名后端开发者，我们在构建和部署可扩展应用时，经常与 Kubernetes 打交道。Kubernetes 本身非常强大，但管理其上的应用程序配置和部署流程有时会变得复杂。这时，Helm 就闪亮登场了！Helm 被称为 "Kubernetes 的包管理器"，它极大地简化了 Kubernetes 应用的查找、分享、安装和升级过程。

这篇博文将作为一份全面的入门指南，带你了解 **什么是 Helm**，**为什么需要它**，如何在你的 Linux 环境中 **手动安装 Helm**，并掌握 **Helm 仓库管理** 和 **Chart 的基本操作**，为高效管理 Kubernetes 应用打下坚实的基础。

<!-- more -->

## 什么是 Helm？

简单来说，Helm 使用一种称为 **Charts** 的打包格式。一个 Chart 包含了运行一个应用（或应用的一部分）所需的所有 Kubernetes 资源定义（如 Deployments, Services, ConfigMaps 等），以及这些资源的配置参数模板。Helm 通过管理这些 Charts，使得部署复杂应用变得像使用 `apt` 或 `yum` 安装软件包一样简单。

## 为什么要使用 Helm？

*   **简化部署：** 将复杂的应用定义打包成一个简单的 Chart 进行部署。
*   **版本管理：** 轻松跟踪、回滚和升级应用版本。
*   **可重用性：** 创建和分享 Charts，避免重复编写 Kubernetes YAML 文件。
*   **依赖管理：** Chart 可以依赖其他 Chart，方便管理复杂应用的依赖关系。

## 安装 Helm 的先决条件

在开始安装 Helm 之前，请确保你满足以下条件：

1.  **一个 Linux 发行版：** 本指南适用于大多数常见的 Linux 发行版（如 Ubuntu, Debian, CentOS, Fedora, Red Hat 等）。
2.  **`kubectl` 已安装并配置：** Helm 需要与你的 Kubernetes 集群通信，因此你需要安装 `kubectl` 并将其配置为指向你的目标集群。你可以通过运行 `kubectl cluster-info` 来验证连接。
3.  **访问 Kubernetes 集群：** 你需要有权限在你配置的 Kubernetes 集群中部署资源。

## 安装方法：下载二进制版本（手动）

这种方法让你对安装过程有更多的控制，适用于不希望或无法执行远程脚本的环境。你需要从 Helm 的 GitHub Releases 页面下载预编译的二进制文件。

1.  **访问 Releases 页面：** 打开浏览器，访问 Helm 的官方 GitHub Releases 页面：[https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)
2.  **选择版本：** 找到你想要安装的 Helm 版本（通常选择最新的稳定版）。
3.  **下载 Linux AMD64 包：** 在该版本的 "Assets" 部分，找到名为 `helm-vX.Y.Z-linux-amd64.tar.gz` 的文件（将 `X.Y.Z` 替换为你选择的实际版本号）并下载。你可以在终端使用 `wget` 或 `curl` 下载：

    ```bash
    # ！！！请务必将 X.Y.Z 替换为你从 GitHub Releases 页面选择的实际版本号
    # 示例：使用 3.14.2 版本，请根据实际情况修改
    HELM_VERSION="3.14.2"
    wget https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz
    ```

4.  **解压下载的文件：**

    ```bash
    tar -zxvf helm-v${HELM_VERSION}-linux-amd64.tar.gz
    ```

    这会解压出一个名为 `linux-amd64/` 的目录，其中包含 `helm` 可执行文件。

5.  **将 Helm 二进制文件移动到 PATH 路径：** 为了能在任何地方直接运行 `helm` 命令，需要将其可执行文件移动到一个包含在你的系统 `PATH` 环境变量中的目录。`/usr/local/bin` 是一个常用且推荐的位置。

    ```bash
    # 你可能需要使用 sudo 获取写入 /usr/local/bin 的权限
    sudo mv linux-amd64/helm /usr/local/bin/helm
    ```
    **解释:**
    *   `mv linux-amd64/helm ...`: 将解压出来的 `helm` 文件移动到目标位置。
    *   `/usr/local/bin/helm`: 目标路径和文件名。大多数 Linux 系统默认将 `/usr/local/bin` 加入了 `PATH`。

6.  **验证 PATH（可选）：** 确认 `/usr/local/bin` 是否在你的 `PATH` 中。
    ```bash
    echo $PATH
    ```
    如果不在，你需要将其添加到你的 `PATH` (编辑 `~/.bashrc`, `~/.zshrc` 等，然后 `source` 它，或者选择 `PATH` 中已有的其他目录如 `/usr/bin`)。

7.  **清理（可选）：** 删除下载的压缩包和解压出的目录，保持环境整洁。

    ```bash
    rm helm-v${HELM_VERSION}-linux-amd64.tar.gz
    rm -rf linux-amd64/
    ```

## 验证安装

安装完成后，最后都需要验证 Helm 是否成功安装并且可以正常工作。在终端运行：

```bash
helm version
```

如果安装成功，你应该能看到类似以下的输出，显示客户端的版本信息：

```
version.BuildInfo{Version:"v3.14.2", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.7"}
# 注意：这里的版本号应与你下载的版本一致
```

注意：Helm v3 不需要像 v2 那样在集群中安装 Tiller 服务端组件，它直接与 Kubernetes API 服务器交互，因此这里主要显示客户端版本信息。

## 管理 Helm 仓库：发现和获取 Charts

Helm Chart 存储在称为“仓库 (Repository)”的地方。你需要先添加仓库，然后才能从中搜索和安装 Charts。

### 1. 查看已配置的仓库列表

要查看当前系统中已经配置的所有 Helm 仓库，可以使用以下命令：

```bash
helm repo list
```
该命令会列出所有 Helm 仓库的名称及对应的地址。

### 2. 删除默认仓库（可选，加速国内访问）

Helm 默认仓库（如果存在，例如旧版的 'stable'）通常位于国外，访问速度较慢。若不需要，可以将其删除，防止更新超时：

```bash
# 如果存在名为 'stable' 的旧仓库，可以这样删除
# helm repo remove stable
```
*注意：较新版本的 Helm 可能不再默认添加 'stable' 仓库。请根据 `helm repo list` 的实际输出来决定是否需要删除。*

### 3. 添加常用的 Helm 仓库

为了提升访问速度和稳定性，建议添加国内或速度较快的镜像仓库。以下是几个常用示例：

```bash
# 官方仓库（需要魔法）
helm repo add stable https://charts.helm.sh/stable

# Bitnami 官方仓库（内容丰富，推荐）
helm repo add bitnami https://charts.bitnami.com/bitnami

# Azure 中国区镜像仓库 (如果在中国区使用 Azure)
helm repo add azure http://mirror.azure.cn/kubernetes/charts
```
你可以根据实际需求自定义仓库名称，并添加其他可靠的仓库源。

### 4. 更新仓库索引数据

添加或删除仓库后，务必运行以下命令来更新本地的仓库索引文件，获取最新的 Chart 信息：

```bash
helm repo update
```
这将从所有已配置的仓库下载最新的 `index.yaml` 文件。

## Helm Chart 基础操作：搜索、安装与管理

配置好仓库后，就可以开始使用 Helm 来管理 Kubernetes 应用了。

### 1. 搜索 Chart

使用 `helm search repo` 命令在所有已添加并更新的仓库中搜索特定的 Chart：

```bash
# 搜索包含 "mysql" 关键字的 Chart
helm search repo mysql

# 搜索 bitnami 仓库中的 redis
helm search repo bitnami/redis
```
搜索结果会显示 Chart 名称、版本、描述等信息。

### 2. 下载 Chart 包到本地（可选）

如果希望先查看 Chart 的内容、进行修改或进行离线安装，可以使用 `helm pull` 命令将 Chart 下载到本地：

```bash
# 下载 bitnami/redis 的最新版本，并解压
helm pull bitnami/redis --untar

# 下载指定版本的 redis chart
# helm pull bitnami/redis --version <具体版本号> --untar
```
*   `--untar` 参数会在下载后自动解压 Chart 包到一个同名目录。

### 3. 安装 Chart 到 Kubernetes 集群

使用 `helm install` 命令将 Chart 部署到你的 Kubernetes 集群中，创建一个 "Release"（Chart 的一个运行实例）：

```bash
# 示例：安装 bitnami/mysql，创建一个名为 my-db 的 Release
# 安装到 'database' 命名空间，如果该命名空间不存在则自动创建 (--create-namespace)
helm install my-db bitnami/mysql --namespace database --create-namespace

# 重要提示：
# 大多数 Chart 需要配置才能正确运行（例如数据库密码、服务类型等）。
# 查阅 Chart 文档非常重要！可以使用以下命令查看默认配置值：
# helm show values bitnami/mysql
#
# 可以通过 --set 参数在安装时覆盖默认值：
# helm install my-db bitnami/mysql --namespace database --create-namespace \
#   --set auth.rootPassword="your-strong-password" \
#   --set primary.service.type=LoadBalancer
#
# 或者，创建一个自定义的 values.yaml 文件，然后使用 -f 参数指定：
# helm install my-db bitnami/mysql -f my-mysql-values.yaml --namespace database --create-namespace
```

### 4. 查看已部署的 Release

使用 `helm list` (或 `helm ls`) 命令查看已部署的 Release：

```bash
# 查看指定命名空间下的 Release
helm list -n database

# 查看所有命名空间下的 Release
helm list -A
```

### 5. 卸载 Release

当不再需要某个应用实例时，使用 `helm uninstall` 命令将其卸载：

```bash
# 卸载名为 my-db 的 Release (位于 database 命名空间)
helm uninstall my-db -n database
```
Helm 会根据 Chart 定义，删除由该 Release 创建的所有 Kubernetes 资源。

## 总结

Helm 是 Kubernetes 生态中管理应用部署和生命周期的强大助手。通过本篇指南，我们从 Helm 的基本概念出发，学习了如何在 Linux 上手动安装 Helm 客户端，掌握了管理 Helm 仓库以获取 Chart 的方法，并实践了包括搜索、安装、查看和卸载 Chart 在内的核心操作。

熟练运用 Helm，可以让你更高效、更规范地管理日益复杂的 Kubernetes 应用。接下来，你可以进一步探索如何创建自己的 Helm Charts，或者深入了解 Helm 的模板语法、依赖管理、升级与回滚等高级功能。

希望这篇合并后的指南对你的 Kubernetes 之旅有所帮助！如果你在实践中遇到任何问题，或有使用心得，欢迎在评论区留言分享！