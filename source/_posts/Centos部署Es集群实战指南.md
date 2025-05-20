---
title: Centos部署Es集群实战指南
tags:
  - 环境搭建
  - Linux
  - Es
  - ElasticSearch
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250515191747078.png'
toc: true
abbrlink: c567c8dd
date: 2023-10-10 19:14:59
---

在当今数据驱动的时代，对海量数据进行快速检索、实时分析和高效管理已成为企业成功的基石。Elasticsearch (ES) 作为一款高度可扩展的开源全文搜索和分析引擎，凭借其卓越的分布式特性、RESTful API 以及对多种数据类型的支持，使您能够轻松实现对PB级数据的实时存储、搜索和分析。它广泛应用于日志与事件数据分析、实时应用监控、全文搜索、业务智能（BI）等多种场景，是现代后端架构中不可或缺的一环。

本教程将作为您的向导，一步步带领您在Linux环境下部署一个基础但功能完备的Elasticsearch集群（以7.6.2版本为例）。我们将深入探讨从环境准备、软件下载安装，到核心配置文件（如`jvm.options`和`elasticsearch.yml`）的精细调校，再到集群的启动、验证及状态监控。此外，我们还将涵盖Kibana的可视化部署以及IK中文分词器的集成，为您提供一个端到端的解决方案。

无论您是初探Elasticsearch的新手，还是寻求优化现有部署的有经验的开发者或系统管理员，本教程都将为您提供实用且深入的指导，助您构建稳定、高效的Elasticsearch集群，以满足您的企业级或个人项目需求。

**让我们开始构建您的专属Elasticsearch集群之旅！**

<!-- more -->

## 环境准备

一个稳定高效的Elasticsearch集群离不开精心准备的运行环境。以下是部署前务必完成的基础环境配置：

