---
title: 常用中间件docker部署文件合集
tags:
  - Docker
  - 中间件
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504270030786.png'
toc: true
abbrlink: c3c5dc6c
date: 2024-07-11 00:19:03
---

在现代应用开发与运维中，中间件的部署与管理是保障系统稳定高效运行的关键环节。Docker作为轻量级容器化技术，极大简化了中间件的安装、配置和升级流程，提高了环境一致性和部署效率。本文汇总了多款常用中间件的Docker Compose部署文件，包括数据库（MySQL、PostgreSQL、MongoDB）、缓存（Redis）、消息队列（Kafka）、搜索引擎（Elasticsearch）、大数据组件（Doris）以及监控系统（Prometheus、Grafana等）。无论是单机部署还是集群搭建，均提供详尽配置示例，方便开发者和运维人员快速上手，实现一键部署和高效管理。如果您正在规划容器化中间件的部署方案，本篇博文将是您实用且全面的参考资源。

<!-- more -->

Openwebui
---

```shell
# .env文件内容
HOST_IP=宿主机IP
```
```yaml
version: '3.8'

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "46000:8080"
    environment:
      - OLLAMA_BASE_URL=http://${HOST_IP}:11434
    volumes:
      - open-webui:/app/backend/data
    restart: always

volumes:
  open-webui:
```

Moments
---

```yaml
services:
  moments:
    image: kingwrcy/moments:latest
    container_name: moments
    restart: always
    environment:
      PORT: 30000
      JWT_KEY: 24ef6995-d9c9-47b4-8a55-2c74a94796c6
    ports:
      - 30000:30000
    volumes:
      - data:/app/data

volumes:
  data:
```


LobeChat
---
```yaml
version: '2.1'

services:
  lobechat:
    image: lobehub/lobe-chat
    container_name: lobechat
    restart: always
    ports:
      - '3210:3210'
    mem_limit: 4g
```

NextChat
---
```yaml
version: '2.1'

services:
  nextchat:
    image: 'yidadaa/chatgpt-next-web'
    container_name: 'nextchat'
    restart: always
    ports:
      - '3000:3000'
    mem_limit: 4g
```

QingLong
---
````yaml
version: '2.1'

services:
  web:
    image: whyour/qinglong:latest
    volumes:
      - ./data:/ql/data
    ports:
      - "0.0.0.0:5700:5700"
    environment:
      QlBaseUrl: '/'
    restart: always
    mem_limit: 4g
````

MySQL
---

### MySQL 5.7
```yaml
version: '2.1'

services:
  mysql57:
    image: docker.io/bitnami/mysql:5.7.42
    container_name: mysql57
    restart: always
    ports:
      - 3307:3306
    volumes:
      - data:/bitnami/mysql/data
      - logs:/opt/bitnami/mysql/logs
    environment:
      # 密码不能包含特殊字符
      - MYSQL_ROOT_PASSWORD=Lbs2024
      - MYSQL_USER=lbs
      # 密码不能包含特殊字符
      - MYSQL_PASSWORD=Lbs2024
      - MYSQL_DATABASE=test
      - TZ=Asia/Shanghai
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mysql/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6
    # 用来限制5.7的资源占用，否则会占用大量内存
    ulimits:
      nproc: 65535
      nofile:
        soft: 26677
        hard: 46677
    mem_limit: 8g

volumes:
  data:
  logs:
```

### MySQL 8.0.3
```yaml
version: '2.1'

services:
  mysql8:
    image: bitnami/mysql:8.0.39
    container_name: mysql8
    restart: always
    ports:
      - 3308:3306
    volumes:
      - data:/bitnami/mysql/data
      - logs:/opt/bitnami/mysql/logs
    environment:
      # 密码
      - MYSQL_ROOT_PASSWORD=Rongshu@2024
      - MYSQL_USER=lbs
      - MYSQL_PASSWORD=Rongshu@2024
      - MYSQL_DATABASE=test
      - TZ=Asia/Shanghai
    healthcheck:
      test: ['CMD', '/opt/bitnami/scripts/mysql/healthcheck.sh']
      interval: 15s
      timeout: 5s
      retries: 6
    mem_limit: 4g

volumes:
  data:
  logs:
```

