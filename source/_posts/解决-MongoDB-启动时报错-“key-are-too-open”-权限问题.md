---
title: 解决 MongoDB 启动时报错 “key are too open” 权限问题
tags:
  - Linux
  - MongoDB
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262324888.png'
toc: true
abbrlink: 283ee8cf
date: 2025-02-10 23:23:24
---

在使用 MongoDB 部署副本集或分片集群时，常常需要配置 keyFile 来实现节点之间的认证。但在启动 MongoDB 服务时，遇到如下报错：

```
2016-03-18T10:39:58.889+0800 I CONTROL [main] ***** SERVER RESTARTED *****
2016-03-18T10:39:58.918+0800 I ACCESS  [main] permissions on /usr/local/mongodb/conf/keyFilers0.key are too open
```

该错误提示表明 MongoDB 对 keyFile 文件的权限要求比较严格，当前文件权限设置不符合安全要求，导致服务启动失败。

<!-- more -->

## MongoDB keyFile 权限要求详解

MongoDB 使用 keyFile 进行内部节点之间的认证，因此对该文件的安全性有较高要求：

- keyFile 必须只对文件所有者可读写，其他用户无任何权限。
- 权限过于开放（例如对所有用户可读）会被 MongoDB 拒绝，提示权限过大。

这是为了防止密钥文件泄漏，保证集群安全，避免未授权访问。

## 造成错误的常见权限设置

错误权限示例：

```shell
-rw-r--r--  1 mongo mongo  100 Mar 18 10:38 keyFilers0.key
```

即权限为 `644`，这意味着除了所有者，其他用户也能读取该文件，安全性不足。

## 解决方法

只需将 keyFile 的权限设置为仅所有者可读写即可：

```bash
chmod 600 /usr/local/mongodb/conf/keyFilers0.key
```

设置后的权限示例：

```shell
-rw-------  1 mongo mongo  100 Mar 18 10:38 keyFilers0.key
```

这样配置后，重启 MongoDB 服务即可正常启动，无权限报错。

## 注意事项

- keyFile 文件的拥有者建议设置为运行 mongod 进程的用户，如 `mongo` 用户。
- 文件权限不能过宽，否则会导致集群认证失败。
- 每次修改 keyFile 后，确保权限正确才能避免启动异常。
- 生产环境尽量避免以 root 用户运行 mongod，降低安全风险。

## 总结

MongoDB keyFile 权限设置必须严格，通常设为 `600`，即仅所有者可读写。过于开放的权限会导致启动时提示 “key are too open” 错误。只需使用 `chmod 600` 命令修改权限，即可解决此问题，保证集群的安全认证机制正常工作。

如果您在配置 MongoDB 副本集或分片集群时遇到类似报错，不妨优先检查并修正 keyFile 的权限设置。