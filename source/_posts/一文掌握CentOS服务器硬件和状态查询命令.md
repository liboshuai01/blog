---
title: 一文掌握CentOS服务器硬件和状态查询命令
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232339386.png'
toc: true
abbrlink: 8a663581
date: 2023-12-07 23:37:24
---

在管理 CentOS 服务器过程中，快速准确获取硬件配置和系统运行状态是运维人员的基础技能。合理使用系统自带和常用工具，能帮助快速定位问题、评估服务器性能和优化资源使用。

本文重点梳理日常服务器查看硬件信息及系统状态的核心命令，结构清晰、实用性强，适合中高级运维和开发人员参考。

<!-- more -->

---

## CPU 信息查看

CPU是服务器性能核心，准确获取 CPU 相关信息非常关键。

### 1.1 物理及逻辑CPU详情

```bash
lscpu
```

该命令简单明了展示：

- 架构类型（如 x86_64）
- 物理 CPU 数（socket 数）
- 每 CPU 核心数与线程数
- 主频及缓存大小（L1、L2、L3）
- 硬件虚拟化支持信息（VT-x 等）

示例：

```
Architecture:        x86_64
CPU(s):              24
Thread(s) per core:  2
Core(s) per socket:  6
Socket(s):           2
Model name:          Intel(R) Xeon(R) CPU E5-2620 v3 @ 2.40GHz
Virtualization:      VT-x
```

### 1.2 查看详细CPU参数

```bash
cat /proc/cpuinfo
```

从该文件获取更细粒度数据，通常用于脚本自动分析。配合`grep`使用效率更佳：

- 查看型号：
  ```bash
  grep "model name" /proc/cpuinfo | uniq
  ```

- 逻辑核心数：
  ```bash
  grep -c ^processor /proc/cpuinfo
  ```

- 物理 CPU 数：
  ```bash
  grep "physical id" /proc/cpuinfo | sort | uniq | wc -l
  ```

### 1.3 CPU 负载情况实时查看

```bash
cat /proc/loadavg
top
htop  # 需安装，交互式进程查看工具
```

`loadavg` 展示系统负载（1、5、15分钟平均），`top`和`htop`则显示进程占用 CPU 和内存情况。

---

## 内存资源监控

内存状态对服务器性能影响巨大，及时掌握内存使用情况防止系统发生交换。

### 2.1 目前内存利用情况

```bash
free -m
```

典型输出：

| 属性          | 说明             |
|---------------|------------------|
| total         | 总内存(MB)       |
| used          | 已用内存         |
| free          | 空闲内存         |
| shared        | 共享内存         |
| buff/cache    | 缓存和缓冲区大小 |
| available     | 可用内存         |

注意：`available` 是可供程序使用的真实可用内存，优于 `free`。

### 2.2 详细内存信息

```bash
cat /proc/meminfo
```

查看细节指标如缓存、交换区状态、内存碎片化等。

---

## 磁盘设备及健康检查

磁盘是数据关键承载，合理管理分区和及时检测磁盘健康至关重要。

### 3.1 磁盘与分区状态

- 查看分区挂载点及容量：
  ```bash
  df -h
  ```

- 查看物理磁盘及分区详情：
  ```bash
  fdisk -l
  lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
  ```

- 交换分区状态：
  ```bash
  swapon -s
  ```

### 3.2 硬盘健康检测（SMART）

安装 `smartmontools`：

```bash
yum install smartmontools -y
```

常用检测命令：

- 查看硬盘型号与基本信息：
  ```bash
  smartctl -i /dev/sda
  ```

- 查看健康状态（`PASSED` 表示正常）：
  ```bash
  smartctl -H /dev/sda
  ```

- 详细健康参数和错误日志：
  ```bash
  smartctl -A /dev/sda
  smartctl --all /dev/sda
  ```

*注意：硬盘若处于 RAID 环境，可能需带上对应控制器参数*，如：

```bash
smartctl -a -d megaraid,1 /dev/sda
```

### 3.3 软件RAID查看

```bash
cat /proc/mdstat
```

查看软件RAID状态及同步情况。

---

## 网络接口及流量状态

网络状态影响服务器数据传输质量，在服务器运行中需实时监控。

### 4.1 网络接口信息

- CentOS 6 及之前：

```bash
ifconfig
```

- CentOS 7 及以后推荐：

```bash
ip addr
ip link show
```

### 4.2 查看网卡PCI设备

```bash
lspci | grep -i ethernet
```

如 `lspci` 未安装：

```bash
yum install pciutils -y
```

### 4.3 网络带宽与端口监控

- 查看网卡驱动与速率：

```bash
ethtool eth0
```

- 路由表：

```bash
route -n
```

- 监听的网络端口及进程：

```bash
netstat -ntpl
lsof -i
```

---

## 系统用户及进程管理

了解系统运行用户及进程是维护服务器安全与资源分配基础。

### 5.1 查看当前用户列表

```bash
cut -d: -f1 /etc/passwd
```

### 5.2 查看登录用户信息

```bash
who
w
last
```

### 5.3 用户权限信息

```bash
id 用户名
```

### 5.4 进程管理

查看进程及资源占用：

```bash
ps -ef
ps aux --sort=-%cpu | head -n 10    # CPU使用最高的10个进程
top
```

通过进程名过滤：

```bash
ps -ef | grep 进程名
```

---

## 服务器硬件与固件信息

掌握服务器硬件概貌和厂商信息，有助于硬件维护与升级。

### 6.1 主板信息

```bash
dmidecode -t baseboard
```

或查看系统信息：

```bash
dmidecode | grep -A16 "System Information"
```

### 6.2 BIOS版本

```bash
dmidecode -t bios
```

---

## 结语

熟练应用以上命令是每位 Linux 系统管理员的必备技能。它们不仅提供硬件资源状况，还帮助及时发现潜在问题，保障服务器稳定高效运行。结合自动化脚本和监控方案，会让您的运维工作更具效率与可控性。

希望本文能作为您工作中的实用参考手册，帮助快速定位和解决问题。

---

**温馨提示：** 多数命令需使用 root 或 sudo 权限运行，请在安全环境下操作，确保系统安全和数据完整。
