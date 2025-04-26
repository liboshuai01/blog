---
title: Redis集群离线滚动升级全流程解析，确保业务零中断
tags:
  - Linux
  - Redis
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261136169.png'
toc: true
abbrlink: ecca5eff
date: 2024-10-16 11:34:22
---

近期质保部安全扫描发现我们现有 Redis 集群存在安全漏洞，需尽快升级至最新版本进行修复。鉴于 Redis
是关键缓存组件，升级过程中需要保证业务系统持续稳定运行，避免任何停机或性能波动。本次升级目标为：

- 将 Redis 从 7.4.1 升级到 7.4.2
- 采用离线滚动升级方案，确保系统业务零中断
- 对升级流程进行标准化总结，方便团队复用

<!-- more -->

## 现有Redis集群架构概览

### 升级前节点角色分布

| IP        | 端口   | 角色                      |
|-----------|------|-------------------------|
| 10.0.0.87 | 6479 | 主节点                     |
| 10.0.0.87 | 6480 | 从节点（主节点：10.0.0.82:6479） |
| 10.0.0.9  | 6479 | 主节点                     |
| 10.0.0.9  | 6480 | 从节点（主节点：10.0.0.87:6479） |
| 10.0.0.82 | 6479 | 主节点                     |
| 10.0.0.82 | 6480 | 从节点（主节点：10.0.0.9:6479）  |

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261134073.png)

集群中每台机器均承载了一个主节点和一个对应从节点，数据分布均衡且互为备份。

### 升级后期望的节点角色调整

| IP        | 端口   | 角色                       |
|-----------|------|--------------------------|
| 10.0.0.87 | 6479 | 从节点（新主节点：10.0.0.9:6480）  |
| 10.0.0.87 | 6480 | 主节点                      |
| 10.0.0.9  | 6479 | 从节点（新主节点：10.0.0.82:6480） |
| 10.0.0.9  | 6480 | 主节点                      |
| 10.0.0.82 | 6479 | 从节点（新主节点：10.0.0.87:6480） |
| 10.0.0.82 | 6480 | 主节点                      |

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261134273.png)

升级完成后，主从角色将发生相互切换，切换后的从节点对应旧主节点，确保集群健康且版本统一。

## 滚动升级方案设计思路

- **编译生成新版本 Redis 二进制文件（bin）**，替换旧版本保证统一
- **先升级所有从节点**，避免因主节点停机导致服务不可用，减小风险
- **逐一完成主节点升级**，先将其对应的从节点提升为主节点，完成主节点切换后进行停服升级，维护业务连贯性
- **升级过程中充分监控主从同步状态和节点健康情况**，确保数据一致性及线上业务稳定
- **升级完成后执行全面业务验证**，确认集群版本、节点状态和业务功能未受影响

该方案注重**平滑上下线**和**主从逻辑切换**，使集群升级无缝过度。

## 详细升级步骤

请根据实际 Redis 安装目录替换文中命令中的路径 `/home/lbs/`。

### 编译新版本 Redis 二进制文件

1. 下载并解压源码包（示例使用7.4.1版本，升级时请替换相应版本）

    ```bash
    wget https://github.com/redis/redis/archive/refs/tags/7.4.1.tar.gz
    tar -zxvf 7.4.1.tar.gz
    cd redis-7.4.1
    ```

2. 编译并安装，生成新的 `bin` 目录

    ```bash
    make && make install PREFIX=/home/lbs/temp/redis
    ```

*说明：此目录下的 `bin` 包含新版本 redis-server 和 redis-cli 等可执行文件。*

### 查看集群当前节点信息和状态

方便了解现有集群分布及节点健康，执行：

```bash
/home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6479 -a admin123456 cluster nodes
```

确保所有节点正常且状态无异常。

### 替换旧版 Redis 二进制文件

在每台集群机器重复以下操作：

```bash
mv -f /home/lbs/software/redis/bin /home/lbs/software/redis/bin_old
mkdir -p /home/lbs/software/redis/bin
cp -rf /home/lbs/temp/redis/bin/* /home/lbs/software/redis/bin
```

替换完成后，`bin` 下的 `redis-server` 和 `redis-cli` 即为新版本。

### 升级所有从节点（二次上线）

逐台执行：

1. 停止当前从节点

    ```bash
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6480 -a admin123456 shutdown
    # 若shutdown受限，使用kill命令强制终止进程
    ```

2. 使用新版本二进制启动从节点

    ```bash
    /home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6480/conf/redis-6480.conf
    ```

3. 验证启动版本和节点状态

    ```bash
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6480 -a admin123456 info | grep version
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6480 -a admin123456 cluster nodes
    ```

4. 等待主从数据同步完成，确认 `repl_offset` 相匹配

    ```bash
    /home/lbs/software/redis/bin/redis-cli -h localhost -p 6480 -a admin123456 info replication | grep 'repl_offset'
    ```

*注：数据量大时此同步过程可能较久，耐心等待。*

### 逐个升级主节点及其从节点角色切换

1. 将主节点对应的从节点通过 `CLUSTER FAILOVER` 提升为主节点

    ```bash
    /home/lbs/software/redis/bin/redis-cli -h [从节点IP] -p [从节点端口] -a admin123456 CLUSTER FAILOVER
    ```

2. 确认角色切换成功，并节点状态健康

    ```bash
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6479 -a admin123456 cluster nodes
    /home/lbs/software/redis/bin/redis-cli -h localhost -p 6479 -a admin123456 info replication | grep 'repl_offset'
    ```

3. 停止旧主节点，使用新版本二进制启动

    ```bash
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6479 -a admin123456 shutdown
    /home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6479/conf/redis-6479.conf
    ```

4. 再次确认启动版本和节点上线

    ```bash
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6479 -a admin123456 info | grep version
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6479 -a admin123456 cluster nodes
    ```

5. 按照以上流程，逐个节点完成主从角色切换及升级。

### 集群升级后检测与业务验证

- 查看集群节点分布是否全部为新版本且状态正常

    ```bash
    /home/lbs/software/redis/bin/redis-cli -c -h localhost -p 6480 -a admin123456 cluster nodes
    ```

- 通过业务系统接口或缓存操作确认功能正常响应，无异常报错。

- 监控线上业务指标数据及日志，确保无性能或稳定性退化。

## 升级过程常见注意事项

- **数据同步监控不可忽略**：切勿在主从同步完成前启动下一个节点避免数据不一致风险。
- **维护期间避免误操作**：严格遵循步骤，防止错停主从节点造成集群不可用。
- **备份关键数据**：升级前务必对 Redis 数据及配置文件进行完整备份。
- **合理安排时间窗口**：根据数据量和业务压力挑选业务低峰时段执行操作。
- **权限和命令安全控制**：保证升级操作账号拥有集群管理权限且命令通过安全渠道执行。

## 总结

本次 Redis 离线滚动升级方案核心在于：

- 灵活利用主从角色切换降低停机风险
- 分阶段替换可执行文件实现平滑升级
- 保证集群数据一致性和业务连续性

通过科学规划和严谨执行，确保了Redis集群安全漏洞修复的同时，业务系统持续稳定运行。这套方案同样适用于其他关键缓存和分布式缓存系统的升级改造。

欢迎在评论区留言交流您的升级经验和疑问！