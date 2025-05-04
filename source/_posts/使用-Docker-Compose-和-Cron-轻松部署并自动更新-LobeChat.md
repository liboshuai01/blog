---
title: 使用 Docker Compose 和 Cron 轻松部署并自动更新 LobeChat
tags:
  - Linux
  - Docker
  - Ai
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050208258.png'
toc: true
abbrlink: 302b2d3b
date: 2025-05-05 02:07:18
---

LobeChat 是一个开源、可扩展的高性能聊天机器人框架，支持多 AI 提供商。对于希望自托管 AI 聊天服务的开发者和用户来说，它是一个很棒的选择。Docker 和 Docker Compose 使得部署 LobeChat 变得异常简单。更进一步，我们可以设置一个简单的 Cron 任务来自动拉取最新镜像并更新我们的 LobeChat 实例，确保我们始终使用最新功能和安全修复。

本文将引导您完成以下步骤：

1.  使用 `docker-compose.yml` 文件快速部署 LobeChat。
2.  创建一个 Bash 脚本来自动检查并应用 LobeChat 更新。
3.  使用 Cron 定时任务实现每日自动更新。

<!-- more -->

### 前提条件

在开始之前，请确保您的服务器（或本地机器）已安装：

*   **Docker**: [安装 Docker](https://docs.docker.com/engine/install/)
*   **Docker Compose**: [安装 Docker Compose](https://docs.docker.com/compose/install/) (通常随 Docker Desktop 一起安装，或作为 Docker 插件)。
*   **Bash环境**: 大多数 Linux 发行版和 macOS 都自带。
*   **(可选) Cron**: Linux/macOS 系统通常自带的定时任务工具。

### 步骤 1: 使用 Docker Compose 部署 LobeChat

Docker Compose 允许我们使用 YAML 文件来定义和运行多容器 Docker 应用程序。对于 LobeChat，虽然目前只有一个主要服务，但使用 Compose 可以方便地管理配置和未来的扩展。

1.  **创建项目目录**:
    在您选择的位置创建一个目录来存放 LobeChat 的配置。我们将使用您提到的路径 `/root/docker/lobe-chat-client` 作为示例。如果您不是 root 用户，请选择您用户主目录下的路径，例如 `~/docker/lobe-chat-client`。

    ```bash
    mkdir -p /root/docker/lobe-chat-client
    cd /root/docker/lobe-chat-client
    ```

2.  **创建 `docker-compose.yml` 文件**:
    在 `lobe-chat-client` 目录下创建一个名为 `docker-compose.yml` 的文件，并将以下内容粘贴进去：

    ```yaml
    version: '3.8' # 指定 Compose 文件版本

    services:
      lobe-chat: # 服务名称
        image: lobehub/lobe-chat:latest # 使用官方 LobeChat 最新镜像
        container_name: lobe-chat # 定义容器的名称
        restart: always # 容器退出时总是自动重启
        ports:
          - '43210:3210' # 将主机的 43210 端口映射到容器的 3210 端口
                         # 您可以根据需要更改主机端口 43210
        environment:
          # --- 核心环境变量 ---
          # OpenAI API Key (必需，替换为您自己的 Key)
          OPENAI_API_KEY: sk-xxxxx
          # OpenAI API 代理地址 (可选，如果需要通过代理访问 OpenAI)
          OPENAI_PROXY_URL: https://aihubmix.com/v1 # 替换为您自己的代理地址或留空
          # LobeChat 访问密码 (可选，设置后需要密码才能访问)
          ACCESS_CODE: xxxxx # 替换为您自己的密码或留空/注释掉

          # --- 可选环境变量 (更多请参考 LobeChat 文档) ---
          # OPENAI_MODEL_LIST: gpt-4-turbo,gpt-3.5-turbo # 自定义可用模型列表
          # 您可以添加更多环境变量来配置其他 AI 提供商或功能...
    ```

    **重要提示**:
    *   **替换 `sk-xxxxx`**: 将 `OPENAI_API_KEY` 的值替换为您真实的 OpenAI API 密钥。
    *   **替换 `xxxxx`**: 如果您想设置访问密码，请将 `ACCESS_CODE` 的值替换为您想要的密码。如果不需要密码，可以删除或注释掉 (`#`) 这一行。
    *   **端口映射**: `43210:3210` 表示您可以通过服务器的 `43210` 端口访问 LobeChat 的 Web 界面（它在容器内运行于 `3210` 端口）。您可以将 `43210` 更改为任何您想要的未被占用的主机端口。
    *   **代理**: `OPENAI_PROXY_URL` 是可选的，仅当您需要通过代理服务器访问 OpenAI API 时才需要设置。

3.  **启动 LobeChat**:
    在包含 `docker-compose.yml` 文件的目录 (`/root/docker/lobe-chat-client`) 中，运行以下命令：

    ```bash
    docker-compose up -d
    ```

    *   `up`: 创建并启动服务。
    *   `-d`: 在后台 (detached) 模式下运行。

    Docker Compose 现在会拉取 `lobehub/lobe-chat:latest` 镜像（如果本地没有）并根据您的配置启动容器。

4.  **验证**:
    等待片刻让容器启动。然后，在您的浏览器中访问 `http://<您的服务器IP>:43210` (或者您设置的其他主机端口)。如果一切顺利，您应该能看到 LobeChat 的界面。如果设置了 `ACCESS_CODE`，您需要先输入密码。

### 步骤 2: 创建自动更新脚本

手动检查更新并重启容器很麻烦。我们可以创建一个简单的 Bash 脚本来自动化这个过程。

1.  **创建脚本文件**:
    在您喜欢的位置创建一个脚本文件，例如 `/usr/local/bin/auto-update-lobe-chat.sh` (或者 `~/bin/auto-update-lobe-chat.sh`)。

    ```bash
    touch /usr/local/bin/auto-update-lobe-chat.sh
    chmod +x /usr/local/bin/auto-update-lobe-chat.sh
    ```
    *   `touch`: 创建空文件。
    *   `chmod +x`: 赋予脚本执行权限。

2.  **编辑脚本**:
    使用您喜欢的文本编辑器 (如 `nano` 或 `vim`) 打开该文件，并粘贴以下脚本内容：

    ```bash
    #!/bin/bash
    # auto-update-lobe-chat.sh

    # --- 配置 ---
    # 设置包含 docker-compose.yml 文件的目录路径
    COMPOSE_DIR="/root/docker/lobe-chat-client"
    # ----------------

    # 脚本出错时立即退出
    set -e

    echo "----------------------------------------"
    echo "Starting LobeChat update check: $(date)"
    echo "----------------------------------------"

    # 可选: 设置代理，如果您的服务器需要代理才能访问 Docker Hub
    # export https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 all_proxy=socks5://127.0.0.1:7890

    echo "Attempting to pull the latest image for lobehub/lobe-chat..."
    # 拉取最新镜像并将输出存储到变量中，同时捕获错误输出
    output=$(docker pull lobehub/lobe-chat:latest 2>&1)
    pull_exit_code=$?

    # 检查拉取命令是否成功执行
    if [ $pull_exit_code -ne 0 ]; then
      echo "Error: Failed to pull docker image. Output:"
      echo "$output"
      echo "Update check failed."
      exit 1
    fi

    # 检查输出是否包含特定字符串 "Image is up to date"
    if echo "$output" | grep -q "Image is up to date for lobehub/lobe-chat:latest"; then
      echo "Lobe-Chat image is already up to date. No action needed."
      echo "Update check finished."
      exit 0
    fi

    # 如果镜像有更新
    echo "Detected Lobe-Chat update."

    # 检查 docker-compose.yml 文件所在的目录是否存在
    if [ ! -d "$COMPOSE_DIR" ]; then
      echo "Error: Docker Compose directory not found: $COMPOSE_DIR"
      echo "Update check failed."
      exit 1
    fi

    echo "Navigating to $COMPOSE_DIR"
    # 切换到 docker-compose.yml 文件所在的目录，如果失败则退出
    cd "$COMPOSE_DIR" || { echo "Error: Failed to change directory to $COMPOSE_DIR"; exit 1; }

    echo "Stopping and removing old containers defined in docker-compose.yml..."
    # 使用 docker-compose down 停止并移除由该 compose 文件定义的旧容器和网络
    # 它会读取 .env 文件（如果存在）和 yml 文件中的配置
    docker-compose down

    echo "Starting updated container(s) using docker-compose..."
    # 使用 docker-compose up -d 重新创建并启动容器
    # 它会自动使用刚刚拉取的最新镜像
    docker-compose up -d

    # 输出更新时间和版本信息
    echo "Update complete."
    echo "Update time: $(date)"
    # 尝试获取新镜像的版本号
    echo "New Version: $(docker inspect lobehub/lobe-chat:latest | grep 'org.opencontainers.image.version' | awk -F'"' '{print $4}' || echo 'N/A - Could not determine version')"


    echo "Cleaning up old, untagged lobe-chat images (optional)..."
    # 清理未被使用的旧版本 lobe-chat 镜像（不包括 latest 和 database 镜像）
    # 使用 xargs -r 避免在没有输入时执行 rmi
    # 使用 > /dev/null 2>&1 抑制正常输出和错误输出
    # 使用 || true 防止因没有旧镜像可删或删除失败（例如镜像还在被缓存使用）而导致脚本退出
    docker images | grep 'lobehub/lobe-chat' | grep -v 'lobehub/lobe-chat-database' | grep -v 'latest' | awk '{print $3}' | xargs -r docker rmi > /dev/null 2>&1 || true
    echo "Cleanup process finished."
    echo "----------------------------------------"


    exit 0
    ```

    **脚本解释**:
    *   `COMPOSE_DIR`: **确保将其值修改为您实际的 `docker-compose.yml` 文件所在的目录路径。**
    *   `set -e`: 任何命令执行失败，脚本将立即退出。
    *   `docker pull`: 拉取最新的 `lobehub/lobe-chat:latest` 镜像。
    *   `grep -q "Image is up to date"`: 检查 `docker pull` 的输出。如果镜像已经是最新，则脚本退出。
    *   `cd "$COMPOSE_DIR"`: **关键步骤**。切换到包含 `docker-compose.yml` 的目录，这是执行 `docker-compose` 命令的前提。
    *   `docker-compose down`: 优雅地停止并移除 `docker-compose.yml` 文件中定义的所有服务（容器、网络等）。
    *   `docker-compose up -d`: 使用更新后的镜像重新创建并启动服务。
    *   `docker inspect`: 获取新镜像的版本信息（如果可用）。
    *   `docker rmi ... || true`: 清理旧的、未标记的 `lobehub/lobe-chat` 镜像，`|| true` 保证即使清理失败或没有可清理项，脚本也不会因此退出。

### 步骤 3: 使用 Cron 设置定时自动更新

现在我们有了更新脚本，可以使用 Cron 来安排它定期运行，例如每天凌晨 3 点。

1.  **编辑 Cron 表**:
    打开当前用户的 Cron 配置文件：

    ```bash
    crontab -e
    ```
    如果是第一次运行，系统可能会提示您选择一个编辑器 (如 `nano`)。

2.  **添加定时任务**:
    在文件的末尾添加以下行：

    ```crontab
    # 每天凌晨 3:00 运行 LobeChat 自动更新脚本
    0 3 * * * /usr/local/bin/auto-update-lobe-chat.sh >> /var/log/lobe-chat-update.log 2>&1
    ```

    **解释**:
    *   `0 3 * * *`: 这是 Cron 的时间表达式，表示“每月的、每周的、每天的第 3 小时的第 0 分钟”，即每天凌晨 3:00。
    *   `/usr/local/bin/auto-update-lobe-chat.sh`: **确保这是您脚本的实际完整路径**。
    *   `>> /var/log/lobe-chat-update.log`: 将脚本的标准输出 (`stdout`) 追加到日志文件 `/var/log/lobe-chat-update.log`。**请确保 `/var/log` 目录（或您选择的其他目录）对运行 Cron 任务的用户是可写的，或者提前创建该日志文件并设置好权限。**
    *   `2>&1`: 将标准错误 (`stderr`) 重定向到标准输出 (`stdout`)，这样错误信息也会被记录到同一个日志文件中。

3.  **保存并退出**:
    保存 Cron 文件并退出编辑器 (在 `nano` 中通常是 `Ctrl+X`, 然后按 `Y`, 再按 `Enter`)。

    Cron 服务会自动加载新的配置。现在，您的 LobeChat 将在每天凌晨 3 点自动检查更新，并在有新版本时自动完成更新过程。您可以检查 `/var/log/lobe-chat-update.log` 文件来查看更新历史和任何潜在的错误。

### 结论

通过结合 Docker Compose 的便捷部署和 Cron 的定时任务，我们实现了一个几乎“一劳永逸”的 LobeChat 自托管方案。它不仅简化了初始设置，还通过自动化更新脚本确保了服务的时效性和安全性，让您可以更专注于使用 LobeChat 本身。记得定期检查更新日志，确保一切按预期运行。
