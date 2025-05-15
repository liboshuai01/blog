---
title: Centos部署Doris集群实战指南
tags:
  - Linux
  - Doris
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250409145732836.png'
toc: true
abbrlink: 1c0d9a3d
date: 2023-10-12 23:11:00
---

本文系统介绍了基于 CentOS 7 的 Apache Doris 集群搭建过程，涵盖从环境准备到集群管理的关键步骤。文中首先列出了操作系统的配置要求，包括关闭防火墙、调整内核参数、配置时钟同步及免密登录等，确保系统能够满足 Doris 的性能需求，同时附上相关配置链接供详细查阅。

集群规划方面，将三台服务器分别设置为 `master`、`node1` 和 `node2`，并明确角色分工，如 FE、BE 和 BROKER。这部分强调了路径和集群 IP 的修改要求，操作优先使用非 `root` 用户。

安装部分是本文的重点，详细展示了模块化的安装步骤，包括 FE、BE 和 BROKER 的配置、启动及节点关联方法，提供了分布式操作工具 `xsync` 的使用及 Mysql 客户端管理 FE 的方式。同时，文中附带了各模块端口调整与内存限制的建议设置。

最后，文章介绍了集群验证、管理功能，包括节点状态检查、Web 管理界面登录及一键式管理脚本的编写，方便集群的启动、停止与监控操作。

本文结合官方文档及实践经验，提供了清晰的操作指南，是搭建高性能数据分析集群的重要参考。

<!-- more -->

环境准备
---

需要有三台 Centos7 服务器，并都需要完成下面的配置要求：

