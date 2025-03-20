---
title: Redis集群一键搭建
date: 2023-10-10 11:54:00
tags: [环境搭建,Linux,Redis]
categories: [环境搭建]
---

本文介绍了在 CentOS 7 系统下，利用普通用户权限从零开始搭建一个由三主三从构成的 Redis 集群的全过程。文章首先说明了在高并发和大数据量场景下，单个 Redis 实例性能受限，需要通过集群实现扩展性与高可用性。接着，详细介绍了环境配置、各节点端口规划及安装前的依赖要求，随后通过提供的 install.sh 脚本实现多实例自动化部署与配置，并说明如何通过 redis-cli 将这些实例组建成果主果从的 Redis 集群。最后，还提供了用于集群管理（启动、停止、状态检查）的脚本，方便日常维护。整体方案适用于对性能和扩展性有高要求的应用场景，同时操作简单迅速，既适合初学者，也能满足实际生产环境的需求。

<!-- more -->

## 前言

在面对高并发和大数据量的场景时，单个Redis实例可能无法满足性能需求，此时就需要使用Redis集群来提升系统的扩展性和可用性。本文将详细介绍如何在Centos 7系统环境下，使用普通用户权限从零开始搭建一个由三主三从构成的Redis集群。通过本文的步骤，你将能够搭建一个可靠的Redis集群，从而解决单实例Redis在处理大规模数据时遇到的瓶颈问题。

## 环境要求
- **Redis版本**：7.0.12、7.4.1
- **系统环境**：Centos 7
- **用户权限**：普通用户权限即可
- **`gcc`编译器**：必须安装
  > 若运行`gcc -v`时提示`-bash: gcc: command not found`，则表示没有安装`gcc`。你可以参考我的另一篇博文[CentOS离线一键安装gcc（附安装包+脚本）](https://juejin.cn/post/7310143510102376457)，来进行`gcc`的安装。

## 集群规划
本文档默认的集群搭建规模如下表所示：

| 服务器 | 示例IP   | 端口      |
| ------ | -------- | --------- |
| node1  | 10.0.0.1 | 6379, 6380 |
| node2  | 10.0.0.2 | 6379, 6380 |
| node3  | 10.0.0.3 | 6379, 6380 |

我们将在三台服务器上部署一个由三个主节点和三个从节点组成的Redis集群。当然，你可以根据实际需求调整主从节点的数量及部署的服务器数量。

资源下载
---
- [redis脚本安装包-蓝奏云](https://liboshuai.lanzouv.com/iZyoJ2ccrydg)
- [redis脚本安装包-百度云](https://pan.baidu.com/s/1yey6vGQQMe-t66QfGGglow?pwd=usx9)

使用教程
---
1. 上传下载到的`redis_install.tar`压缩包到指定的三台服务器`node1`、`node2`、`node3`中。
   > 如果你希望将Redis集群部署到六台不同的服务器上，则需要将tar包上传到所有六台服务器上。

2. 在三台服务器`node1`、`node2`、`node3`上，将压缩包放置在同一目录下，然后分别执行以下命令解压并安装：
    ```shell
    tar -xvf redis_install.tar && cd redis_install && ./install.sh 7.4.1 /home/lbs/software/redis 1gb admin123456 6379 6380
    ```

   `install.sh`脚本的使用说明如下：
    ```
    [root@localhost redis_install]# ./install.sh --help
    用法: ./install.sh <版本> <安装路径> <内存限制> <密码> <端口1> <端口2> ...

    参数:
      指定版本       Redis指定版本，目前仅支持7.0.12和7.4.1。
      安装路径       Redis将要被安装的目录。
      内存限制       Redis单节点最大使用内存。
      密码           Redis实例使用的密码。
      端口1          第一个Redis实例的端口号。
      端口2          第二个Redis实例的端口号。
      ...            更多Redis实例的额外端口号。

    选项:
      --help         显示此帮助信息并退出。

    示例:
      ./install.sh 7.4.1 /home/lbs/software/redis 1gb admin123456 6379 6380
    ```

3. 登录任意一台服务器节点，执行下面的命令，将各个节点加入到Redis集群中
   > 注意：在执行此步骤前，请确保已经关闭防火墙或者开放了相应的Redis端口。
    ```shell
    # 进入到Redis安装目录，并执行加入Redis节点的命令
    cd /home/lbs/software/redis && yes yes | ./bin/redis-cli --cluster create 10.0.0.1:6379 10.0.0.1:6380 10.0.0.2:6379 10.0.0.2:6380 10.0.0.3:6379 10.0.0.3:6380 --cluster-replicas 1 -a admin123456
    ```
   > 创建集群的命令中，`admin123456`是设置的认证密码。`--cluster-replicas 1`表示每个主节点有一个对应的从节点。执行此命令后，脚本会自动为这六个Redis实例分配主从角色和槽位。
   > 如果你希望手动指定节点的主从关系，可以参考[Redis集群指定主从关系及动态增删节点](https://blog.csdn.net/guotianqing/article/details/119778684)。

4. 登录任意一台服务器节点，执行以下命令以验证集群状态：
    ```shell
    # 进入到Redis安装目录，并验证Redis集群状态及信息
    cd /home/lbs/software/redis && ./bin/redis-cli -h localhost -c -p 6379 -a admin123456 cluster nodes
    ```

   ![image.png](images/Centos 离线傻瓜式一键式部署 Redis 集群（附脚本+安装包）_1.jpg)

5. 最终的目录结构

   ![image.png](images/Centos 离线傻瓜式一键式部署 Redis 集群（附脚本+安装包）_2.jpg)
   > 如果你觉得脚本生成的redis集群结构和配置内容不符合你的预期，你可以参考[# redis 主从模式、哨兵模式、集群模式 离线搭建](https://juejin.cn/post/7257734526138744889)的集群模式部分，进行手动搭建

## 集群管理

> 注意使用下面脚本，需先完成集群各节点机器的免密登陆配置

### 脚本

> redis集群的启动、停止都是基于此脚本的，请务必首先执行下面创建脚本的命令。

```shell
cat > /home/lbs/software/redis/bin/redis-cluster.sh <<EOF
#!/bin/bash
case $1 in
"start"){
        for i in 10.0.0.87 10.0.0.81 10.0.0.82
        do
                 echo -------------------------------- $i redis 启动 ---------------------------
                ssh $i "source /etc/profile; /home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6379/conf/redis-6379.conf; /home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6380/conf/redis-6380.conf"
        done
}
;;
"stop"){
        for i in 10.0.0.87 10.0.0.81 10.0.0.82
        do
                echo -------------------------------- $i redis 停止 ---------------------------
                ssh $i "/home/lbs/software/redis/bin/redis-cli -p 6379 -a Rongshu@2024 shutdown; /home/lbs/software/redis/bin/redis-cli -p 6380 -a Rongshu@2024 shutdown"
        done
}
;;
"status"){
        for i in 10.0.0.87 10.0.0.81 10.0.0.82
        do
                echo -------------------------------- $i redis 状态 ---------------------------
                status=$(ssh $i "source /etc/profile; ps -ef | grep redis | grep -i cluster")
                if [[ -z "$status" ]]
                then
                    echo "Redis not running!"
                else
                    echo "$status"
                fi
        done
}
;;
esac
EOF

chmod +x /home/lbs/software/redis/bin/redis-cluster.sh
```

### 启动

```shell
/home/lbs/software/redis/bin/redis-cluster.sh start
```

### 停止

```shell
/home/lbs/software/redis/bin/redis-cluster.sh stop
```

### 状态

```shell
/home/lbs/software/redis/bin/redis-cluster.sh status
```
