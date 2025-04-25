---
title: 如何直接在Doris导入Kafka中的数据
abbrlink: 98250a3
date: 2024-05-19 01:22:13
tags:
  - Doris
  - Kafka
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260125214.png'
toc: true
---

在现代大数据架构中，Kafka作为分布式消息系统广泛应用于流式数据传输，而Doris作为高性能的MPP数据库，支持实时数据分析与查询。本文将详细介绍如何利用Doris的Routine Load功能，直接从Kafka导入数据，实现实时数据同步，涵盖CSV和JSON格式的数据导入，并提供相关操作指令及调优建议。

<!-- more -->

## Doris中创建Routine Load导入任务

Doris支持通过`CREATE ROUTINE LOAD`命令创建常驻的Routine Load任务，持续从Kafka消费数据并导入指定表。该功能支持CSV和JSON两种主流数据格式，满足不同的数据结构需求。

### 导入CSV格式数据

#### Kafka示例数据

假设Kafka主题`test-routine-load-csv`中有如下CSV格式的记录：

```
1,Emily,25
2,Benjamin,35
3,Olivia,28
4,Alexander,60
5,Ava,17
6,William,69
7,Sophia,32
8,James,64
9,Emma,37
10,Liam,64
```

你可以使用如下命令查看数据（需安装Kafka客户端）：

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-routine-load-csv --from-beginning
```

#### 创建目标表结构

在Doris中创建表，接收导入数据：

```sql
CREATE TABLE testdb.test_streamload(
    user_id BIGINT NOT NULL COMMENT "用户 ID",
    name VARCHAR(20) COMMENT "用户姓名",
    age INT COMMENT "用户年龄"
)
DUPLICATE KEY(user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 10;
```

此表设计确保主键`user_id`唯一性，同时合理分布数据以支撑高并发读取。

#### 创建Routine Load导入任务

执行以下命令创建Routine Load任务，消费Kafka中的CSV数据：

```sql
CREATE ROUTINE LOAD testdb.example_routine_load_csv ON testdb.test_streamload
COLUMNS TERMINATED BY ","
COLUMNS(user_id, name, age)
FROM KAFKA(
    "kafka_broker_list" = "192.168.88.62:9092",
    "kafka_topic" = "test-routine-load-csv",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

关键参数说明：

- `COLUMNS TERMINATED BY ","` 指定CSV字段分割符。
- `kafka_broker_list` 指定Kafka集群地址。
- `kafka_topic` 指定消费的主题名。
- `property.kafka_default_offsets` 设置从头开始消费。

### 导入JSON格式数据

#### Kafka示例数据

Kafka主题`test-routine-load-json`中的示例JSON数据为：

```
{"user_id":1,"name":"Emily","age":25}
{"user_id":2,"name":"Benjamin","age":35}
{"user_id":3,"name":"Olivia","age":28}
{"user_id":4,"name":"Alexander","age":60}
{"user_id":5,"name":"Ava","age":17}
{"user_id":6,"name":"William","age":69}
{"user_id":7,"name":"Sophia","age":32}
{"user_id":8,"name":"James","age":64}
{"user_id":9,"name":"Emma","age":37}
{"user_id":10,"name":"Liam","age":64}
```

同样使用Kafka客户端读取数据：

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-routine-load-json --from-beginning
```

#### 创建目标表结构

表结构与CSV导入一致：

```sql
CREATE TABLE testdb.test_streamload(
    user_id BIGINT NOT NULL COMMENT "用户 ID",
    name VARCHAR(20) COMMENT "用户姓名",
    age INT COMMENT "用户年龄"
)
DUPLICATE KEY(user_id)
DISTRIBUTED BY HASH(user_id) BUCKETS 10;
```

#### 创建JSON格式的Routine Load任务

使用如下命令定义JSON数据导入：

```sql
CREATE ROUTINE LOAD testdb.example_routine_load_json ON testdb.test_streamload
COLUMNS(user_id, name, age)
PROPERTIES(
    "format" = "json",
    "jsonpaths" = '["$.user_id","$.name","$.age"]'
)
FROM KAFKA(
    "kafka_broker_list" = "192.168.88.62:9092",
    "kafka_topic" = "test-routine-load-json",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
```

重点说明：

- `format = json` 告诉Doris数据格式为JSON。
- `jsonpaths`精确定位每一字段在JSON中的路径。

## 导入作业的状态监控

### 查看Routine Load作业信息

执行命令查看作业的整体状态，包括导入目标表信息、当前导入任务数、运行状态以及错误信息：

```sql
SHOW ROUTINE LOAD FOR testdb.example_routine_load_csv\G
```

输出信息详细展示了数据接收量、任务执行状态以及错误日志等，便于快速定位问题。

### 查看Routine Load的子任务详情

可通过以下命令获取各个子任务的执行情况及分配的BE节点：

```sql
SHOW ROUTINE LOAD TASK WHERE jobname = 'example_routine_load_csv';
```

该命令展示TaskId、状态、执行时间和分配的BE节点等细节，方便做任务调度和监控。

## 管理Routine Load导入任务

### 暂停Routine Load任务

当需要临时停用导入任务时，可以执行：

```sql
PAUSE ROUTINE LOAD FOR testdb.example_routine_load_csv;
```

暂停后作业进入`PAUSED`状态，但任务可恢复。

### 恢复Routine Load任务

任务暂停后可随时用如下命令恢复：

```sql
RESUME ROUTINE LOAD FOR testdb.example_routine_load_csv;
```

恢复消费Kafka数据，将继续导入。

### 修改Routine Load配置

修改作业配置前需先暂停任务，示例如下：

```sql
PAUSE ROUTINE LOAD FOR testdb.example_routine_load_csv;

ALTER ROUTINE LOAD FOR testdb.example_routine_load_csv
PROPERTIES(
    "desired_concurrent_number" = "3"
)
FROM KAFKA(
    "kafka_broker_list" = "192.168.88.60:9092",
    "kafka_topic" = "test-topic"
);

RESUME ROUTINE LOAD FOR testdb.example_routine_load_csv;
```

以上示例调整了导入任务并行度并更新了Kafka Broker列表及Topic。

### 停止并删除导入任务

当不再需要Routine Load任务时，可彻底删除：

```sql
STOP ROUTINE LOAD FOR testdb.example_routine_load_csv;
```

删除后，该任务无法恢复，也无法查询。

## 优化建议与实操提醒

- **选择合适的并发数**：根据Kafka数据量和Doris集群资源合理调整`desired_concurrent_number`配置，避免资源竞争。
- **注意数据格式一致性**：Routine Load对数据格式敏感，切忌Kafka中数据格式混杂，否则会导致导入失败或数据错误。
- **合理配置错误允许度**：如数据质量不稳，可通过`max_error_number`和`max_filter_ratio`调整容错参数。
- **监控延迟与错误**：定期查看`SHOW ROUTINE LOAD`结果，监控`Lag`和错误日志，确保导入实时性与正确性。
- **结合Doris其他导入方式**：在数据量峰值或协同ETL阶段可结合Stream Load和Batch Load使用，灵活应对不同业务需求。

## 结语

Doris的Routine Load功能为Kafka数据实时入库提供了稳定高效的解决方案，且易于配置管理。掌握上述步骤和注意事项，将助力你实现流式数据的快速同步与分析。更多技术细节和功能介绍，欢迎查阅[Doris官方文档 - Routine Load数据导入](https://doris.apache.org/zh-CN/docs/dev/data-operate/import/routine-load-manual)。

---

通过本文的示范，您已经具备了在Doris环境中快速搭建Kafka数据同步的能力，祝您构建业务大数据实时分析平台顺利！