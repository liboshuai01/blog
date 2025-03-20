---
title: Zookeeper集群搭建
date: 2023-10-10 08:08:00
tags: [环境搭建,Linux,Zookeeper]
categories: [环境搭建]
---

本文简要介绍了如何在三台 CentOS 7 服务器上快速部署 Zookeeper 集群。主要流程包括：

1. 环境准备：关闭防火墙与 SELinux、创建普通用户、配置时间同步和免密登录、安装 JDK8，以及配置 xsync/xcall 同步脚本，确保必要端口空闲。

2. 安装配置：下载并解压 Zookeeper，修改 zoo.cfg 文件（包括 dataDir 和各节点配置），创建 myid 文件，并将 Zookeeper 目录分发到所有节点。

3. 集群管理：编写 zkCluster.sh 脚本实现集群的启动、停止和状态监控，最后通过 jps 和 zkCli.sh 验证系统运行状态。

该教程适合希望快速部署 Zookeeper 集群的运维和开发人员。

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
