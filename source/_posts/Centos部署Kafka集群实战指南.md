---
title: Centos部署Kafka集群实战指南
tags:
  - 环境搭建
  - Linux
  - Kafka
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408113627328.png'
toc: true
abbrlink: '1267253'
date: 2023-10-10 10:34:00
---

本篇技术博文详细介绍了如何在 Linux 环境下完成 Kafka 集群的搭建，适用于需要部署高可用分布式消息队列的场景。文章内容采用循序渐进的方式，从环境准备到安装配置，再到操作脚本编写和集群管理，以及最终的验证集群运行状态，涵盖了 Kafka 集群搭建的每一个关键环节。

首先，文章对环境准备步骤进行了说明，包含服务器配置、用户管理、时钟同步、系统防护设置等多个细节，确保每台机器处于稳定的状态并具备必要的运行条件。所有基础配置均配有参考链接，方便读者扩展学习。

接着，文章规划了 Kafka 集群的节点分布，依据实际需求，提供具体 IP 配置及相应的 `server.properties` 文件参数调整方法。通过分发安装目录和逐节点配置，达成集群协作的目标，并强调了 IP 地址的自定义替换。

最后，博文分享了针对集群管理的操作脚本及相关命令，并通过 Kafka 进程和 Broker 状态验证了集群的成功搭建。无论您是 Kafka 的初学者还是有经验的开发人员，这篇文章都能为您提供完整、实用的指导与参考。

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
-   安装配置`zookeeper`集群，并启动`zookeeper`集群
-   确保端口`9092,9999`端口没有被占用

