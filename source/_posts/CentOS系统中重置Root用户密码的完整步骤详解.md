---
title: CentOS系统中重置Root用户密码的完整步骤详解
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425144815357.png'
toc: true
abbrlink: bbe1dc4
date: 2024-05-15 14:45:42
---

在实际运维过程中，由于各种原因，可能会忘记或丢失 CentOS 系统中 root 用户的登录密码。传统方法需要借助安装介质或者复杂的恢复工具，这对于部分用户来说较为繁琐。本文将详细介绍一种简便且高效的方式，通过修改启动参数进入单用户模式，从而重新设置 root 密码，恢复对系统的完全控制权限。整个过程无需额外工具，适用于 CentOS 系统的常见版本，步骤清晰易操作，非常适合系统管理员和运维工程师参考学习。

<!-- more -->

## 具体步骤

1.  重启系统
2.  在这个选择界面，按`e`\
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425144409924.png)
3.  找到如下位置，插入`init=/bin/sh`。\
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425144409940.png)
4.  填写完成后按`Ctrl+x`引导启动
5.  输入`mount -o remount, rw /`\
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425144409981.png)
6.  重置密码\
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425144409937.png)\
    出现以下为重置成功\
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425144409928.png)
7.  执行`touch /.autorelabel`
8.  退出`exec /sbin/init`
9.  输入你的新密码即可登录，到此重置密码完成！

## 结语

重置 root 密码是确保系统安全和稳定运行的重要手段之一。通过本文介绍的方法，用户可以快速、有效地解决因密码遗忘导致的无法登录问题。需要注意的是，进行密码重置操作时应保持系统操作的谨慎，避免误操作引发其他系统异常。同时，为防止安全隐患，重置密码后建议及时更新相关安全策略和访问权限。希望本文能帮助您顺利恢复对 CentOS 系统的管理权限，提升系统运维效率。