### MySQL一主二从
```yaml
version: '2.1'

services:
  mysql_master:
    image: bitnami/mysql:8.0.39
    container_name: mysql_master
    restart: always
    ports:
      - "3310:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=Rongshu@2024
      - MYSQL_REPLICATION_MODE=master
      - MYSQL_REPLICATION_USER=repl_user
      - MYSQL_REPLICATION_PASSWORD=Rongshu@2024
      - MYSQL_USER=lbs
      - MYSQL_PASSWORD=Rongshu@2024
      - MYSQL_DATABASE=test
      - TZ=Asia/Shanghai
    volumes:
      - master_data:/bitnami/mysql/data
      - master_logs:/opt/bitnami/mysql/logs
    mem_limit: 4g

  mysql_slave1:
    image: bitnami/mysql:8.0.39
    container_name: mysql_slave1
    restart: always
    ports:
      - "3311:3306"
    environment:
      - MYSQL_REPLICATION_MODE=slave
      - MYSQL_REPLICATION_USER=repl_user
      - MYSQL_REPLICATION_PASSWORD=Rongshu@2024
      - MYSQL_MASTER_HOST=mysql_master
      - MYSQL_MASTER_PORT_NUMBER=3306
      - MYSQL_MASTER_ROOT_PASSWORD=Rongshu@2024
      - TZ=Asia/Shanghai
    volumes:
      - slave1_data:/bitnami/mysql/data
      - slave1_logs:/opt/bitnami/mysql/logs
    mem_limit: 4g

  mysql_slave2:
    image: bitnami/mysql:8.0.39
    container_name: mysql_slave2
    restart: always
    ports:
      - "3312:3306"
    environment:
      - MYSQL_REPLICATION_MODE=slave
      - MYSQL_REPLICATION_USER=repl_user
      - MYSQL_REPLICATION_PASSWORD=Rongshu@2024
      - MYSQL_MASTER_HOST=mysql_master
      - MYSQL_MASTER_PORT_NUMBER=3306
      - MYSQL_MASTER_ROOT_PASSWORD=Rongshu@2024
      - TZ=Asia/Shanghai
    volumes:
      - slave2_data:/bitnami/mysql/data
      - slave2_logs:/opt/bitnami/mysql/logs
    mem_limit: 4g

volumes:
  master_data:
  slave1_data:
  slave2_data:
  master_logs:
  slave1_logs:
  slave2_logs:
```

Redis
---

### Redis 单机

```yaml
version: '2'

services:
  redis:
    image: bitnami/redis:7.4.2
    container_name: redis
    restart: always
    ports:
      - '6390:6379'
    volumes:
      - 'data:/bitnami/redis/data'
      - /etc/localtime:/etc/localtime:ro
    environment:
      - 'REDIS_PASSWORD=Rongshu@2024'
    mem_limit: 4g

volumes:
  data:
```

### Redis 集群

```shell
# .env文件内容
HOST_IP=宿主机IP
```

