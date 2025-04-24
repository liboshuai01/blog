---
title: Linux命令行使用阿里云盘入门指南
tags:
  - Linux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242151778.png'
toc: true
abbrlink: 8cf8c083
date: 2024-03-25 21:50:40
---

阿里云盘CLI是一款基于命令行的阿里云盘客户端，模拟了Linux Shell中的文件处理命令，极大地方便了Linux用户进行云端文件管理。它不仅支持交互式命令操作，还具备JavaScript插件扩展能力，支持同步备份及批量下载相册等实用功能，是阿里云盘用户的高效工具。

<!-- more -->

## 安装步骤

在CentOS系统中，可以通过官方提供的RPM仓库快速安装阿里云盘CLI客户端。具体步骤如下：

```bash
sudo curl -fsSL http://file.tickstep.com/rpm/aliyunpan/aliyunpan.repo | sudo tee /etc/yum.repos.d/tickstep-aliyunpan.repo > /dev/null
sudo yum install aliyunpan -y
```

这将添加阿里云盘的yum仓库，并安装最新版本的aliyunpan命令行工具。

## 使用指南

安装成功后，即可开始使用阿里云盘CLI。以下是常用操作示例：

### 启动交互模式

```bash
aliyunpan
```

启动后进入交互式命令界面，方便用户连续执行文件管理命令。

### 登录阿里云盘

在交互界面输入：

```bash
login
```

系统会输出一个授权链接，默认会打开浏览器访问该链接。按照提示完成账号授权后，扫描二维码即可完成登录。

### 切换云盘空间

阿里云盘CLI支持切换不同的云盘空间，包括备份盘和资源库：

```bash
drive
```

执行后会列出可用云盘驱动器，选择相应编号切换当前工作盘。

### 文件管理基础命令

- **查看当前目录**

  ```bash
  pwd
  ```

- **列出当前目录文件**

  ```bash
  ls
  ```

- **切换目录**

  ```bash
  cd <目录名>
  ```

### 文件传输命令

- **下载文件**

  ```bash
  d <文件名>
  ```

- **上传文件**

  ```bash
  u <本地路径> <阿里云盘目标路径>
  ```

例如，将本地文件`/home/user/picture.jpg`上传到云盘根目录：

```bash
u /home/user/picture.jpg /
```

## 进阶功能及资源

阿里云盘CLI支持丰富的扩展功能，像JavaScript插件开发、同步备份设置以及相册批量下载等，极大提升文件管理效率。

推荐深入阅读官方文档及开源项目主页，获取最新命令用法和插件开发指南：

[阿里云盘CLI GitHub仓库](https://github.com/tickstep/aliyunpan)

## 总结

通过阿里云盘命令行客户端，Linux用户能够在熟悉的命令行环境下轻松管理阿里云盘文件，提升工作效率。本文介绍了安装流程及核心命令，帮助你快速上手阿里云盘CLI，欢迎尝试并根据需求定制更多自动化脚本体验。

---

希望这篇指南能帮助你在CentOS系统上高效使用阿里云盘命令行工具！