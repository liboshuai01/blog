---
title: 浅谈Dockerfile中RUN、CMD与ENTRYPOINT
tags:
  - Centos
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105158019.png'
toc: true
abbrlink: 4393d8b3
date: 2024-04-11 10:50:18
---

在容器化开发中，理解 Dockerfile 的指令至关重要，尤其是与命令执行相关的 `RUN`、`CMD` 和 `ENTRYPOINT`。它们虽关联紧密，但作用时机与目的存在本质差异。本文将详细剖析这三个指令的功能、区别及最佳实践，助力你编写高效且灵活的 Dockerfile。

<!-- more -->

## RUN：构建镜像时执行命令

- 执行阶段：`RUN` 指令在镜像构建期间运行，执行结果会被写入生成的镜像中。
- 主要用途：安装软件包、配置环境、修改文件系统。例如安装依赖、创建文件夹等。
- 语法示例：

  ```Dockerfile
  RUN apt-get update && apt-get install -y curl
  RUN mkdir /app
  ```

- 镜像层影响：每个 `RUN` 指令都会生成一个新的镜像层，因此合理合并命令可以减少镜像层数，优化镜像体积。

## CMD：指定容器启动时的默认命令与参数

- 执行阶段：`CMD` 指令是在容器运行时执行，用以设置默认启动命令或参数。
- 可被覆盖：通过 `docker run` 提供命令参数时会覆盖 `CMD` 指令。
- 用途场景：
    - 如果 Dockerfile 无 `ENTRYPOINT`，则 `CMD` 表示容器启动的完整命令。
    - 如果有 `ENTRYPOINT`，`CMD` 充当默认参数。
- 语法形式：

    1. exec 形式（推荐）：
       ```Dockerfile
       CMD ["python", "app.py"]
       ```
    2. shell 形式（通过 `/bin/sh -c` 执行）：
       ```Dockerfile
       CMD python app.py
       ```
- 注意：Dockerfile 中如果存在多个 `CMD`，仅最后一个生效。

## ENTRYPOINT：定义容器启动的固定入口命令

- 执行阶段：`ENTRYPOINT` 同样在容器启动时执行，且对容器命令有更高的控制权。
- 特点：
    - 将容器塑造成特定的可执行程序。
    - 即使在 `docker run` 命令中传入参数，这些参数会作为参数传递给 `ENTRYPOINT`，而非替换它。
    - 可被 `docker run --entrypoint` 命令行选项临时覆盖。
- 两种形式：

    1. exec 形式（推荐）：
       ```Dockerfile
       ENTRYPOINT ["python", "app.py"]
       ```
    2. shell 形式：
       ```Dockerfile
       ENTRYPOINT python app.py
       ```
- 推荐用途：
    - 当需要固定执行程序时，如数据库初始化脚本、应用启动命令。
    - 配合 `CMD` 作为默认参数，用于支持参数灵活传递。

## 深入理解三者协同关系

以示例说明它们的交互机制：

```Dockerfile
FROM ubuntu:20.04

RUN apt-get update && apt-get install -y python3

ENTRYPOINT ["python3"]

CMD ["--version"]
```

- 镜像构建阶段：通过 `RUN` 安装了 python3。
- 容器启动时：
    - 默认执行命令为 `python3 --version`。
    - 如果使用 `docker run <image>`，显示 python 版本信息。
    - 如果使用 `docker run <image> -c "print('Hello, Docker')"`，容器执行 `python3 -c "print('Hello, Docker')"`。
- 若通过 `docker run --entrypoint /bin/bash <image>`，则忽略默认 `ENTRYPOINT`，进入 bash 环境。

## 总结与最佳实践建议

- 使用 `RUN` 指令准备镜像环境，安装应用依赖和配置文件。
- 利用 `ENTRYPOINT` 固定容器运行的主程序，提升容器用途的确定性。
- 将灵活的参数放置于 `CMD`，方便用户通过命令行覆盖默认行为。
- 推荐选择 exec 形式来避免 shell 解析带来的问题。
- 合理合并 `RUN` 指令，减少镜像层和体积。
- 避免 Dockerfile 中存在多个 `CMD`，以消除歧义。
- 当需要让容器像应用一样执行单一任务时，`ENTRYPOINT + CMD` 组合是最佳设计。

理解并灵活应用 `RUN`、`CMD` 和 `ENTRYPOINT`，能够极大提升 Docker 镜像的可维护性和容器的运行灵活性，为你的微服务应用构建坚实的基础。