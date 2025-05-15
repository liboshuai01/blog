---
title: Centos部署Redis主从/哨兵/集群实战指南
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
mkdir -p /app/redis/packet
mkdir -p /app/redis/conf
mkdir -p /app/redis/logs
mkdir -p /app/redis/pid
mkdir -p /app/redis/data
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

| 服务器 IP 地址   | 端口  | 角色   |
| :------------- | :---- | :----- |
| 172.27.199.218 | 54801 | Master |
| 172.27.199.219 | 54802 | Slave  |

### 配置文件准备

**Master 节点配置 (172.27.199.218)**

1.  复制原始配置文件：
    ```bash
    cp /app/redis/packet/redis-7.0.12/redis.conf /app/redis/conf/redis-master-54801.conf
    ```
2.  编辑 `/app/redis/conf/redis-master-54801.conf` 文件，修改或确认以下关键配置：

    ```ini
    # 允许远程连接 (注释掉或改为 0.0.0.0)
    # bind 127.0.0.1
    bind 0.0.0.0 # 或者指定具体的 IP 地址

    # 后台运行
    daemonize yes

    # 关闭保护模式，允许外部访问 (需配合密码或 bind 使用)
    protected-mode no

    # 实例监听端口
    port 54801

    # 设置密码 (!!!请务必修改为强密码!!!)
    requirepass your_strong_password

    # 设置主节点连接从节点时的认证密码 (如果从节点也设置了 requirepass)
    # masterauth your_strong_password # 主连从时用，这里是Master配置，此行通常不需要，但写上无妨

    # PID 文件路径
    pidfile /app/redis/pid/redis_54801.pid

    # 日志文件路径
    logfile /app/redis/logs/redis_54801.log

    # 数据目录
    dir /app/redis/data/54801 # 建议为每个实例创建独立数据目录

    # RDB 文件名
    dbfilename dump_54801.rdb

    # 开启 AOF 持久化 (可选，建议开启)
    # appendonly yes
    # AOF 文件名
    # appendfilename "appendonly_54801.aof"

    # 最大内存限制 (示例: 2GB)
    maxmemory 2147483648
    # 内存淘汰策略
    maxmemory-policy allkeys-lru

    # 从节点默认只读
    replica-read-only yes
    ```

**Slave 节点配置 (172.27.199.219)**

1.  复制原始配置文件：
    ```bash
    cp /app/redis/packet/redis-7.0.12/redis.conf /app/redis/conf/redis-slave-54802.conf
    ```
2.  编辑 `/app/redis/conf/redis-slave-54802.conf` 文件，修改或确认以下关键配置：

    ```ini
    # 允许远程连接
    # bind 127.0.0.1
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
    pidfile /app/redis/pid/redis_54802.pid

    # 日志文件路径
    logfile /app/redis/logs/redis_54802.log

    # 数据目录
    dir /app/redis/data/54802

    # RDB 文件名
    dbfilename dump_54802.rdb

    # AOF 文件名 (如果开启 AOF)
    # appendfilename "appendonly_54802.aof"

    # 最大内存限制 (示例: 2GB)
    maxmemory 2147483648
    # 内存淘汰策略
    maxmemory-policy allkeys-lru

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

| 服务器 IP 地址   | 端口  | 角色        | 备注                      |
| :------------- | :---- | :---------- | :------------------------ |
| 172.27.199.218 | 54801 | Redis Master | Redis 数据节点            |
| 172.27.199.218 | 44801 | Sentinel 1  | 哨兵监控节点              |
| 172.27.199.219 | 54801 | Redis Slave 1 | Redis 数据节点            |
| 172.27.199.219 | 44801 | Sentinel 2  | 哨兵监控节点              |
| 172.27.199.220 | 54801 | Redis Slave 2 | Redis 数据节点            |
| 172.27.199.220 | 44801 | Sentinel 3  | 哨兵监控节点              |

*   **注意:** 实际生产中，建议将 Master、Slave 和 Sentinel 部署在不同的物理机或虚拟机上以提高容灾能力。本示例为简化部署，将 Redis 实例和 Sentinel 部署在同一台服务器上。

### Redis 主从节点配置 (Master, Slave1, Slave2)

**Master 节点配置 (172.27.199.218:54801)**

*   参考主从模式的 Master 配置，确保端口为 `54801`，设置好 `requirepass`。
*   配置文件路径示例: `/app/redis/conf/redis-master-54801.conf`
*   数据目录示例: `/app/redis/data/master_54801`
*   PID 文件示例: `/app/redis/pid/redis_master_54801.pid`
*   日志文件示例: `/app/redis/logs/redis_master_54801.log`

**Slave 节点配置 (172.27.199.219:54801 和 172.27.199.220:54801)**

*   参考主从模式的 Slave 配置，确保端口为 `54801`。
*   核心配置:
    *   `replicaof 172.27.199.218 54801`
    *   `masterauth your_strong_password` (与 Master 的 `requirepass` 一致)
    *   `requirepass your_strong_password` (建议 Slave 也设置密码)
*   配置文件路径示例:
    *   Slave1: `/app/redis/conf/redis-slave1-54801.conf`
    *   Slave2: `/app/redis/conf/redis-slave2-54801.conf`
*   数据目录示例:
    *   Slave1: `/app/redis/data/slave1_54801`
    *   Slave2: `/app/redis/data/slave2_54801`
*   PID/日志文件依此类推。

**启动 Redis 主从节点**

分别在对应服务器上使用各自的配置文件启动 Master 和两个 Slave 实例。

```bash
# 在 218 上启动 Master
/app/redis/bin/redis-server /app/redis/conf/redis-master-54801.conf

