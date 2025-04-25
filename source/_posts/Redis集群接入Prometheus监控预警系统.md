---
title: Redis集群接入Prometheus监控预警系统
tags:
  - Linux
  - Redis
  - Prometheus
categories:
  - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142906518.png'
toc: true
abbrlink: b0a4529e
date: 2024-04-27 14:27:54
---

在当前的企业信息化快速发展背景下，高效的系统监控变得尤为重要。特别是对于数据库和缓存服务，如何确保其性能和稳定性是每个系统管理员和开发者必须面对的挑战。`Redis`作为一种高性能的键值对存储系统，广泛应用于各种业务场景中，因此对其进行有效的监控显得尤为关键。本文将详细介绍如何将`Redis`集群接入到`Prometheus`和`Grafana`中，实现数据的实时监控和可视化，确保系统的健康和性能。

<!-- more -->

## 环境准备

- `Prometheus`搭建部署完毕，版本推荐`v2.36.2`
- `Grafan`搭建部署完毕，版本推荐`9.1.2`
- - `Redis`集群搭建部署完毕，版本推荐`7.12`

> 环境准备可参考我的博文：
>
> [# Docker 安装 Prometheus 和 Grafana](https://juejin.cn/post/7360629255258046475)
>
> [# Centos 离线傻瓜式一键式部署 Redis 集群](https://juejin.cn/post/7310412252552953865)

## 安装`redis-exporter`

1. 首先进入到需要安装`redis-exporter`的目录中

2. 创建`docker-compose.yaml`文件，内容如下：

   > `-redis.addr`为集群中的任意节点地址
   >
   > `-redis.password`为对应节点的认证密码

    ```yaml
    version: "3.8"
    services:
      redis-exporter:
        image: oliver006/redis_exporter:v1.58.0
        container_name: redis-exporter
        restart: unless-stopped
        command:
          - "-redis.addr=10.0.0.87:8098"
          - "-redis.password=Rongshu@2024"
        environment:
          TZ: Asia/Shanghai
        ports:
          - "9121:9121"
    ```

3. 执行命令启动`redis-exporter`服务
    ```shell
    docker-compose up -d
    ```

## 配置`Prometheus`

1. 编辑`prometheus.yaml`配置文件，`scrape_configs`块下新增如下内容：

   > 注意：
   > 1. 缩进格式保持一致
   > 2. `redis_exporter`下的`targets`里面的内容替换为自己`redis-exporter`的IP端口
   > 3. `redis_exporter_targets`下的`targets`里面的内容替换为自己`redis`集群的所有节点的IP端口
   > 4. `redis_exporter_targets`下的`replacement`的值为自己`redis-exporter`的IP端口

    ```yaml
        - job_name: "redis_exporter"
          static_configs:           
            - targets: ["10.0.0.87:9121"]
        - job_name: 'redis_exporter_targets'
          static_configs:
            - targets:              
                - redis://10.0.0.87:8098
                - redis://10.0.0.87:8099 
                - redis://10.0.0.81:8098    
                - redis://10.0.0.81:8099
                - redis://10.0.0.82:8098
                - redis://10.0.0.82:8099
          metrics_path: /scrape         
          relabel_configs:              
            - source_labels: [__address__]
              target_label: __param_target
            - source_labels: [__param_target]
              target_label: instance
            - target_label: __address__
              replacement: 10.0.0.87:9121
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142809274.png)

2. 重启`prometheus`服务
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
    > `

3. 验证`prometheus`中是否可以查看到`redis`相关的信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142809308.png)

## 配置`Grafana`

> 面板ID为: `11835`

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142809346.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142809387.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142810511.png)

## 结语

通过本文的介绍，我们成功将`Redis`集群与`Prometheus`和`Grafana`相结合，实现了对`Redis`集群性能的深入监控和数据可视化。这不仅提高了运维团队对系统状态的响应速度，也为系统的优化和调整提供了数据支持。希望本文能为需要进行类似监控部署的读者提供帮助和指导。

在未来的工作中，我们可以进一步探索更多的`Prometheus`和`Grafana`高级功能，如设置更为复杂的警报规则，或者自定义更多细粒度的监控指标。此外，持续优化和调整监控系统的配置，以适应业务发展和系统扩展的需要，也是我们需要重点关注的方向。

通过持续的技术积累和实践探索，我们能够更好地把握系统的全局性能，确保业务的稳定运行。希望未来能有更多的技术分享，与大家一起进步，共同推动技术的发展与创新。