```yaml
version: "2"

services:
  redis01:
    image: bitnami/redis-cluster:7.4.2
    container_name: redis01
    restart: always
    ports:
      - 6381:6381 # redis连接用端口
      - 16381:16381 # redis之间消息连接的端口 默认是redis连接端口+10000
    volumes:
      - data1:/bitnami/redis/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - 'REDIS_CLUSTER_ANNOUNCE_IP=${HOST_IP}'
      - 'REDIS_PORT_NUMBER=6381' # 替换原始6379端口
      - 'REDIS_CLUSTER_DYNAMIC_IPS=no' # 标明不使用动态IP
      - 'REDIS_PASSWORD=Rongshu@2024' # 认证密码
      - 'REDIS_NODES=${HOST_IP}:6381 ${HOST_IP}:6382 ${HOST_IP}:6383 ${HOST_IP}:6384 ${HOST_IP}:6385 ${HOST_IP}:6386' # 集群节点IP端口
    mem_limit: 2g

  redis02:
    image: bitnami/redis-cluster:7.4.2
    container_name: redis02
    restart: always
    ports:
      - 6382:6382
      - 16382:16382
    volumes:
      - data2:/bitnami/redis/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - 'REDIS_CLUSTER_ANNOUNCE_IP=${HOST_IP}'
      - 'REDIS_PORT_NUMBER=6382'
      - 'REDIS_CLUSTER_DYNAMIC_IPS=no'
      - 'REDIS_PASSWORD=Rongshu@2024'
      - 'REDIS_NODES=${HOST_IP}:6381 ${HOST_IP}:6382 ${HOST_IP}:6383 ${HOST_IP}:6384 ${HOST_IP}:6385 ${HOST_IP}:6386'
    mem_limit: 2g

  redis03:
    image: bitnami/redis-cluster:7.4.2
    container_name: redis03
    restart: always
    ports:
      - 6383:6383
      - 16383:16383
    volumes:
      - data3:/bitnami/redis/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - 'REDIS_PASSWORD=Rongshu@2024'
      - 'REDIS_CLUSTER_ANNOUNCE_IP=${HOST_IP}'
      - 'REDIS_CLUSTER_DYNAMIC_IPS=no'
      - 'REDIS_PORT_NUMBER=6383'
      - 'REDIS_NODES=${HOST_IP}:6381 ${HOST_IP}:6382 ${HOST_IP}:6383 ${HOST_IP}:6384 ${HOST_IP}:6385 ${HOST_IP}:6386'
    mem_limit: 2g

  redis04:
    image: bitnami/redis-cluster:7.4.2
    container_name: redis04
    restart: always
    ports:
      - 6384:6384
      - 16384:16384
    volumes:
      - data4:/bitnami/redis/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - 'REDIS_PASSWORD=Rongshu@2024'
      - 'REDIS_CLUSTER_ANNOUNCE_IP=${HOST_IP}'
      - 'REDIS_CLUSTER_DYNAMIC_IPS=no'
      - 'REDIS_PORT_NUMBER=6384'
      - 'REDIS_NODES=${HOST_IP}:6381 ${HOST_IP}:6382 ${HOST_IP}:6383 ${HOST_IP}:6384 ${HOST_IP}:6385 ${HOST_IP}:6386'
    mem_limit: 2g

  redis05:
    image: bitnami/redis-cluster:7.4.2
    container_name: redis05
    restart: always
    ports:
      - 6385:6385
      - 16385:16385
    volumes:
      - data5:/bitnami/redis/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - 'REDIS_PASSWORD=Rongshu@2024'
      - 'REDIS_CLUSTER_ANNOUNCE_IP=${HOST_IP}'
      - 'REDIS_CLUSTER_DYNAMIC_IPS=no'
      - 'REDIS_PORT_NUMBER=6385'
      - 'REDIS_NODES=${HOST_IP}:6381 ${HOST_IP}:6382 ${HOST_IP}:6383 ${HOST_IP}:6384 ${HOST_IP}:6385 ${HOST_IP}:6386'
    mem_limit: 2g

  redis06:
    image: bitnami/redis-cluster:7.4.2
    container_name: redis06
    restart: always
    ports:
      - 6386:6386
      - 16386:16386
    volumes:
      - data6:/bitnami/redis/data
      - /etc/localtime:/etc/localtime:ro
    depends_on:
      - redis01
      - redis02
      - redis03
      - redis04
      - redis05
    environment:
      - 'REDIS_PASSWORD=Rongshu@2024'
      - 'REDIS_CLUSTER_ANNOUNCE_IP=${HOST_IP}'
      - 'REDIS_CLUSTER_DYNAMIC_IPS=no'
      - 'REDIS_PORT_NUMBER=6386'
      - 'REDISCLI_AUTH=Rongshu@2024'
      - 'REDIS_CLUSTER_REPLICAS=1'
      - 'REDIS_NODES=${HOST_IP}:6381 ${HOST_IP}:6382 ${HOST_IP}:6383 ${HOST_IP}:6384 ${HOST_IP}:6385 ${HOST_IP}:6386'
      - 'REDIS_CLUSTER_CREATOR=yes'
    mem_limit: 2g

volumes:
  data1:
  data2:
  data3:
  data4:
  data5:
  data6:
```

### Redis Insight

```yaml
version: '2.1'

services:
  redisinsight:
    image: redis/redisinsight:2.60
    container_name: redisinsight
    ports:
      - "5541:5540"
    volumes:
      - data:/data
      - /etc/localtime:/etc/localtime:ro
    restart: always
    mem_limit: 4g

volumes:
  data:
```