# 在 219 上启动 Slave1
/app/redis/bin/redis-server /app/redis/conf/redis-slave1-54801.conf

# 在 220 上启动 Slave2
/app/redis/bin/redis-server /app/redis/conf/redis-slave2-54801.conf
```

### 哨兵节点配置 (Sentinel 1, 2, 3)

在三台服务器 (172.27.199.218, 219, 220) 上分别配置哨兵。

1.  复制原始哨兵配置文件：
    ```bash
    # 在每台哨兵服务器上执行
    cp /app/redis/packet/redis-7.0.12/sentinel.conf /app/redis/conf/sentinel-44801.conf
    ```
2.  编辑 `/app/redis/conf/sentinel-44801.conf` 文件 (**三台服务器上的此文件内容应保持一致**)，修改或确认以下关键配置：

    ```ini
    # 哨兵监听端口
    port 44801

    # 后台运行
    daemonize yes

    # 关闭保护模式 (如果需要远程访问哨兵)
    # protected-mode no # 哨兵通常不需要关闭保护模式，客户端连接的是它发现的 Master

    # PID 文件路径
    pidfile /app/redis/pid/sentinel_44801.pid

    # 日志文件路径
    logfile /app/redis/logs/sentinel_44801.log

    # 工作目录 (哨兵会在该目录下写入状态信息)
    dir /app/redis/data/sentinel_44801 # 建议为哨兵创建独立数据目录

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
    # sentinel down-after-milliseconds mymaster 30000 # 示例: 30秒

    # 发生故障转移时，最多允许多少个 Slave 同时向新的 Master 发起同步
    # sentinel parallel-syncs <master-group-name> <numslaves>
    # sentinel parallel-syncs mymaster 1 # 示例: 一次只允许一个 Slave 同步
    ```

### 启动哨兵服务

在三台配置了哨兵的服务器上，分别启动哨兵进程：

```bash
# 在 218, 219, 220 上分别执行
/app/redis/bin/redis-sentinel /app/redis/conf/sentinel-44801.conf
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

*   **结构:** N 个 Master 节点，每个 Master 可以有 M 个 Slave 节点。为了保证高可用，通常建议 N >= 3 且 N 为奇数。至少需要 3 主 3 从（共 6 个节点）才能保证基本的容错能力。
*   **数据分片:** 整个数据库被分为 16384 个哈希槽 (slot)，每个 Master 节点负责处理一部分槽。
*   **故障检测:** 节点间通过 PING/PONG 消息检测存活状态。如果一个节点超过半数的其他 Master 节点认为其 PFAIL (Possible Fail)，则该节点被标记为 FAIL (客观下线)。
*   **故障转移:** 如果一个 Master 节点 FAIL，其 Slave 节点会尝试发起选举，接管 Master 负责的槽并成为新的 Master。

