---
title: Docker重启策略restart参数值的对比与选择
tags:
  - Linux
  - Docker
  - 容器化
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505180341119.png'
toc: true
abbrlink: cecb4f00
date: 2025-05-15 22:57:57
---

在现代后端应用部署中，容器化技术已成为标配。Docker 作为其中的佼佼者，提供了强大的容器管理功能。其中，容器的重启策略（Restart Policy）是确保应用高可用性和自我修复能力的关键一环。当容器因错误、资源问题或其他原因意外退出时，一个配置得当的重启策略能够自动尝试恢复服务，减少人工干预，提升系统韧性。

本文将深入探讨 Docker 支持的几种主要重启策略：`no` (默认)、`always`、`unless-stopped` 和 `on-failure`，并通过实例演示它们的行为差异，帮助您在不同场景下做出最佳选择。

<!-- more -->

## 理解重启策略的核心价值

配置重启策略的核心价值在于实现容器的**自我修复**。这意味着：
*   **提升应用可用性**：在容器异常退出后自动重启，减少服务中断时间。
*   **简化运维**：自动化处理常见的瞬时故障导致的容器停止。
*   **增强系统韧性**：即使在 Docker守护进程（daemon）重启后，也能根据策略恢复关键容器的运行状态。

## Docker 支持的重启策略概览

重启策略可以应用于单个容器，在 `docker container run` 命令中通过 `--restart` 参数指定，或在 Docker Compose 及 Docker Stacks 的 `docker-compose.yml` 文件中声明。

截至目前，Docker 主要支持以下几种重启策略：

1.  **`no`** (默认策略)
2.  **`always`**
3.  **`unless-stopped`**
4.  **`on-failure[:max-retries]`**

接下来，我们将逐一解析。

### 1. `no` (默认不重启)

这是 Docker 容器的默认重启策略。如果未显式指定任何重启策略，则应用此策略。
*   **行为**：容器退出后，无论退出状态码是什么，Docker 都不会自动重启该容器。
*   **适用场景**：
    *   一次性任务或批处理作业，完成后即停止。
    *   开发和调试阶段，需要手动控制容器生命周期。
    *   由外部编排工具（如 Kubernetes）管理的容器，其重启逻辑由编排工具负责。

```bash
# 示例：运行一个容器，不指定重启策略 (默认为 no)
$ docker container run -d --name no-restart-container alpine sleep 60

# 容器将在 60 秒后正常退出，或者可以手动停止
# $ docker container stop no-restart-container
# 之后，该容器不会自动重启
```

### 2. `always` (始终重启)

`always` 策略会确保容器始终尝试保持运行状态。
*   **行为**：
    *   当容器停止时（无论是正常退出还是因错误退出），Docker 会立即尝试重启它。
    *   **一个关键特性**：即使容器是通过 `docker container stop` 命令被明确停止的，在 Docker 守护进程重启后，该容器仍会被自动启动。
*   **适用场景**：
    *   关键应用服务，如 Web 服务器、数据库等，需要尽可能保证持续运行。
    *   需要注意，如果容器启动后因配置错误等原因立即失败，`always` 策略可能导致无限重启循环，消耗系统资源。

**示例：`always` 策略的即时重启效果**

```bash
$ docker container run --name neversaydie -it --restart always alpine sh

# 进入容器Shell后，输入 exit 命令
/# exit

# 容器会立即退出，但 Docker 会马上重启它
# 查看容器状态，注意 CREATED 和 STATUS 时间戳的差异
$ docker container ls
CONTAINER ID    IMAGE     COMMAND     CREATED           STATUS                 NAMES
0901afb84439    alpine    "sh"        35 seconds ago    Up 1 second            neversaydie
```
可以看到，容器于 35 秒前创建，但状态显示 1 秒前启动，这是因为它在 `exit` 后被自动重启了。

### 3. `unless-stopped` (除非明确停止，否则重启)

