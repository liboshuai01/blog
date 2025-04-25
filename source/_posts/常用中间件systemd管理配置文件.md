---
title: 常用中间件systemd管理配置文件
tags:
  - Linux
  - GCC
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260129801.png'
toc: true
abbrlink: 152a4fcc
date: 2025-07-10 01:28:53
---

zookeeper
---

```shell
sudo tee /etc/systemd/system/zookeeper.service <<'EOF'
[Unit]
Description=zookeeper
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/zookeeper/bin/zkServer.sh start
ExecStop=/home/lbs/software/zookeeper/bin/zkServer.sh stop
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

kafka
---

```shell
sudo tee /etc/systemd/system/kafka.service <<'EOF'
[Unit]
Description=kafka
After=zookeeper.service
Requires=zookeeper.service

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/kafka/bin/kafka-server-start.sh -daemon /home/lbs/software/kafka/config/server.properties
ExecStop=/home/lbs/software/kafka/bin/kafka-server-stop.sh
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

redis
---

### 节点1

```sheel
sudo tee /etc/systemd/system/redis-6379.service <<'EOF'
[Unit]
Description=redis-6379
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
ExecStart=/home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6379/conf/redis-6379.conf
ExecStop=/home/lbs/software/redis/bin/redis-cli -p 6379 -a Rongshu@2024 shutdown
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

### 节点2

```shell
sudo tee /etc/systemd/system/redis-6380.service <<'EOF'
[Unit]
Description=redis-6380
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
ExecStart=/home/lbs/software/redis/bin/redis-server /home/lbs/software/redis/6380/conf/redis-6380.conf
ExecStop=/home/lbs/software/redis/bin/redis-cli -p 6380 -a Rongshu@2024 shutdown
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

elasticSearch（es)
---

```shell
sudo tee /etc/systemd/system/es.service <<'EOF'
[Unit]
Description=elasticsearch
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
ExecStart=/home/lbs/software/elasticsearch/bin/elasticsearch -d
PrivateTmp=true
# 进程可以打开的最大文件数
LimitNOFILE=65535
# 进程可以打开的最大进程数
LimitNPROC=65535
# 最大虚拟内存
LimitAS=infinity
# 最大文件大小
LimitFSIZE=infinity
# 超时设置 0-永不超时
TimeoutStopSec=0
# SIGTERM是停止java进程的信号
KillSignal=SIGTERM
# 信号只发送给给JVM
KillMode=process
# java进程不会被杀掉
SendSIGKILL=no
# 正常退出状态
SuccessExitStatus=143

# 开机自启动
[Install]
WantedBy=multi-user.target
EOF
```

flink集群
```bash
sudo tee /etc/systemd/system/flink-cluster.service <<'EOF'
[Unit]
Description=flink集群
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/flink/bin/start-cluster.sh
ExecStop=/home/lbs/software/flink/bin/stop-cluster.sh
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

kibana
---

```shell
sudo tee /etc/systemd/system/kibana.service <<'EOF'
[Unit]
Description=Kibana Service
After=es.service
Requires=es.service

[Service]
Type=simple
User=lbs
Group=lbs
ExecStart=/home/lbs/software/kibana/bin/kibana
ExecStop=/usr/bin/kill -15 $MAINPID
ExecReload=/usr/bin/kill -HUP $MAINPID
RemainAfterExit=yes
PrivateTmp=true
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=65535
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

[Install]
WantedBy=multi-user.target
EOF
```

hadoop集群
---

```shell
sudo tee /etc/systemd/system/hadoop-cluster.service <<'EOF'
[Unit]
Description=hadoop-cluster
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/hadoop/sbin/start-all.sh
ExecStop=/home/lbs/software/hadoop/sbin/stop-all.sh
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

doris
---

### fe

