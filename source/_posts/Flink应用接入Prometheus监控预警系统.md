---
title: Flink应用接入Prometheus监控预警系统
tags:
   - Linux
   - Kafka
   - Prometheus
categories:
   - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425143129514.png'
toc: true
abbrlink: 9585b017
date: 2024-04-28 14:35:01
---

在现代数据处理和监控领域，Apache Flink 作为实时流处理框架，已经在众多企业和场景中得到广泛应用。为了更好地监控和管理 Flink 应用的性能与资源利用情况，将 Flink 集成至 Prometheus 和 Grafana 是一个非常有效的方法。本文详细介绍了如何搭建和配置这一监控系统，确保你可以实时跟踪和优化你的 Flink 应用。

<!-- more -->

## 环境准备

- `flink`任务启动并运行
- `Prometheus`搭建部署完毕，版本推荐`v2.36.2`
- `Grafan`搭建部署完毕，版本推荐`9.1.2`

> 环境准备可参考我的博文：
>
> [# Docker 安装 Prometheus 和 Grafana](https://juejin.cn/post/7360629255258046475)
>
> [# 最详细且简单的Flink集群搭建教程](https://juejin.cn/post/7288963700538753078)
>
> [# 最详细且简单的Hadoop高可用集群搭建教程](https://juejin.cn/post/7357888522333077556)

## 安装`pushgateway`

1. 首先进入到需要安装`pushgateway`的目录中

2. 创建`docker-compose.yaml`文件，内容如下：

    ```yaml
    version: "3.8"
    services:
      pushgateway:
        image: prom/pushgateway:v1.5.0
        container_name: pushgateway
        ports:
          - "9091:9091"
        restart: unless-stopped
    ```

3. 执行命令启动`pushgateway`服务
    ```shell
    docker-compose up -d
    ```

## 配置`flink`

> 如果你的 flink 应用会部署到多个节点，请所有的节点都同样执行下面的步骤

1. 编辑`flink-conf.yaml`文件，在文件最后追加以下内容：
   > host、port 替换为自己`pushgateway`的对应信息

    ```yaml
    metrics.reporter.promgateway.class: org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
    metrics.reporter.promgateway.host: 10.0.0.87
    metrics.reporter.promgateway.port: 9091
    metrics.reporter.promgateway.jobName: flink_pushgateway
    metrics.reporter.promgateway.randomJobNameSuffix: true
    metrics.reporter.promgateway.deleteOnShutdown: false
    metrics.reporter.promgateway.interval: 15 SECONDS
    ```
2. 拷贝`flink`目录下的`plugins/metrics-prometheus`jar包到`lib`目录下面
    ```shell
    # 进入 flink 目录
    cd /home/lbs/software/flink
    
    # 拷贝 jar 包
    cp plugins/metrics-prometheus/flink-metrics-prometheus-1.14.2.jar lib/
    ```

3. 重启`flink`集群，或者`yarn`集群（取决于你 flink 任务运行的环境）

## 配置`Prometheus`

1. 编辑`prometheus.yaml`配置文件，`scrape_configs`块下新增如下内容：

   > 注意：
   > 1. 缩进格式保持一致
   > 2. `pushgateway`下的`targets`里面的内容替换为自己`pushgateway`的IP端口

    ```yaml
        - job_name: "pushgateway"
          static_configs:
            - targets: ["10.0.0.87:9091"]
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260109939.png)

2. 重启`prometheus`服务
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
   > `

3. 验证`prometheus`中是否可以查看到`flink`相关的信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260109083.png)

## 配置`Grafana`

> 面板ID为: `14911`

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260109980.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260109034.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260109063.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260109162.png)

> 注意：想要看到图表及数据，需要至少运行一个 flink 任务

## 结语

通过本文的指南，你已经可以成功地将 Flink 任务与 Prometheus 和 Grafana 监控系统集成。这不仅提高了问题发现和解决的效率，也为系统的性能优化提供了数据支持。务必确保按照文中步骤准确配置，以便最大程度地发挥监控系统的功效。后续可以根据实际监控数据，继续调整和优化 Flink 配置，进一步提升系统的稳定性和处理能力。