### 环境规划 (三主三从示例)

| 服务器 IP 地址   | 端口  | 角色        | 备注                      |
| :------------- | :---- | :---------- | :------------------------ |
| 192.168.5.248  | 7001  | Node 1 (M1) | Master                    |
| 192.168.5.248  | 7002  | Node 2 (S3) | Slave of Node 5 (M3)      |
| 192.168.5.221  | 7003  | Node 3 (M2) | Master                    |
| 192.168.5.221  | 7004  | Node 4 (S1) | Slave of Node 1 (M1)      |
| 192.168.5.102  | 7005  | Node 5 (M3) | Master                    |
| 192.168.5.102  | 7006  | Node 6 (S2) | Slave of Node 3 (M2)      |

*   **重要:**
    *   主节点 (7001, 7003, 7005) 必须分布在不同的服务器上。
    *   同一槽位的主从节点 (如 7001 和 7004) 也必须分布在不同的服务器上，以提高容灾能力。
    *   集群节点间通信需要额外端口：`数据端口 + 10000` (例如 7001 对应 17001)。请确保防火墙允许这些端口的通信。

### 节点配置 (所有 6 个节点)

为每个节点创建独立的配置和数据目录。

```bash
# 在 192.168.5.248 上创建目录
mkdir -p /app/redis/cluster_nodes/7001 /app/redis/cluster_nodes/7002
mkdir -p /app/redis/cluster_nodes/7001/{data,pid,logs}
mkdir -p /app/redis/cluster_nodes/7002/{data,pid,logs}

# 在 192.168.5.221 上创建目录
mkdir -p /app/redis/cluster_nodes/7003 /app/redis/cluster_nodes/7004
mkdir -p /app/redis/cluster_nodes/7003/{data,pid,logs}
mkdir -p /app/redis/cluster_nodes/7004/{data,pid,logs}

# 在 192.168.5.102 上创建目录
mkdir -p /app/redis/cluster_nodes/7005 /app/redis/cluster_nodes/7006
mkdir -p /app/redis/cluster_nodes/7005/{data,pid,logs}
mkdir -p /app/redis/cluster_nodes/7006/{data,pid,logs}
```

创建基础配置文件模板 `/app/redis/conf/redis-cluster-base.conf` (此文件仅作模板，实际使用需复制修改)：

```ini
# 允许远程连接
bind 0.0.0.0

# 后台运行
daemonize yes

# 关闭保护模式
protected-mode no

# 节点端口 (!!!每个节点不同!!!)
port 7001 # 需要修改

# 设置密码 (!!!所有节点建议设置相同密码!!!)
requirepass your_cluster_password
# 集群内部节点间通信认证密码
masterauth your_cluster_password

# PID 文件路径 (!!!每个节点不同!!!)
pidfile /app/redis/cluster_nodes/7001/pid/redis_7001.pid # 需要修改

# 日志文件路径 (!!!每个节点不同!!!)
logfile /app/redis/cluster_nodes/7001/logs/redis_7001.log # 需要修改

# 数据目录 (!!!每个节点不同!!!)
dir /app/redis/cluster_nodes/7001/data # 需要修改

# RDB 文件名 (!!!每个节点不同!!!)
dbfilename dump_7001.rdb # 需要修改

# AOF 文件名 (如果开启 AOF, 每个节点不同)
# appendfilename "appendonly_7001.aof" # 需要修改

# 最大内存限制 (根据需要设置)
# maxmemory 2147483648
# 内存淘汰策略
# maxmemory-policy allkeys-lru

# 开启集群模式 (!!!核心配置!!!)
cluster-enabled yes

# 集群配置文件 (!!!每个节点不同, Redis 自动维护!!!)
cluster-config-file nodes-7001.conf # 需要修改

# 集群节点超时时间 (毫秒)
cluster-node-timeout 15000

# 开启 AOF (集群模式下强烈建议开启)
appendonly yes
```

**为每个节点创建并修改配置文件：**

