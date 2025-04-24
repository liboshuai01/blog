---
title: 常用Hadoop HDFS命令详解与实用指南
tags:
  - Linux
  - Hadoop
  - Hdfs
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250424134712836.png'
toc: true
abbrlink: 28d329f5
date: 2023-12-25 13:18:48
---

Hadoop HDFS（分布式文件系统）通过命令行工具提供了丰富的文件管理功能。使用 `hadoop fs` 命令可以方便地操作 HDFS 文件和目录，类似于 Linux 系统的文件管理命令。本文将围绕常用的 HDFS 操作命令进行详细讲解，并用实例展示实用技巧，帮助你快速上手。

<!-- more -->

## 基本命令结构理解

所有 HDFS 文件操作命令均以 `hadoop fs` 开头，例如：
- `hadoop fs -ls`  用于列出目录内容
- `hadoop fs -mkdir` 用于创建目录
- `hadoop fs -cat` 用于查看文件内容

这个结构类似于 Linux 命令，`hadoop fs` 相当于执行文件系统的操作入口。

## 查看目录与文件内容

### 列出目录下的文件和目录

```bash
hadoop fs -ls /path/to/directory
```

此命令显示指定路径（如果省略，则默认当前用户的 HDFS 根目录）下所有文件和目录的详细信息，包括权限、拥有者、大小和修改时间。

### 读取并输出文件内容

```bash
hadoop fs -cat /path/to/file.txt
```

将指定文件的内容直接输出到终端，方便实时查看文件数据。

## 目录和文件管理

### 创建目录

```bash
hadoop fs -mkdir /path/to/new_directory
```

在指定路径下创建新目录。可以多级创建，但需要确保父目录存在。

### 上传本地文件至 HDFS

```bash
hadoop fs -put /local/path/file.txt /hdfs/path/
```

将本地文件上传到 HDFS 指定目录。命令也支持上传文件夹。

### 下载文件或目录到本地

```bash
hadoop fs -get /hdfs/path/file_or_dir /local/path/
```

将 HDFS 上的文件或目录下载到本地指定位置。

### 合并文件并下载（getmerge）

Hadoop 作业输出通常是分布式多个 `part-` 文件，`-getmerge` 命令可将这些文件合并成为一个本地文件。

```bash
hadoop fs -getmerge /hdfs/path/dir /local/path/merged_file.txt
```

所有 `part-` 文件会依次合并输出到本地文件中。

#### 合并时文件间加空行

加上 `-nl` 参数，合并的各个文件内容之间会自动插入换行，方便区分：

```bash
hadoop fs -getmerge -nl /hdfs/path/dir /local/path/merged_file.txt
```

## 查看文件和目录信息

### 统计文件和子目录数量

```bash
hadoop fs -count /hdfs/path/dir
```

显示目录下的目录数量、文件数量和文件总大小（单位为字节），有助于了解存储结构和容量。

### 查看文件或目录大小

```bash
hadoop fs -du /hdfs/path/file_or_dir
```

显示指定路径下所有文件的大小列表，若指定为单个文件，则显示该文件大小。可以用来判断文件占用空间情况。

## 删除文件和目录

### 删除文件

```bash
hadoop fs -rm /hdfs/path/file
```

删除指定文件，操作谨慎，通常会移动至回收站（Trash）。

### 删除目录

```bash
hadoop fs -rmr /hdfs/path/directory
```

递归删除目录及其所有内容。

> **注意：** 删除操作不可逆，建议确认路径正确，误删后可尝试从 Trash 恢复。

## 其他实用提示

- 输入 `hadoop` 命令可以查看 Hadoop 的安装路径及环境配置信息。
- 多使用 `hadoop fs -help` 查看所有支持的文件系统命令及其用法。
- 结合 Shell 脚本自动化操作可提升工作效率，如定时清理旧数据、批量上传下载等。

---

通过熟练掌握以上 HDFS 命令，您可以高效地管理大数据存储，便利地完成文件上传、查看、下载和删除等核心操作。希望本指南能够帮助您快速启动 Hadoop 文件系统的应用场景，提升开发和运维效率。