-   关闭防火墙
-   新建普通用户`me`
-   阿里云时钟同步服务器
-   配置免密登陆
-   永久关闭`selinux`
-   永久关闭`swap`分区
-   永久调整句柄数和进程限制
-   永久调整虚拟内存区域限制
-   临时禁用透明大页
-   配置`xsync`、`xcall`同步脚本
-   配置`jdk8`环境

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
> [centos关闭swap分区](https://juejin.cn/spost/7358259851611045926)
>
> [centos调整句柄数和进程限制](https://juejin.cn/post/7357998495483379721)
>
> [centos调整虚拟内存区域限制](https://juejin.cn/post/7358270777983860774)
>
> [centos禁用透明大页](https://juejin.cn/spost/7358083295241240626)
>
> [centos配置xsync和xcall同步脚本](https://juejin.cn/post/7295962144750813221 "https://juejin.cn/post/7295962144750813221")
>
> [centos安装jdk8](https://juejin.cn/post/7173667982051606558 "https://juejin.cn/post/7173667982051606558")

## 集群规划

> 说明：
> 1. 除特别说明外，本文的所有操作均在master节点、使用me这个非root用户执行
> 2. 命令中出现的IP，均需要替换为自己集群中的IP【必须】
> 3. 命令中出现的`/home/lbs/software`路径，可选择替换为自定义路径【可选】

| hostname | IP        | 角色                        |
| -------- | --------- | --------------------------- |
| master   | 10.0.0.87 | FE（LEADER）+ BE + BROKER   |
| node1    | 10.0.0.81 | FE（FOLLOWER）+ BE + BROKER |
| node2    | 10.0.0.82 | FE（FOLLOWER）+ BE + BROKER |


## 正式安装

下载资源包`apache-doris-2.1.6-bin-x64.tar.gz`，并解压到指定安装路径
```shell
wget https://apache-doris-releases.oss-accelerate.aliyuncs.com/apache-doris-2.1.6-bin-x64.tar.gz
tar -zxvf apache-doris-2.1.6-bin-x64.tar.gz -C /home/lbs/software
mv /home/lbs/software/apache-doris-2.1.6-bin-x64 /home/lbs/software/doris
```

### 安装`FE`
1. 修改`fe.conf`配置文件
    ```shell
    vi /home/lbs/software/doris/fe/conf/fe.conf
    
    # 修改各种端口，防止与hadoop集群冲突（可选，端口不冲突不用）
    http_port = 8130
    rpc_port = 9120
    query_port = 9130
    edit_log_port = 9110
    
    # 绑定集群ip网段
    priority_networks = 10.0.0.0/24
    ```

2. 分发`fe`目录到集群的其他的两个机器上
    ```shell
    xsync /home/lbs/software/doris/fe
    ```
### 安装`BE`

1. 修改`be.conf`配置文件
    ```shell
    vi /home/lbs/software/doris/be/conf/be.conf
    
    # 修改各种端口，防止与hadoop集群冲突（可选，端口不冲突不用）
    be_port = 9160
    webserver_port = 8140
    heartbeat_service_port = 9150
    brpc_port = 8160
    
    # 绑定集群ip网段
    priority_networks = 10.0.0.0/24
    
    # 新增配置内容，限制be内存占用
    mem_limit = 30%
    ```

2. 分发`be`目录到集群的其他的两个机器上
    ```shell
    xsync /home/lbs/software/doris/be
    ```

### 安装`BROKER`

1. 修改`apache_hdfs_broker.conf`配置文件
    ```shell
    vi /home/lbs/software/doris/extensions/apache_hdfs_broker/conf/apache_hdfs_broker.conf
    
    # 修改内容如下（默认为8000）：
    broker_ipc_port = 8100 
    ```

2. 分发`extensions`目录到集群的其他的两个机器上
    ```shell
    xsync /home/lbs/software/doris/extensions
    ```

## 创建集群

### 启动并关联`FE`
1. 登录master节点启动 FE 作为 LEADER，并查看`doris`进程
   ```
   # 启动FE
   /home/lbs/software/doris/fe/bin/start_fe.sh --daemon
   
   # 查看 Doris 进程
   ps -ef | grep doris
   ```
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112858.png)

2. 再分别登录到`node1`与`node2`两个机器执行下面命令，用于分别启动 FE 作为两个 FOLLOWER，并查看`doris`进程
   > 注意：
   >
   > `--helper 10.0.0.87:9110` 仅在首次启动时需要添加，后续无需添加
   >
   > `10.0.0.87` 为 `LEADER` 节点的IP，`9110` 为 FE 的 `edit_log_port`。

    ```
    # 启动FE，并连接到LEADER
    /home/lbs/software/doris/fe/bin/start_fe.sh --helper 10.0.0.87:9110 --daemon
    
    # 查看 Doris 进程
    ps -ef | grep doris
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112861.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112846.png)

3. 回到`master`机器，安装`mysql`客户端
    ```shell
    sudo yum install -y mysql
    ```

4. 通过`Mysql-client`连接 FE LEADER 的 Mysql，将 两个 FOLLOWER 添加进去。
    ```
    # 密码为空
    mysql -h 10.0.0.87 -P 9130 -uroot
    
    # 添加three、four为follower节点
    ALTER SYSTEM ADD FOLLOWER "10.0.0.81:9110";
    ALTER SYSTEM ADD FOLLOWER "10.0.0.82:9110";
    
    # 查看状态（需要多等待一会儿，直到Alive列值都为true）
    SHOW PROC '/frontends';
    ```

   > 添加节点：`ALTER SYSTEM ADD FOLLOWER[OBSERVER] "fe_host:edit_log_port";`
   >
   > 删除节点：`ALTER SYSTEM DROP FOLLOWER[OBSERVER] "fe_host:edit_log_port";`

### 启动并关联`BE`

1. 启动集群中的三台机器上的`BE`，并查看doris进程
   > 先退出`mysql`客户端

    ```shell
    # 启动BE
    xcall /home/lbs/software/doris/be/bin/start_be.sh --daemon
    
    # 查看 Doris 进程
    ps -ef | grep doris
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112849.png)
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112841.png)
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112853.png)

2. 添加集群中的三台机器上的`BE`到`FE`的`MySQL`数据库中。
    ```
    # 登录到FE的Mysql数据库
    mysql -h 10.0.0.87 -P 9130 -uroot
    
    # 添加集群中三个节点的BE
    ALTER SYSTEM ADD BACKEND "10.0.0.87:9150";
    ALTER SYSTEM ADD BACKEND "10.0.0.81:9150";
    ALTER SYSTEM ADD BACKEND "10.0.0.82:9150";
      
    # 查看状态（需要多等待一会儿，直到Alive列值都为true）
    SHOW PROC '/backends';
    ```

   > 优雅删除节点（将此BE数据迁移到其他BE节点）：`ALTER SYSTEM DECOMMISSION BACKEND "be_host:be_heartbeat_service_port";`
   >
   > 强制删除节点（会直接删除BE及上面的数据）：`ALTER SYSTEM DROPP BACKEND "be_host:be_heartbeat_service_port";`


### 启动并关联`BROKER`
1. 启动集群中的三台机器上的`BROKER`，并查看`Doris`进程
   > 先退出`mysql`客户端

    ```
    # 启动broker
    xcall /home/lbs/software/doris/extensions/apache_hdfs_broker/bin/start_broker.sh --daemon
    
    # 查看 Doris 进程
    ps -ef | grep doris
    ```
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112132.png)
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112138.png)
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112154.png)