Kafka
---

### Kafka单机 + zookeeper单机

```shell
# .env文件内容
HOST_IP=宿主机IP
```

```yaml
version: '2.1'

services:
  zookeeper:
    image: 'bitnami/zookeeper:3.6.1'
    container_name: zookeeper
    restart: always
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    ports:
      - "2183:2181"
    volumes:
      - zoo_data:/bitnami/zookeeper/data
      - /etc/localtime:/etc/localtime:ro
    mem_limit: 2g

  kafka:
    image: 'bitnami/kafka:2.5.0'
    container_name: kafka
    restart: always
    ports:
      - "9089:9092"
    volumes:
      - kafka_data:/bitnami/kafka/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://${HOST_IP}:9089
    mem_limit: 2g
    depends_on:
      - zookeeper

volumes:
  zoo_data:
  kafka_data:
```

### Kafka集群 + zookeeper单机

```shell
# .env文件内容
HOST_IP=宿主机IP
```

```yaml
version: '2.1'

services:
  zookeeper:
    image: 'bitnami/zookeeper:3.6.1'
    container_name: zookeeper
    restart: always
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes
    ports:
      - "2182:2181"
    volumes:
      - zoodata:/bitnami/zookeeper/data
      - /etc/localtime:/etc/localtime:ro
    mem_limit: 2g


  kafka01:
    image: 'bitnami/kafka:2.5.0'
    container_name: kafka01
    restart: always
    ports:
      - "9096:9092"
    volumes:
      - data1:/bitnami/kafka/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_BROKER_ID=1
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://${HOST_IP}:9096
    mem_limit: 2g
    depends_on:
      - zookeeper

  kafka02:
    image: 'bitnami/kafka:2.5.0'
    container_name: kafka02
    restart: always
    ports:
      - "9097:9092"
    volumes:
      - data2:/bitnami/kafka/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_BROKER_ID=2
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://${HOST_IP}:9097
    mem_limit: 2g
    depends_on:
      - zookeeper

  kafka03:
    image: 'bitnami/kafka:2.5.0'
    container_name: kafka03
    restart: always
    ports:
      - "9098:9092"
    volumes:
      - data3:/bitnami/kafka/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ALLOW_PLAINTEXT_LISTENER=yes
      - KAFKA_BROKER_ID=3
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://${HOST_IP}:9098
    mem_limit: 2g
    depends_on:
      - zookeeper

volumes:
  zoodata:
  data1:
  data2:
  data3:
```

Es
---

> 注意：需要调整宿主机虚拟内存vm.max_map_count参数，可参考：centos调整虚拟内存区域限制

```yaml
version: '2.1'

services:
  es01:
    image: "docker.elastic.co/elasticsearch/elasticsearch:7.5.2"
    container_name: es01
    restart: always
    ports:
      - "9201:9200"
    volumes:
      - data01:/usr/share/elasticsearch/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      # -Xms与-Xmx参数值要一致
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    # 此限制要大于ES_JAVA_OPTS的参数1g
    mem_limit: 6g
  
  es02:
    image: "docker.elastic.co/elasticsearch/elasticsearch:7.5.2"
    container_name: es02
    restart: always
    ports:
      - "9202:9200"
    volumes:
      - data02:/usr/share/elasticsearch/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      # -Xms与-Xmx参数值要一致
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    # 此限制要大于ES_JAVA_OPTS的参数1g
    mem_limit: 6g
  
  es03:
    image: "docker.elastic.co/elasticsearch/elasticsearch:7.5.2"
    container_name: es03
    restart: always
    ports:
      - "9203:9200"
    volumes:
      - data03:/usr/share/elasticsearch/data
      - /etc/localtime:/etc/localtime:ro
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      # -Xms与-Xmx参数值要一致
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    # 此限制要大于ES_JAVA_OPTS的参数1g
    mem_limit: 6g

volumes:
  data01:
  data02:
  data03:
```

Doris
---

### Doris单机（1fe+1be）

> 注意：需要调整宿主机虚拟内存vm.max_map_count参数和关闭swap