- 关闭防火墙：[Firewalld防火墙常用命令手册](https://lbs.wiki/pages/7cdf467c/)
- 创建普通用户：[CentOS系统用户管理入门指南](https://lbs.wiki/pages/ccf5da00/)
- 时钟同步：[CentOS7与RockyLinux系统时间同步指南](https://lbs.wiki/pages/669fb41a/)
- 配置免密登录：[Linux多台服务器之间免密登录](https://lbs.wiki/pages/f7c16155/)
- 关闭SELinux、关闭Swap分区、调整文件句柄数和进程数限制、调整虚拟内存区域限制：[优化Centos关闭SELinux/Swap及资源限制调整](https://lbs.wiki/pages/d040eabd/)
- 配置`xsync`、`xcall`同步脚本：[Linux编写xcall命令执行与xsync文件同步脚本](https://lbs.wiki/pages/c8cd0634/)
- 配置JDK 8环境：[Linux安装Jdk8](https://lbs.wiki/pages/42f5694/)

> **重要提示**：环境准备是集群稳定运行的基石，请务必仔细按照参考博文完成每一项配置。

## 集群规划

清晰的集群规划有助于后续的部署与维护。

**操作约定**：
1.  除特别说明外，本文所有操作均在主操作节点 `node-1` 上、使用非`root`用户 `lbs` 执行。
2.  命令中出现的IP地址（`10.0.0.X`），**必须**替换为您自己集群中节点的实际IP地址。
3.  命令中出现的路径 `/home/lbs/software`，可根据您的实际情况替换为自定义路径，但需确保后续所有相关路径一致。

**节点规划示例**：

| 节点名称 (Node Name) | IP 地址    | 角色说明                     |
| :------------------- | :--------- | :--------------------------- |
| `node-1`             | `10.0.0.87` | Master-eligible, Data, Ingest |
| `node-2`             | `10.0.0.81` | Master-eligible, Data, Ingest |
| `node-3`             | `10.0.0.82` | Master-eligible, Data, Ingest |

> Для生产环境，建议至少3个Master-eligible节点以保证高可用性。节点角色可以根据需求精细配置。

## Elasticsearch 安装与配置

### 下载与解压

在 `node-1` 节点上执行以下操作：
```bash
# 创建软件存放目录 (如果尚不存在)
mkdir -p /home/lbs/software

# 进入该目录
cd /home/lbs/software

# 下载Elasticsearch 7.6.2
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.2-linux-x86_64.tar.gz

# 解压到指定安装路径
tar -zxvf elasticsearch-7.6.2-linux-x86_64.tar.gz -C /home/lbs/software

# 重命名为更简洁的目录名 (可选，但推荐)
mv /home/lbs/software/elasticsearch-7.6.2 /home/lbs/software/elasticsearch
```

### 配置JVM参数 (`jvm.options`)

Elasticsearch的性能与JVM堆内存大小密切相关。
编辑 `jvm.options` 文件：
```bash
vim /home/lbs/software/elasticsearch/config/jvm.options
```
修改JVM初始堆大小 (`-Xms`) 和最大堆大小 (`-Xmx`)。**建议将两者设置为相同的值**，以避免运行时JVM堆调整带来的性能开销。
根据服务器可用内存进行配置，**但不应超过物理内存的50%**，且最大不超过30-31GB（具体取决于JVM是否开启指针压缩）。
```properties
# 修改内容如下（示例为4GB，默认值为1g，根据服务器资源设置即可）:
-Xms4g
-Xmx4g
```
> **注意**：Elasticsearch 7.x自带了JDK，但如果系统中配置了`JAVA_HOME`，ES会优先使用系统JDK。确保您使用的JDK与ES版本兼容。此版本(7.6.2)自带的JDK是Java 13，但它也支持您系统已安装的Java 8 或 Java 11。

### 配置ES核心参数 (`elasticsearch.yml`)

这是Elasticsearch的核心配置文件，决定了集群的行为。
编辑 `/home/lbs/software/elasticsearch/config/elasticsearch.yml` 文件：
```bash
vim /home/lbs/software/elasticsearch/config/elasticsearch.yml
```
根据以下说明修改或添加配置项：
```yaml
# 集群名称，同一集群下所有节点的此名称必须一致。
cluster.name: my-es-application # 修改为您期望的集群名

# 节点名称，集群中每个节点的名称必须唯一。
# 在 node-1 上配置为 node-1。后续分发到其他节点时，再分别修改为 node-2, node-3。
node.name: node-1

# ES数据存储路径。确保该目录存在且lbs用户有读写权限。
path.data: /home/lbs/software/elasticsearch/data

# ES运行日志文件路径。确保该目录存在且lbs用户有读写权限。
path.logs: /home/lbs/software/elasticsearch/logs

# 绑定IP地址，允许哪些IP访问ES。
# 设置为 0.0.0.0 表示允许任何网络接口的IP访问。生产环境请谨慎配置，建议绑定内网IP。
network.host: 0.0.0.0

# ES HTTP服务的访问端口号 (REST API)。
http.port: 9200

# 节点间通信端口号 (TCP)。默认为9300。
# transport.port: 9300 # 如果需要可取消注释并修改

# 集群发现配置：用于节点发现和主节点选举的初始主机列表。
# 列出集群中所有 master-eligible 节点的地址和传输端口 (默认为9300)。
# 确保IP和端口正确无误。
discovery.seed_hosts: ["10.0.0.87:9300", "10.0.0.81:9300", "10.0.0.82:9300"]

# 首次启动集群时，用于选举主节点的初始 master-eligible 节点列表。
# 列出所有 master-eligible 节点的 node.name。
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# (可选，但推荐) 防止脑裂的最小主节点数配置
# 对于3节点集群，通常设置为 (3/2) + 1 = 2。
# discovery.zen.minimum_master_nodes: 2 # ES 7.x中被移除, 依赖 discovery.seed_hosts 和 cluster.initial_master_nodes

# (可选，为了允许跨域访问，如Kibana在不同域或本地开发时需要)
# http.cors.enabled: true
# http.cors.allow-origin: "*" # 生产环境应指定具体域名
```
> **配置解读**：
> *   `cluster.name`: 逻辑上组织节点的关键。
> *   `node.name`: 节点的唯一标识符。
> *   `path.data` & `path.logs`: 分离数据和日志，便于管理和备份。
> *   `network.host`: 控制ES服务的可访问性。`0.0.0.0`表示监听所有可用网络接口。
> *   `http.port`: RESTful API 端口。
> *   `discovery.seed_hosts`: 节点如何找到彼此形成集群。
> *   `cluster.initial_master_nodes`: 帮助集群在首次启动时安全地选举出第一个主节点。**此配置仅在集群首次形成时使用，一旦集群形成，后续添加或重启节点时应将其注释掉或忽略，或者确保其列表与实际主节点候选者一致。** 对于本教程，保持即可。

### 创建数据与日志目录

根据`elasticsearch.yml`中的配置，在 `node-1` 上创建相应的目录：
```bash
mkdir -p /home/lbs/software/elasticsearch/data
mkdir -p /home/lbs/software/elasticsearch/logs
```
确保`lbs`用户对这些目录有读写权限。

### 分发到集群其他节点

使用之前配置的`xsync`脚本，将整个`elasticsearch`安装目录从`node-1`分发到`node-2`和`node-3`：
```bash
# 确保xsync脚本在$PATH中，或者使用其绝对路径
# 假设xsync位于/usr/local/bin/xsync
# xsync /home/lbs/software/elasticsearch
# 或者如果xsync在当前用户bin目录且已加入PATH：
xsync /home/lbs/software/elasticsearch
```
> 此命令会将 `/home/lbs/software/elasticsearch` 目录完整地复制到 `node-2` 和 `node-3` 上的相同路径。

### 配置各节点特定参数

分别登录到 `node-2` 和 `node-3`，修改各自的 `elasticsearch.yml` 文件。**唯一需要修改的是 `node.name`**。

**在 `node-2` 上**：
```bash
ssh lbs@10.0.0.81 # 替换为node-2的IP和lbs用户名
vim /home/lbs/software/elasticsearch/config/elasticsearch.yml
```
修改 `node.name` 为：
```yaml
node.name: node-2
```
保存并退出。

**在 `node-3` 上**：
```bash
ssh lbs@10.0.0.82 # 替换为node-3的IP和lbs用户名
vim /home/lbs/software/elasticsearch/config/elasticsearch.yml
```
修改 `node.name` 为：
```yaml
node.name: node-3
```
保存并退出。

> 所有其他配置（如`cluster.name`, `discovery.seed_hosts`, `cluster.initial_master_nodes`等）在所有节点上应保持一致。

## 集群管理脚本

为了方便地启停和查看集群中所有ES节点的状态，我们在 `node-1` 上创建一个管理脚本。

```bash
# 在 node-1 上执行
cat > /home/lbs/software/elasticsearch/bin/elasticsearch-cluster.sh <<EOF
#!/bin/bash

# 定义集群节点IP列表
NODES=("10.0.0.87" "10.0.0.81" "10.0.0.82") # 替换为你的节点IP
ES_HOME="/home/lbs/software/elasticsearch" # ES安装目录
ES_USER="lbs" # 运行ES的用户

# 函数：在指定节点上执行命令
# 用法: run_on_node <node_ip> <command_string>
run_on_node() {
    local node_ip=\$1
    shift
    local cmd_string=\$*
    echo "---- Executing on \$node_ip ----"
    ssh "\$ES_USER@\$node_ip" "source /etc/profile; source ~/.bash_profile; \$cmd_string"
    # 添加 ~/.bash_profile 以确保用户环境变量被加载
}

# 函数：启动单个ES节点
start_es_node() {
    local node_ip=\$1
    echo "-------------------------------- \$node_ip Elasticsearch 启动 ---------------------------"
    # 检查是否已运行
    local pid=\$(run_on_node "\$node_ip" "ps -ef | grep '[o]rg.elasticsearch.bootstrap.Elasticsearch' | awk '{print \$2}'")
    if [[ -n "\$pid" ]]; then
        echo "Elasticsearch is already running on \$node_ip with PID \$pid."
    else
        run_on_node "\$node_ip" "\$ES_HOME/bin/elasticsearch -d -p \$ES_HOME/es.pid"
        # 短暂等待启动
        sleep 5
        pid=\$(run_on_node "\$node_ip" "cat \$ES_HOME/es.pid 2>/dev/null")
        if [[ -n "\$pid" ]] && run_on_node "\$node_ip" "ps -p \$pid > /dev/null"; then
            echo "Elasticsearch started successfully on \$node_ip with PID \$pid."
        else
            echo "Failed to start Elasticsearch on \$node_ip. Check logs: \$ES_HOME/logs/"
        fi
    fi
}

# 函数：停止单个ES节点
stop_es_node() {
    local node_ip=\$1
    echo "-------------------------------- \$node_ip Elasticsearch 停止 ---------------------------"
    local pid=\$(run_on_node "\$node_ip" "cat \$ES_HOME/es.pid 2>/dev/null")
    if [[ -n "\$pid" ]] && run_on_node "\$node_ip" "ps -p \$pid > /dev/null"; then
        run_on_node "\$node_ip" "kill \$pid"
        # 等待进程结束
        timeout=30
        while run_on_node "\$node_ip" "ps -p \$pid > /dev/null" && [[ \$timeout -gt 0 ]]; do
            echo "Waiting for Elasticsearch to stop on \$node_ip (PID \$pid)..."
            sleep 2
            timeout=\$((timeout - 2))
        done
        if ! run_on_node "\$node_ip" "ps -p \$pid > /dev/null"; then
            echo "Elasticsearch stopped on \$node_ip."
            run_on_node "\$node_ip" "rm -f \$ES_HOME/es.pid"
        else
            echo "Elasticsearch (PID \$pid) did not stop gracefully on \$node_ip after 30s. Sending SIGKILL."
            run_on_node "\$node_ip" "kill -9 \$pid"
            run_on_node "\$node_ip" "rm -f \$ES_HOME/es.pid"
            echo "Elasticsearch (PID \$pid) killed on \$node_ip."
        fi
    else
        # 尝试通过进程名查找并杀死（如果PID文件不存在或进程已不是该PID）
        local pids_by_name=\$(run_on_node "\$node_ip" "ps -ef | grep '[o]rg.elasticsearch.bootstrap.Elasticsearch' | awk '{print \$2}'")
        if [[ -n "\$pids_by_name" ]]; then
            echo "Found running Elasticsearch process(es) by name: \$pids_by_name. Stopping them..."
            run_on_node "\$node_ip" "kill \$pids_by_name" # 尝试优雅关闭
            sleep 5
            run_on_node "\$node_ip" "kill -9 \$pids_by_name 2>/dev/null" # 强制关闭
            echo "Elasticsearch process(es) \$pids_by_name killed on \$node_ip."
            run_on_node "\$node_ip" "rm -f \$ES_HOME/es.pid"
        else
            echo "Elasticsearch not running on \$node_ip (or PID file \$ES_HOME/es.pid missing/stale)."
        fi
    fi
}

# 函数：检查单个ES节点状态
status_es_node() {
    local node_ip=\$1
    echo "-------------------------------- \$node_ip Elasticsearch 状态 ---------------------------"
    local pid=\$(run_on_node "\$node_ip" "cat \$ES_HOME/es.pid 2>/dev/null")
    if [[ -n "\$pid" ]] && run_on_node "\$node_ip" "ps -p \$pid > /dev/null"; then
        echo "Elasticsearch is RUNNING on \$node_ip with PID \$pid (from PID file)."
    else
        # PID文件不存在或PID失效，尝试通过进程名查找
        local pids_by_name=\$(run_on_node "\$node_ip" "ps -ef | grep '[o]rg.elasticsearch.bootstrap.Elasticsearch' | awk '{print \$2}'")
        if [[ -n "\$pids_by_name" ]]; then
            echo "Elasticsearch is RUNNING on \$node_ip with PID(s): \$pids_by_name (found by process name)."
        else
            echo "Elasticsearch is NOT RUNNING on \$node_ip."
        fi
    fi
}

case \$1 in
"start")
    for node in "\${NODES[@]}"; do
        start_es_node "\$node"
    done
    echo "-------------------------------- Cluster Start command finished ---------------------------"
    echo "Wait a moment for the cluster to form, then check status or logs."
    ;;
"stop")
    # 倒序停止，可以先停非主节点，但对于简单场景顺序停止也可
    for (( idx=\${#NODES[@]}-1 ; idx>=0 ; idx-- )) ; do
        stop_es_node "\${NODES[idx]}"
    done
    echo "-------------------------------- Cluster Stop command finished ----------------------------"
    ;;
"status")
    for node in "\${NODES[@]}"; do
        status_es_node "\$node"
    done
    echo "-------------------------------- Cluster Status command finished --------------------------"
    ;;
"restart")
    echo "Restarting Elasticsearch cluster..."
    \$0 stop
    sleep 10 # 等待节点完全停止
    \$0 start
    echo "Elasticsearch cluster restart initiated."
    ;;
*)
    echo "Usage: \$0 {start|stop|status|restart}"
    exit 1
    ;;
esac
EOF

# 赋予脚本执行权限
chmod +x /home/lbs/software/elasticsearch/bin/elasticsearch-cluster.sh
```
> **脚本改进说明**：
> *   使用 PID 文件 (`-p $ES_HOME/es.pid`) 进行更精确的进程管理。
> *   启动时检查是否已运行。
> *   停止时首先尝试优雅关闭 (SIGTERM)，超时后才使用 SIGKILL。
> *   状态检查优先使用 PID 文件，失败则尝试通过进程名查找。
> *   确保远程执行命令时加载了用户的环境变量 (`source /etc/profile; source ~/.bash_profile;`)。
> *   增加了`restart`命令。
> *   `NODES` 和 `ES_USER` 变量方便修改。

## 集群启停与验证

### 启动集群

在 `node-1` 上执行启动脚本：
```bash
/home/lbs/software/elasticsearch/bin/elasticsearch-cluster.sh start
```
> **Java版本提示**：您可能会看到类似 `future versions of Elasticsearch will require Java 11; your Java version from [/home/lbs/software/jdk/jre] does not meet this requirement` 的提示（如果使用JDK 8）。对于ES 7.6.2，JDK 8是支持的，此提示可暂时忽略，不影响当前版本使用。但计划未来升级时，需要注意JDK版本的兼容性。

启动后，请耐心等待几十秒到一两分钟，以便各节点启动完成并组成集群。

### 停止集群

在 `node-1` 上执行停止脚本：
```bash
/home/lbs/software/elasticsearch/bin/elasticsearch-cluster.sh stop
```

### 集群健康检查与验证

1.  **查看各节点日志** (在每个节点上或通过`xcall`在`node-1`上统一执行)：
    重点关注日志末尾部分，检查是否有错误信息（`ERROR`）、异常堆栈（`Exception`）或启动失败的指示。
    ```bash
    # 在任一节点上执行 (或使用 xcall 批量查看)
    tail -fn 200 /home/lbs/software/elasticsearch/logs/my-es-application.log # 注意日志文件名是基于cluster.name
    
    # 例如在 node-1 上使用 xcall:
    # xcall "tail -n 100 /home/lbs/software/elasticsearch/logs/my-es-application.log"
    ```

2.  **通过REST API检查节点信息** (可在任一节点或有权限访问ES HTTP端口的机器上执行)：
    ```bash
    curl -X GET "http://10.0.0.87:9200"
    curl -X GET "http://10.0.0.81:9200"
    curl -X GET "http://10.0.0.82:9200"
    ```
    如果成功，将返回类似以下格式的JSON响应，包含节点和集群信息：
    ```json
    {
      "name" : "node-1",
      "cluster_name" : "my-es-application",
      "cluster_uuid" : "some_uuid",
      "version" : { ... },
      "tagline" : "You Know, for Search"
    }
    ```

3.  **查看集群健康状态**：
    ```bash
    # 在 node-1 (master) 或任一节点上执行:
    curl -X GET "http://10.0.0.87:9200/_cluster/health?pretty"
    ```
    预期输出：
    ```json
    {
      "cluster_name" : "my-es-application",
      "status" : "green",                 оптимально "green"
      "timed_out" : false,
      "number_of_nodes" : 3,             // 节点数量
      "number_of_data_nodes" : 3,        // 数据节点数量
      "active_primary_shards" : 0,       // 初始状态无索引则为0
      "active_shards" : 0,               // 初始状态无索引则为0
      "relocating_shards" : 0,
      "initializing_shards" : 0,
      "unassigned_shards" : 0,
      "delayed_unassigned_shards" : 0,
      "number_of_pending_tasks" : 0,
      "number_of_in_flight_fetch" : 0,
      "task_max_waiting_in_queue_millis" : 0,
      "active_shards_percent_as_number" : 100.0
    }
    ```
    > **集群状态解读**：
    > *   `green`: 所有主分片和副本分片都已成功分配。集群健康。
    > *   `yellow`: 所有主分片已分配，但至少有一个副本分片未分配。集群功能正常，但数据冗余性不足。
    > *   `red`: 至少有一个主分片（及其所有副本）未分配。集群部分功能不可用，可能存在数据丢失风险。
          > 初始空集群状态为 `green` 是正常的。

4.  **查看集群节点列表及角色**：
    ```bash
    # 在 node-1 (master) 或任一节点上执行:
    curl -X GET "http://10.0.0.87:9200/_cat/nodes?v&pretty"
    ```
    预期输出 (列名和顺序可能略有差异，IP和name会根据您的配置变化)：
    ```
    ip        heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
    10.0.0.82            9          58   0    0.29    0.22     0.17 dilm      -      node-3
    10.0.0.81           10          58   0    0.02    0.11     0.16 dilm      -      node-2
    10.0.0.87           13          64   0    0.19    0.28     0.30 dilm      *      node-1
    ```
    > `*` 表示当前的主节点 (master)。`node.role` 中的 `d`=data, `i`=ingest, `l`=machine learning(若启用), `m`=master-eligible。

## Kibana 安装与配置

Kibana 是 Elasticsearch 的官方可视化工具，提供数据探索、仪表盘、管理等功能。我们将其安装在 `node-1` 节点上。

### 下载与解压 (Kibana)

在 `node-1` 上执行：
```bash
cd /home/lbs/software # 进入软件存放目录

# 下载Kibana 7.6.2 (应与ES版本匹配)
wget https://artifacts.elastic.co/downloads/kibana/kibana-7.6.2-linux-x86_64.tar.gz

# 解压
tar -zxvf kibana-7.6.2-linux-x86_64.tar.gz -C /home/lbs/software

# 重命名
mv /home/lbs/software/kibana-7.6.2-linux-x86_64 /home/lbs/software/kibana
```

### 配置Kibana (`kibana.yml`)

编辑 Kibana 配置文件：
```bash
vim /home/lbs/software/kibana/config/kibana.yml
```
修改或添加以下配置：
```yaml
# Kibana服务端口号
server.port: 5601

# Kibana绑定的主机地址，0.0.0.0允许外部访问
server.host: "0.0.0.0"

# Elasticsearch集群的访问地址 (可以是任一节点，Kibana会自动发现其他节点)
# 如果有多个协调节点或网关，可以配置多个，用逗号分隔
elasticsearch.hosts: ["http://10.0.0.87:9200", "http://10.0.0.81:9200", "http://10.0.0.82:9200"]

# (可选) Kibana实例名称，显示在浏览器标签页等处
# server.name: "my-kibana"

# (可选) 设置Kibana界面语言为中文
i18n.locale: "zh-CN"

# (可选) 如果Elasticsearch启用了X-Pack安全特性并设置了用户名密码
# elasticsearch.username: "kibana_system_user" # 替换为实际用户
# elasticsearch.password: "your_password"      # 替换为实际密码
```

### Kibana管理脚本
在 `node-1` 上为Kibana创建一个简单的管理脚本：
```bash
# 在 node-1 上执行
cat > /home/lbs/software/kibana/bin/kibana-manage.sh <<EOF
#!/bin/bash

KIBANA_HOME="/home/lbs/software/kibana"
KIBANA_EXEC="\$KIBANA_HOME/bin/kibana"
LOG_DIR="\$KIBANA_HOME/logs"
LOG_FILE="\$LOG_DIR/kibana-`date +%Y-%m-%d`.log"
PID_PATH_NAME="\$KIBANA_HOME/kibana.pid" # Kibana不直接创建pid，我们用pgrep

# 创建日志目录
mkdir -p \$LOG_DIR

get_pid() {
    pgrep -f "\$KIBANA_HOME/node/bin/node .* \$KIBANA_HOME/src/cli"
}

start_kibana() {
    if [ -n "\$(get_pid)" ]; then
        echo "Kibana is already running with PID(s): \$(get_pid)."
        return 0
    fi
    echo "Starting Kibana..."
    # Kibana 7.x中，nohup和后台执行 (&) 由kibana脚本自身管理较好。
    # 直接执行，它会自行处理日志（默认输出到stdout，或根据配置到文件）。
    # 若要完全后台运行且记录日志，可nohup，但建议让Kibana自身或systemd管理。
    # 这里使用 nohup 简单示例：
    nohup \$KIBANA_EXEC > "\$LOG_FILE" 2>&1 &
    # 等待一小会儿让进程启动
    sleep 5 
    if [ -n "\$(get_pid)" ]; then
        echo "Kibana started successfully. Logging to \$LOG_FILE. PID(s): \$(get_pid)."
    else
        echo "Kibana failed to start. Check logs at \$LOG_FILE or run in foreground for more details."
    fi
}

stop_kibana() {
    local pids=\$(get_pid)
    if [ -z "\$pids" ]; then
        echo "Kibana is not running."
        return 0
    fi
    echo "Stopping Kibana (PID(s): \$pids)..."
    kill \$pids
    # 等待优雅关闭
    sleep 5
    pids=\$(get_pid)
    if [ -n "\$pids" ]; then
        echo "Kibana did not stop gracefully. Force killing (PID(s): \$pids)..."
        kill -9 \$pids
        sleep 2
    fi
    if [ -z "\$(get_pid)" ]; then
        echo "Kibana stopped."
    else
        echo "Failed to stop Kibana. PID(s) still active: \$(get_pid)."
    fi
}

status_kibana() {
    local pids=\$(get_pid)
    if [ -n "\$pids" ]; then
        echo "Kibana is RUNNING. PID(s): \$pids"
    else
        echo "Kibana is NOT RUNNING."
    fi
}

case "\$1" in
    start)
        start_kibana
        ;;
    stop)
        stop_kibana
        ;;
    status)
        status_kibana
        ;;
    restart)
        stop_kibana
        start_kibana
        ;;
    *)
        echo "Usage: \$0 {start|stop|status|restart}"
        exit 1
esac
EOF

# 赋予脚本执行权限
chmod +x /home/lbs/software/kibana/bin/kibana-manage.sh
```
> **Kibana脚本说明**：
> * `pgrep` 用于查找Kibana的Node.js进程。Kibana是用Node.js编写的。
> * 启动时nohup后台运行，并将日志输出到`logs/kibana-YYYY-MM-DD.log`。
> * 停止时先尝试优雅关闭，失败则强制关闭。

### 启动与访问Kibana

1.  **启动Kibana** (在 `node-1` 上):
    ```bash
    /home/lbs/software/kibana/bin/kibana-manage.sh start
    ```

2.  **查看Kibana状态** (在 `node-1` 上):
    ```bash
    /home/lbs/software/kibana/bin/kibana-manage.sh status
    # 查看日志
    tail -f /home/lbs/software/kibana/logs/kibana-$(date +%Y-%m-%d).log
    ```
    当日志中出现类似 `{"type":"log","@timestamp":"...","tags":["listening","info"],"pid":...,"message":"Server running at http://0.0.0.0:5601"}` 的信息时，表示Kibana已成功启动并连接到Elasticsearch。

3.  **浏览器访问Kibana**:
    打开浏览器，访问 `http://10.0.0.87:5601` (将IP替换为`node-1`的实际IP，或您配置了`server.host`的任何可访问IP)。
    您应该能看到Kibana的欢迎界面。
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250515184841277.png)
    *(示例图片，实际界面可能因版本略有不同)*

4.  **停止Kibana** (可选，在 `node-1` 上):
    ```bash
    /home/lbs/software/kibana/bin/kibana-manage.sh stop
    ```

## IK中文分词器安装与验证

对于中文内容搜索，默认的Standard Analyzer分词效果不佳。IK Analyzer是一款广泛使用的Elasticsearch中文分词插件。

### 下载与安装 (IK)

1.  **下载IK分词器** (版本需与Elasticsearch版本完全对应，此处为7.6.2):
    访问IK GitHub Releases页面: [https://github.com/infinilabs/analysis-ik/releases](https://github.com/infinilabs/analysis-ik/releases)
    找到 `elasticsearch-analysis-ik-7.6.2.zip` 并下载。
    ```bash
    # 在 node-1 上执行
    cd /home/lbs/software # 或其他临时下载目录
    wget https://github.com/infinilabs/analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
    ```

2.  **在每个Elasticsearch节点上安装IK插件**:
    IK分词器插件需要在集群中的**每一个**Elasticsearch节点上安装。

    **在 `node-1` 上操作**:
    ```bash
    # 创建IK插件目录 (如果用elasticsearch-plugin工具安装则会自动创建，手动解压需先创建)
    mkdir -p /home/lbs/software/elasticsearch/plugins/ik

    # 解压到ik目录
    unzip elasticsearch-analysis-ik-7.6.2.zip -d /home/lbs/software/elasticsearch/plugins/ik
    ```
    **分发到其他节点** (或在每个节点重复下载解压步骤):
    如果您的`xsync`配置正确，可以在`node-1`上执行（确保`plugins/ik`目录已创建并在`elasticsearch`主目录下）：
    ```bash
    # (1) 确保node-2, node-3上 /home/lbs/software/elasticsearch/plugins/ 目录存在
    xcall "mkdir -p /home/lbs/software/elasticsearch/plugins/"
    # (2) 分发ik插件目录
    xsync /home/lbs/software/elasticsearch/plugins/ik 
    ```
    或者，分别登录`node-2`和`node-3`，重复下载和解压IK分词器的步骤，确保路径为各自节点的 `/home/lbs/software/elasticsearch/plugins/ik`。

    > **另一种安装方式 (推荐)**: Elasticsearch提供了`elasticsearch-plugin`工具来管理插件。
    > ```bash
    > # 在每个ES节点上执行 (替换为zip包的URL或本地文件路径)
    > # sudo -u lbs /home/lbs/software/elasticsearch/bin/elasticsearch-plugin install file:///path/to/elasticsearch-analysis-ik-7.6.2.zip
    > # 或直接从URL安装 (需要节点联网)
    > # sudo -u lbs /home/lbs/software/elasticsearch/bin/elasticsearch-plugin install https://github.com/infinilabs/analysis-ik/releases/download/v7.6.2/elasticsearch-analysis-ik-7.6.2.zip
    > ```
    > 使用 `elasticsearch-plugin` 工具可以更好地处理权限和路径问题。这里我们沿用原文的手动解压方式。

### 重启ES集群使分词器生效
安装（或卸载）插件后，必须重启Elasticsearch集群中的所有节点。
```bash
# 在 node-1 上执行
/home/lbs/software/elasticsearch/bin/elasticsearch-cluster.sh restart
```
等待集群重启完成，并再次通过 `_cluster/health` 确认集群状态为 `green`。

### 验证分词效果
通过Kibana的Dev Tools (开发工具) 或 `curl` 命令来测试IK分词器的效果。

打开Kibana -> Dev Tools，执行以下命令：

1.  **智能分词 (`ik_smart`)**: 它会做最粗粒度的拆分。
    ```json
    GET /_analyze
    {
      "text": "中华人民共和国国歌",
      "analyzer": "ik_smart"
    }
    ```
    预期结果 (tokens可能略有不同)：
    ```json
    {
      "tokens" : [
        { "token" : "中华人民共和国", "start_offset" : 0, "end_offset" : 7, "type" : "CN_WORD", "position" : 0 },
        { "token" : "国歌", "start_offset" : 7, "end_offset" : 9, "type" : "CN_WORD", "position" : 1 }
      ]
    }
    ```
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250515184822481.png)

2.  **最细粒度分词 (`ik_max_word`)**: 它会将文本做最细粒度的拆分，适合搜索。
    ```json
    GET /_analyze
    {
      "text": "中华人民共和国国歌",
      "analyzer": "ik_max_word"
    }
    ```
    预期结果 (tokens可能略有不同)：
    ```json
    {
      "tokens" : [
        { "token" : "中华人民共和国", "start_offset" : 0, "end_offset" : 7, "type" : "CN_WORD", "position" : 0 },
        { "token" : "中华人民", "start_offset" : 0, "end_offset" : 4, "type" : "CN_WORD", "position" : 1 },
        { "token" : "中华", "start_offset" : 0, "end_offset" : 2, "type" : "CN_WORD", "position" : 2 },
        { "token" : "华人", "start_offset" : 1, "end_offset" : 3, "type" : "CN_WORD", "position" : 3 },
        // ... 更多细分的词 ...
        { "token" : "国歌", "start_offset" : 7, "end_offset" : 9, "type" : "CN_WORD", "position" : 10 } // position可能变化
      ]
    }
    ```
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250515184830294.png)

如果能看到类似上述的分词结果，则表明IK分词器已成功安装并工作。

## 重要注意事项与优化建议

1.  **安全性**：
    *   **网络层面**：切勿将Elasticsearch的9200/9300端口直接暴露于公网，除非有严格的防火墙规则和安全措施。使用VPC、安全组或防火墙限制访问来源。
    *   **X-Pack Security**：对于生产环境，强烈建议启用Elasticsearch的X-Pack安全特性（7.x版本中基础版免费），配置用户认证、授权 (RBAC)、HTTPS加密通信 (TLS/SSL) 等。
    *   **Kibana安全**：同样为Kibana配置HTTPS，并考虑启用登录认证。

2.  **资源调优**：
    *   **JVM堆内存**：`jvm.options`中的`-Xms`和`-Xmx`设置至关重要。监控JVM堆使用情况，避免OOM。
    *   **文件句柄**：确保`ulimit -n`（打开文件数）足够大（如65536或更高）。
    *   **mmap计数**：如日志中提示`max virtual memory areas vm.max_map_count [xxxxx] is too low, increase to at least [262144]`，则需调整 `/etc/sysctl.conf` 中的 `vm.max_map_count`。
    *   **磁盘**：使用高性能磁盘 (SSD优先)，监控磁盘I/O和可用空间。Elasticsearch对磁盘性能敏感。

3.  **数据备份与恢复**：
    *   定期使用Snapshot API进行数据备份到共享文件系统 (NFS) 或对象存储 (S3, OSS等)。制定并测试恢复策略。

4.  **监控**：
    *   使用Kibana自带的Stack Monitoring功能监控集群健康、性能指标、资源使用情况。
    *   可集成Prometheus、Grafana等第三方监控系统进行更全面的监控和告警。

5.  **集群规模与分片设计**：
    *   根据数据量、写入/查询并发量合理规划节点数量和硬件配置。
    *   索引的分片数 (number_of_shards) 一旦设定后通常不可更改 (除非reindex)。副本数 (number_of_replicas) 可动态调整。合理设计分片策略对性能和扩展性至关重要。

6.  **升级**：
    *   关注Elastic官方的发布和升级指南。小版本升级通常较平滑，大版本升级需仔细规划和测试。

7.  **日志管理**：
    *   定期检查ES和Kibana的日志，及时发现和处理问题。
    *   配置日志滚动策略，避免日志文件过大。

## 总结

恭喜您！通过本教程，您已经成功部署了一个包含3个节点的Elasticsearch 7.6.2集群，并配置了Kibana作为可视化管理工具，以及IK中文分词器以支持中文搜索。您还掌握了集群的基本管理（启停、状态检查）和验证方法。

这仅仅是Elasticsearch强大功能的冰山一角。作为一名后端开发者，您现在可以基于这个集群，进一步探索索引创建、数据导入、复杂查询、聚合分析、以及与Spring Boot等后端框架的集成。将Elasticsearch应用于日志分析、应用性能监控(APM)、全文搜索等场景，无疑将极大提升您的Web应用能力和用户体验。

请记住，持续学习和实践是精通任何技术的关键。建议您深入阅读Elasticsearch官方文档，并根据您的实际需求不断优化和调整集群配置。

祝您在Elasticsearch的探索之旅中一切顺利！

---