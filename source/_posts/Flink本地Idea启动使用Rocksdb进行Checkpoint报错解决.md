---
title: Flink本地Idea启动使用Rocksdb进行Checkpoint报错解决
tags:
  - Flink
  - Idea
  - RocksDB
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261339889.png'
toc: true
abbrlink: d541c343
date: 2024-12-19 13:37:05
---

在使用 Flink 与 RocksDB 进行状态管理时，可能会遇到如下异常报错：

```
Caused by: org.rocksdb.RocksDBException: Failed to create a NewWriteableFile: C:\Users\Administrator\AppData\Local\Temp\flink-io-xxxxxx\job_xxx\chk-2.tmp/MANIFEST-000004: 系统找不到指定的路径
    at org.rocksdb.Checkpoint.createCheckpoint(Native Method)
    at org.rocksdb.Checkpoint.createCheckpoint(Checkpoint.java:51)
    at org.apache.flink.contrib.streaming.state.snapshot.RocksIncrementalSnapshotStrategy.takeDBNativeCheckpoint(RocksIncrementalSnapshotStrategy.java:292)
    ...
```

错误信息提示 RocksDB 在创建写入文件时失败，原因是系统找不到指定的路径。该问题通常由 Flink 默认临时目录的路径不可写或路径不合法引起。

<!-- more -->

## 原因分析

Flink 在执行 RocksDB 状态快照时，会在系统临时目录（通常由 `java.io.tmpdir` 指定）下创建临时文件和文件夹。如果操作系统对该临时路径权限不足，或者路径名称包含非英文字符、特殊符号，便可能导致 RocksDB 无法正确创建文件，进而抛出异常。

特别是在 Windows 环境下，默认的临时路径可能包含中文用户名或者特殊字符，造成路径访问失败。

## 解决方法

### 配置自定义临时目录

为了避免使用系统默认临时目录，可自行定义一个全英文且权限充足的临时目录，例如：

```
C:\Me\flink\temp
```

### 修改 JVM 启动参数

将 `java.io.tmpdir` 指向自定义目录，确保 RocksDB 在快照过程中使用该路径。 修改启动参数为：

```bash
-Djava.io.tmpdir=C:/Me/flink/temp
```

注意：路径中使用正斜杠 `/`，而非反斜杠 `\`，避免转义或路径解析错误。

### 高版本 Flink 配置调整

如果您使用的是 Flink 新版本（例如 1.15 及以上），需要额外配置进程工作目录参数：

```bash
-Dprocess.working-dir=C:/Me/flink/temp
```

该参数用于控制 Flink 子进程的工作目录，保证 RocksDB 及相关组件操作路径正确。

### 在 IDE 中添加参数配置示例

以下为 IntelliJ IDEA 中 JVM 启动参数配置示例截图，方便快速定位参数填写位置：

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261337940.png)

确保参数配置生效后，重新启动 Flink 作业。

## 验证效果

启动 Flink 作业后，进入配置的临时目录，即可看到 RocksDB 快照产生的文件和目录：

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261337119.png)

此时快照生成正常，说明问题已解决。

## 额外建议

- 确保自定义目录磁盘空间充足，避免快照过程中因存储不足导致失败。
- 尽量使用英文路径，避免 Windows 下特殊字符引起的兼容性问题。
- 给予临时目录充足的读写权限，特别是在生产环境中。
- 关注 Flink 和 RocksDB 版本更新日志，了解相关参数和行为的调整。
- 如果问题仍未解决，可尝试清空临时文件夹后重试，或者变更其他目录路径。

## 总结

遇到 Flink 使用 RocksDB 产生快照时报 “Failed to create a NewWriteableFile” 错误时，最常见根因是临时目录权限或路径问题。通过自定义临时路径并配置 JVM 启动参数，可有效规避此类异常，确保 Flink 状态快照顺利完成。

欢迎分享、点赞和关注，以获取更多 Flink 与 RocksDB 实践经验。祝您开发顺利！