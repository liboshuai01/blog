---
title: 在 Linux 上安装 Helm：Kubernetes 包管理利器入门指南
tags:
  - Linux
  - k8s
  - Docker
  - Helm
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050317333.png'
toc: true
abbrlink: 2eda4ff
date: 2025-05-05 03:15:02
---

大家好！作为一名后端开发者，我们在构建和部署可扩展应用时，经常与 Kubernetes 打交道。Kubernetes 本身非常强大，但管理其上的应用程序配置和部署流程有时会变得复杂。这时，Helm 就闪亮登场了！Helm 被称为 "Kubernetes 的包管理器"，它极大地简化了 Kubernetes 应用的查找、分享、安装和升级过程。

今天这篇博文，我们将专注于如何在你的 Linux 环境中 **通过下载预编译的二进制文件来手动安装** Helm 客户端，为后续高效管理 Kubernetes 应用打下基础。

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
    HELM_VERSION="3.14.2" # 这是一个示例版本，请检查并使用最新稳定版或你需要的特定版本
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

注意：此时主要显示客户端版本 (`Client`)。Helm v3 不需要像 v2 那样在集群中安装 Tiller 服务端组件，它直接与 Kubernetes API 服务器交互。

## 开始使用 Helm

安装完成后，你可以开始探索 Helm 的世界了！以下是一些基础命令：

*   **添加 Chart 仓库：** Helm Chart 存储在称为“仓库 (Repository)”的地方。添加一个常用的仓库，比如 Bitnami：
    ```bash
    helm repo add bitnami https://charts.bitnami.com/bitnami
    helm repo update # 更新本地仓库缓存
    ```
*   **搜索 Chart：** 在已添加的仓库中搜索应用，例如搜索 MySQL：
    ```bash
    helm search repo mysql
    ```
*   **安装 Chart：** 部署一个应用（例如，部署一个名为 `my-release` 的 MySQL 实例）：
    ```bash
    # 部署到 'mysql' 命名空间，如果不存在则创建它
    helm install my-release bitnami/mysql --namespace mysql --create-namespace
    ```
    （重要提示：实际部署通常需要查阅该 Chart 的文档 (`helm show values bitnami/mysql`) 并通过 `--set` 参数或自定义 `values.yaml` 文件来配置必要的参数，如密码等。）
*   **查看已部署的版本 (Releases)：**
    ```bash
    helm list -n mysql # 查看 mysql 命名空间下的 releases
    helm list -A         # 查看所有命名空间下的 releases
    ```
*   **卸载 Release：**
    ```bash
    helm uninstall my-release -n mysql
    ```

## 总结

Helm 是 Kubernetes 生态中不可或缺的工具，它极大地简化了应用的打包、分发和管理。通过本篇博文，我们学习了如何在 Linux 系统上手动下载并安装 Helm 的二进制文件。这种方法提供了更多的安装控制权。安装完成后，你就可以利用 Helm 的强大功能来管理你的 Kubernetes 应用了。

希望这篇指南对你有帮助！接下来，你可以开始探索如何创建自己的 Helm Charts，或者深入学习 Helm 的高级用法。

如果你在安装过程中遇到任何问题，或者有关于 Helm 的使用心得，欢迎在评论区留言交流！