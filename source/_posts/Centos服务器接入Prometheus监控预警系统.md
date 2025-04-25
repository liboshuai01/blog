---
title: Centos服务器接入Prometheus监控预警系统
tags:
  - Linux
  - Docker
  - Prometheus
categories:
  - 监控预警
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142250834.png'
toc: true
abbrlink: c49f934d
date: 2024-04-23 14:21:37
---

在这篇技术博文中，我将详细介绍如何在 CentOS 系统上安装和配置 Prometheus 的一个重要组件 —— `node_exporter`。`node_exporter` 负责收集主机级别的性能指标，并将这些指标暴露给 Prometheus 服务器。我们还将介绍如何将这些监控数据接入到 Prometheus，并使用 Grafana 来创建一个漂亮的监控面板。

接下来，我们将一步步通过实际操作来完成整个过程。我会从 `node_exporter` 的安装和配置开始，然后配置 Prometheus 以收集 `node_exporter` 暴露的指标，最后通过 Grafana 创建仪表板来可视化这些指标。无论你是系统管理员还是开发人员，掌握这些技能都将有助于你更好地维护和优化你的系统。

<!-- more -->

## 环境准备

> 一定要注意各个组件的版本对应问题，博文在这方面吃了大亏，浪费了很多时间。
>
> 特别是 Grafana 版本 与 kafka-exporter 版本不能过高，不然不兼容，图标数据不展示

-   Prometheus版本: `v2.36.2`
-   Grafana版本: `9.1.2`
-   Kafka版本: `2.5.0`
-   Kafka-exporter版本: `v1.0.0`

> 环境准备可参考我的博文：
>
> [# Docker 安装 Prometheus 和 Grafana](https://juejin.cn/post/7360629255258046475 "https://juejin.cn/post/7360629255258046475")

## 安装`node_exporter`

### 本地安装【推荐】

1. 下载安装包并上传到服务器
    ```shell
    https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
    ```

2. 解压安装包到指定安装路径，并重命名
    ```shell
    mkdir -p /home/lbs/software/exporter
    tar -zxvf node_exporter-1.7.0.linux-amd64.tar.gz -C /home/lbs/software/exporter
    mv /home/lbs/software/exporter/node_exporter-1.7.0.linux-amd64 /home/lbs/software/exporter/node_exporter
    ```

3. 在`node_exporter`目录下新增管理脚本`manage_script.sh`
   > 注意替换`NODE_EXPORTER_ROOT`的值为自己的安装路径

    ```
    echo '#!/bin/bash
    # node_exporter的根目录路径
    NODE_EXPORTER_ROOT="/home/lbs/software/exporter/node_exporter"
    # node_exporter的pid文件名称
    PID_FILE="node_exporter.pid"
    # node_exporter的日志文件名称
    LOG_FILE="node_exporter.log"
   
    function start {
        # 从根目录路径组合出完整的命令路径
        local cmd="$NODE_EXPORTER_ROOT/node_exporter"
        nohup $cmd > $LOG_FILE 2>&1 &
        echo $! > $PID_FILE
        echo "Started Node Exporter, pid: $(cat $PID_FILE)"
    }
   
    function stop {
        kill $(cat $PID_FILE)
        rm $PID_FILE
        echo "Stopped Node Exporter"
    }
   
    function status {
        if [ -f $PID_FILE ]; then
            local pid=$(cat $PID_FILE)
   
            if ps -p $pid > /dev/null; then
                echo "Node Exporter is running, pid: $pid"
                echo "Start time: $(ps -p $pid -o lstart=)"
                echo "Running time: $(ps -p $pid -o etime=)"
                echo "Started by user: $(ps -p $pid -o user=)"
            else
                echo "Node Exporter is not running"
            fi
        else
            echo "Node Exporter is not running"
        fi
    }
   
    case $1 in
        "start")
            start
            ;;
        "stop")
            stop
            ;;
        "status")
            status
            ;;
        *)
            echo "Usage: $0 {start|stop|status}"
            ;;
    esac' > manage_script.sh
    chmod +x manage_script.sh
    ```

4. 使用`manage_script.sh`脚本启动`node_exporter`
    ```shell
    /home/lbs/software/exporter/node_exporter/manage_script.sh start
    ```

5. 使用`manage_script.sh`查看`node_exporter`状态
    ```shell
    /home/lbs/software/exporter/node_exporter/manage_script.sh status
    ```

### docker安装

> `node exporter`官方更推荐使用本地部署，而不是docker

1. 编写docker-compose.yml文件
    ```yaml
    version: '3.8'
    
    services:
      node_exporter:
        image: prom/node-exporter:v1.6.1
        container_name: node_exporter
        restart: always
        network_mode: host
        pid: host
        volumes:
          - '/:/host:ro,rslave'
        command:
          - '--path.rootfs=/host'
    ```
2. 启动容器

    ```bash
    docker-compose up -d
    ```

## 配置`promethues`

> 需要先进入`prometheus`容器内部

1. 编辑`/etc/prometheus/prometheus.yml`配置文件，变动内容部分如下：

    ```yml
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              # 配置alertmanagers的ip与端口
              - 10.0.0.7:9093
    rule_files:
      # 新增的预警规则文件
      - /etc/prometheus/alert_rule.yml
    ......
    ......
    ......
    scrape_configs:
      ......
      ......
      ......
      - job_name: 'node1'
        static_configs:
        # 第一台服务器ndoe_exporter的ip端口
        - targets: ['10.0.0.7:9100']
      - job_name: 'node2'
        static_configs:
        # 第二器ndoe_exporter的ip端口
        - targets: ['10.0.0.7:9100']
      ......
      ......
      ......
    ```

2. 新增的`/etc/prometheus/alert_rule.yml`配置文件，内容如下：

    ```yaml
    groups:
      - name: Node Monitor
        rules:
          - alert: nodeSurvival
            expr: up == 0
            for: 30s
            labels:
              level: 高
            annotations:
              summary: "宕机预警"
              description: "已经宕机30s,请尽快处理!"
          - alert: cpuUsage
            expr: 100 * (1 - avg(irate(node_cpu_seconds_total{mode="idle"}[2m])) by(instance)) > 90
            for: 1m
            labels:
              level: 高
            annotations:
              summary: "主机CPU预警"
              description: "主机CPU使用率超过90% 已经持续1m"
          - alert: memoryUsage
            expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
            for: 1m
            labels:
              level: 高
            annotations:
              summary: "主机内存预警"
              description: "主机内存使用率超过90% 已经持续1m"
          - alert: diskUsage
            expr: 100 * (node_filesystem_size_bytes{fstype=~"xfs|ext4"} - node_filesystem_avail_bytes) / node_filesystem_size_bytes > 90
            for: 1m
            labels:
              level: 高
            annotations:
              summary: "主机磁盘预警"
              description: "主机磁盘使用率超过90% 已经持续1m"
    ```

3. 重启`promethues`使其新增配置生效
   > 也可以采用热加载配置文件的方式：`curl  -XPOST localhost:9090/-/reload
   > `

4. 验证`node-exporter`是否接入`prometheus`成功

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142013199.png)

