---
title: 优化Centos关闭SELinux/Swap及资源限制调整
abbrlink: d040eabd
date: 2023-11-24 18:06:08
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250422180722180.png'
toc: true
---

在部署高性能、高并发的后端服务时，操作系统的合理配置是必不可少的环节。合理关闭不必要的安全机制、释放资源占用、提升系统的文件描述符和进程数限制，同时调整内核相关参数，可以极大提升系统性能和稳定性。

本文将系统且详细地介绍在 **CentOS** 系统中：

- 如何关闭 SELinux
- 关闭 Swap 分区
- 调整最大打开文件数与进程数限制
- 调整虚拟内存映射限制 (`vm.max_map_count`)
- 禁用透明大页（Transparent HugePage）

帮助您快速完成生产环境下的系统基础优化。

<!-- more -->

---

## 关闭 SELinux

SELinux （Security-Enhanced Linux）是 Linux 内核的安全模块，提供强制访问控制。默认情况下，CentOS 开启了
SELinux，但对于某些高性能环境或兼容性需求，可能需要关闭它。

### 查看 SELinux 当前状态

```bash
getenforce
```

- `Enforcing`：强制执行，SELinux 开启
- `Permissive`：宽容模式，记录而不拦截（等同于临时关闭）
- `Disabled`：永久关闭

### 临时关闭 SELinux

该方法只在当前运行时有效，重启后恢复。

```bash
sudo setenforce 0
```

确认关闭：

```bash
getenforce  # 应显示 Permissive
```

### 永久关闭 SELinux

1. 编辑配置文件：

    ```bash
    sudo vi /etc/selinux/config
    ```

2. 找到 `SELINUX=` 行，修改为：

    ```conf
    SELINUX=disabled
    ```

3. 保存后重启系统使其生效：

    ```bash
    sudo reboot
    ```

重启后执行 `getenforce`，应显示 `Disabled`。

---

## 关闭 Swap 分区

Swap 是 Linux 系统用来扩展物理内存的虚拟内存空间，但对于需要高性能和低延迟的应用，如数据库和 Elasticsearch，建议关闭 Swap。

### 临时关闭 Swap

即时生效，重启后失效：

```bash
sudo swapoff -a
```

### 永久关闭 Swap

1. 编辑挂载配置：

    ```bash
    sudo vi /etc/fstab
    ```

2. 找到所有关于 `swap` 的行，注释掉（在行首添加 `#`）：

    ```conf
    #/dev/mapper/centos-swap swap                    swap    defaults        0 0
    ```

3. 保存文件后，重启系统，或使用 `swapoff -a` 立即关闭。

### 验证 Swap 状态

```bash
free -h
```

Swap 一列应显示为 `0`，表示已关闭。

---

## 调整最大打开文件数和最大进程数限制

高并发服务经常遇到文件描述符（file descriptors）和进程数限制不足的问题。默认 CentOS 系统通常较低（1024），需要调高。

### 当前限制查询

```bash
ulimit -n        # 查看最大打开文件数
ulimit -u        # 查看最大进程数
```

### 永久修改方法

1. 修改 `/etc/security/limits.conf`

   ```bash
   sudo vi /etc/security/limits.conf
   ```

   文件末尾添加如下内容，对所有用户生效：

   ```conf
   *       soft    nofile  65536
   *       hard    nofile  65536
   *       soft    nproc   65536
   *       hard    nproc   65536
   ```

    - `nofile`：最大打开文件数
    - `nproc`：最大可创建进程数
    - `soft`、`hard`：软限制与硬限制，软限制可被用户临时调低；硬限制是最大不可逾越值。

2. 查看并修改 `/etc/security/limits.d/` 下配置

   某些系统会单独维护进程数限制文件，如 `20-nproc.conf`。也同步设置：

   ```bash
   sudo vi /etc/security/limits.d/20-nproc.conf
   ```

   确保内容类似：

   ```conf
   *       soft    nofile  65536
   *       hard    nofile  65536
   *       soft    nproc   65536
   *       hard    nproc   65536
   ```