```shell
sudo tee /etc/systemd/system/doris-fe.service <<'EOF'
[Unit]
Description=doris-fe
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/doris/fe/bin/start_fe.sh --daemon
ExecStop=/home/lbs/software/doris/fe/bin/stop_fe.sh --daemon
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

### be

```shell
sudo tee /etc/systemd/system/doris-be.service <<'EOF'
[Unit]
Description=doris-be
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/doris/be/bin/start_be.sh --daemon
ExecStop=/home/lbs/software/doris/be/bin/stop_be.sh --daemon
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

### broker

```shell
sudo tee /etc/systemd/system/doris-broker.service <<'EOF'
[Unit]
Description=doris-broker
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/doris/extensions/apache_hdfs_broker/bin/start_broker.sh --daemon
ExecStop=/home/lbs/software/doris/extensions/apache_hdfs_broker/bin/stop_broker.sh --daemon
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

nacos
---

```shell
sudo tee /etc/systemd/system/nacos.service <<'EOF'
[Unit]
Description=nacos
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/nacos/bin/startup.sh
ExecStop=/home/lbs/software/nacos/bin/shutdown.sh
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```

harbor
---

```shell
sudo tee /etc/systemd/system/harbor.service <<'EOF'
[Unit]
Description=harbor
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /root/software/harbor/harbor/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /root/software/harbor/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
EOF
```

mysql
---

```shell
sudo tee /etc/systemd/system/mysql.service <<'EOF'
[Unit]
Description=mysql
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /opt/1panel/apps/mysql/mysql/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /opt/1panel/apps/mysql/mysql/docker-compose.yml down

[Install]
WantedBy=multi-user.target
EOF
```

portainer
---

```shell
sudo tee /etc/systemd/system/portainer.service <<'EOF'
[Unit]
Description=portainer
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /home/lbs/software/portainer/docker-compose.yaml up
ExecStop=/usr/local/bin/docker-compose -f /home/lbs/software/portainer/docker-compose.yaml down

[Install]
WantedBy=multi-user.target
EOF
```

prometheus等
---

```shell
sudo tee /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=prometheus, grafana, alertmanager, prometheus-alert, pushgateway
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /home/lbs/software/prometheus/docker-compose.yaml up
ExecStop=/usr/local/bin/docker-compose -f /home/lbs/software/prometheus/docker-compose.yaml down

[Install]
WantedBy=multi-user.target
EOF
```

jenkins
---

```shell
sudo tee /etc/systemd/system/jenkins.service <<'EOF'
[Unit]
Description=jenkins
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /home/lbs/software/jenkins/jenkins/docker-compose.yaml up
ExecStop=/usr/local/bin/docker-compose -f /home/lbs/software/jenkins/jenkins/docker-compose.yaml down

[Install]
WantedBy=multi-user.target
EOF
```

sonarqube
---

```shell
sudo tee /etc/systemd/system/sonarqube.service <<'EOF'
[Unit]
Description=sonarqube, postgres
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /home/lbs/software/sonarqube/sonarqube/docker-compose.yaml up
ExecStop=/usr/local/bin/docker-compose -f /home/lbs/software/sonarqube/sonarqube/docker-compose.yaml down

[Install]
WantedBy=multi-user.target
EOF
```

node_exporter
---

```shell
sudo tee /etc/systemd/system/node_exporter.service <<'EOF'
[Unit]
Description=node_exporter
After=network.target

[Service]
User=lbs
Group=lbs
ExecStart=/home/lbs/software/exporter/node_exporter/node_exporter
Restart=on-failure
RestartSec=20

[Install]
WantedBy=multi-user.target
EOF
```

flume
---

```shell
sudo tee /etc/systemd/system/flume.service <<'EOF'
[Unit]
Description=flume
After=network.target

[Service]
Type=forking
User=lbs
Group=lbs
KillMode=control-group
Environment="JAVA_HOME=/home/lbs/software/jdk8"
ExecStart=/home/lbs/software/flume/bin/flume-test.sh start
ExecStop=/home/lbs/software/flume/bin/flume-test.sh stop
SuccessExitStatus=0 143
PrivateTmp=false
LimitNOFILE=1000000
LimitNPROC=100000
TimeoutStopSec=10s
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```