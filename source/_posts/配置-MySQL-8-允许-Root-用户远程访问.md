---
title: 配置 MySQL 8 允许 Root 用户远程访问
tags:
  - Linux
  - mysql
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250414141557514.png'
toc: true
abbrlink: 69b2f848
date: 2023-11-12 01:01:01
---

作为后端开发人员，我们经常需要在本地连接到远程服务器上的 MySQL 数据库进行开发、测试或维护。
然而，MySQL 8 默认情况下禁止 root 用户从远程主机连接，这给我们带来了不便。
本文将详细介绍如何配置 MySQL 8，使其允许 root 用户从任何 IP 地址进行连接，这在很多开发场景中非常方便。

<!-- more -->

> **风险提示：** 允许 root 用户从远程访问是一个安全风险，请谨慎操作。建议在生产环境中使用更安全的身份验证方式，如创建具有特定权限的用户并限制其访问。

前提条件
---

*   已安装 MySQL 8
*   具有 root 用户权限

步骤详解
---

1. 登录 MySQL 服务器

   首先，通过本地终端以 root 用户身份登录到 MySQL 服务器:

    ```bash
    mysql -u root -p
    ```

   输入 root 用户的密码以完成登录。

2. 更新 `user` 表

   接下来，我们需要更新 `mysql.user` 表，允许 root 用户从任何主机连接。请执行以下 SQL 命令：

    ```mysql
    use mysql;
    update user set host='%' where user ='root';
    ```

   这条命令将 `mysql.user` 表中 `user` 列为 'root' 的所有行的 `host` 列更新为 '%'。`%` 作为通配符，表示允许来自任何主机的连接。

3. 授予所有权限

   为了确保 root 用户拥有足够的权限进行操作，我们需要授予其所有权限。执行以下 SQL 命令：

    ```mysql
    GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
    ```

   这条命令授予了 `root` 用户在所有数据库 (第一个 `*`) 和所有表 (第二个 `*`) 上的所有权限。`WITH GRANT OPTION` 允许 root 用户将这些权限授予其他用户。

4. 刷新权限

   最后，我们需要刷新 MySQL 的权限缓存，使修改生效。执行以下 SQL 命令：

    ```mysql
    FLUSH PRIVILEGES;
    ```

至此，您已经成功配置了 MySQL 8 允许 root 用户从任何 IP 地址进行远程连接。

验证连接
---

您可以使用 MySQL 客户端（如 MySQL Workbench、Navicat 或命令行工具）从您的本地机器尝试连接到远程 MySQL 服务器，并使用 root 用户和密码进行验证。

结语
---

本文介绍了如何配置 MySQL 8 允许 root 用户从远程访问。虽然这种方法在开发和测试环境中可能很方便，但在生产环境中需要谨慎使用，并采取额外的安全措施。希望本文对您有所帮助！