```shell
# .env文件内容
HOST_IP=宿主机IP
```

```yaml
version: "3"
services:
  fe:
    image: liboshuai01/apache-doris:2.0.13-fe
    hostname: fe
    environment:
      - FE_SERVERS=fe1:${HOST_IP}:9010
      - FE_ID=1
    volumes:
      - fe_doris_meta:/opt/apache-doris/fe/doris-meta/
      - fe_log:/opt/apache-doris/fe/log/
    ports:
      - "8030:8030"
      - "9020:9020"
      - "9030:9030"
      - "9010:9010"
  be:
    image: liboshuai01/apache-doris:2.0.13-be
    hostname: be
    environment:
      - FE_SERVERS=fe1:${HOST_IP}:9010
      - BE_ADDR=${HOST_IP}:9050
    volumes:
      - be_storage:/opt/apache-doris/be/storage/
      - be_script:/docker-entrypoint-initdb.d/
    ports:
      - "9060:9060"
      - "8040:8040"
      - "9050:9050"
      - "8060:8060"
    depends_on:
      - fe

volumes:
  fe_doris_meta:
  fe_log:
  be_storage:
  be_script:
```

### Doris 集群（3fe+3be）

> 注意：需要调整宿主机虚拟内存vm.max_map_count参数和关闭swap
> 
> 另外docker的ip段不能调整，会报错的。

```yaml
version: '3'
services:
  docker-fe-01:
    image: "apache/doris:doris-fe-2.1.7"
    container_name: "doris-fe-01"
    hostname: "fe-01"
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010,fe2:172.20.80.3:9010,fe3:172.20.80.4:9010
      - FE_ID=1
    ports:
      - 8031:8030
      - 9031:9030
    volumes:
      - doris_fe01_meta:/opt/apache-doris/fe/doris-meta
      - doris_fe01_log:/opt/apache-doris/fe/log
    networks:
      doris_net:
        ipv4_address: 172.20.80.2

  docker-fe-02:
    image: "apache/doris:doris-fe-2.1.7"
    container_name: "doris-fe-02"
    hostname: "fe-02"
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010,fe2:172.20.80.3:9010,fe3:172.20.80.4:9010
      - FE_ID=2
    ports:
      - 8032:8030
      - 9032:9030
    volumes:
      - doris_fe02_meta:/opt/apache-doris/fe/doris-meta
      - doris_fe02_log:/opt/apache-doris/fe/log
    networks:
      doris_net:
        ipv4_address: 172.20.80.3

  docker-fe-03:
    image: "apache/doris:doris-fe-2.1.7"
    container_name: "doris-fe-03"
    hostname: "fe-03"
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010,fe2:172.20.80.3:9010,fe3:172.20.80.4:9010
      - FE_ID=3
    ports:
      - 8033:8030
      - 9033:9030
    volumes:
      - doris_fe03_meta:/opt/apache-doris/fe/doris-meta
      - doris_fe03_log:/opt/apache-doris/fe/log
    networks:
      doris_net:
        ipv4_address: 172.20.80.4

  docker-be-01:
    image: "apache/doris:doris-be-2.1.7"
    container_name: "doris-be-01"
    hostname: "be-01"
    depends_on:
      - docker-fe-01
      - docker-fe-02
      - docker-fe-03
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010,fe2:172.20.80.3:9010,fe3:172.20.80.4:9010
      - BE_ADDR=172.20.80.5:9050
    ports:
      - 8041:8040
    volumes:
      - doris_be01_storage:/opt/apache-doris/be/storage
      - doris_be01_script:/docker-entrypoint-initdb.d
      - doris_be01_log:/opt/apache-doris/be/log
    networks:
      doris_net:
        ipv4_address: 172.20.80.5

  docker-be-02:
    image: "apache/doris:doris-be-2.1.7"
    container_name: "doris-be-02"
    hostname: "be-02"
    depends_on:
      - docker-fe-01
      - docker-fe-02
      - docker-fe-03
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010,fe2:172.20.80.3:9010,fe3:172.20.80.4:9010
      - BE_ADDR=172.20.80.6:9050
    ports:
      - 8042:8040
    volumes:
      - doris_be02_storage:/opt/apache-doris/be/storage
      - doris_be02_script:/docker-entrypoint-initdb.d
      - doris_be02_log:/opt/apache-doris/be/log
    networks:
      doris_net:
        ipv4_address: 172.20.80.6

  docker-be-03:
    image: "apache/doris:doris-be-2.1.7"
    container_name: "doris-be-03"
    hostname: "be-03"
    depends_on:
      - docker-fe-01
      - docker-fe-02
      - docker-fe-03
    environment:
      - FE_SERVERS=fe1:172.20.80.2:9010,fe2:172.20.80.3:9010,fe3:172.20.80.4:9010
      - BE_ADDR=172.20.80.7:9050
    ports:
      - 8043:8040
    volumes:
      - doris_be03_storage:/opt/apache-doris/be/storage
      - doris_be03_script:/docker-entrypoint-initdb.d
      - doris_be03_log:/opt/apache-doris/be/log
    networks:
      doris_net:
        ipv4_address: 172.20.80.7

networks:
  doris_net:
    ipam:
      config:
        - subnet: 172.20.80.0/24

volumes:
  doris_fe01_meta:
  doris_fe01_log:
  doris_fe02_meta:
  doris_fe02_log:
  doris_fe03_meta:
  doris_fe03_log:
  doris_be01_storage:
  doris_be01_script:
  doris_be01_log:
  doris_be02_storage:
  doris_be02_script:
  doris_be02_log:
  doris_be03_storage:
  doris_be03_script:
  doris_be03_log:
```

