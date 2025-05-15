---
title: Centos一键部署Redis集群实战指南
tags:
  - 环境搭建
  - Linux
  - Redis
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408113442272.png'
toc: true
abbrlink: a1366697
date: 2023-10-10 11:54:00
---

在高并发和大数据量应用场景中，单机Redis实例往往成为系统性能瓶颈。为了提升Redis的扩展性和可靠性，搭建Redis集群是常见且有效的解决方案。本文将结合CentOS 7环境，以普通用户权限为例，详细介绍如何从零开始搭建一个包含三主三从节点的Redis集群，帮助你实现稳定、高效的分布式缓存服务。

<!-- more -->
---

## 环境准备

**操作系统及依赖**
- 操作系统：CentOS 7
- Redis版本支持：7.0.12、7.4.1
- 用户权限：普通用户权限即可完成所有操作
- 必须安装`gcc`编译器
  > 执行 `gcc -v` 若提示 `-bash: gcc: command not found`，请先自行安装`gcc`。建议参考：[CentOS离线一键安装gcc（含安装包+脚本）](https://juejin.cn/post/7310143510102376457)。

**网络**
- 确保三台服务器之间端口通信互通（包括Redis端口）
- 关闭防火墙或放开Redis相关端口（如6379、6380）

---

## 集群规划与资源获取

### 集群结构规划

| 服务器 | IP地址     | Redis端口          |
|--------|------------|--------------------|
| node1  | 10.0.0.1   | 6379, 6380         |
| node2  | 10.0.0.2   | 6379, 6380         |
| node3  | 10.0.0.3   | 6379, 6380         |

本方案将在三台CentOS 7服务器上部署共6个Redis实例，构成3主3从架构。你也可以根据资源和需求调整节点数量及端口。

### 资源下载链接

- [Redis安装脚本包（蓝奏云）](https://liboshuai.lanzouv.com/iZyoJ2ccrydg)
- [Redis安装脚本包（百度云）](https://pan.baidu.com/s/1yey6vGQQMe-t66QfGGglow?pwd=usx9)

---

## Redis安装与配置

### 1. 上传安装包

将下载好的`redis_install.tar`压缩包分别上传至三台服务器的同一目录（例如：`/home/lbs/`）。

### 2. 运行安装脚本

SSH登录每台服务器，执行以下解压并安装命令（以Redis 7.4.1版本为例）：

```bash
tar -xvf redis_install.tar && cd redis_install
./install.sh 7.4.1 /home/lbs/software/redis 1gb admin123456 6379 6380
```

安装脚本参数说明：

| 参数       | 说明                             |
|------------|----------------------------------|
| 版本       | Redis版本，支持7.0.12或7.4.1     |
| 安装路径   | Redis安装目录                    |
| 内存限制   | 单实例最大内存，例如`1gb`        |
| 密码       | Redis连接密码                   |
| 端口       | 一台机器多个Redis实例的端口号   |

> 可执行 `./install.sh --help` 查看详细帮助。

---

## Redis集群创建

### 1. 确认网络连通与端口开放

请确保机器间6379和6380端口开放，且防火墙配置允许访问。

### 2. 创建集群

选择任意一台服务器，进入Redis安装目录，执行：

```bash
cd /home/lbs/software/redis
yes yes | ./bin/redis-cli --cluster create \
10.0.0.1:6379 10.0.0.1:6380 10.0.0.2:6379 10.0.0.2:6380 10.0.0.3:6379 10.0.0.3:6380 \
--cluster-replicas 1 -a admin123456
```

- `--cluster-replicas 1` 表示每个主节点有一个从节点
- `-a admin123456` 为前面设置的密码
- 执行后，redis-cli会自动进行槽位分配和主从节点配置分配

如果想手动指定主从关系，可参考官方文档或[Redis集群管理](https://blog.csdn.net/guotianqing/article/details/119778684)。

### 3. 验证集群状态

执行以下命令检查各节点状态：

```bash
cd /home/lbs/software/redis
./bin/redis-cli -h localhost -p 6379 -a admin123456 cluster nodes
```

返回结果应显示6个节点及其角色状态，如 `master`、`slave` 等。

---

## 集群管理脚本

为方便操作集群的启动、停止与状态查看，创建管理脚本：

```bash
cat > /home/lbs/software/redis/bin/redis-cluster.sh <<'EOF'
#!/bin/bash
case $1 in
"start")
  for i in 10.0.0.1 10.0.0.2 10.0.0.3
  do
    echo "------- $i Redis 启动中 -------"
    ssh "$i" "source /etc/profile; /home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6379/conf/redis-6379.conf; /home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6380/conf/redis-6380.conf"
  done
  ;;
"stop")
  for i in 10.0.0.1 10.0.0.2 10.0.0.3
  do
    echo "------- $i Redis 停止中 -------"
    ssh "$i" "/home/lbs/software/redis/bin/redis-cli -p 6379 -a admin123456 shutdown; /home/lbs/software/redis/bin/redis-cli -p 6380 -a admin123456 shutdown"
  done
  ;;
"status")
  for i in 10.0.0.1 10.0.0.2 10.0.0.3
  do
    echo "------- $i Redis 状态 -------"
    status=$(ssh "$i" "pgrep -fl redis-server || echo 'Redis未运行'")
    echo "$status"
  done
  ;;
*)
  echo "用法: $0 {start|stop|status}"
  ;;
esac
EOF

chmod +x /home/lbs/software/redis/bin/redis-cluster.sh
```

**使用示例：**

- 启动集群：`./redis-cluster.sh start`
- 停止集群：`./redis-cluster.sh stop`
- 查看状态：`./redis-cluster.sh status`

> 使用该脚本前确保三台机器之间已配置免密登录（SSH无密码登录）。

---

## 常见问题与进阶建议

### 1. Redis端口未开放或防火墙阻断
请确认防火墙关闭或允许对应端口访问，例如：

```bash
firewall-cmd --permanent --add-port=6379/tcp
firewall-cmd --permanent --add-port=6380/tcp
firewall-cmd --reload
```

### 2. 性能调优
根据实际资源适当调整`redis.conf`里的内存限制、持久化策略以及连接数等参数。

### 3. 监控与告警
建议结合`redis-exporter`与Prometheus、Grafana实现监控和告警。

### 4. 集群扩容和节点替换
可参考官方文档修改槽位分配，实现动态扩容和节点替换维护。

---

## 总结

通过以上步骤，你已经成功在三台CentOS 7服务器上使用普通用户权限搭建了一个三主三从的高可用Redis集群。该集群可以有效解决单实例承载瓶颈，提升系统稳定性和扩展能力。配合脚本管理，提高运维便捷性。如果你需要更细粒度的配置，也可以结合源码自行编译部署或选择手工配置方式。

如想学习更深入的Redis集群原理、自动化运维或与AI系统结合，请持续关注我的分享，欢迎交流！

---

## 参考链接

- [Redis官方集群教程](https://redis.io/docs/manual/scaling/)
- [Redis集群管理技巧分享](https://blog.csdn.net/guotianqing/article/details/119778684)
- [自动化部署及监控Redis集群实践](https://juejin.cn/post/7257734526138744889)

---

恭喜你！你已经具备搭建并管理生产级Redis集群的能力，祝你在项目中高效应用Redis，实现业务性能飞跃！