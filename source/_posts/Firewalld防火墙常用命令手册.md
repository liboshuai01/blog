---
title: Firewalld防火墙常用命令手册
tags:
  - Linux
  - 防火墙
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130240764.png'
toc: true
abbrlink: 7cdf467c
date: 2022-07-01 09:59:00
---

这篇博文详细介绍了 firewalld 防火墙的常用命令及操作方法。首先，文中介绍了如何查看防火墙状态、刷新配置以及启动、停止和设置防火墙服务的开机自启。

接着，文章重点解析了新增和移除开放端口的命令使用，包括针对特定IP和端口范围的 rich-rule 配置，最后还说明了如何列出当前开放的端口和规则设置。整个内容适合于初学者掌握基本防火墙管理技巧。

<!-- more -->

查看防火墙状态
---

```
firewall-cmd --state
```

刷新防火墙
---

```
firewall-cmd --reload
```

启动/停止 firewall
---

```
systemctl start firewalld.service
systemctl stop firewalld.service
```

启用/禁止 firewall 开机启动
---

```
systemctl enable firewalld.service 
systemctl disable firewalld.service 
```

新增开放端口
---

```
firewall-cmd --zone=public --add-port=端口号或端口范围/tcp --permanent

# 示例
firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port port="1-65535" protocol="tcp" accept'
```

移除开放端口
---

```
firewall-cmd --zone=public --remove-port=端口号或端口范围/tcp --permanent

# 示例
firewall-cmd --permanent --zone=public --remove-rich-rule='rule family="ipv4" source address="10.0.0.0/24" port port="1-65535" protocol="tcp" accept'
```

查看开放的端口
---

```
firewall-cmd --zone=public --list-ports

firewall-cmd --list-rich-rules --zone=public
```