MongoDB
---

### MongoDB 单机

```yaml
version: '2.1'

services:
  mongodb:
    image: docker.io/bitnami/mongodb:4.4.2
    restart: always
    ports:
      - "27018:27017"
    volumes:
      - 'mongodb_data:/bitnami/mongodb'
    environment:
      - MONGODB_ROOT_USER=root
      - MONGODB_ROOT_PASSWORD=Rongshu@2024
      - MONGODB_USERNAME=lbs
      - MONGODB_PASSWORD=Rongshu@2024
      - MONGODB_DATABASE=starlink_risk
    mem_limit: 4g

volumes:
  mongodb_data:
```

Portainer
---

```yaml
version: '2.1'

services:
  portainer:
    image: "portainer/portainer-ce:2.19.5"
    container_name: portainer
    restart: always
    ports:
      - "9001:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - data:/data
      - /etc/localtime:/etc/localtime:ro
    mem_limit: 2g

volumes:
  data:
```

Xxl-job
---

```yaml
version: '3.8'

services:
  xxl-job-admin:
    image: xuxueli/xxl-job-admin:2.3.0
    container_name: xxl-job-admin
    ports:
      - "8181:8181"
    environment:
      PARAMS: '--server.port=8181
              --server.servlet.context-path=/xxl-job-admin
              --spring.datasource.url=jdbc:mysql://windows:3308/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
              --spring.datasource.username=lbs
              --spring.datasource.password=Rongshu@2024
              --xxl.job.accessToken=Rongshu@2024'
    volumes:
      - applogs:/data/applogs
    restart: always
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "windows:192.168.69.244"

volumes:
  applogs:
```

监控组件
---

### Prometheus、Grafana等

```yaml
version: "3"

services:
  prometheus:
    image: prom/prometheus:v2.36.2
    container_name: prometheus
    volumes:
      - prometheus_conf:/etc/prometheus
      - prometheus_data:/prometheus
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "9091:9090"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--web.enable-lifecycle"
    restart: always
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"

  pushgateway:
    image: prom/pushgateway:v1.5.0
    container_name: pushgateway
    ports:
      - "9092:9091"
    volumes:
      - /etc/localtime:/etc/localtime:ro
    restart: always
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"

  alertmanager:
    image: prom/alertmanager:v0.25.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - alertmanager_data:/etc/alertmanager
      - /etc/localtime:/etc/localtime:ro
    restart: always
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"

  prometheus-alert:
    image: feiyu563/prometheus-alert:master
    container_name: prometheus-alert
    ports:
      - "9094:8080"
    volumes:
      - prometheus_alert_data:/app
      - /etc/localtime:/etc/localtime:ro
    environment:
      - PA_LOGIN_USER=alertuser
      - PA_LOGIN_PASSWORD=123456
      - PA_TITLE=prometheusAlert
      - PA_OPEN_FEISHU=1
      - PA_OPEN_DINGDING=0
      - PA_OPEN_WEIXIN=1
    restart: always
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"

  grafana:
    image: grafana/grafana:9.1.2
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=Rongshu@2024
    volumes:
      - grafana_data:/var/lib/grafana
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "9095:3000"
    restart: always
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"

volumes:
  prometheus_conf:
  prometheus_data:
  prometheus_alert_data:
  alertmanager_data:
  grafana_data:
```

