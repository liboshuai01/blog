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

   ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77720b53b6c3451c8209f46d8c151d6b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=291&h=63&s=4885&e=png&b=191919)

2. 重启`prometheus`服务
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
`

3. 验证`prometheus`中是否可以查看到`flink`相关的信息

   ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/586d1fb044f940fb8a62b3d1a2bbff54~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3024&h=1888&s=365165&e=png&b=ffffff)

## 配置`Grafana`

> 面板ID为: `14911`

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9251ffe913e541139330d7506fa2e6bd~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3024&h=1888&s=780437&e=png&b=1b1c21)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b29f1c78368f4881bd3bb89c2c010aa9~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3024&h=1888&s=300693&e=png&b=16171c)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72a5d6b839dc445ebdc8715e96ce3425~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3024&h=1888&s=408283&e=png&b=17181d)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8847d363b194dd787b17c7bb563fd3e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=3024&h=1888&s=508826&e=png&b=17191d)

> 注意：想要看到图表及数据，需要至少运行一个 flink 任务

## 结语

通过本文的指南，你已经可以成功地将 Flink 任务与 Prometheus 和 Grafana 监控系统集成。这不仅提高了问题发现和解决的效率，也为系统的性能优化提供了数据支持。务必确保按照文中步骤准确配置，以便最大程度地发挥监控系统的功效。后续可以根据实际监控数据，继续调整和优化 Flink 配置，进一步提升系统的稳定性和处理能力。