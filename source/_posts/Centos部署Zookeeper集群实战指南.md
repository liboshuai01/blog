---
title: Centos部署Zookeeper集群实战指南
tags:
  - 环境搭建
  - Linux
  - Zookeeper
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408113556002.png'
toc: true
abbrlink: '89127441'
date: 2023-10-10 08:08:00
---

在分布式系统领域，Zookeeper 是一种流行的分布式协调服务，具备高可用性和可靠性，广泛应用于分布式系统的配置管理、名称服务以及分布式锁控制等场景。然而，为了更好地支持分布式环境，Zookeeper 集群的搭建至关重要。通过集群模式，系统可以实现更高的容错性，同时提供更稳定的服务。

本文详细讲解了在三台 CentOS7 服务器上部署 Zookeeper 集群的步骤。我们从环境准备开始，包括关闭防火墙、配置免密登录，以及安装必要的工具和依赖项。接下来，根据集群规划，逐步完成 Zookeeper 的安装、配置，以及节点之间的数据分发。此外，还分享了如何编写脚本来简化 Zookeeper 集群的启动、停止与状态监控操作，使运维更加高效。

最后，通过多种验证步骤确认集群的成功运行，包括查看节点进程和连接到集群执行命令。无论是新手还是有一定经验的开发者，都可以根据本文的分步指南轻松搭建一个完整的 Zookeeper 集群，为分布式应用的开发和部署提供坚实基础。

<!-- more -->

# 环境准备

需要有三台 Centos7 服务器，并都需要完成下面的配置要求：

-   关闭防火墙
-   新建普通用户`me`
-   阿里云时钟同步服务器
-   配置免密登陆
-   关闭`selinux`
-   配置`xsync`、`xcall`同步脚本
-   配置`jdk8`环境
-   确保端口`2188,2888`端口没有被占用

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

1. 下载安装包，并解压到指定路径
    ```shell
    wget https://archive.apache.org/dist/zookeeper/zookeeper-3.6.3/apache-zookeeper-3.6.3-bin.tar.gz
    tar -zxvf apache-zookeeper-3.6.3-bin.tar.gz -C /home/lbs/software
    mv /home/lbs/software/apache-zookeeper-3.6.3-bin /home/lbs/software/zookeeper
    ```

2. 修改`zoo.cfg`配置文件
    ```shell
    cp /home/lbs/software/zookeeper/conf/zoo_sample.cfg /home/lbs/software/zookeeper/conf/zoo.cfg
    
    vim /home/lbs/software/zookeeper/conf/zoo.cfg
    
    # 修改如下内容
    dataDir=/home/lbs/software/zookeeper/data
    
    # 追加如下内容
    server.1=10.0.0.87:2188:2888
    server.2=10.0.0.81:2188:2888
    server.3=10.0.0.82:2188:2888
    ```

3. 创建数据存储目录和`myid`文件
    ```shell
    mkdir -p /home/lbs/software/zookeeper/data
    echo "1" > /home/lbs/software/zookeeper/data/myid
    ```

4. 将`zookeeper`目录分发到其他机器节点
    ```
    xsync /home/lbs/software/zookeeper
    ```

5. 登录到`node1`节点，执行下面的命令
    ```shell
    echo "2" > /home/lbs/software/zookeeper/data/myid
    ```

6. 登录到`node2`节点，执行下面的命令
    ```shell
    echo "3" > /home/lbs/software/zookeeper/data/myid
    ```
7. 回到`master`节点，编写创建操作`zookeeper`集群的脚本`zkCluster.sh`

    ```bash
    tee /home/lbs/software/zookeeper/bin/zkCluster.sh <<'EOF'
    #!/bin/bash
    case $1 in
    "start"){
         for i in master node1 node2
         do
                  echo -------------------------------- $i zookeeper 启动 ---------------------------
                 ssh $i "source /etc/profile;/home/lbs/software/zookeeper/bin/zkServer.sh start"
         done
    }
    ;;
    "stop"){
         for i in master node1 node2
         do
                 echo -------------------------------- $i zookeeper 停止 ---------------------------
                 ssh $i "source /etc/profile;/home/lbs/software/zookeeper/bin/zkServer.sh stop"
         done
    }
    ;;
    "status"){
         for i in master node1 node2
         do
                 echo -------------------------------- $i zookeeper 状态 ---------------------------
                 ssh $i "source /etc/profile;/home/lbs/software/zookeeper/bin/zkServer.sh status"
         done
    }
    ;;
    esac
    EOF
    chmod +x /home/lbs/software/zookeeper/bin/zkCluster.sh
    ```

## 操作集群

```
# 启动集群
/home/lbs/software/zookeeper/bin/zkCluster.sh start

# 停止集群
/home/lbs/software/zookeeper/bin/zkCluster.sh stop

# 查看集群状态
/home/lbs/software/zookeeper/bin/zkCluster.sh status
```

> 如果启动报错`JAVA_HOME is not set and java could not be found in PATH`，[查看解决方法](https://blog.csdn.net/HACKERRONGGE/article/details/102485260)
>
> 如需设置`zookeeper`集群密码，请参考[# zookeeper集群设置密码](https://zhuanlan.zhihu.com/p/560809198)

## 验证集群

1. 查看`zookeeper`集群进程情况
    ```shell
    [me@master bin]$ xcall jps
    ================current host is master=================
    --> execute command "jps"
    29777 Jps
    28340 QuorumPeerMain
    Command executed successfully on master
    ================current host is node1=================
    --> execute command "jps"
    6435 QuorumPeerMain
    6710 Jps
    Command executed successfully on node1
    ================current host is node2=================
    --> execute command "jps"
    6786 Jps
    6516 QuorumPeerMain
    Command executed successfully on node2
    All commands executed successfully!
    ```
2. 连接到`zookeeper`集群中
    ```shell
    /home/lbs/software/zookeeper/bin/zkCli.sh -server 127.0.0.1:2181
    
    # 输入ls，查看
    [zk: 127.0.0.1:2181(CONNECTED) 0] ls
    ls [-s] [-w] [-R] path
    ```