> prometheus.yml文件

```yaml
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - docker:9093

rule_files:
  - /etc/prometheus/alter_rule.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: 'docker'
    static_configs:
      - targets: ['docker:9100']
  - job_name: 'one'
    static_configs:
      - targets: ['one:9100']
  - job_name: 'two'
    static_configs:
      - targets: ['two:9100']
  - job_name: 'three'
    static_configs:
      - targets: ['three:9100']

  - job_name: "kafka_exporter"
    static_configs:
      - targets: ["docker:9308"]

  - job_name: 'doirs'
    static_configs:
      - targets: ['one:8130', 'two:8130', 'three:8130']
        labels:
          group: fe
      - targets: ['one:8140', 'two:8140', 'three:8140']
        labels:
          group: be

  - job_name: "redis_exporter"
    static_configs:
      - targets: ["docker:9121"]
  - job_name: 'redis_exporter_targets'
    static_configs:
      - targets:
          - redis://one:6379
          - redis://one:6380
          - redis://two:6379
          - redis://two:6380
          - redis://three:6379
          - redis://three:6380
    metrics_path: /scrape
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: docker:9121
        
  - job_name: 'elasticsearch-exporter'
    static_configs:
      - targets:
          - 'docker:9114'
          
  - job_name: "pushgateway"
    static_configs:
      - targets: ["docker:9092"]
```

### Node_exporter

```yaml
version: '3.8'

services:
  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    restart: always
    network_mode: host
    pid: host
    volumes:
      - '/:/host:ro,rslave'
    command:
      - '--path.rootfs=/host'
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"
```

### Redis-exporter

```yaml
version: "3.8"

services:
  redis-exporter:
    image: oliver006/redis_exporter:v1.58.0
    container_name: redis-exporter
    restart: always
    command:
      - "-redis.addr=one:6379"
      - "-redis.password=Rongshu@2024"
    environment:
      TZ: Asia/Shanghai
    ports:
      - "9121:9121"
    volumes:
      - /etc/localtime:/etc/localtime
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"
```

### Kafka-exporter

```yaml
version: '3.8'

services:
  kafka-exporter:
    image: danielqsj/kafka-exporter:v1.0.0
    container_name: kafka-exporter
    ports:
      - "9308:9308"
    command: ["--kafka.server=10.0.0.7:9096","--kafka.server=10.0.0.7:9097","--kafka.server=10.0.0.7:9098"]
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"
```

### Es-exporter

```yaml
version: "3.8"

services:
  elasticsearch-exporter:
    image: quay.io/prometheuscommunity/elasticsearch-exporter:v1.5.0
    container_name: "elasticsearch-exporter"
    command:
      - '--es.uri=http://10.0.0.7:9201'
    restart: always
    ports:
      - "9114:9114"
    volumes:
      - /etc/localtime:/etc/localtime
    deploy:
      resources:
        limits:
          memory: 4g
    extra_hosts:
      - "docker:10.0.0.95"
      - "one:10.0.0.98"
      - "two:10.0.0.99"
      - "three:10.0.0.93"
```

结语
---
通过Docker容器化部署中间件，不仅能够提升系统的弹性和扩展性，还能显著简化环境搭建和运维复杂度。本文所列的中间件Docker Compose配置涵盖了从开发测试到生产环境的多种场景，兼顾性能和稳定性，且配置灵活易于定制。希望这些示例能助力您的项目快速落地，实现高可用、高性能的服务架构。欢迎根据自身需求调整参数，将容器化中间件融入您的技术栈，并持续探索Docker在微服务和云原生中的更多应用。同时，期待大家分享使用心得和优化建议，共同促进社区生态发展。