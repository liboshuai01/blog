---
title: 使用 Shell 脚本管理 Java 应用程序
tags:
  - Java
  - Shell
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261330259.png'
toc: true
abbrlink: adfad19
date: 2024-11-14 13:28:35
---

在现代软件开发中，使用脚本来管理应用程序的启动、停止和监控变得越来越重要。本文将详细解析一个用于管理 Java 应用程序的 Bash 脚本，帮助读者理解其功能和实现细节。

<!-- more -->

## 脚本概述

这个 Bash 脚本的主要功能是启动、停止、重启和检查一个 Java 应用程序的状态。脚本中使用了 Java 进程的相关命令和参数，确保应用能够在指定的环境中运行。

## 脚本内容

```shell
#!/bin/bash

# 定义变量
MEMORY_SIZE="4096m" # 占用内存大小
PROCESS_NAME="yudao-server.jar" # 进程名称
ROOT_PATH="/home/lbs/project/ruoyi-pro" # 项目根路径
GC_LOG_PATH="${ROOT_PATH}/backend/logs/service-gc.log" # jvm gc日志路径
HEAP_DUMP_PATH="${ROOT_PATH}/backend/logs" # oom dump文件路径
JAR_PATH="${ROOT_PATH}/backend/jar/yudao-server.jar" # jar包路径

start() {
    # 检查是否有相同的进程在运行
    if pgrep -f "${PROCESS_NAME}" > /dev/null; then
        echo "应用已经在运行。"
        exit 1
    fi

    nohup java \
        -Xms"${MEMORY_SIZE}" -Xmx"${MEMORY_SIZE}" \
        -Xloggc:"${GC_LOG_PATH}" \
        -verbose:gc \
        -XX:+PrintGCDetails \
        -XX:+PrintGCDateStamps \
        -XX:+PrintGCTimeStamps \
        -XX:+UseGCLogFileRotation \
        -XX:NumberOfGCLogFiles=10 \
        -XX:GCLogFileSize=100M \
        -XX:+HeapDumpOnOutOfMemoryError \
        -XX:HeapDumpPath=${HEAP_DUMP_PATH} \
        -server \
        -XX:+UseG1GC \
        -XX:MaxGCPauseMillis=20 \
        -XX:InitiatingHeapOccupancyPercent=35 \
        -XX:+ExplicitGCInvokesConcurrent \
        -XX:MaxInlineLevel=15 \
        -Djava.awt.headless=true \
        -jar "${JAR_PATH}" >/dev/null 2>&1 &

    echo "应用已启动。"
}

stop() {
    PIDS=$(pgrep -f "${PROCESS_NAME}")

    if [ -n "${PIDS}" ]; then
        for pid in ${PIDS}; do
            echo "正在停止进程，PID: ${pid}"
            kill ${pid}
        done

        # 等待进程停止
        for pid in ${PIDS}; do
            while kill -0 ${pid} 2>/dev/null; do
                echo "正在停止应用，PID: ${pid}"
                sleep 1
            done
        done

        echo "应用已停止。"
    else
        echo "未找到正在运行的应用。"
    fi
}

restart() {
    stop
    start
}

status() {
    PIDS=$(pgrep -f "${PROCESS_NAME}")

    if [ -n "${PIDS}" ]; then
        for pid in ${PIDS}; do
            START_TIME=$(ps -o lstart= -p ${pid})
            RUNNING_TIME=$(ps -o etime= -p ${pid})
            USER=$(ps -o user= -p ${pid})

            echo "应用正在运行："
            echo "PID: ${pid}"
            echo "启动时间: ${START_TIME}"
            echo "启动用户: ${USER}"
            echo "运行时长: ${RUNNING_TIME}"
        done
    else
        echo "应用未在运行。"
    fi
}

help() {
    echo "用法: $0 {start|stop|restart|status|--help}"
    echo ""
    echo "命令:"
    echo "  start      启动应用"
    echo "  stop       停止应用"
    echo "  restart    重启应用"
    echo "  status     查看应用状态"
    echo "  --help     显示帮助信息"
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        restart
        ;;
    status)
        status
        ;;
    --help)
        help
        ;;
    *)
        echo "用法: $0 {start|stop|restart|status|--help}"
        exit 1
        ;;
esac
```

## 脚本解析

### 变量定义

在脚本的开头，定义了一些关键变量：

```shell
PROCESS_NAME="yudao-server.jar" # 进程名称
ROOT_PATH="/home/lbs/project/ruoyi-pro" # 项目根路径
GC_LOG_PATH="${ROOT_PATH}/backend/logs/service-gc.log" # jvm gc日志路径
HEAP_DUMP_PATH="${ROOT_PATH}/backend/logs" # oom dump文件路径
JAR_PATH="${ROOT_PATH}/backend/jar/yudao-server.jar" # jar包路径
```

这些变量提供了应用程序所需的基本信息，例如项目环境、进程名称和日志路径等。

### 启动应用程序

`start` 函数负责启动 Java 应用程序。首先，它检查是否已有相同的进程在运行：

```bash
if pgrep -f "${PROCESS_NAME}" > /dev/null; then
    echo "应用已经在运行。"
    exit 1
fi
```

如果应用未在运行，使用 `nohup` 命令启动 Java 应用，并传递了一些 JVM 参数和 Spring 配置：

```bash
nohup java xxxxxx -jar  ...
```

这些参数包括内存设置、垃圾回收策略、GC 日志配置等，以优化应用的性能和稳定性。

### 停止应用程序

`stop` 函数用于停止正在运行的应用程序。它通过 `pgrep` 获取进程 ID（PID），并使用 `kill` 命令停止进程：

```bash
PIDS=$(pgrep -f "${PROCESS_NAME}")
```

如果找到进程，脚本会逐个停止它们，并在停止过程中输出相关信息：

```bash
for pid in ${PIDS}; do
    echo "正在停止进程，PID: ${pid}"
    kill ${pid}
done
```

### 重启和状态检查

`restart` 函数简单地调用 `stop` 和 `start` 函数实现重启功能。而 `status` 函数则用于检查应用程序的当前状态：

```bash
if [ -n "${PIDS}" ]; then
    for pid in ${PIDS}; do
        START_TIME=$(ps -o lstart= -p ${pid})
        RUNNING_TIME=$(ps -o etime= -p ${pid})
        USER=$(ps -o user= -p ${pid})
        ...
    done
else
    echo "应用未在运行。"
fi
```

该函数输出每个运行中的进程的 PID、启动时间、运行时长和用户信息。

### 帮助信息

最后，脚本提供了帮助信息，指导用户如何使用这些命令：

```bash
help() {
    echo "用法: $0 {start|stop|restart|status|--help}"
    ...
}
```

## 友情提醒

请注意，由于脚本是通过 JAR 包名称来查找进程的，这可能会导致误杀其他同名进程。在使用时，请确保处理好相关问题，以避免不必要的影响。

## 总结

这个 Bash 脚本为 Java 应用程序提供了一个简单而有效的管理工具。通过使用脚本，开发者可以方便地控制应用程序的生命周期，同时也能实时监控应用的状态。这种自动化的管理方式提高了开发和运维的效率，是现代应用管理中不可或缺的一部分。

希望这篇文章能帮助你更好地理解和使用 Bash 脚本来管理 Java 应用程序！