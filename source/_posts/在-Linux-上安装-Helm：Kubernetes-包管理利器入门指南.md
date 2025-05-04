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

今天这篇博文，我们将专注于如何在你的 Linux 环境中安装 Helm 客户端，为后续高效管理 Kubernetes 应用打下基础。

<!-- more -->

什么是 Helm？
---

简单来说，Helm 使用一种称为 **Charts** 的打包格式。一个 Chart 包含了运行一个应用（或应用的一部分）所需的所有 Kubernetes 资源定义（如 Deployments, Services, ConfigMaps 等），以及这些资源的配置参数模板。Helm 通过管理这些 Charts，使得部署复杂应用变得像使用 `apt` 或 `yum` 安装软件包一样简单。

为什么要使用 Helm？
---

*   **简化部署：** 将复杂的应用定义打包成一个简单的 Chart 进行部署。
*   **版本管理：** 轻松跟踪、回滚和升级应用版本。
*   **可重用性：** 创建和分享 Charts，避免重复编写 Kubernetes YAML 文件。
*   **依赖管理：** Chart 可以依赖其他 Chart，方便管理复杂应用的依赖关系。

安装 Helm 的先决条件
---

在开始安装 Helm 之前，请确保你满足以下条件：

1.  **一个 Linux 发行版：** 本指南适用于大多数常见的 Linux 发行版（如 Ubuntu, Debian, CentOS, Fedora, Red Hat 등）。
2.  **`kubectl` 已安装并配置：** Helm 需要与你的 Kubernetes 集群通信，因此你需要安装 `kubectl` 并将其配置为指向你的目标集群。你可以通过运行 `kubectl cluster-info` 来验证连接。
3.  **访问 Kubernetes 集群：** 你需要有权限在你配置的 Kubernetes 集群中部署资源。

安装方法
---

我们推荐以下两种在 Linux 上安装 Helm 的常用方法：

### 方法一：使用官方安装脚本（推荐）

这是最简单、最快捷的方法，由 Helm 官方提供。它会自动下载最新稳定版的 Helm，进行校验，并将其安装到 `/usr/local/bin` 目录下。

打开你的终端，执行以下命令：

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

**代码解释：**

1.  `curl -fsSL ...`: 从 Helm 的 GitHub 仓库下载 `get-helm-3` 脚本，并保存为 `get_helm.sh`。
    *   `-f`: 在 HTTP 错误时不显示错误页面内容。
    *   `-s`: 静默模式，不显示进度条。
    *   `-S`: 与 `-s` 结合使用，但仍然显示错误信息。
    *   `-L`: 跟随重定向。
    *   `-o get_helm.sh`: 将下载内容输出到 `get_helm.sh` 文件。
2.  `chmod 700 get_helm.sh`: 赋予脚本执行权限（仅限当前用户）。
3.  `./get_helm.sh`: 执行下载的脚本来安装 Helm。

脚本执行完成后，Helm CLI 就安装好了。你可能需要 **关闭并重新打开你的终端**，或者运行 `source ~/.bashrc` (或 `~/.zshrc` 等，取决于你的 shell) 来让 shell 识别新的 `helm` 命令。

**注意：** 如果你没有 `/usr/local/bin` 的写入权限，脚本可能会提示你需要使用 `sudo` 来执行最后一步安装。

### 方法二：下载二进制版本（手动）

如果你倾向于手动控制安装过程，或者你的环境无法直接运行 `curl | bash` 脚本，可以从 Helm 的 GitHub Releases 页面下载预编译的二进制文件。

1.  **访问 Releases 页面：** 打开浏览器，访问 Helm 的官方 GitHub Releases 页面：[https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)
2.  **选择版本：** 找到你想要安装的 Helm 版本（通常选择最新的稳定版）。
3.  **下载 Linux AMD64 包：** 在该版本的 "Assets" 部分，找到名为 `helm-vX.Y.Z-linux-amd64.tar.gz` 的文件（将 `X.Y.Z` 替换为实际版本号）并下载。你可以使用 `wget` 或 `curl` 在终端下载：

    ```bash
    # 替换 X.Y.Z 为你选择的版本号
    HELM_VERSION="3.14.2" # 示例版本，请检查最新版
    wget https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz
    ```

4.  **解压下载的文件：**

    ```bash
    tar -zxvf helm-v${HELM_VERSION}-linux-amd64.tar.gz
    ```

    这会解压出一个名为 `linux-amd64/` 的目录。

5.  **将 Helm 二进制文件移动到 PATH 路径：** 为了能在任何地方直接运行 `helm` 命令，需要将其移动到一个包含在你的系统 `PATH` 环境变量中的目录。`/usr/local/bin` 是一个常用且推荐的位置。

    ```bash
    # 使用 sudo 获取权限（如果需要）
    sudo mv linux-amd64/helm /usr/local/bin/helm
    ```

6.  **清理（可选）：** 删除下载的压缩包和解压出的目录。

    ```bash
    rm helm-v${HELM_VERSION}-linux-amd64.tar.gz
    rm -rf linux-amd64/
    ```

## 验证安装

无论使用哪种方法安装，最后都需要验证 Helm 是否成功安装并且可以正常工作。在终端运行：

```bash
helm version
```

如果安装成功，你应该能看到类似以下的输出，显示客户端的版本信息：

```
version.BuildInfo{Version:"v3.14.2", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.7"}
```

注意：此时可能只会显示客户端版本 (`Client`)。如果你已经配置好 `kubectl` 并连接到集群，Helm 还会尝试连接 Tiller (Helm v2) 或 Kubernetes API (Helm v3) 来获取服务端信息。对于 Helm v3，通常不需要额外的服务端组件。

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
    helm install my-release bitnami/mysql --namespace mysql --create-namespace # 在指定命名空间安装
    ```
    （注意：实际部署可能需要根据 Chart 文档配置参数）
*   **查看已部署的版本 (Releases)：**
    ```bash
    helm list -n mysql # 查看 mysql 命名空间下的 releases
    ```
*   **卸载 Release：**
    ```bash
    helm uninstall my-release -n mysql
    ```

## 总结

Helm 是 Kubernetes 生态中不可或缺的工具，它极大地简化了应用的打包、分发和管理。通过本篇博文，我们学习了如何在 Linux 系统上使用官方脚本或手动下载二进制文件这两种主要方式来安装 Helm。安装完成后，你就可以利用 Helm 的强大功能来管理你的 Kubernetes 应用了。

希望这篇指南对你有帮助！接下来，你可以开始探索如何创建自己的 Helm Charts，或者深入学习 Helm 的高级用法。

如果你在安装过程中遇到任何问题，或者有关于 Helm 的使用心得，欢迎在评论区留言交流！
