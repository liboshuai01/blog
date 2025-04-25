---
title: Docker快速部署完整监控预警系统
tags:
  - Linux
  - Docker
  - Prometheus
categories:
  - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141920379.png'
toc: true
abbrlink: e352145d
date: 2024-04-23 14:17:41
---

在本篇博文中，我们将探讨如何使用 Docker 快速部署 Prometheus 和 Grafana。Prometheus 是一个开源的监控和警报工具，而 Grafana 则是一个用于可视化监控数据的开源平台。通过结合使用这两个工具，您可以有效地监控您的 IT 基础设施，并通过美观的仪表板展示关键性能指标。

在现代的云计算和微服务架构中，有效的监控系统是不可或缺的。Prometheus 提供了强大的数据采集能力，而 Grafana 则提供了强大的数据可视化支持。使用 Docker 部署这些服务不仅可以简化安装过程，还能保证环境的一致性和可复制性。接下来，我们将一步步通过 Docker Compose 来安装和配置 Prometheus、 Grafana 和 Alertmanager。

<!-- more -->

## 环境准备

> 注意：版本最好与本文中的版本保持一致，避免后续集成监控kafka、redis等组件出现版本不兼容的问题（作者因为版本问题足足搞了一天）

- Centos7系统
- docker
- docker-compose

> 环境准备可参考我的博文：
>
> [# 一键在线/离线安装docker及docker-compose](https://juejin.cn/post/7355389990531678260)

## 安装 Prometheus、Alertmanger、Grafana

1. 进入安装指定目录下
2. 编写`docker-compose.yaml`文件
    ```yaml
    version: "3"
    services:
      prometheus:
        image: prom/prometheus:v2.36.2
        container_name: prometheus
        volumes:
          - prometheus_conf:/etc/prometheus
          - prometheus_data:/prometheus
        ports:
          - "9090:9090"
        command:
          - "--config.file=/etc/prometheus/prometheus.yml"
          - "--web.enable-lifecycle"
        restart: unless-stopped
        extra_hosts:
          - "master:10.0.0.87"
          - "node1:10.0.0.81"
          - "node2:10.0.0.82"
      pushgateway:
        image: prom/pushgateway:v1.5.0
        container_name: pushgateway
        ports:
          - "9091:9091"
        restart: unless-stopped
        extra_hosts:
          - "master:10.0.0.87"
          - "node1:10.0.0.81"
          - "node2:10.0.0.82"
      alertmanager:
        image: prom/alertmanager:v0.25.0
        container_name: alertmanager
        restart: unless-stopped
        ports:
          - "9093:9093"
        volumes:
          - alertmanager_data:/etc/alertmanager
        extra_hosts:
          - "master:10.0.0.87"
          - "node1:10.0.0.81"
          - "node2:10.0.0.82"
      prometheus-alert:
        image: feiyu563/prometheus-alert:master
        container_name: prometheus-alert
        restart: unless-stopped
        ports:
          - "9094:8080"
        volumes:
          - prometheus_alert_data:/app
        environment:
          - PA_LOGIN_USER=alertuser
          - PA_LOGIN_PASSWORD=123456
          - PA_TITLE=prometheusAlert
          - PA_OPEN_FEISHU=1
          - PA_OPEN_DINGDING=0
          - PA_OPEN_WEIXIN=1
        extra_hosts:
          - "master:10.0.0.87"
          - "node1:10.0.0.81"
          - "node2:10.0.0.82"
      grafana:
        image: grafana/grafana:9.1.2
        container_name: grafana
        environment:
          - GF_SECURITY_ADMIN_PASSWORD=Rongshu@2024
        volumes:
          - grafana_data:/var/lib/grafana
        ports:
          - "3000:3000"
        restart: unless-stopped
        extra_hosts:
          - "master:10.0.0.87"
          - "node1:10.0.0.81"
          - "node2:10.0.0.82"
    volumes:
      prometheus_conf:
      prometheus_data:
      prometheus_alert_data:
      alertmanager_data:
      grafana_data:
    ```
3. 执行命令启动服务
    ```shell
    docker-compose up -d
    ```

## 配置`grafana`

1. 浏览器打开`http://IP:3000`，输入账号密码

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141528704.png)

2. 新增数据源为`promethues`

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141528531.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141528550.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141529580.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425141528489.png)

## 结语

通过以上步骤，您可以轻松地在 Docker 上部署 Prometheus 和 Grafana。这种部署方式不仅快速而且易于管理，非常适合快速搭建测试环境或者小型生产环境。在完成部署后，您可以开始配置 Prometheus 的监控目标和 Grafana 的数据源，进而构建出强大的监控和数据可视化解决方案。