### 使更改生效

- 登出当前用户，再重新登录。
- 对于服务，重启对应进程。
- 不需要重启整个系统。

### 验证限制

```bash
ulimit -Sn  # 软限制文件数
ulimit -Hn  # 硬限制文件数
ulimit -Su  # 软限制进程数
ulimit -Hu  # 硬限制进程数

ulimit -a   # 查看所有限制
```

---

## 调整 `vm.max_map_count`（虚拟内存最大映射数）

`vm.max_map_count` 是 Linux 内核参数，限制一个进程可以拥有的虚拟内存映射区域的数量。许多高性能数据库和搜索引擎（如
Elasticsearch）要求此值较高。

### 临时修改

```bash
sudo sysctl -w vm.max_map_count=2000000
```

该设置立即生效，但重启失效。

### 永久修改

1. 编辑配置文件：

   ```bash
   sudo vi /etc/sysctl.conf
   ```

2. 添加以下行：

   ```conf
   vm.max_map_count=2000000
   ```

3. 使配置生效：

   ```bash
   sudo sysctl -p
   ```

### 确认修改

```bash
sysctl vm.max_map_count
# 输出：vm.max_map_count = 2000000
```

---

## 禁用透明大页（Transparent HugePage）

透明大页是 Linux 内核的内存管理特性，用于自动管理大页内存提高性能。但对于数据库和 JVM 堆内存，透明大页可能带来性能反而下降和延迟波动。

### 临时禁用

```bash
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag
```

### 永久禁用（修改 GRUB 启动参数）

1. 编辑 grub 默认配置文件：

   ```bash
   sudo vi /etc/default/grub
   ```

2. 找到 `GRUB_CMDLINE_LINUX` 行，追加参数：

   ```conf
   GRUB_CMDLINE_LINUX="... transparent_hugepage=never"
   ```

   确保引号内参数间有空格分隔。

3. 更新 grub 配置：

   ```bash
   sudo grub2-mkconfig -o /boot/grub2/grub.cfg
   ```

4. 重启系统：

   ```bash
   sudo reboot
   ```

### 验证禁用状态

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

输出应包含：

```
always madvise [never]
always madvise [never]
```

表示已禁用。

---

## 推荐工具与命令汇总

| 操作                  | 命令或路径                                                                  |
|---------------------|------------------------------------------------------------------------|
| 查看 SELinux 状态       | `getenforce`                                                           |
| 临时关闭 SELinux        | `sudo setenforce 0`                                                    |
| 永久关闭 SELinux 配置     | 编辑 `/etc/selinux/config`                                               |
| 临时关闭 Swap           | `sudo swapoff -a`                                                      |
| 永久关闭 Swap           | 编辑 `/etc/fstab` 注释 `swap` 行                                            |
| 修改文件描述符与进程限制        | 编辑 `/etc/security/limits.conf` 和 `/etc/security/limits.d/*.conf`       |
| 修改 vm.max_map_count | `sudo sysctl -w vm.max_map_count=2000000` <br> 永久写入 `/etc/sysctl.conf` |
| 禁用透明大页临时            | 写入 `/sys/kernel/mm/transparent_hugepage/enabled` 和 `defrag`            |
| 禁用透明大页永久（GRUB）      | 编辑 `/etc/default/grub` 并执行 `grub2-mkconfig`                            |

---

## 总结

经过以上配置和调优，您的 CentOS 系统将：

- 关闭可能影响性能的 SELinux 和 Swap
- 大幅提升文件描述符与进程限制，满足高并发需求
- 调高内核内存映射参数，优化数据库与搜索引擎性能
- 禁用透明大页，避免内存管理带来的延迟和性能抖动

这些基础配置通常是生产环境部署前的必备步骤，能够为后端服务的稳定运行和高效响应打下坚实基础。

如果您是使用 Java 或 Elasticsearch 类服务的开发者，强烈建议参考本文内容对系统进行优化！