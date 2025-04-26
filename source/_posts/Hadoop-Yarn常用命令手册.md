---
title: Hadoop Yarn常用命令手册
tags:
  - Hadoop
  - Yarn
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261115968.png'
toc: true
abbrlink: feade61
date: 2024-08-29 11:15:09
---

在现代大数据平台中，Hadoop YARN（Yet Another Resource Negotiator）作为资源管理和任务调度的核心组件，极大提升了集群资源的利用率和作业执行效率。为了更好地管理和监控集群应用，掌握常用的 YARN 命令是必不可少的技能。本文将详尽介绍各类实用的 YARN 命令，涵盖应用程序管理、日志查看、容器和节点管理，以及配置更新，助你高效运维 Hadoop 集群。

<!-- more -->

## 管理 YARN 应用程序

### 查看当前所有应用程序

通过以下命令可以列出 Hadoop 集群中所有正在运行及已完成的 YARN 应用程序，便于对作业整体状况进行监控：

```bash
yarn application -list
```

### 按应用状态筛选应用程序

YARN 支持根据应用程序的状态进行筛选。例如，查看所有已经完成的应用：

```bash
yarn application -list -appStates FINISHED
```

此外，常见状态还有 RUNNING、FAILED、KILLED 等，根据需求灵活使用。

### 终止指定应用程序

当某个应用表现异常或需要提前停止时，可以通过如下命令杀死指定作业：

```bash
yarn application -kill application_1654496324557_0001
```

该命令确保作业安全中断，释放资源供其他任务使用。

## 精细日志查看技巧

日志是了解作业运行状况和排查问题的重要依据。

### 查询整个应用程序的日志

一条命令即可获取指定应用程序在所有容器上的日志聚合：

```bash
yarn logs -applicationId application_1654496324557_0001
```

该日志包括 Driver 和 Executor 的输出，涵盖 stdout、stderr 等信息。

### 指定容器查看日志

若需精确定位某个容器的运行日志，执行：

```bash
yarn logs -applicationId application_1654496324557_0001 -containerId container_1654496324557_0001_01_000001
```

这样可以针对具体的容器环境和任务表现进行深入分析。

## 应用程序尝试的管理与监控

### 列举应用程序所有尝试

YARN 允许为同一应用启动多个尝试来保证作业成功。查看某个应用所有尝试信息：

```bash
yarn applicationattempt -list application_1654496324557_0001
```

### 查询特定应用尝试的状态详情

掌握某一次尝试的运行详情，方便判断失败原因或资源分配情况：

```bash
yarn applicationattempt -status appattempt_1654496324557_0001_000001
```

## 容器监控与管理

### 列出某次应用尝试下的容器

一个应用尝试中包含若干容器实例。使用以下命令查看：

```bash
yarn container -list appattempt_1654496324557_0004_000001
```

### 获取容器实时状态

关注容器的运行状态，有助于及时发现资源异常：

```bash
yarn container -status container_1654496324557_0006_01_000001
```

> ⚠️ 仅在任务执行期间，此命令才能返回容器状态信息。

## 集群节点状态查看

掌握集群节点的健康状态和资源分配，可指导调度和扩容：

```bash
yarn node -list -all
```

这条命令详细列出了所有节点的运行状态、可用内存、CPU 核数等关键指标。

## 队列管理与配置刷新

### 刷新 YARN 队列配置

当动态调整队列资源配额或参数时，使用以下命令让 ResourceManager 重新加载配置文件：

```bash
yarn rmadmin -refreshQueues
```

### 查看队列详细状态

了解队列任务调度情况、资源使用率和作业排队情况：

```bash
yarn queue -status default
```

可根据业务需求监控不同队列性能。

## 进阶建议

- **自动化运维**：结合 Shell 脚本和定时任务定期收集日志和状态信息，实现自动化报警。
- **日志聚合与分析**：结合 ELK 或 Grafana 构建日志及指标可视化平台，直观发现异常。
- **安全加固**：合理配置访问权限，使用 Kerberos 等认证机制保障集群安全。
- **性能调优**：结合应用特点合理设置内存和 CPU，提升资源分配效率。

## 结语

掌握和灵活运用以上 YARN 命令，能够帮助大数据开发和运维人员实现对 Hadoop 集群的深度监控和高效管理。无论是运行中的监测、日志分析还是资源调度调整，这些命令都是日常不可或缺的利器。希望本文能助你快速提升 YARN 管理能力，保障大数据业务稳定顺畅运行。