---
title: Centos部署Redis主从、哨兵、集群实战指南
tags:
  - 环境搭建
  - Linux
  - Redis
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262355142.png'
toc: true
abbrlink: 61ac89f6
date: 2023-07-20 23:50:40
---

Redis 作为一款高性能的键值数据库，在现代 Web 应用中扮演着至关重要的角色，常用于缓存、消息队列、会话管理等场景。为了满足不同的业务需求，特别是对高可用和可扩展性的要求，Redis 提供了多种部署模式：主从复制（Master-Slave）、哨兵（Sentinel）和集群（Cluster）。本文旨在详细介绍如何在离线环境中，逐步搭建这三种模式的 Redis 服务。

<!-- more -->

## 环境准备与要求

在开始之前，请确保满足以下基本条件：

*   **操作系统:** 本文以 CentOS 为例，其他 Linux 发行版类似。
*   **GCC 编译器:** Redis 源码安装需要 GCC 编译器。
    *   在线安装: `yum install -y gcc`
    *   离线安装: 参考 [CentOS离线安装gcc](https://www.cnblogs.com/niceyoo/p/14532228.html) 或准备相应的离线 `rpm` 包。
*   **Redis 源码包:** 下载所需的 Redis 版本（本文以 `7.0.12` 为例）。
    *   下载地址: `https://github.com/redis/redis/archive/7.0.12.tar.gz` 或从 [Redis 官网](http://download.redis.io/releases/) 下载稳定版。
    *   将下载好的 `redis-7.0.12.tar.gz` 文件上传到所有需要部署 Redis 的服务器上。
*   **服务器资源:** 准备相应数量的服务器或虚拟机，确保网络互通。
*   **基础目录规划:** 建议规划统一的安装和数据目录，例如 `/app/redis`。

## Redis 通用安装步骤 (所有服务器执行)

以下步骤在每台需要运行 Redis 实例的服务器上执行一次，用于编译和安装 Redis。

**创建基础目录**

```bash
# 规划统一的父目录，如果不存在则创建
mkdir -p /app/redis/bin  # 用于存放编译后的可执行文件
mkdir -p /app/redis/packet
mkdir -p /app/redis/conf
mkdir -p /app/redis/logs
mkdir -p /app/redis/pid
mkdir -p /app/redis/data # 通用数据父目录，各实例再分子目录
```

**上传并解压源码包**

将下载的 `redis-7.0.12.tar.gz` 上传到 `/app/redis/packet` 目录下。

```bash
# 进入源码包所在目录
cd /app/redis/packet

# 解压源码包
tar -zxvf redis-7.0.12.tar.gz
```

**编译和安装**

```bash
# 进入解压后的源码目录
cd redis-7.0.12

# 清理之前的编译产物（可选，确保干净编译）
make distclean

# 编译 Redis
make

# 安装 Redis 可执行文件到指定目录 /app/redis/bin
# PREFIX 指定安装路径，这样 redis-server, redis-cli 等命令会安装在 /app/redis/bin 目录下
make install PREFIX=/app/redis
```

*   **注意:** 如果 `make` 时提示 `cc: command not found`，表示缺少 `gcc` 编译器，请根据“环境准备”部分的指引安装 `gcc`。

## 主从模式 (Master-Slave) 部署

主从模式提供数据冗余和读写分离的能力。一个 Master 节点负责写操作，并将数据同步给一个或多个 Slave 节点，Slave 节点通常负责读操作。

### 环境规划

| 服务器 IP 地址   | 端口  | 角色   | 数据目录示例             | 配置文件名                 |
| :------------- | :---- | :----- | :----------------------- | :------------------------- |
| 172.27.199.218 | 54801 | Master | `/app/redis/data/ms_54801` | `redis-master-54801.conf` |
| 172.27.199.219 | 54802 | Slave  | `/app/redis/data/ms_54802` | `redis-slave-54802.conf`  |

### 配置文件准备

**Master 节点配置 (172.27.199.218)**

1.  创建数据目录：
    ```bash
    mkdir -p /app/redis/data/ms_54801
    ```
2.  复制原始配置文件并重命名：
    ```bash
    cp /app/redis/packet/redis-7.0.12/redis.conf /app/redis/conf/redis-master-54801.conf
    ```
3.  编辑 `/app/redis/conf/redis-master-54801.conf` 文件，内容如下：

    ```ini
    # 允许远程连接
    bind 0.0.0.0

    # 后台运行
    daemonize yes

    # 关闭保护模式，允许外部访问 (需配合密码或 bind 使用)
    protected-mode no

    # 实例监听端口
    port 54801

    # 设置密码 (!!!请务必修改为强密码!!!)
    requirepass your_strong_password

    # PID 文件路径
    pidfile /app/redis/pid/redis_master_54801.pid

    # 日志文件路径
    logfile /app/redis/logs/redis_master_54801.log

    # 数据目录
    dir /app/redis/data/ms_54801

    # RDB 文件名
    dbfilename dump_54801.rdb

    # 开启 AOF 持久化 (可选，建议开启)
    # appendonly yes
    # AOF 文件名 (如果开启 AOF)
    # appendfilename "appendonly_54801.aof"

    # 最大内存限制 (示例: 2GB)
    # maxmemory 2gb
    # 内存淘汰策略
    # maxmemory-policy allkeys-lru

    # 从节点默认只读 (Master 配置中此项通常不影响行为，但在Slave中重要)
    replica-read-only yes
    ```

**Slave 节点配置 (172.27.199.219)**

1.  创建数据目录：
    ```bash
    mkdir -p /app/redis/data/ms_54802
    ```
2.  复制原始配置文件并重命名：
    ```bash
    cp /app/redis/packet/redis-7.0.12/redis.conf /app/redis/conf/redis-slave-54802.conf
    ```
3.  编辑 `/app/redis/conf/redis-slave-54802.conf` 文件，内容如下：

    ```ini
    # 允许远程连接
    bind 0.0.0.0

    # 后台运行
    daemonize yes

    # 关闭保护模式
    protected-mode no

    # 实例监听端口
    port 54802

    # 设置密码 (!!!与 Master 保持一致或单独设置!!!)
    requirepass your_strong_password

    # 指定 Master 节点的 IP 和端口 (!!!核心配置!!!)
    replicaof 172.27.199.218 54801

    # 连接 Master 节点的认证密码 (!!!必须与 Master 的 requirepass 一致!!!)
    masterauth your_strong_password

    # PID 文件路径
    pidfile /app/redis/pid/redis_slave_54802.pid

    # 日志文件路径
    logfile /app/redis/logs/redis_slave_54802.log

    # 数据目录
    dir /app/redis/data/ms_54802

    # RDB 文件名
    dbfilename dump_54802.rdb

    # AOF 文件名 (如果开启 AOF)
    # appendfilename "appendonly_54802.aof"

    # 最大内存限制 (示例: 2GB)
    # maxmemory 2gb
    # 内存淘汰策略
    # maxmemory-policy allkeys-lru

    # 从节点只读 (保持默认)
    replica-read-only yes
    ```

    *   **重要:** 请将 `your_strong_password` 替换为您实际使用的强密码。

### 启动服务

**启动 Master 节点 (172.27.199.218)**
```bash
/app/redis/bin/redis-server /app/redis/conf/redis-master-54801.conf
```

**启动 Slave 节点 (172.27.199.219)**
```bash
/app/redis/bin/redis-server /app/redis/conf/redis-slave-54802.conf
```

### 验证主从关系

登录 Master 或 Slave 节点，查看复制信息：

```bash
# 登录 Master
/app/redis/bin/redis-cli -h 172.27.199.218 -p 54801 -a your_strong_password

# 登录 Slave
/app/redis/bin/redis-cli -h 172.27.199.219 -p 54802 -a your_strong_password

# 在 redis-cli 中执行
info replication
```

*   在 Master 节点 `info replication` 输出中，应能看到 `role:master` 和连接的 slave 信息。
*   在 Slave 节点 `info replication` 输出中，应能看到 `role:slave` 和 `master_host`, `master_port`, `master_link_status:up` 等信息。

## 哨兵模式 (Sentinel) 部署

哨兵模式在主从模式的基础上增加了自动故障转移能力。哨兵进程监控主从节点状态，当 Master 节点故障时，能自动将一个 Slave 节点提升为新的 Master。通常需要部署奇数个哨兵进程（>=3）以保证选举的可靠性。

### 环境规划

| 服务器 IP 地址   | 端口  | 角色        | 数据目录示例                   | 配置文件名                         |
| :------------- | :---- | :---------- | :----------------------------- | :--------------------------------- |
| 172.27.199.218 | 54801 | Redis Master | `/app/redis/data/sentinel_m_54801` | `redis-sentinel-master-54801.conf`  |
| 172.27.199.218 | 44801 | Sentinel 1  | `/app/redis/data/sentinel1_44801`| `sentinel1-44801.conf`             |
| 172.27.199.219 | 54801 | Redis Slave 1 | `/app/redis/data/sentinel_s1_54801`| `redis-sentinel-slave1-54801.conf`|
| 172.27.199.219 | 44801 | Sentinel 2  | `/app/redis/data/sentinel2_44801`| `sentinel2-44801.conf`             |
| 172.27.199.220 | 54801 | Redis Slave 2 | `/app/redis/data/sentinel_s2_54801`| `redis-sentinel-slave2-54801.conf`|
| 172.27.199.220 | 44801 | Sentinel 3  | `/app/redis/data/sentinel3_44801`| `sentinel3-44801.conf`             |

*   **注意:** 实际生产中，建议将 Master、Slave 和 Sentinel 部署在不同的物理机或虚拟机上以提高容灾能力。本示例为简化部署，部分 Redis 实例和 Sentinel 部署在同一台服务器上。

### Redis 主从节点配置 (Master, Slave1, Slave2)

**Master 节点配置 (172.27.199.218:54801)**

1.  创建数据目录：
    ```bash
    # 在 172.27.199.218 上执行
    mkdir -p /app/redis/data/sentinel_m_54801
    ```
2.  创建配置文件 `/app/redis/conf/redis-sentinel-master-54801.conf` 内容如下:

    ```ini
    bind 0.0.0.0
    daemonize yes
    protected-mode no
    port 54801
    requirepass your_strong_password # 哨兵模式下主节点必须设置密码
    pidfile /app/redis/pid/redis_sentinel_master_54801.pid
    logfile /app/redis/logs/redis_sentinel_master_54801.log
    dir /app/redis/data/sentinel_m_54801
    dbfilename dump_sentinel_master_54801.rdb
    # maxmemory 2gb
    # maxmemory-policy allkeys-lru
    ```

**Slave1 节点配置 (172.27.199.219:54801)**

1.  创建数据目录：
    ```bash
    # 在 172.27.199.219 上执行
    mkdir -p /app/redis/data/sentinel_s1_54801
    ```
2.  创建配置文件 `/app/redis/conf/redis-sentinel-slave1-54801.conf` 内容如下:

    ```ini
    bind 0.0.0.0
    daemonize yes
    protected-mode no
    port 54801 # Slave 端口可以和 Master 不同机上的端口相同
    requirepass your_strong_password # 从节点也建议设置密码，与主节点一致或独立设置
    replicaof 172.27.199.218 54801
    masterauth your_strong_password  # 必须与 Master 的 requirepass 一致
    pidfile /app/redis/pid/redis_sentinel_slave1_54801.pid
    logfile /app/redis/logs/redis_sentinel_slave1_54801.log
    dir /app/redis/data/sentinel_s1_54801
    dbfilename dump_sentinel_slave1_54801.rdb
    replica-read-only yes
    # maxmemory 2gb
    # maxmemory-policy allkeys-lru
    ```

**Slave2 节点配置 (172.27.199.220:54801)**

1.  创建数据目录：
    ```bash
    # 在 172.27.199.220 上执行
    mkdir -p /app/redis/data/sentinel_s2_54801
    ```
2.  创建配置文件 `/app/redis/conf/redis-sentinel-slave2-54801.conf` 内容如下:

    ```ini
    bind 0.0.0.0
    daemonize yes
    protected-mode no
    port 54801
    requirepass your_strong_password
    replicaof 172.27.199.218 54801
    masterauth your_strong_password
    pidfile /app/redis/pid/redis_sentinel_slave2_54801.pid
    logfile /app/redis/logs/redis_sentinel_slave2_54801.log
    dir /app/redis/data/sentinel_s2_54801
    dbfilename dump_sentinel_slave2_54801.rdb
    replica-read-only yes
    # maxmemory 2gb
    # maxmemory-policy allkeys-lru
    ```
    *   **重要:** 再次提醒，`your_strong_password` 需要替换。

**启动 Redis 主从节点**

分别在对应服务器上使用各自的配置文件启动 Master 和两个 Slave 实例。

```bash
# 在 172.27.199.218 上启动 Master
/app/redis/bin/redis-server /app/redis/conf/redis-sentinel-master-54801.conf

# 在 172.27.199.219 上启动 Slave1
/app/redis/bin/redis-server /app/redis/conf/redis-sentinel-slave1-54801.conf

# 在 172.27.199.220 上启动 Slave2
/app/redis/bin/redis-server /app/redis/conf/redis-sentinel-slave2-54801.conf
```
启动后，可以通过 `redis-cli` 连接并使用 `info replication` 检查主从状态。

### 哨兵节点配置 (Sentinel 1, 2, 3)

在三台服务器 (172.27.199.218, 219, 220) 上分别配置哨兵。

1.  创建哨兵数据目录并复制原始哨兵配置文件：
    ```bash
    # 在 172.27.199.218 上执行:
    mkdir -p /app/redis/data/sentinel1_44801
    cp /app/redis/packet/redis-7.0.12/sentinel.conf /app/redis/conf/sentinel1-44801.conf

    # 在 172.27.199.219 上执行:
    mkdir -p /app/redis/data/sentinel2_44801
    cp /app/redis/packet/redis-7.0.12/sentinel.conf /app/redis/conf/sentinel2-44801.conf

    # 在 172.27.199.220 上执行:
    mkdir -p /app/redis/data/sentinel3_44801
    cp /app/redis/packet/redis-7.0.12/sentinel.conf /app/redis/conf/sentinel3-44801.conf
    ```
2.  编辑哨兵配置文件。**三台服务器上的哨兵配置文件内容应保持一致，但 `pidfile`, `logfile`, `dir` 需要根据规划的路径确保唯一性，或者保持相同路径（Redis会自动区分）。为清晰起见，我们这里为每个哨兵指定不同但有规律的路径。**

    **通用哨兵配置内容 (例如 `/app/redis/conf/sentinel1-44801.conf` on 172.27.199.218):**
    ```ini
    # 哨兵监听端口
    port 44801

    # 后台运行
    daemonize yes

    # PID 文件路径 (每个哨兵实例应有独立的 PID 文件)
    pidfile /app/redis/pid/sentinel_44801_node1.pid # sentinel1-44801.conf
    # 对于 sentinel2-44801.conf: pidfile /app/redis/pid/sentinel_44801_node2.pid
    # 对于 sentinel3-44801.conf: pidfile /app/redis/pid/sentinel_44801_node3.pid

    # 日志文件路径 (每个哨兵实例应有独立的日志文件)
    logfile /app/redis/logs/sentinel_44801_node1.log # sentinel1-44801.conf
    # 对于 sentinel2-44801.conf: logfile /app/redis/logs/sentinel_44801_node2.log
    # 对于 sentinel3-44801.conf: logfile /app/redis/logs/sentinel_44801_node3.log

    # 工作目录 (哨兵会在该目录下写入状态信息，每个哨兵实例应有独立的目录)
    dir /app/redis/data/sentinel1_44801 # sentinel1-44801.conf
    # 对于 sentinel2-44801.conf: dir /app/redis/data/sentinel2_44801
    # 对于 sentinel3-44801.conf: dir /app/redis/data/sentinel3_44801

    # 监控主节点配置 (!!!核心配置!!!)
    # sentinel monitor <master-group-name> <master-ip> <master-port> <quorum>
    # <master-group-name>: 主节点组名，自定义，例如 mymaster
    # <master-ip>: 当前 Master 节点的 IP
    # <master-port>: 当前 Master 节点的 Port
    # <quorum>: 判定 Master 客观下线 (ODOWN) 所需的最少哨兵同意数。建议设置为 (哨兵总数 / 2) + 1
    sentinel monitor mymaster 172.27.199.218 54801 2

    # Master 节点的认证密码 (!!!如果 Master 设置了 requirepass!!!)
    # sentinel auth-pass <master-group-name> <password>
    sentinel auth-pass mymaster your_strong_password

    # Master 被判定为主观下线 (SDOWN) 后，超过多少毫秒未恢复，则判定为客观下线 (ODOWN)
    # sentinel down-after-milliseconds <master-group-name> <milliseconds>
    sentinel down-after-milliseconds mymaster 5000 # 示例: 5秒（生产环境根据情况调整，默认30秒）

    # 发生故障转移时，最多允许多少个 Slave 同时向新的 Master 发起同步
    # sentinel parallel-syncs <master-group-name> <numslaves>
    sentinel parallel-syncs mymaster 1 # 示例: 一次只允许一个 Slave 同步

    # 故障转移超时时间 (毫秒)
    # sentinel failover-timeout <master-name> <milliseconds>
    # sentinel failover-timeout mymaster 180000 # 默认3分钟
    ```
    **请确保为每个哨兵实例（`sentinel1-44801.conf`, `sentinel2-44801.conf`, `sentinel3-44801.conf`）修改 `pidfile`, `logfile`, 和 `dir` 指向其各自规划的路径。**

### 启动哨兵服务

在三台配置了哨兵的服务器上，分别启动哨兵进程：

```bash
# 在 172.27.199.218 上执行
/app/redis/bin/redis-sentinel /app/redis/conf/sentinel1-44801.conf

# 在 172.27.199.219 上执行
/app/redis/bin/redis-sentinel /app/redis/conf/sentinel2-44801.conf

# 在 172.27.199.220 上执行
/app/redis/bin/redis-sentinel /app/redis/conf/sentinel3-44801.conf
```

### 验证哨兵状态

连接到任意一个哨兵进程，查看监控信息：

```bash
# 连接到 Sentinel 1 (示例)
/app/redis/bin/redis-cli -p 44801

# 在 redis-cli 中执行哨兵命令
# 查看所有被监控的 Master 及其状态
SENTINEL masters

# 查看指定 Master 的详细信息
SENTINEL master mymaster

# 查看指定 Master 的所有 Slave 及其状态
SENTINEL slaves mymaster

# 查看监控该 Master 的所有 Sentinel 信息
SENTINEL sentinels mymaster

# 获取当前 Master 的地址
SENTINEL get-master-addr-by-name mymaster

# 检查 Sentinel 配置是否能达到 quorum (正常返回 OK)
SENTINEL ckquorum mymaster
```

## 集群模式 (Cluster) 部署

Redis Cluster 提供数据分片（Sharding）和高可用性。数据自动分布在多个 Master 节点上，每个 Master 可以有多个 Slave。集群采用无中心架构，节点间通过 Gossip 协议通信。

### 集群原理简介

*   **结构:** N 个 Master 节点，每个 Master 可以有 M 个 Slave 节点。为了保证高可用，通常建议 N >= 3。至少需要 3 主 3 从（共 6 个节点）才能保证基本的容错能力。
*   **数据分片:** 整个数据库被分为 16384 个哈希槽 (slot)，每个 Master 节点负责处理一部分槽。
*   **故障检测:** 节点间通过 PING/PONG 消息检测存活状态。如果一个节点超过半数的其他 Master 节点认为其 PFAIL (Possible Fail)，则该节点被标记为 FAIL (客观下线)。
*   **故障转移:** 如果一个 Master 节点 FAIL，其 Slave 节点会尝试发起选举，接管 Master 负责的槽并成为新的 Master。

### 环境规划 (三主三从示例)

| 服务器 IP 地址   | 端口  | 角色        | 备注                      | 配置文件名          |
| :------------- | :---- | :---------- | :------------------------ | :------------------ |
| 192.168.5.248  | 7001  | Node 1 (M1) | Master                    | `redis-7001.conf` |
| 192.168.5.248  | 7002  | Node 2 (S3) | Slave of Node 5 (M3)      | `redis-7002.conf` |
| 192.168.5.221  | 7003  | Node 3 (M2) | Master                    | `redis-7003.conf` |
| 192.168.5.221  | 7004  | Node 4 (S1) | Slave of Node 1 (M1)      | `redis-7004.conf` |
| 192.168.5.102  | 7005  | Node 5 (M3) | Master                    | `redis-7005.conf` |
| 192.168.5.102  | 7006  | Node 6 (S2) | Slave of Node 3 (M2)      | `redis-7006.conf` |

*   **重要:**
    *   主节点 (7001, 7003, 7005) 建议分布在不同的物理服务器上。
    *   同一槽位的主从节点 (例如，M1 和 S1) 也必须分布在不同的物理服务器上，以提高容灾能力。本示例为简化，部分节点在同一台机器，生产环境请务必分开。
    *   集群节点间通信需要额外端口：`数据端口 + 10000` (例如 7001 对应 17001)。请确保防火墙允许这些端口的通信。

### 节点配置 (所有 6 个节点)

为每个节点创建独立的配置和数据/日志/PID目录。

**在每台服务器上创建对应节点的目录结构：**

```bash
# 在 192.168.5.248 上创建目录
mkdir -p /app/redis/cluster_nodes/7001/{data,pid,logs}
mkdir -p /app/redis/cluster_nodes/7002/{data,pid,logs}

# 在 192.168.5.221 上创建目录
mkdir -p /app/redis/cluster_nodes/7003/{data,pid,logs}
mkdir -p /app/redis/cluster_nodes/7004/{data,pid,logs}

# 在 192.168.5.102 上创建目录
mkdir -p /app/redis/cluster_nodes/7005/{data,pid,logs}
mkdir -p /app/redis/cluster_nodes/7006/{data,pid,logs}
```

**为每个节点创建并修改配置文件：**
将原始的 `redis.conf` 复制 6 份到 `/app/redis/conf/` 目录下，分别命名为 `redis-7001.conf`, `redis-7002.conf`, ..., `redis-7006.conf`。然后仔细修改每个配置文件。

**示例: `/app/redis/conf/redis-7001.conf` (Master 节点 192.168.5.248:7001)**
```ini
# 允许远程连接
bind 0.0.0.0

# 后台运行
daemonize yes

# 关闭保护模式
protected-mode no

# 节点端口
port 7001

# 设置密码 (!!!所有节点建议设置相同密码!!!)
requirepass your_cluster_password
# 集群内部节点间通信认证密码
masterauth your_cluster_password

# PID 文件路径
pidfile /app/redis/cluster_nodes/7001/pid/redis_7001.pid

# 日志文件路径
logfile /app/redis/cluster_nodes/7001/logs/redis_7001.log

# 数据目录
dir /app/redis/cluster_nodes/7001/data

# RDB 文件名
dbfilename dump_7001.rdb

# 开启集群模式 (!!!核心配置!!!)
cluster-enabled yes

# 集群配置文件 (Redis 自动维护)
cluster-config-file nodes-7001.conf # 此文件会在 dir 目录下生成

# 集群节点超时时间 (毫秒)
cluster-node-timeout 5000 # 建议值，默认15000

# 开启 AOF (集群模式下强烈建议开启)
appendonly yes
# AOF 文件名
appendfilename "appendonly_7001.aof"

# 最大内存限制 (根据需要设置)
# maxmemory 2gb
# 内存淘汰策略
# maxmemory-policy allkeys-lru
```

**示例: `/app/redis/conf/redis-7004.conf` (Slave 节点 192.168.5.221:7004)**
```ini
# 允许远程连接
bind 0.0.0.0

# 后台运行
daemonize yes

# 关闭保护模式
protected-mode no

# 节点端口
port 7004

# 设置密码 (与 Master 一致)
requirepass your_cluster_password
masterauth your_cluster_password

# PID 文件路径
pidfile /app/redis/cluster_nodes/7004/pid/redis_7004.pid

# 日志文件路径
logfile /app/redis/cluster_nodes/7004/logs/redis_7004.log

# 数据目录
dir /app/redis/cluster_nodes/7004/data

# RDB 文件名
dbfilename dump_7004.rdb

# 开启集群模式
cluster-enabled yes

# 集群配置文件
cluster-config-file nodes-7004.conf

# 集群节点超时时间
cluster-node-timeout 5000

# 开启 AOF
appendonly yes
appendfilename "appendonly_7004.aof"

# 最大内存限制 (根据需要设置)
# maxmemory 2gb
# 内存淘汰策略
# maxmemory-policy allkeys-lru
```

**请为其他四个节点 (`redis-7002.conf`, `redis-7003.conf`, `redis-7005.conf`, `redis-7006.conf`) 以此类推，确保以下参数针对每个节点唯一且正确：**
*   `port`
*   `pidfile` (路径中的端口号)
*   `logfile` (路径中的端口号)
*   `dir` (路径中的端口号)
*   `dbfilename` (文件名中的端口号)
*   `cluster-config-file` (文件名中的端口号)
*   `appendfilename` (文件名中的端口号，如果启用 AOF)

所有节点的 `requirepass`, `masterauth`, `cluster-enabled`, `cluster-node-timeout`应保持一致（`your_cluster_password` 替换为实际强密码）。

### 启动所有节点

在对应的服务器上，使用各自的配置文件启动全部 6 个 Redis 实例。

```bash
# 在 192.168.5.248 上启动 7001, 7002
/app/redis/bin/redis-server /app/redis/conf/redis-7001.conf
/app/redis/bin/redis-server /app/redis/conf/redis-7002.conf

# 在 192.168.5.221 上启动 7003, 7004
/app/redis/bin/redis-server /app/redis/conf/redis-7003.conf
/app/redis/bin/redis-server /app/redis/conf/redis-7004.conf

# 在 192.168.5.102 上启动 7005, 7006
/app/redis/bin/redis-server /app/redis/conf/redis-7005.conf
/app/redis/bin/redis-server /app/redis/conf/redis-7006.conf
```

### 创建集群

选择**任意一台安装了 Redis 的服务器**（确保 `redis-cli` 可用），执行集群创建命令。

```bash
# 进入 Redis bin 目录
cd /app/redis/bin/

# 执行创建集群命令
# --cluster-replicas 1 表示为每个 Master 创建 1 个 Slave
# -a 指定所有节点的密码 (需要 Redis 5 或更高版本支持在 create 命令中直接使用 -a)
# 调整节点顺序，确保前三个是期望的主节点，后三个是期望的从节点，
# redis-cli 会尝试将从节点分配给不同IP的主节点。
# 根据环境规划，我们将 248:7001, 221:7003, 102:7005 作为主节点
# 将 221:7004 作为 248:7001 的从，102:7006 作为 221:7003 的从，248:7002 作为 102:7005 的从
./redis-cli --cluster create \
  192.168.5.248:7001 \
  192.168.5.221:7003 \
  192.168.5.102:7005 \
  192.168.5.221:7004 \
  192.168.5.102:7006 \
  192.168.5.248:7002 \
  --cluster-replicas 1 \
  -a your_cluster_password
```

*   **命令解释:**
    *   `--cluster create`: 表示执行创建集群操作。
    *   列出所有节点的 `IP:Port`。**顺序很重要**：先列出所有期望成为 Master 的节点，然后列出所有期望成为 Slave 的节点。`redis-cli` 会根据这个顺序和 `--cluster-replicas` 参数来分配主从角色。
    *   `--cluster-replicas 1`: `redis-cli` 会自动将列表中的前 N 个节点设为 Master， 后 N * replicas 个节点设为 Slave，并尽量将 Master 和 Slave 分配到不同的 IP 地址上。这里 N=3, replicas=1, 所以前 3 个是 Master，后 3 个是 Slave。
    *   `-a your_cluster_password`: 提供集群节点的密码。

*   **确认配置:**
    执行命令后，`redis-cli` 会计算出槽分配方案并请求确认。仔细检查方案是否符合预期（例如，Master 和其 Slave 不在同一台服务器上）。如果无误，输入 `yes` 并回车。

*   **防火墙注意:** 再次提醒，确保服务器间 `7001-7006` 端口以及 `17001-17006` 端口（集群总线端口，即 `数据端口 + 10000`）互相开放。

### 验证集群状态

使用 `redis-cli` 连接到集群中的任意一个节点，并使用 `-c` 参数（表示启用集群模式，客户端会自动处理 MOVED/ASK 重定向）。

```bash
# 连接到集群节点 7001 (示例)
/app/redis/bin/redis-cli -c -h 192.168.5.248 -p 7001 -a your_cluster_password

# 在 redis-cli 中执行
# 查看集群节点信息和槽分配情况
cluster nodes

# 查看集群整体信息
cluster info

# 尝试读写数据 (集群模式下会被自动路由到正确的节点)
set mykey myvalue
get mykey
```

输出 `cluster nodes` 时，应能看到所有 6 个节点，状态为 `connected`，并显示了每个 Master 节点负责的槽范围以及其 Slave 节点。

## 安全注意事项

*   **强密码:** 无论是 `requirepass` 还是 `masterauth`，都必须使用复杂且难以猜测的强密码。切勿使用示例中的 `your_strong_password` 或 `your_cluster_password`。
*   **网络绑定 (`bind`):** 除非确实需要从任何地方访问 Redis，否则应将 `bind` 指令设置为明确的内网 IP 地址，限制访问来源。对于对外提供服务的机器，`0.0.0.0` 是可以的，但务必配合强密码和防火墙。
*   **保护模式 (`protected-mode`):** 如果 Redis 不需要对外提供服务（例如，仅供本机或特定内网应用访问），建议保持 `protected-mode yes`，并配合 `bind 127.0.0.1` 使用。关闭 `protected-mode no` 时，必须设置密码 (`requirepass`)。
*   **防火墙:** 配置严格的防火墙规则，仅允许必要的端口（Redis 数据端口、哨兵端口、集群总线端口）和来源 IP 访问。
*   **非 Root 用户运行:** 考虑创建专门的 `redis` 用户和组，并以该用户身份运行 Redis 服务，以减小潜在的安全风险。

## 常用运维命令

**通用命令**

```bash
# 启动 Redis 服务 (后台模式)
/app/redis/bin/redis-server /path/to/your/redis.conf

# 启动 Sentinel 服务 (后台模式)
/app/redis/bin/redis-sentinel /path/to/your/sentinel.conf

# 连接 Redis 实例
/app/redis/bin/redis-cli -h <ip> -p <port> -a <password>

# 连接 Sentinel 实例
/app/redis/bin/redis-cli -p <sentinel_port> # 如果哨兵设置了密码, 使用 redis-cli -p <port> AUTH <password>

# 连接 Redis Cluster (启用集群模式)
/app/redis/bin/redis-cli -c -h <node_ip> -p <node_port> -a <password>

# 关闭 Redis 实例 (通过客户端)
/app/redis/bin/redis-cli -h <ip> -p <port> -a <password> shutdown

# 检查 Redis 服务是否存活
/app/redis/bin/redis-cli -h <ip> -p <port> -a <password> ping # 应返回 PONG
```

**模式特定命令 (在 redis-cli 中执行)**

*   **主从/哨兵模式 (连接 Redis 节点):**
    *   `info replication`: 查看主从复制信息。
*   **哨兵模式 (连接 Sentinel):**
    *   `SENTINEL masters`: 查看所有被监控的 Master 及其状态。
    *   `SENTINEL slaves <master-name>`: 查看指定 Master 的所有 Slave 及其状态。
    *   `SENTINEL sentinels <master-name>`: 查看监控该 Master 的所有 Sentinel 实例。
    *   `SENTINEL get-master-addr-by-name <master-name>`: 获取指定 Master 的当前 IP 和端口。
    *   `SENTINEL ckquorum <master-name>`: 检查当前 Sentinel 配置是否能达到法定数量。
*   **集群模式 (连接 Cluster 节点，带 `-c` 参数):**
    *   `cluster nodes`: 查看集群节点状态和槽分配。
    *   `cluster info`: 查看集群整体信息。
    *   `cluster meet <ip> <port>`: 手动添加节点到集群 (在节点未加入集群时)。
    *   `cluster forget <node-id>`: 从集群中移除一个节点。
    *   `cluster replicate <master-node-id>`: 将当前连接的节点设置为指定 Master 节点的 Slave。
    *   `cluster addslots <slot> [<slot> ...]` : 给当前节点分配哈希槽。
    *   `cluster reshard <node-ip>:<node-port> --cluster-from <source-node-id> --cluster-to <target-node-id> --cluster-slots <number-of-slots> --cluster-yes`: 进行槽迁移。

## 结语

本文详细介绍了 Redis 主从、哨兵和集群三种模式的离线搭建过程，涵盖了环境准备、安装、配置、启动和验证等关键步骤，并提供了具体的配置文件示例。根据您的业务场景对可用性、可扩展性和一致性的不同要求，可以选择合适的模式进行部署。请务必在实际部署中关注安全配置，并根据具体硬件资源调整内存、持久化等相关参数。希望这篇指南能帮助您成功搭建稳定可靠的 Redis 服务。

---
