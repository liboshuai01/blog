---
title: Linux回收站trash-cli详解
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261322213.png'
toc: true
abbrlink: b00127c5
date: 2024-11-11 13:21:22
---

在 Linux 系统中，误删文件时常令人心痛，却缺少 Windows 那样直观的回收站机制。为避免“手误”导致文件的永久丢失，`trash-cli` 作为一个简单有效的命令行回收站工具，能帮助我们安全地“删除”文件，同时便于恢复和管理。本文将带你全面了解 `trash-cli` 的安装、使用及自动清理方案，实现 Linux 文件管理的安全升级。

<!-- more -->

---

## trash-cli 简介

`trash-cli` 是一个基于 FreeDesktop.org 回收站标准的命令行工具，允许用户将删除的文件移入回收站，而非立即从磁盘清除。这样可以最大限度地降低误删带来的损失，支持文件恢复、查看回收站列表、单独删除回收站文件以及清空回收站等操作。

它兼容 Linux 各大主流发行版，尤其适合喜欢在命令行下操作的用户及服务器环境。

---

## 安装指南

### 通过包管理器快速安装

#### CentOS 系统

```bash
yum install -y python-setuptools.noarch
easy_install trash-cli
```

> 说明：`easy_install` 是 Python 的包安装工具，这样安装相对简单，无需额外依赖。

#### Arch Linux / Manjaro

```bash
sudo pacman -S trash-cli
```

此方式适合 Arch 类发行版，安装快捷，版本更新较及时。

### 源码安装适配旧系统

对于内核版本较旧的环境（例如 CentOS 6/7）：

```bash
wget https://github.com/andreafrancia/trash-cli/archive/master.zip
unzip master.zip
cd trash-cli-master
python setup.py install
```

> 若安装后命令无法识别，执行以下命令建立软连接：

```bash
ln -s /usr/local/python/bin* /usr/bin/
```

此操作可保证命令在系统路径下被识别。

---

## 常用命令及功能介绍

安装完成后，系统中会出现以下主要命令：

```
trash-put         # 把文件或目录移入回收站
trash-list        # 查看回收站中的文件列表
trash-restore     # 选择文件从回收站还原到原路径
trash-rm          # 彻底删除回收站中的指定文件
trash-empty       # 清空回收站中的所有文件
```

---

## rm 命令别名及使用建议

为了习惯性使用 `rm` 时不致直接删除文件，部分用户会将 `rm` 命令统一映射为 `trash-put`，示例：

```bash
echo "alias rm='trash-put'" >> /etc/bashrc
source /etc/bashrc
```

**但官方并不推荐此做法**，因为：

- `rm` 命令的参数语义与 `trash-put` 存在差异，如 `rm -r` 用于递归删除目录，而 `trash-put` 不需该参数。
- 误用别名可能导致脚本执行异常。

官方建议替代方案是给 `rm` 命令设置警示：

```bash
alias rm='echo "请谨慎使用 rm 命令！建议使用 trash-put"'
```

需要确定执行原始 `rm`，可以通过前置反斜杠：

```bash
\rm 文件名
```

使用别名只影响交互式 shell，脚本中的 `rm` 不受干扰。

---

## 使用示例演示

假设当前目录中有以下文件和文件夹：

```bash
touch a b c
mkdir 1 2 3
ls
# 输出: 1 2 3 a b c
```

删除所有文件和目录：

```bash
rm -rf ./*
ls
# 输出: (空)
```

这时文件实际上被移入回收站，通过 `trash-list` 查看：

```bash
trash-list
# 2018-10-30 16:21:00 /当前目录/1
# 2018-10-30 16:21:00 /当前目录/2
# 2018-10-30 16:21:00 /当前目录/3
# 2018-10-30 16:21:00 /当前目录/a
# 2018-10-30 16:21:00 /当前目录/b
# 2018-10-30 16:21:00 /当前目录/c
```

回收站文件实际保存在：

```
$HOME/.local/share/Trash/files/
```

可进入该目录核实。

### 文件还原操作

执行：

```bash
trash-restore
```

系统会列出回收站文件及索引，例如：

```
0 2018-10-30 16:21:00 /当前目录/1
1 2018-10-30 16:21:00 /当前目录/2
...
```

输入对应数字确认恢复，文件将回到原路径。

### 删除回收站中的单个文件

```bash
trash-rm 2  # 删除索引为 2 的文件
trash-list  # 查看剩余文件列表
```

### 一键清空回收站

```bash
trash-empty
trash-list  # 此时应无任何文件显示
```

---

## 普通用户也可使用

普通用户使用时，回收站目录在自己的家目录下，操作方式一致，无需 root 权限。

---

## 自动清理回收站文件方案 —— autotrash

长时间不清理回收站会占用大量磁盘空间。为此，`autotrash` 能自动根据设定规则清理过期回收文件。

### 安装方法

```bash
wget https://github.com/bneijt/autotrash/archive/v0.2.1.zip
unzip v0.2.1.zip
cd autotrash-0.2.1
python setup.py install
```

确认安装成功：

```bash
whereis autotrash
# 输出示例: /usr/bin/autotrash
```

### 使用示例

- 删除当前用户回收站中，超过 30 天的文件：

```bash
autotrash -d 30
```

- 删除所有用户回收站中过期文件（需 root 权限）：

```bash
autotrash -td 30
```

- 当回收站剩余空间小于 1GB 时，自动删除超过 30 天的文件：

```bash
autotrash --max-free 1024 -d 30
```

### 定时任务自动执行

将清理规则加入定时任务，实现自动周期清理：

```bash
echo "# 自动清理回收站过期文件" >> /var/spool/cron/root
echo "@daily /usr/bin/autotrash -d 7" >> /var/spool/cron/root
```

或者为普通用户:

```bash
echo "# 自动清理回收站过期文件" >> /var/spool/cron/用户名
echo "@daily /usr/bin/autotrash -d 7" >> /var/spool/cron/用户名
```

每天执行，清理回收站中保留超过 7 天的文件。

---

## 总结

`trash-cli` 让 Linux 系统增添了类似于图形界面的回收站功能，更加安全、容错性强；而 `autotrash` 自动清理机制，则避免了回收站文件长期堆积带来的磁盘压力。结合使用这两个工具，可以大幅提升 Linux 环境下文件管理的安全性与便捷性，推荐部署于个人工作站及关键服务器上为你的数据保驾护航。

---

未来，您还可以结合更丰富的脚本、日志监控，实现更加智能的文件管理与备份策略。希望本文对您了解和应用 `trash-cli` 及相关工具有所帮助！祝使用愉快。