`unless-stopped` 策略与 `always` 类似，但对于被用户明确停止的容器，在 Docker 守护进程重启后会有不同表现。
*   **行为**：
    *   当容器停止时（无论是正常退出还是因错误退出），Docker 会立即尝试重启它。
    *   **与 `always` 的核心区别**：如果容器是通过 `docker container stop` 命令被明确停止的，那么在 Docker 守护进程重启后，该容器**不会**被自动启动，它会保持停止状态。
*   **适用场景**：
    *   希望容器在大多数情况下自动重启，但在维护或特定场景下手动停止后，不希望它在宿主机或 Docker 服务重启后意外启动。这通常是生产环境中一个更安全、更可控的选择。

### 4. `on-failure[:max-retries]` (仅在失败时重启)

`on-failure` 策略只在容器以非零退出状态码（表示发生错误）退出时才尝试重启。
*   **行为**：
    *   如果容器正常退出（退出状态码为 0），则不会重启。
    *   如果容器异常退出（退出状态码非 0），Docker 会尝试重启它。
    *   可以任选指定最大重启次数 `:max-retries`。例如，`on-failure:5` 表示最多尝试重启 5 次。如不指定次数，则会一直尝试重启。
    *   如果容器因错误退出并被此策略安排重启，即使它在 Docker 守护进程重启前处于停止状态（等待重启），在守护进程重启后，它也会被尝试重启。
*   **适用场景**：
    *   执行可能会间歇性失败的任务的 Worker 进程。
    *   对正常完成即停止的脚本或应用，不希望在它们成功执行后还被重启。

**示例：`on-failure` 策略**

```bash
# 示例1: 容器将因错误退出并自动重启 (不限次数)
$ docker container run -d --name fail-infinite --restart on-failure alpine sh -c "echo 'Attempting execution...' && exit 1"

# 示例2: 容器将因错误退出，最多重启3次
$ docker container run -d --name fail-limited --restart on-failure:3 alpine sh -c "echo 'Attempting execution...' && exit 1"

# 示例3: 容器将正常退出 (exit 0)，不会重启
$ docker container run -d --name success-no-restart --restart on-failure alpine sh -c "echo 'Successful execution.' && exit 0"
```

## Daemon 重启时 `always` 与 `unless-stopped` 的关键区别演示

这是理解 `always` 和 `unless-stopped` 策略差异的核心。

**(1) 创建两个新容器，分别使用 `always` 和 `unless-stopped` 策略**

```bash
$ docker container run -d --name always-policy-demo \
  --restart always \
  alpine sleep 1d

$ docker container run -d --name unless-stopped-policy-demo \
  --restart unless-stopped \
  alpine sleep 1d

$ docker container ls
CONTAINER ID   IMAGE    COMMAND      STATUS        NAMES
3142bd91ecc4   alpine   "sleep 1d"   Up 2 secs     unless-stopped-policy-demo
4f1b431ac729   alpine   "sleep 1d"   Up 17 secs    always-policy-demo
```
现在有两个运行中的容器：“always-policy-demo” 和 “unless-stopped-policy-demo”。

**(2) 手动停止这两个容器**

```bash
$ docker container stop always-policy-demo unless-stopped-policy-demo

$ docker container ls -a  # -a 参数显示所有容器，包括已停止的
CONTAINER ID   IMAGE     COMMAND      STATUS                        NAMES
3142bd91ecc4   alpine    "sleep 1d"   Exited (137) 3 seconds ago    unless-stopped-policy-demo
4f1b431ac729   alpine    "sleep 1d"   Exited (137) 3 seconds ago    always-policy-demo
```
两个容器现在都处于 `Exited` 状态。

**(3) 重启 Docker 守护进程**

重启 Docker 的命令因操作系统而异：
*   **Linux (systemd)**: `sudo systemctl restart docker`
*   **Windows Server**: `Restart-Service docker`
*   **macOS (Docker Desktop)**: 通常通过 Docker Desktop 应用界面重启。

**(4) Docker 重启完成后，检查容器状态**

