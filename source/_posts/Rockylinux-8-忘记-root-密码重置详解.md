---
title: Rockylinux 8 忘记 root 密码重置详解
abbrlink: dde3d9e5
date: 2025-02-10 23:17:03
tags:
  - Linux
  - Rockylinux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262320140.png'
toc: true
---

在使用 Rockylinux 8 系统过程中，如果忘记了 root 密码，会导致无法进入系统进行操作。本文详尽介绍通过单用户模式修改 root 密码的步骤，并保留了操作截图，帮助您轻松恢复管理员权限。

<!-- more -->

---

## 进入 GRUB 编辑模式

重启进入系统时，在 GRUB 选单页面按上下方向键，阻止系统自动进入操作系统，接着按下 **e** 键进入启动项编辑页面。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262317978.png)

---

## 修改启动参数，进入单用户命令行

将启动参数中的 `ro`（只读模式）替换为：

```
rw init=/sysroot/bin/sh
```

之后，按下 **Ctrl + X** 继续启动。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262317987.png)

---

## 切换根目录环境

系统启动会进入 shell，执行以下命令切换根目录环境：

```bash
chroot /sysroot
```

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262317736.png)


随后，设置环境语言为英语方便观察提示：

```bash
LANG=en
```

---

## 修改 root 密码

执行命令修改 root 密码：

```bash
passwd root
```

系统会提示输入新密码，请输入并确认。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262317771.png)

---

## 处理 SELinux 重新标记

为了避免 SELinux 策略限制，执行命令创建重标记标志文件：

```bash
touch /.autorelabel
```

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262318799.png)


---

## 退出重启并登录

按 **Ctrl + D** 或输入 `exit` 命令退出 chroot 环境，回到初始 shell，执行重启：

```bash
reboot
```

系统重新启动后，使用新设置的 root 密码登录即可。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262318817.png)

---

## 总结

本文采用 GRUB 编辑启动参数的方式进入单用户模式，通过 chroot 环境修改 root 密码，并处理了 SELinux 重标记问题，确保系统安全恢复登录权限。此法适用于 Rockylinux 8 及类似的 RHEL 8 系统发行版。

如有任何疑问或遇到特殊情况，欢迎留言交流。祝系统管理顺利！