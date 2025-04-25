---
title: Doris集群接入Prometheus监控预警系统
tags:
  - Linux
  - Kafka
  - Prometheus
categories:
  - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425143333683.png'
toc: true
abbrlink: cb9ef302
date: 2024-04-28 14:32:30
---

在本文中，我们将详细探讨如何将 Doris 集群与 Prometheus 和 Grafana 监控系统集成。通过对这些技术的整合，我们可以实现对 Doris 集群的实时监控，从而有效地监控集群的性能和健康状态。此外，通过图形化的监控界面，我们能更直观地理解和分析数据，对于维护和优化集群运行至关重要。

<!-- more -->

## 环境准备

- `Doris`集群搭建部署完毕，版本推荐`2.1.0`
- `Prometheus`搭建部署完毕，版本推荐`v2.36.2`
- `Grafan`搭建部署完毕，版本推荐`9.1.2`

> 环境准备可参考我的博文：
>
> [# Docker 安装 Prometheus 和 Grafana](https://juejin.cn/post/7360629255258046475)
>
> [# 最详细且简单的Doris集群搭建教程](https://juejin.cn/post/7302023698722471977)

## 配置`Prometheus`

> doris接入到 `prometheus` 中不需要`exporter`

1. 编辑`prometheus.yaml`配置文件，`scrape_configs`块下新增如下内容：

   > 注意：
   > 1. `fe_host1`与`be_host1`修改为自己 doris 集群对应的 IP
   > 2. `http_port`需替换为自己 doris 集群的端口，默认端口为`8030`
   > 3. `webserver_port`需替换为自己 doris 集群的端口，默认端口为`8040`

    ```yaml
      - job_name: 'doirs'
        static_configs:
          - targets: ['fe_host1:http_port', 'fe_host2:http_port', 'fe_host3:http_port']
            labels:
              group: fe 
          - targets: ['be_host1:webserver_port', 'be_host2:webserver_port', 'be_host3:webserver_port']
            labels:
              group: be
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111724.png)

2. 重启`prometheus`服务
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
   > `

3. 验证`prometheus`中是否可以查看到`doris`相关的信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111796.png)

## 配置`Grafana`

> 面板ID为: `9734`

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111826.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111785.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111810.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260111864.png)

## 结语

通过本文的指导，您可以有效地将 Doris 集群与 Prometheus 和 Grafana 集成，实现高效、实时的监控解决方案。此外，监控系统的配置和优化是一个持续的过程，需要根据实际运行情况不断调整和完善。希望本文能为您在构建和优化 Doris 集群监控体系提供帮助和启示。

> 参考于：[doris 官网](https://doris.apache.org/zh-CN/docs/dev/admin-manual/maint-monitor/monitor-alert/)