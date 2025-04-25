---
title: 关闭CentOS上MySQL的二进制日志功能详解
tags:
  - Linux
  - Centos
  - Mysql
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141343106.png'
toc: true
abbrlink: af0f2a22
date: 2024-04-22 14:12:25
---

MySQL 的二进制日志（binlog）用于记录数据库的所有更改操作，是主从复制和数据恢复的重要机制。但在某些场景下，比如不需要复制或想减轻系统负担时，可以选择关闭 binlog 功能。本文针对 CentOS 操作系统上的 MySQL 8.0 及以上版本，详细说明如何关闭二进制日志，并介绍相关注意事项。

<!-- more -->

## 定位 MySQL 配置文件  
MySQL 的主要配置文件通常存放在以下位置：

- `/etc/my.cnf`
- `/etc/my.cnf.d/` 目录下的多个配置文件，如 `server.cnf`

你可以通过以下命令查找所有相关配置文件：

```bash
sudo find /etc -name my.cnf
sudo find /etc -name "*.cnf"
```

建议优先检查 `/etc/my.cnf`，如果配置较复杂，可能会分散到 `/etc/my.cnf.d/` 下的多个文件。

## 修改配置关闭二进制日志  
使用你熟悉的文本编辑器打开目标配置文件。若使用 `vi`，例如：

```bash
sudo vi /etc/my.cnf
```
或者

```bash
sudo vi /etc/my.cnf.d/server.cnf
```

在文件中找到 `[mysqld]` 段，该段为 MySQL 服务器的主要配置节。

如果存在类似 `log_bin` 的配置项，如：

```ini
log_bin=mysql-bin
```
请将其注释掉或删除。

在 MySQL 8.0 中，官方支持使用配置项 `disable_log_bin` 来显式关闭二进制日志功能，推荐添加以下内容：

```ini
[mysqld]
disable_log_bin
```

这是一个布尔类型选项，只需声明，表示禁用 binlog。

注意，旧版本可能使用：

```ini
skip-log-bin
```

但 MySQL 8.0 以后更推荐 `disable_log_bin`。

## 应用配置修改  
保存并关闭配置文件后，需要重启 MySQL 服务使配置生效。

在 CentOS 7 及以上版本，通常使用 systemd 管理服务：

```bash
sudo systemctl restart mysqld
```

若系统使用传统 init 脚本：

```bash
sudo service mysqld restart
```

## 确认二进制日志状态  
登录 MySQL 客户端验证配置：

```bash
mysql -u root -p
```

执行查询：

```sql
SHOW VARIABLES LIKE 'log_bin';
```

正常关闭时，返回结果应显示：

```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | OFF   |
+---------------+-------+
```

表示二进制日志功能已被禁用。

额外检查：

- 查看 `SHOW BINARY LOGS;` 应报错或无日志列表。
- 检查 MySQL 错误日志文件中是否有与 binlog 相关的启动信息。

## 操作注意事项

- **备份配置**：修改前建议备份现有配置文件，防止误操作导致服务异常。
- **权限要求**：确保使用的帐号对配置文件和 MySQL 服务拥有编辑和重启权限。
- **业务影响**：禁用 binlog 后，主从复制将无法正常工作，备份和恢复策略可能需要调整。
- **服务中断**：重启 MySQL 服务会造成短暂断线，请在业务低峰期执行。
- **配置冲突**：确保没有其他配置文件启用了 binlog，避免配置互相覆盖。
- **日志依赖**：某些监控和审计工具可能依赖 binlog，关闭前确认影响范围。

## 总结  
关闭 MySQL 二进制日志对于降低系统负荷和简化部署有帮助，尤其是在无复制需求的单机环境下。通过编辑 `/etc/my.cnf` 或相关配置文件中的 `[mysqld]` 段，添加 `disable_log_bin` 选项，并重启服务，即可有效关闭 binlog 功能。务必结合业务需求和备份策略，确保安全与稳定平衡。

这样操作后，你的 MySQL 服务将不再产生二进制日志文件，节省存储空间，提高性能，但也失去了基于 binlog 的复制和点时间恢复能力，请谨慎使用。