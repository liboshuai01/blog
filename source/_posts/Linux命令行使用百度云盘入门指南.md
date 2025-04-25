---
title: Linux命令行使用百度云盘入门指南
tags:
  - Linux
  - Centos
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504242148212.png'
toc: true
abbrlink: 4262bd57
date: 2024-03-24 21:47:10
---

在服务器或无图形界面的环境下使用百度网盘，传统的网页方式不够便捷。本文介绍一款基于 Linux Shell 操作风格的百度网盘命令行客户端——BaiduPCS-Go，帮助你在 CentOS 系统中实现文件的快速管理与下载，操作简单，功能丰富，适合开发者和运维人员使用。

<!-- more -->

---

## 安装指南

确保使用具备 root 权限的账户执行以下操作，方便将程序放置于系统路径中：

```bash
# 下载 BaiduPCS-Go 压缩包
wget https://github.com/qjfoidnh/BaiduPCS-Go/releases/download/v3.9.5/BaiduPCS-Go-v3.9.5-linux-amd64.zip

# 解压文件
unzip BaiduPCS-Go-v3.9.5-linux-amd64.zip && cd BaiduPCS-Go-v3.9.5-linux-amd64

# 移动并赋予执行权限，将可执行文件放入系统路径
mv BaiduPCS-Go /usr/local/bin/baidu-pcs && chmod +x /usr/local/bin/baidu-pcs
```

执行后，在终端输入 `baidu-pcs` 可以查看是否安装成功及支持的命令帮助。

---

## 登录百度云盘

百度网盘登录采用 BDUSS 认证令牌进行身份验证，省去了二次密码输入，自动登录稳定高效。

1. 打开百度网盘网页版，使用 Chrome 浏览器按 `F12` 打开开发者工具。

2. 进入“应用程序（Application）”栏目下的“Cookie”，找到包含“BDUSS”字段的条目。

3. 复制 BDUSS 对应的 `Value` 内容，这就是你的登录令牌。

4. 在终端执行登录命令：

```bash
baidu-pcs login -bduss=<你的BDUSS值>
```

登录成功后，即可通过命令管理你的百度网盘文件。

> **提示**：为了安全，避免将 BDUSS 令牌暴露给他人。

---

## 常用命令操作

BaiduPCS-Go 提供了丰富且直观的命令集合，以下是部分常用命令示例：

### 查看当前路径

```bash
baidu-pcs pwd
```

### 列出当前目录内容

```bash
baidu-pcs ls
```

### 切换到指定目录

```bash
baidu-pcs cd <目录名称>
```

### 以树状图形式显示目录结构

```bash
baidu-pcs tree <目录路径>
```

### 下载文件或目录

支持一次下载多个文件或目录：

```bash
baidu-pcs d <网盘文件或目录路径1> <文件或目录路径2> ...
```

### 上传本地文件或目录

批量上传本地资源到指定网盘目录：

```bash
baidu-pcs u <本地文件或目录路径1> <路径2> ... <目标网盘目录>
```

---

## 进阶应用和注意事项

- **断点续传**：BaiduPCS-Go 支持断点续传，适合下载大文件。

- **自动化脚本集成**：结合 Linux shell 脚本，可以实现自动定时备份、批量下载等功能。

- **多账户切换**：可以通过多次登录管理不同百度账户，适合团队使用。

- **安全性**：BDUSS 为重要凭证，建议妥善保管，避免泄露。

- **帮助命令**：

```bash
baidu-pcs help
```

查看完整命令列表与使用说明。

---

## 参考资料

- 官方项目地址及源码：[BaiduPCS-Go GitHub](https://github.com/qjfoidnh/BaiduPCS-Go)

---

通过这款工具，你能够轻松实现在纯命令行环境下对百度网盘文件的浏览、下载和上传，大大提高了服务器环境中文件管理的便捷性。希望本文的介绍能帮助你快速上手，打造效率更高的工作流程。