2. 同时添加集群中的三台机器上的`BROKER`到`FE`的`MySQL`数据库中。
    ```
    # 登录到FE的Mysql数据库
    mysql -h 10.0.0.87 -P 9130 -uroot
    
    # 添加集群中三个节点的broker
    ALTER SYSTEM ADD BROKER broker_name "10.0.0.87:8100","10.0.0.81:8100","10.0.0.82:8100";
    
    # 查看状态（需要多等待一会儿，直到Alive列值都为true）
    SHOW PROC "/brokers";
    ```

   > 添加 BROKER 节点：`ALTER SYSTEM ADD BROKER broker_name "broker_host1:broker_ipc_port1","broker_host2:broker_ipc_port2",...;`
   >
   > 删除 BROKER 节点：`ALTER SYSTEM DROP BROKER broker_name "broker_host1:broker_ipc_port1","broker_host2:broker_ipc_port2",...;`
   >
   > 删除所有 BROKER 节点：`ALTER SYSTEM DROP ALL BROKER broker_name;`

## 验证集群

1. 登录`FE`的数据库，查看`FE`、`BE`、`BROKER`各节点的状态
    ```shell
    # 登录到FE的数据库
    mysql -h 10.0.0.87 -P 9130 -uroot
    
    # 查看集群中所有的FE节点状态
    SHOW PROC '/frontends';
    
    # 查看集群中所有的BE节点状态
    SHOW PROC '/backends';
    
    # 查看集群中所有的BROKER节点状态
    SHOW PROC "/brokers";
    ```

2. 浏览器输入下面的地址、账号，进入`Doris`的`Web`页面。

   | 地址                                                    | 账号 | 密码 |
      | ------------------------------------------------------- | ---- | ---- |
   | http://10.0.0.87:8130 (http://fe_hostname:fe_http_port) | root | 空   |


    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260115243.png)

## 集群管理

### 启动集群

```shell
# 启动集群中所有的FE节点
xcall /home/lbs/software/doris/fe/bin/start_fe.sh --daemon

# 启动集群中所有的BE节点
xcall /home/lbs/software/doris/be/bin/start_be.sh --daemon

# 启动集群中所有的BROKER节点
xcall /home/lbs/software/doris/extensions/apache_hdfs_broker/bin/start_broker.sh --daemon
```

### 停止集群

```shell
# 停止集群中所有的BROKER节点
xcall /home/lbs/software/doris/extensions/apache_hdfs_broker/bin/stop_broker.sh --daemon

# 停止集群中所有的BE节点
xcall /home/lbs/software/doris/be/bin/stop_be.sh --daemon

# 停止集群中所有的FE节点
xcall /home/lbs/software/doris/fe/bin/stop_fe.sh --daemon
```

### 集群状态

```shell
# 登录到FE的数据库
mysql -h 10.0.0.87 -P 9130 -uroot

# 查看集群中所有的FE节点状态
SHOW PROC '/frontends';

# 查看集群中所有的BE节点状态
SHOW PROC '/backends';

# 查看集群中所有的BROKER节点状态
SHOW PROC "/brokers";
```

### 管理脚本

```shell
cat > /home/lbs/software/doris/doris-cluster.sh <<EOF
#!/bin/bash
case $1 in
"start"){
        for i in 10.0.0.87 10.0.0.81 10.0.0.82
        do
                 echo -------------------------------- $i doris 启动 ---------------------------
                ssh $i "source /etc/profile; /home/lbs/software/doris/fe/bin/start_fe.sh --daemon; /home/lbs/software/doris/be/bin/start_be.sh --daemon; /home/lbs/software/doris/extensions/apache_hdfs_broker/bin/start_broker.sh --daemon"
        done
}
;;
"stop"){
        for i in 10.0.0.87 10.0.0.81 10.0.0.82
        do
                echo -------------------------------- $i doris 停止 ---------------------------
                ssh $i "source /etc/profile; /home/lbs/software/doris/extensions/apache_hdfs_broker/bin/stop_broker.sh --daemon; /home/lbs/software/doris/be/bin/stop_be.sh --daemon; /home/lbs/software/doris/fe/bin/stop_fe.sh --daemon"
        done
}
;;
"status"){
        for i in 10.0.0.87 10.0.0.81 10.0.0.82
        do
                echo -------------------------------- $i doris 状态 ---------------------------
                status=$(ssh $i "source /etc/profile; ps -ef | grep doris")
                if [[ -z "$status" ]]
                then
                    echo "Doris not running!"
                else
                    echo "$status"
                fi
        done
}
;;
esac
EOF
```

> 参考文献：
>
> [Doris官网文档-标准部署](https://doris.apache.org/zh-CN/docs/install/standard-deployment)
>
> [Doris官网文档-弹性扩缩容](https://doris.apache.org/zh-CN/docs/admin-manual/cluster-management/elastic-expansion)