5. 验证`alertmanager`是否接入`prometheus`成功

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142011953.png)


## 配置`grafana`

1. 登录`grafana`页面，新增`node_exporter`面板
   > 面版ID号为: `15172`

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142012969.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142013186.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142012944.png)

## 配置`alertmanager`

1. 浏览器打开`http://IP:9094`
   > 账号/密码: `alertuser/123456`

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142012925.png)

2. 模板管理中新增自定义模板

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142016865.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142017365.png)

   其实模板内容为:
    ```
    {{ range $k,$v:=.alerts }}{{if eq $v.status "resolved"}}<font color="green">**告警恢复信息**</font>
    级别: {{$v.labels.level}}
    类型：{{$v.status}}
    任务: {{$v.labels.job}}
    实例: {{$v.labels.instance}}
    事件: {{$v.annotations.summary}}
    详情: {{$v.annotations.description}}
    开始时间: {{GetCSTtime $v.startsAt}}
    结束时间: {{GetCSTtime $v.endsAt}}
    {{else}}<font color="red">**告警信息**</font>
    级别: {{$v.labels.level}}
    类型：{{$v.status}}
    任务: {{$v.labels.job}}
    实例: {{$v.labels.instance}}
    事件: {{$v.annotations.summary}}
    详情: {{$v.annotations.description}}
    开始时间: {{GetCSTtime $v.startsAt}}
    {{end}}
    {{ end }}
    ```
   >  最后直接点击《保存模板》即可

3. 申请飞书机器人，获取`webhook`地址
   > 需要新建群组

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142018883.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142018407.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142019022.png)

   > 复制图中红色圈起来的`webhook`地址

4. 浏览器打开`http://IP:9094`，获取`模板url`

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142019142.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142021228.png)

   > 复制图中红色圈起来的`url`地址，并将`fsurl=`后的内容替换为第一步中获取的`webhook`地址

5. 编辑`alertmanager`容器`/etc/alertmanager`下的`alertmanager.yml`

    ```yaml
    global:
      resolve_timeout: 5m
    route:
      group_by: ['instance']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 10m
      receiver: 'web.hook.prometheusalert'
    receivers:
      - name: 'web.hook.prometheusalert'
        webhook_configs:
          - url: 'http://10.0.0.87:9094/prometheusalert?type=fs&tpl=prometheus-feishu&fsurl=https://open.feishu.cn/open-apis/bot/v2/hook/xxxxxxxxxxxxxxxxxxxx'
    ```

   > `url`的内容为第二步骤获取到的`模块url`

6. 重启`alertmanager`容器

7. 关闭监控的主机上的`node-exporter`, 30s 后飞书中接收到预警信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142022008.png)

8. 重新启动监控的主机上的`node-exporter`，再次接收到飞书中接收到恢复预警信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425142023920.png)

## 总结

通过本文的介绍，我们了解了如何在 CentOS 环境中安装和配置 `node_exporter`，以及如何将其与 Prometheus 和 Grafana 结合使用。通过这些步骤，我们可以轻松地将系统的监控工作自动化，更加直观地通过 Grafana 面板来观察系统的健康状况和性能指标。

如今，随着云计算和微服务架构的流行，系统监控变得更加重要。一个可靠的监控系统能够帮助我们在问题发生之前就进行预警，从而减少系统的停机时间，提高用户满意度。希望你能将本文的内容应用到实际的工作中，构建出强大的监控系统以支撑你的业务运行。

感谢你的阅读，如果你有任何问题或建议，欢迎在评论区留言讨论。