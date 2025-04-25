---
title: Es集群接入Prometheus监控预警系统
tags:
   - Linux
   - Kafka
   - Prometheus
categories:
   - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425143129514.png'
toc: true
abbrlink: d6995e86
date: 2024-04-28 14:30:23
---

在当前的云计算和大数据时代，监控系统的健康和性能变得尤为重要。本文将详细介绍如何使用`ElasticSearch`作为数据存储后端，通过`Prometheus`和`Grafana`进行有效的监控和可视化，以确保您的服务可靠性和性能优化。

<!-- more -->

## 环境准备

- `ElasticSearch`集群搭建部署完毕，版本推荐`7.6.2`
- `Prometheus`搭建部署完毕，版本推荐`v2.36.2`
- `Grafan`搭建部署完毕，版本推荐`9.1.2`

> 环境准备可参考我的博文：
>
> [# Docker 安装 Prometheus 和 Grafana](https://juejin.cn/post/7360629255258046475)
>
> [# 最详细且简单的ElasticSearch (es)集群搭建教程](https://juejin.cn/post/7358109207994646539)

## 安装`elasticsearch-exporter`

1. 首先进入到需要安装`elasticsearch-exporter`的目录中

2. 创建`docker-compose.yaml`文件，内容如下：

   > `--es.uri`为集群中的任意节点IP端口

    ```yaml
    version: "3.8"
    services:
      elasticsearch_exporter:
        image: quay.io/prometheuscommunity/elasticsearch-exporter:v1.5.0
        container_name: "elasticsearch-exporter"
        command:
          - '--es.uri=http://10.0.0.87:9200'
        restart: unless-stopped
        ports:
          - "9114:9114"
    ```

3. 执行命令启动`elasticsearch-exporter`服务
    ```shell
    docker-compose up -d
    ```

## 配置`Prometheus`

1. 编辑`prometheus.yaml`配置文件，`scrape_configs`块下新增如下内容：

   > 注意：
   > 1. 缩进格式保持一致
   > 2. `elasticsearch-exporter`下的`targets`里面的内容替换为自己`elasticsearch-exporter`的IP端口

    ```yaml
      - job_name: 'elasticsearch-exporter'
        static_configs:
          - targets:
            - '10.0.0.87:9114'
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111352.png)

2. 重启`prometheus`服务
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
   > `

3. 验证`prometheus`中是否可以查看到`elasticsearch`相关的信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111387.png)

## 配置`Grafana`

> 面板ID为: `2322`

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111339.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111371.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111421.png)


## 结语

通过本文的介绍，您应该对如何将`ElasticSearch`集群接入`Prometheus`进行监控，并通过`Grafana`进行数据可视化有了详细的了解。希望这些内容能够帮助您在实际工作中更好地部署和优化监控系统。未来，您还可以探索更多高级特性和最佳实践，以进一步提升监控系统的效能和可靠性。