> 环境准备可以参考我下面的博文：
>
> [centos防火墙常用命令](https://juejin.cn/post/7178874541744062522)
>
> [centos新建普通用户](https://juejin.cn/post/7357917741908787215)
>
> [centos系统时间同步](https://juejin.cn/post/7357917741908656143)
>
> [centos配置免密登录](https://juejin.cn/post/7277395904217939968)
>
> [centos关闭SElinux](https://juejin.cn/post/7322518787424305162)
>
> [centos配置xsync和xcall同步脚本](https://juejin.cn/post/7295962144750813221)
>
> [centos安装jdk8](https://juejin.cn/post/7173667982051606558)
>
> [centos安装zookeeper集群](https://juejin.cn/post/7259188760486477879)

## 集群规划


| hostname | IP | 
| --- | --- |
| master | 10.0.0.87 |
| node1 | 10.0.0.81 |
| node2 | 10.0.0.82 |


## 正式安装

> 安装说明：
> 1. 除特别说明外，所有操作均在master节点、使用me这个非root用户执行
> 2. 命令中出现的IP，均需要替换为自己集群中的IP【必须】
> 3. 命令中出现的`/home/lbs/software`路径，可选择替换为自定义路径【可选】

1. 下载安装包，并解压安装包到安装kafka的指定路径（这里设定为`/home/lbs/software/kafka`）

   ```
   wget https://archive.apache.org/dist/kafka/2.5.0/kafka_2.12-2.5.0.tgz
   tar -zxvf kafka_2.12-2.5.0.tgz -C /home/lbs/software
   mv /home/lbs/software/kafka_2.12-2.5.0 /home/lbs/software/kafka
   ```

2. 创建存放 kafka 消息的目录

   ```shell
   mkdir -p /home/lbs/software/kafka/kafka-logs
   ```

3. 修改`server.properties`配置文件

   ```
   vim /home/lbs/software/kafka/config/server.properties
   
   # 修改如下参数
   broker.id=0 
   ......
   listeners=PLAINTEXT://:9092
   ......
   advertised.listeners=PLAINTEXT://10.0.0.87:9092
   ......
   log.dirs=/home/lbs/software/kafka/kafka-logs
   ......
   zookeeper.connect=10.0.0.87:2181,10.0.0.81:2181,10.0.0.82:2181
   
   # 新增如下参数
   # 关闭自动创建topic
   auto.create.topics.enable=false
   # 开启topic删除
   delete.topic.enable=true
   ```
   
   > 参数说明：
   >
   > 1.`broker.id` ： 集群内全局唯一标识；
   >
   > 2.每个节点上需要设置不同的值 `listeners`：这个`IP`地址也是与本机相关的，每个节点上设置为自己的`IP`地址;
   >
   > 3.`log.dirs`：存放`kafka`消息的
   >
   > 4.`zookeeper.connect`: 配置的是 zookeeper 集群地址
   >
   > 如果你内外网卡地址不一样，listeners就需要特殊的配置了，查看[Kafka学习理解-listeners配置](https://blog.51cto.com/u_14286115/4745727)

4. 分发`kafka`安装目录

   ```shell
   # 分发kafka安装目录给其他集群节点
   xsync /home/lbs/software/kafka
   ```
   
5. 分别登录到其他两台机器，修改其`server.properties`的如下内容

    - `broker.id`: 各机器必须保持唯一，例如`master`为`1`，那么`node1`、`node2`为`2`、`3`。
    - `advertised.listeners`: 各机器修改为自身的IP地址

6. 编写 `kafka` 集群操作脚本

   > 替换IP地址，然后直接执行下面命令即可
   
   ```shell
    tee /home/lbs/software/kafka/bin/kafka-cluster.sh <<'EOF'
    #!/bin/bash
    case $1 in
    "start"){
            for i in master node1 node2
            do
                     echo -------------------------------- $i kafka 启动 ---------------------------
                    ssh $i "source ~/.bash_profile;/home/lbs/software/kafka/bin/kafka-server-start.sh -daemon /home/lbs/software/kafka/config/server.properties"
            done
    }
    ;;
    "stop"){
            for i in master node1 node2
            do
                    echo -------------------------------- $i kafka 停止 ---------------------------
                    ssh $i "/home/lbs/software/kafka/bin/kafka-server-stop.sh"
            done
    }
    ;;
    "status"){
            for i in master node1 node2
            do
                    echo -------------------------------- $i kafka 状态 ---------------------------
                    status=$(ssh $i "source ~/.bash_profile;jps | grep -i kafka")
                    if [[ -z "$status" ]]
                    then
                        echo "Kafka not running!"
                    else
                        echo "$status"
                    fi
            done
    }
    ;;
    esac
    EOF
    chmod +x /home/lbs/software/kafka/bin/kafka-cluster.sh
   ```

## 操作集群

```shell
# 启动集群
/home/lbs/software/kafka/bin/kafka-cluster.sh start

# 停止集群
/home/lbs/software/kafka/bin/kafka-cluster.sh stop

# 查看集群状态
/home/lbs/software/kafka/bin/kafka-cluster.sh status
```

> 更多`kafka`操作命令，请参考我的博文[# Kafka集群管理：常用命令速览](https://juejin.cn/post/7348352600462016512)

## 验证集群

1. 查看`kafka`进程情况
    ```shell
    [me@master bin]$ xcall jps
    ================current host is master=================
    --> execute command "jps"
    28340 QuorumPeerMain
    32121 Jps
    31770 Kafka
    Command executed successfully on master
    ================current host is node1=================
    --> execute command "jps"
    6435 QuorumPeerMain
    7331 Jps
    7177 Kafka
    Command executed successfully on node1
    ================current host is node2=================
    --> execute command "jps"
    6516 QuorumPeerMain
    7480 Jps
    7289 Kafka
    Command executed successfully on node2
    All commands executed successfully!
    ```

2. 查看`broker`在线情况
    ```shell
    [admin@master kafka]$ ./bin/zookeeper-shell.sh localhost:2181 ls /brokers/ids
    Connecting to localhost:2181

    WATCHER::

    WatchedEvent state:SyncConnected type:None path:null
    [0, 1, 2]
    ```
