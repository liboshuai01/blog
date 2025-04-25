---
title: CentOS7与RockyLinux系统时间同步指南
tags:
  - Linux
  - Centos
  - RockyLinux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110909439.png'
toc: true
abbrlink: 669fb41a
date: 2024-04-15 11:08:07
---

系统时间的准确性对于服务器的稳定运行至关重要。无论是日志记录的准确性、安全认证的时效性，还是数据库事务与分布式系统的协调，时间均扮演关键角色。本文将全面介绍如何在 CentOS 7 和 Rocky Linux 9.4 上配置时间同步方案，重点对比传统工具 `ntpdate` 与现代工具 `chrony` 的使用方法和最佳实践，助您打造高效、稳定的服务器环境。

<!-- more -->

---

### CentOS 7 上使用 ntpdate 进行时间同步

在 CentOS 7 中，`ntpdate` 是一个常用但渐显过时的时间同步工具，适合快速手动同步系统时间，简洁易用但不适合长期自动同步。

#### 安装 ntpdate

```bash
sudo yum install -y ntpdate
```

#### 手动同步系统时间

下面命令使用阿里云的 NTP 服务器作为时间源，立即同步系统时间：

```bash
sudo ntpdate ntp.aliyun.com
```

#### 同步硬件时钟

为了确保硬件时钟（RTC）与系统时钟一致，建议执行：

```bash
sudo hwclock --systohc
```

这有助于避免重启后时间偏差。

#### 配置定时同步任务（可选）

虽然不推荐频繁使用 `ntpdate` 自动同步，但若确实需要定时同步，可通过 `cron` 设置：

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/sbin/ntpdate ntp.aliyun.com >/dev/null 2>&1") | crontab -
```

此处设置为每5分钟同步一次，建议频率不要太高，避免系统负载飙升。

#### 注意事项

- `ntpdate` 适合短期手动校时，无法智能管理时间漂移。
- 频繁运行可能会对网络和系统造成额外负担。
- CentOS 7 默认自带 `ntpd` 服务，推荐使用 `ntpd` 或升级使用 `chrony`。

---

### Rocky Linux 9.4 上使用 chrony 进行时间同步

`chrony` 作为现代时间同步方案，相较于传统 `ntp` 更高效稳定，尤其适合现代内核和复杂网络环境。

#### 安装 chrony

Rocky Linux 9.4 使用 `dnf` 管理软件包：

```bash
sudo dnf install -y chrony
```

#### 配置 NTP 服务器

编辑主配置文件 `/etc/chrony.conf`，指定阿里云 NTP 服务器：

```bash
sudo vi /etc/chrony.conf
```

找到并修改（或添加）以下内容：

```conf
server ntp4.aliyun.com iburst
```

推荐使用 `iburst` 参数以加速首次同步。

可以注释掉默认的其它 `server` 行，保证只使用指定的阿里云时间源。

#### 启动并开启开机自启

```bash
sudo systemctl enable --now chronyd
```

这条命令同时启动服务并设置随系统启动。

#### 立即校准系统时间

执行：

```bash
sudo chronyc makestep
```

令时间立即校正，无需等待自动同步。

#### 查看同步状态

通过以下命令监控时间同步情况：

```bash
chronyc tracking
chronyc sources
```

输出示例：

```
Reference ID    : D4E1A8B6 (ntp4.aliyun.com)
Stratum         : 3
Ref time (UTC)  : Sat Apr 27 12:34:56 2024
System time     : 0.000123 seconds fast of NTP time
...

210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* ntp4.aliyun.com               3   6    17    64    +0.000123
```

其中，`^*` 表示当前稳定的同步源。

#### 其他建议

- 定期查看 `chronyc tracking`，确保漂移和延迟在合理范围。
- 对于严格时间要求的系统，可结合硬件时钟校验。
- 安装 `chrony` 后，建议禁用或卸载旧的 NTP 服务，避免冲突。

---

### 时间同步工具对比与优化建议

| 特性                | ntpdate                    | chrony                      |
|---------------------|----------------------------|-----------------------------|
| 适用场景            | 手动快速校时               | 自动持续同步，长时间稳定    |
| 资源消耗            | 较高，频繁调用会增加负载   | 低，智能监控与调整         |
| 网络适应能力        | 较差                       | 出色，适应断网与休眠等情况 |
| 配置复杂度          | 简单                       | 略复杂，功能丰富            |
| 支持系统版本        | 旧版Linux（如CentOS 7）   | 现代Linux发行版（如Rocky 9.4）|

基于上述考虑，推荐未来所有服务器逐步迁移至 `chrony`，以获得持续、高效且稳定的时间同步体验。

---

### 总结

准确的系统时间对于服务器运维和应用表现都是根本保障。本文展示了两款主要工具在不同系统上的应用方法：

- **CentOS 7**：`ntpdate` 适合单次紧急校时，但不适合长时间自动同步。
- **Rocky Linux 9.4**：`chrony` 是首选工具，支持智能、持续且稳定的时间同步。

#### 最佳实践

- 利用 `chrony` 进行自动时间管理，避免手动频繁调整。
- 定期检查同步状态，及时发现并处理异常。
- 保持硬件时钟与系统时钟一致，减少误差积累。
- 确保系统时间与业务需求保持一致，防止因时间偏差造成系统异常。

通过科学的时间同步管理，确保服务器持续稳定运行，提升整体服务的可靠性和安全性。

---

如果您有更深入的时间同步经验或遇到任何问题，欢迎在评论区分享讨论。祝您运维顺利！