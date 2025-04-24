---
title: Kafka常用命令手册
tags:
  - Linux
  - Kafka
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130247633.png'
toc: true
abbrlink: d88bccc7
date: 2023-11-10 11:24:02
---

Apache Kafka作为当前最流行的分布式消息队列之一，拥有一整套命令行工具来帮助维护和管理集群。在这篇博文中，我们将概览一些最常用的Kafka管理命令，为运维人员提供一个快速参考。

<!-- more -->

## 集群信息查询

- **查看在线的Broker列表**：

  ```shell
  ./bin/zookeeper-shell.sh localhost:2181 ls /brokers/ids
  ```

- **获取Broker详细信息**：

  ```shell
  ./bin/zookeeper-shell.sh localhost:2181 get /brokers/ids/<broker_id>
  ```

## 主题管理

- **创建主题**：

  ```shell
  ./bin/kafka-topics.sh --create --bootstrap-server <broker_list> --replication-factor <n> --partitions <m> --topic <topic_name>
  ```

- **查看主题列表**：

  ```shell
  ./bin/kafka-topics.sh --list --bootstrap-server <broker_list>
  ```

- **查看主题详情**：

  ```shell
  ./bin/kafka-topics.sh --describe --bootstrap-server <broker_list> --topic <topic_name>
  ```

- **修改主题配置**：

  ```shell
  ./bin/kafka-configs.sh --bootstrap-server <broker_list> --entity-type topics --entity-name <topic_name> --alter --add-config <config-name>=<config-value>
  ```

## 数据生产与消费

- **向主题发送数据**：

  ```shell
  ./bin/kafka-console-producer.sh --broker-list <broker_list> --topic <topic_name>
  ```

- **消费主题数据**：

  ```shell
  ./bin/kafka-console-consumer.sh --bootstrap-server <broker_list> --topic <topic_name> --from-beginning
  ```

## 消费者组管理

- **查看消费者组列表**：

  ```shell
  ./bin/kafka-consumer-groups.sh --bootstrap-server <broker_list> --list
  ```

- **查看消费者组详情**：

  ```shell
  ./bin/kafka-consumer-groups.sh --bootstrap-server <broker_list> --describe --group <group_name>
  ```

- **查看消费者组的消费积压**：

  ```shell
  ./bin/kafka-consumer-groups.sh --bootstrap-server <broker_list> --describe --group <group_name> --members --verbose
  ```

## 高级管理与维护

- **更改分区数量**：

  ```shell
  ./bin/kafka-topics.sh --alter --bootstrap-server <broker_list> --topic <topic_name> --partitions <new_number>
  ```

- **删除主题**：

  ```shell
  ./bin/kafka-topics.sh --delete --bootstrap-server <broker_list> --topic <topic_name>
  ```

## 结语

以上列出的命令是Kafka集群管理中的基础，涵盖了集群信息查询、主题管理、数据生产与消费以及消费者组管理。掌握这些命令对于日常的Kafka集群运维是非常有帮助的。当然，为了更高效地进行Kafka管理，建议使用图形化管理工具如CMAK或Confluent Control Center，这些工具提供了更为直观和强大的管理功能。最后，随着Kafka版本的迭代，建议定期查看官方文档来获取最新的命令和特性。
