---
title: Kettle性能测试报告
tags:
  - Linux
  - Kettle
  - Elt
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261003635.png'
toc: true
abbrlink: 73fd6885
date: 2024-08-01 10:00:57
---

在大数据处理和业务系统集成中，ETL（Extract-Transform-Load）工具起着至关重要的作用。Kettle 作为一款开源且功能强大的 ETL 工具，被广泛应用于数据抽取和加载的场景中。为了更好地理解 Kettle 在不同数据源和目标环境下的性能表现，本报告针对 两个典型场景展开详细测试。通过对比 MySQL 到 MySQL 及 Excel 到 MySQL 的数据迁移流程，我们分析了 Kettle 在处理千万级数据时的运行时间和系统资源消耗。这些测试结果为实际项目中的性能优化提供了参考依据，同时也揭示了参数配置对性能的影响，为后续优化提供了思路。

<!-- more -->

> 由于作业中不同的参数设置会极大的影响运行耗时与资源消耗，而此报告的参数设置还有其他优化空间，估测试结果仅供参考。

# 测试环境

*   Centos7（32G 内存，8 核 CPU）
*   MySQL8.2.0
*   Kettle9.2

# 场景一

## 测试场景

一千万数据从 A 机器的 MySQL 库抽取到 B 机器的 MySQL 库中。

## 测试数据

### A 机 MySQL

*   数据条数：一千万
*   单表大小：652 MB

```sql
CREATE TABLE `users` (
  `id` int NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `email` varchar(100) DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261004142.png)

### B 机 MySQL

*   数据条数：一千万
*   单表大小：652 MB

```sql
CREATE TABLE `users` (
  `id` int NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `email` varchar(100) DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261004748.png)

## 测试结果

### 运行耗时

**总耗时 6 分 11 秒。**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005596.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005968.png)

### 资源消耗

**内存占用 5.8g，CPU 占用 1 核的百分之 61.5。**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005272.png)

# 场景二

## 测试场景

一千万数据从 A 机器的 Excel 文件中抽取后写入 A 机器的 MySQL 数据库。

## 测试数据

### Excel

*   数据条数：一千万
*   文件大小：239 MB

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005853.png)

### MySQL

*   数据条数：一千万
*   单表大小：652 MB

```sql
CREATE TABLE `users` (
  `id` int NOT NULL,
  `name` varchar(100) DEFAULT NULL,
  `email` varchar(100) DEFAULT NULL,
  `age` int DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

## 测试结果

### 运行耗时

**总耗时 7 分钟。**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005322.png)

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005998.png)

### 资源消耗

**占用 5.8g 内存，CPU 不到 1 核的百分之 80。**

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261005445.png)

## 结语

本次性能测试展现了 Kettle 在大规模数据处理场景中的稳定性和资源利用情况。尽管测试环境和参数尚有进一步优化空间，测试结果已为实际应用提供了宝贵的性能数据支撑。在未来的工作中，可以基于此次测试结果，进一步调整 Kettle 参数配置，提升数据处理效率，降低系统资源占用。此外，结合具体业务需求，合理选择数据源和目标存储也有助于优化整体 ETL 流程。希望本报告能够为大数据开发和运维人员在运用 Kettle 进行数据集成时提供切实可行的参考与指导。