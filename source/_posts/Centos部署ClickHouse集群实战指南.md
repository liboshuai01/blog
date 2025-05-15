---
title: Centos部署ClickHouse集群实战指南
tags:
  - Linux
  - ClickHouse
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504200417825.png'
toc: true
abbrlink: 71df73fa
date: 2023-09-28 04:16:54
---

在大数据分析与实时查询需求日益增长的背景下，ClickHouse 作为一款高性能的列式数据库，凭借其出色的并发能力和极致的查询效率，已成为数据处理和分析领域的重要技术选择。本文将详细介绍如何在 CentOS7 操作系统上，使用离线 tgz 包部署方式，搭建一个高可用、可扩展的 ClickHouse 集群环境。通过整合 Zookeeper 实现副本同步控制，构建包含两个分片双副本的 ClickHouse 集群，为后续的大规模数据分析打下坚实基础。指南适用于网络限制严格或生产环境对稳定性要求较高的用户。

<!-- more -->

## 集群环境

> 注意本教程是基于`tgz`包离线方式搭建

*   系统环境`Centos 7`
*   集群的各个机器配置好免密登录
*   搭建好`Zookeeper`集群，集群机器为：`two`、`three`、`four`

| 机器  | 描述         |
| ----- | ------------ |
| one   | 节点1，副本1 |
| two   | 节点1，副本2 |
| three | 节点2，副本1 |
| four  | 节点2，副本2 |

## 清楚旧版本残余

> 仅限于`tgz`包安装的方式

    rm -rf /var/lib/clickhouse
    rm -rf /etc/clickhouse-*
    rm -rf /var/log/clickhouse-server
    rm -rf /usr/bin/clickhouse*
    systemctl stop clickhouse-server.service
    systemctl disable clickhouse-server.service
    rm -rf /etc/systemd/system/clickhouse-server.service
    systemctl daemon-reload

## 搭建集群

1.  下载下面四个安装包（一个都不要少，且版本要一致），地址[clickhouse阿里镜像站](https://mirrors.aliyun.com/clickhouse/tgz/lts/?spm=a2c6h.25603864.0.0.1d466dc58b2gQW).

    ```shell
    clickhouse-common-static-21.3.20.1.tgz
    clickhouse-common-static-dbg-21.3.20.1.tgz
    clickhouse-server-21.3.20.1.tgz
    clickhouse-client-21.3.20.1.tgz
    ```

2.  上传包到服务器`one`，并解压四个压缩包

3.  执行安装语句（一定要按顺序执行！）

    ```shell
    # cd到四个安装包解压后的路径
    ./clickhouse-common-static-21.3.20.1/install/doinst.sh
    ./clickhouse-common-static-dbg-21.3.20.1/install/doinst.sh
    # 需要输入确定clickhouse登录的默认密码
    ./clickhouse-server-21.3.20.1.tgz/install/doinst.sh
    ./clickhouse-client-21.3.20.1.tgz/install/doinst.sh
    ```

4.  将clickhouse相关文件目前权限都给予root用户

    ```shell
    chown -R root:root /var/lib/clickhouse /var/log/clickhouse-server /etc/clickhouse-server /etc/clickhouse-client
    ```

5.  更改`/etc/clickhouse-server/config.xml`配置文件

    ```xml
    <!--取消此行注释-->
    <listen_host>::</listen_host>

    <!--添加此行，指定集群配置文件路径（后续需创建）-->
    <include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
    ```

6.  新增`/etc/clickhouse-server/config.d/metrika.xml`集群配置文件

    ```xml
    <yandex>
        <!--ck集群节点-->
        <remote_servers>
            <test_ck_cluster>
                <!--分片1-->
                <shard>
                    <weight>1</weight>
                    <internal_replication>true</internal_replication>
                    <replica>
                        <host>one</host>
                        <port>9000</port>
                        <user>root</user>
                        <password>wOv3TDOv</password>
                        <compression>true</compression>
                    </replica>
                    <replica>
                        <host>two</host>
                        <port>9000</port>
                        <user>root</user>
                        <password>wOv3TDOv</password>
                        <compression>true</compression>
                    </replica>
                </shard>
                <!--分片2-->
                <shard>
                    <weight>1</weight>
                    <internal_replication>true</internal_replication>
                    <replica>
                        <host>three</host>
                        <port>9000</port>
                        <user>root</user>
                        <password>wOv3TDOv</password>
                        <compression>true</compression>
                    </replica>
                    <replica>
                        <host>four</host>
                        <port>9000</port>
                        <user>root</user>
                        <password>wOv3TDOv</password>
                        <compression>true</compression>
                    </replica>
                </shard>
            </test_ck_cluster>
        </remote_servers>
        <!--zookeeper相关配置-->
        <zookeeper>
            <node index="1">
                <host>two</host>
                <port>2181</port>
            </node>
            <node index="2">
                <host>three</host>
                <port>2181</port>
            </node>
            <node index="3">
                <host>four</host>
                <port>2181</port>
            </node>
        </zookeeper>
        <macros>
            <!--当前节点主机名，每台机器改成自己的ip-->
            <replica>one</replica>
        </macros>
        <networks>
            <ip>::/0</ip>
        </networks>
        <!--压缩相关配置-->
        <clickhouse_compression>
            <case>
                <min_part_size>10000000000</min_part_size>
                <min_part_size_ratio>0.01</min_part_size_ratio>
                <!--压缩算法lz4压缩比zstd快, 更占磁盘-->
                <method>lz4</method>
            </case>
        </clickhouse_compression>
    </yandex>
    ```

    > macros-replica块 每个节点都需要改成自己集群的ip

7.  命令行生成`sha256`密码

    ```shell
    PASSWORD=$(base64 < /dev/urandom | head -c8); echo "$PASSWORD"; 
    echo -n "$PASSWORD" | sha256sum | tr -d '-'
    ```
    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504200414351.png)

