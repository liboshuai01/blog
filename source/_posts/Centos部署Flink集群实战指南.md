---
title: Centos部署Flink集群实战指南
tags:
  - Linux
  - Flink
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250407192319146.png'
toc: true
abbrlink: f0a658ce
date: 2023-10-12 16:40:00
---

本文系统化介绍了如何在 CentOS 7 环境下搭建 Apache Flink 集群，从环境准备到集群验证，详细覆盖了每个操作步骤与配置细节。首先，列出了安装前的环境准备要求，包括关闭防火墙、配置免密登录、安装 JDK8 和同步脚本，以及确保 Hadoop 集群已配备高可用特性，为 Flink 的运行奠定基础。

接着，规划了三台服务器的角色分工：`master`节点负责 JobManager 和 TaskManager，`node1`和`node2`节点作为 TaskManager。明确了资源包上传与解压路径，并进行了必要的配置文件修改，如 `flink-conf.yaml` 文件和集群节点的 `workers` 与 `masters` 文件。配置中包含内存分配、任务槽数及并行度等核心参数，确保分布式计算的高效性。

本文还详细说明了如何通过 `xsync` 工具分发 Flink 安装目录到其他节点，并启动集群。集群验证部分通过 `jps` 命令检查每个节点的相关进程及状态，并提供 Web UI 访问方式，确保集群稳定运行。

最后，介绍了集群的关闭方法，为全流程的操作提供了完整参考。这篇博文是 Flink 初学者和分布式系统运维人员的重要指南，内容详尽而易懂。

<!-- more -->

## 环境准备

需要有三台 Centos7 服务器，并都需要完成下面的配置要求：

-   关闭防火墙
-   新建普通用户`me`
-   阿里云时钟同步服务器
-   配置免密登陆
-   关闭`selinux`
-   配置`xsync`、`xcall`同步脚本
-   配置`jdk8`环境
-   确保端口`8081,8082`端口没有被占用
-   `hadoop`高可用集群安装配置完毕

> 环境准备可以参考我下面的博文：
>
> [centos防火墙常用命令](https://juejin.cn/post/7178874541744062522 "https://juejin.cn/post/7178874541744062522")
>
> [centos新建普通用户](https://juejin.cn/post/7357917741908787215 "https://juejin.cn/post/7357917741908787215")
>
> [centos系统时间同步](https://juejin.cn/post/7357917741908656143 "https://juejin.cn/post/7357917741908656143")
>
> [centos配置免密登录](https://juejin.cn/post/7277395904217939968 "https://juejin.cn/post/7277395904217939968")
>
> [centos关闭SElinux](https://juejin.cn/post/7322518787424305162 "https://juejin.cn/post/7322518787424305162")
>
> [centos配置xsync和xcall同步脚本](https://juejin.cn/post/7295962144750813221 "https://juejin.cn/post/7295962144750813221")
>
> [centos安装jdk8](https://juejin.cn/post/7173667982051606558 "https://juejin.cn/post/7173667982051606558")
>
> [centos安装hadoop集群](https://juejin.cn/spost/7357888522333077556)


## 集群规划

> 说明：
> 1. 除特别说明外，本文的所有操作均在master节点、使用me这个非root用户执行
> 2. 命令中出现的IP，均需要替换为自己集群中的IP【必须】
> 3. 命令中出现的`/home/lbs/software`路径，可选择替换为自定义路径【可选】

| hostname | IP | 角色 |
| --- | --- | --- |
| master | master | JobManager + TaskManager |
| node1 | node1 | TaskManager |
| node2 | node2 | TaskManager |

## 正式安装

1. 下载资源包[flink-1.14.2-bin-scala_2.12.tgz](https://archive.apache.org/dist/flink/flink-1.14.2/flink-1.14.2-bin-scala_2.12.tgz)
    ```
    https://pan.baidu.com/s/17s6WdDoxAf0gMVcYiIuIjw?pwd=49kv
    ```
2. 上传资源包到`master`节点，并解压到指定路径
    ```shell
    tar -zxvf flink-1.14.2-bin-scala_2.12.tgz -C /home/lbs/software
    mv /home/lbs/software/flink-1.14.2 /home/lbs/software/flink
    ```

3. 配置`HADOOP_CLASSPATH`环境变量
    ```shell
    sudo vim /etc/profile
    # 内容最后，追加下面内容
    export HADOOP_CLASSPATH=`hadoop classpath`
    ```

4. 修改`flink-conf.yaml`文件
    ```shell
    vim /home/lbs/software/flink/conf/flink-conf.yaml
    
    # jobmanager地址
    jobmanager.rpc.address: master
    ......
    # jobmanager 内存（可选）
    jobmanager.memory.process.size: 2048m
    # taskmanager 内存（可选，要小于物理内存）
    taskmanager.memory.process.size: 16384m
    # 槽、并行度
    taskmanager.numberOfTaskSlots: 3
    parallelism.default: 1
    ......
    # 类加载机制
    classloader.resolve-order: parent-first
    classloader.check-leaked-classloader: false
    ......
    # 开启火焰图（新增）
    rest.flamegraph.enabled: true
    ```

5. 配置`workers`文件
    ```
    echo 'master
    node1
    node2' > /home/lbs/software/flink/conf/workers
    ```

6. 配置`masters`文件
    ```
    echo 'master:8081' > /home/lbs/software/flink/conf/masters
    ```

7. 分发`/home/lbs/software/flink`目录到集群中的其他两台机器
    ```shell
    xsync /home/lbs/software/flink
    ```

## 启动集群

```shell
/home/lbs/software/flink/bin/start-cluster.sh
```

## 验证集群

1. jps查询三台服务器的进程情况如下：
    ```shell
    =============== master ===============
    4453 StandaloneSessionClusterEntrypoint
    4458 TaskManagerRunner
    4533 Jps
    
    =============== node1 ===============
    2872 TaskManagerRunner
    2941 Jps
    
    =============== node2 ===============
    2948 Jps
    2876 TaskManagerRunner
    ```
2. 访问`flink`的`Web UI`页面`http://master:8081`.

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202503181105676.jpg)

## 关闭集群

```shell
/home/lbs/software/flink/bin/stop-cluster.sh
```