```bash
$ docker container ls -a
CONTAINER ID   IMAGE     COMMAND      CREATED           STATUS                        NAMES
3142bd91ecc4   alpine    "sleep 1d"   2 minutes ago     Exited (137) 2 minutes ago    unless-stopped-policy-demo
4f1b431ac729   alpine    "sleep 1d"   2 minutes ago     Up 9 seconds                  always-policy-demo
```
**结果分析**：
*   “always-policy-demo” 容器（指定了 `--restart always`）在 Docker 守护进程重启后**已自动重启**。
*   “unless-stopped-policy-demo” 容器（指定了 `--restart unless-stopped`）在 Docker 守护进程重启后**仍保持停止状态**。

这个演示清晰地展示了 `always` 和 `unless-stopped` 在 Docker 守护进程重启场景下，对于被手动停止过的容器的不同处理方式。

## 在 Docker Compose 中配置重启策略

如果您使用 Docker Compose 或 Docker Stacks 管理多容器应用，可以在 `docker-compose.yml` 文件的服务定义中配置重启策略。

```yaml
version: "3.8" # 或更高版本

services:
  myservice:
    image: myapp-image
    # ... 其他配置 ...
    deploy: # 对于 Swarm 模式下的 Stacks
      restart_policy:
        condition: on-failure # 可选: none, on-failure, always, unless-stopped (swarm中any等同于always)
        delay: 5s
        max_attempts: 3
        window: 120s
    # 或者直接用于 Compose (非 Swarm 模式)
    # restart: always
    # restart: unless-stopped
    # restart: on-failure
    # restart: "on-failure:5" # 带重试次数

  another_service:
    image: another-app
    restart: unless-stopped # 普遍推荐
```
注意：在 Compose 文件中，`restart` 顶级键（如 `restart: always`）适用于 `docker-compose up`（非 Swarm 模式）。而 `deploy.restart_policy` 结构则用于 Docker Swarm 模式下的服务部署 (`docker stack deploy`)，它提供了更细致的控制，如 `delay` (重启延迟), `max_attempts` (最大尝试次数), 和 `window` (评估重启是否成功的窗口时间)。对于简单场景，顶级 `restart` 键更常用。

## 选择合适的重启策略：总结与建议

| 策略                | 容器正常退出 (exit 0) | 容器异常退出 (exit non-0) | 手动 `stop` 后，Docker Daemon 重启 | Docker Daemon 重启 (容器之前在运行) | 适用场景建议                                                                 |
| :------------------ | :-------------------- | :------------------------ | :--------------------------------- | :-------------------------------- | :--------------------------------------------------------------------------- |
| `no` (默认)         | 不重启                 | 不重启                     | 保持停止                            | 不重启                             | 一次性任务、开发调试、外部编排系统管理                                           |
| `always`            | 重启                   | 重启                       | **重启**                            | 重启                               | 关键服务，需绝对保证运行；小心无限重启循环                                       |
| `unless-stopped`    | 重启                   | 重启                       | **保持停止**                        | 重启                               | 推荐用于大多数生产环境服务，兼顾高可用和手动维护的可控性                         |
| `on-failure[:N]`    | 不重启                 | 重启 (可限次数 N)          | 保持停止 (除非上次是失败退出)      | 重启 (如果上次是失败退出)          | 可能间歇性失败的 Worker 进程、批处理任务，不希望成功后还重启                    |

**核心建议**：
*   对于大多数需要持续运行的后端服务（如 API 服务器、消息队列消费者），**`unless-stopped` 通常是最佳选择**。它提供了良好的自动恢复能力，同时尊重运维人员的手动停止操作。
*   对于那些无论如何都必须运行的、且内部有良好错误处理和启动逻辑的“顽固”服务，可以使用 `always`，但要监控其重启行为。
*   对于会自然完成并退出的任务，或可能因暂时性问题失败但重试后能成功的任务，`on-failure` (可配合最大重试次数) 非常合适。
*   避免在没有充分理由的情况下使用默认的 `no` 策略于生产环境的常驻服务。

通过合理配置 Docker 容器的重启策略，您可以显著提升应用的健壮性和运维效率，让系统具备一定程度的自我修复能力，从而更专注于业务逻辑的开发和创新。

---