8.  更改`/etc/clickhouse-server/users.xml`配置文件

    ```xml
    <!--在<users></users>块中，添加新用户root，并指定密码-->
    <root>
        <!--明文：IrlBz+AO-->
        <password_sha256_hex>730b74192d3b846acbb367343bca95796c58c455c251d41e89ce1fcb021ef410</password_sha256_hex>
        <networks>
            <ip>::/0</ip>
        </networks>
        <profile>default</profile>
        <quota>default</quota>
    </root>
    ```

9.  创建`/app/clickhouse/script`目录，并在其目录下编写脚本`clickhouse.sh`和`clickhouse-cluster.sh`。

    > `clickhouse.sh`内容如下：

    ```shell
    #!/bin/bash
    case $1 in
    "start"){
            echo -------------------------------- $i clickhouse 启动 ---------------------------
            nohup clickhouse-server --config-file=/etc/clickhouse-server/config.xml start >/dev/null 2>&1 &
    }
    ;;
    "stop"){
            echo -------------------------------- $i clickhouse 停止 ---------------------------
            ps -ef | grep clickhouse-server| awk '{print $2}' | xargs kill -9
    }
    ;;
    "status"){
            echo -------------------------------- $i clickhouse 状态 ---------------------------
            ps -ef | grep clickhouse-server | grep -v grep
    }
    ;;
    esac
    ```

    > `clickhouse-cluster.sh`内容如下：

    ```shell
    #!/bin/bash
    case $1 in
    "start"){
            # 这里可以直接使用ip地址
            for i in one two three four
            do
                     echo -------------------------------- $i clickhouse 启动 ---------------------------
                    ssh $i "/app/clickhouse/script/clickhouse.sh start"
            done
    }
    ;;
    "stop"){
            for i in one two three four
            do
                    echo -------------------------------- $i clickhouse 停止 ---------------------------
                    ssh $i "/app/clickhouse/script/clickhouse.sh stop"
            done
    }
    ;;
    "status"){
            for i in one two three four
            do
                    echo -------------------------------- $i clickhouse 状态 ---------------------------
                    ssh $i "/app/clickhouse/script/clickhouse.sh status"
            done
    }
    ;;
    esac
    ```

10. 将上面的`2、3、4、5、6、7、8、9`步骤在集群机器`two、three、four`上重复执行（唯有第6步每台机器需要自定义修改，其余步骤集群的每台集群都相同）。

11. 启动`zookeeper`集群

12. 启动`clickhouse`集群（任意一台机器上执行）

    ```shell
    /app/clickhouse/script/clickhouse-cluster.sh start
    ```

13. 查看`clickhouse`集群状态（任意一台机器上执行）

    ```shell
    /app/clickhouse/script/clickhouse-cluster.sh status
    ```

14. 登录到`clickhouse`集群（在任意一台机器即可）

    ```shell
    # 回车后需要输入密码
    clickhouse-client -u root --password
    ```

15. 停止`clickhouse`集群（任意一台机器上执行）

    ```shell
    /app/clickhouse/script/clickhouse-cluster.sh stop
    ```

## 验证集群环境

1.  登录到`clickhouse`集群（在任意一台机器即可）

    ```shell
    # 回车后需要输入密码
    clickhouse-client -u root --password
    ```

2.  查看集群信息

    ```sql
    select * from system.clusters;
    ```
    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504200414383.png)

3.  验证zookeeper是否与当前数据库clickhouse进行了正确的配置

    ```sql
    SELECT * FROM system.zookeeper WHERE path = '/clickhouse';
    ```
    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504200414387.png)

## 结语

至此，ClickHouse 集群环境已经成功搭建并完成验证，具备了多节点分布、复制容错和高可用配置能力。通过结合 Zookeeper 负责集群副本协调，可以有效提升系统在高并发查询和数据写入场景下的表现。无论是用于构建企业级数仓支撑，还是实时 BI 报表后端，ClickHouse 都提供了强大而可靠的支撑能力。后续可根据实际场景进一步调整压缩算法、资源调度策略及安全性配置，持续优化性能，释放 ClickHouse 的全部潜能。