将 `redis-cluster-base.conf` 复制 6 份，分别命名为 `redis-7001.conf`, `redis-7002.conf`, ..., `redis-7006.conf`，并将它们放置在 `/app/redis/conf/` 目录下。然后，**仔细修改每个配置文件**中带有 `# 需要修改` 注释的行，确保 `port`, `pidfile`, `logfile`, `dir`, `dbfilename`, `appendfilename` (如果启用 AOF), `cluster-config-file` 都使用了与节点端口对应的正确值。

例如，`redis-7002.conf` 应包含：
```ini
port 7002
pidfile /app/redis/cluster_nodes/7002/pid/redis_7002.pid
logfile /app/redis/cluster_nodes/7002/logs/redis_7002.log
dir /app/redis/cluster_nodes/7002/data
dbfilename dump_7002.rdb
cluster-config-file nodes-7002.conf
appendfilename "appendonly_7002.aof" # 如果启用了 appendonly yes
```
其他配置项（如 `bind`, `requirepass`, `cluster-enabled` 等）保持一致。

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
    *   列出所有节点的 `IP:Port`。**顺序很重要**：先列出所有期望成为 Master 的节点，然后列出所有期望成为 Slave 的节点。
    *   `--cluster-replicas 1`: `redis-cli` 会自动将列表中的前 N 个节点设为 Master， 后 N * replicas 个节点设为 Slave，并尽量将 Master 和 Slave 分配到不同的 IP 地址上。这里 N=3, replicas=1, 所以前 3 个是 Master，后 3 个是 Slave。
    *   `-a your_cluster_password`: 提供集群节点的密码。

*   **确认配置:**
    执行命令后，`redis-cli` 会计算出槽分配方案并请求确认。仔细检查方案是否符合预期（例如，Master 和其 Slave 不在同一台服务器上）。如果无误，输入 `yes` 并回车。

*   **防火墙注意:** 再次提醒，确保服务器间 `7001-7006` 端口以及 `17001-17006` 端口（集群总线端口）互相开放。

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

*   **强密码:** 无论是 `requirepass` 还是 `masterauth`，都必须使用复杂且难以猜测的强密码。切勿使用示例中的 `admin123456` 或 `your_strong_password`。
*   **网络绑定 (`bind`):** 除非确实需要从任何地方访问 Redis，否则应将 `bind` 指令设置为明确的内网 IP 地址，限制访问来源。
*   **保护模式 (`protected-mode`):** 如果 Redis 不需要对外提供服务（例如，仅供本机或特定内网应用访问），建议保持 `protected-mode yes`，并配合 `bind` 使用。关闭 `protected-mode no` 时，必须设置密码 (`requirepass`)。
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
/app/redis/bin/redis-cli -p <sentinel_port>

# 连接 Redis Cluster (启用集群模式)
/app/redis/bin/redis-cli -c -h <node_ip> -p <node_port> -a <password>

# 关闭 Redis 实例 (通过客户端)
/app/redis/bin/redis-cli -h <ip> -p <port> -a <password> shutdown

# 检查 Redis 服务是否存活
/app/redis/bin/redis-cli -h <ip> -p <port> -a <password> ping # 应返回 PONG
```

**模式特定命令 (在 redis-cli 中执行)**

*   **主从/哨兵模式:**
    *   `info replication`: 查看主从复制信息。
*   **哨兵模式 (连接 Sentinel):**
    *   `SENTINEL masters`, `SENTINEL slaves <master_name>`, `SENTINEL get-master-addr-by-name <master_name>`
*   **集群模式 (连接 Cluster 节点):**
    *   `cluster nodes`: 查看集群节点状态和槽分配。
    *   `cluster info`: 查看集群整体信息。

## 结语

本文详细介绍了 Redis 主从、哨兵和集群三种模式的离线搭建过程，涵盖了环境准备、安装、配置、启动和验证等关键步骤。根据您的业务场景对可用性、可扩展性和一致性的不同要求，可以选择合适的模式进行部署。请务必在实际部署中关注安全配置，并根据具体硬件资源调整内存、持久化等相关参数。希望这篇指南能帮助您成功搭建稳定可靠的 Redis 服务。