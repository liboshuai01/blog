---
title: Kafka集群接入Prometheus监控预警系统
tags:
   - Linux
   - Kafka
   - Prometheus
categories:
   - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142636317.png'
toc: true
abbrlink: d83294c3
date: 2024-04-24 14:25:23
---

本文将指导您如何配置 Kafka 集群的监控系统，使用 Prometheus 和 Grafana 进行数据可视化。该过程包括安装 Kafka exporter、配置 Prometheus 和 Grafana，以便有效监控 Kafka 集群的性能和状态。

<!-- more -->

## 环境准备

> 一定要注意各个组件的版本对应问题，博文在这方面吃了大亏，浪费了很多时间。
>
> 特别是 Grafana 版本 与 kafka-exporter 版本不能过高，不然不兼容，图标数据不展示

- Prometheus版本: `v2.36.2`
- Grafana版本: `9.1.2`
- Kafka版本: `2.5.0`
- Kafka-exporter版本: `v1.0.0`

> 环境准备可参考我的博文：
>
> [# Docker 安装 Prometheus 和 Grafana](https://juejin.cn/post/7360629255258046475)
>
> [# 最详细且简单的kafka集群搭建教程](https://juejin.cn/post/7259208528530833465)

## 安装`kafka-exporter`

1. 进入到安装`kafka exporter`的目录

2. 创建`docker-compose.yaml`文件，内容如下：
   > 注意：`--kafka.server`的内容替换为自己集群的IP端口

    ```yaml
    version: '3.8'
    
    services:
      kafka-exporter:
        image: danielqsj/kafka-exporter:v1.0.0
        container_name: kafka-exporter
        ports:
          - "9308:9308"
        command: ["--kafka.server=10.0.0.87:9092","--kafka.server=10.0.0.81:9092","--kafka.server=10.0.0.82:9092"]
        restart: always
        volumes:
          - /etc/localtime:/etc/localtime
        deploy:
          resources:
            limits:
              memory: 4g
    ```

3. 执行命令启动`kafka-exporter`服务

    ```shell
    docker-compose up -d
    ```

## 配置`Prometheus`

1. 编辑`prometheus.yaml`配置文件，`scrape_configs`块下新增如下内容：

   > 注意：
   > 1. 缩进格式保持一致
   > 2. `targets`里面的内容替换为自己`kafka-exporter`的IP端口

    ```yaml
        - job_name: "kafka_exporter"
          static_configs:
            - targets: ["10.0.0.87:9308"]
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112942.png)

2. 重启`prometheus`服务
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
   > `

3. 验证`prometheus`中是否可以查看到`kafka`相关的信息


    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260116534.png)

## 配置`Grafana`

> 面板ID号为: `7589`

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112996.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112983.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260112967.png)


> 如果面板中没有数据图表展示，则很可能是因为`kafka`集群中没有数据被消费，没有产生消费偏移量。需要往`kafka`中多发送一些数据，并进行消费一段时间即可。

## 结语

至此，您已成功设置了 Kafka 集群的监控系统。您可以通过 Grafana 仪表板实时查看 Kafka 的性能指标，并据此进行调优和维护。这将极大地帮助您确保 Kafka 集群的健康和高效运行。