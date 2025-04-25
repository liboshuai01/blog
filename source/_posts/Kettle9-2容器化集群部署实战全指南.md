---
title: Kettle9.2容器化集群部署实战全指南
tags:
  - Linux
  - Kettle
  - Docker
  - Etl
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260135938.png'
toc: true
abbrlink: c9f07f4a
date: 2024-07-26 01:34:10
---

本文详细介绍如何基于 Docker 容器技术，搭建 Kettle 9.2 版本的分布式集群环境，包含环境准备、镜像制作、配置编排和多节点部署等完整流程，适用于需要实现 ETL 及数据集成作业分布式调度与管理的应用场景。

<!-- more -->

---

## 集群架构与基础环境规划

- **Kettle版本**：9.2
- **JDK版本**：1.8
- **集群拓扑结构**：

| 主机    | 角色     |
| ------- | -------- |
| master  | 主节点   |
| node1   | 从节点1  |
| node2   | 从节点2  |

---

## 项目目录结构概览

```bash
[lbs@master kettle]$ tree -L 3
.
├── centos7_openjdk8
│ ├── Dockerfile
│ └── init
│     ├── Centos-7.repo
│     ├── nux-dextop-release-0-5.el7.nux.noarch.rpm
│     └── openlogic-openjdk-8u422-b05-linux-x64.tar.gz
└── kettle9.2
    ├── conf
    │ └── carte-config.xml
    ├── docker-compose.yml
    ├── Dockerfile
    ├── init
    │ ├── kettle.tar.gz
    │ └── repositories.xml
    └── logs
        └── pdi.log
```

该结构清晰划分了基础镜像及 Kettle 镜像相关配置和存储目录，方便各节点统一管理和维护。

---

## 制作基础系统镜像（CentOS 7 + OpenJDK 8）

### 环境依赖资源下载

首先准备 CentOS 7 基础镜像所需的资源包：

```bash
mkdir -p centos7_openjdk8/init && cd centos7_openjdk8

wget -P ./init https://mirrors.aliyun.com/repo/Centos-7.repo
wget -P ./init https://privatefilesbucket-community-edition.s3.us-west-2.amazonaws.com/9.4.0.0-343/ce/client-tools/pdi-ce-9.4.0.0-343.zip
wget -P ./init https://mirrors.coreix.net/li.nux.ro//nux/dextop/el7/x86_64/nux-dextop-release-0-5.el7.nux.noarch.rpm
wget -P ./init https://builds.openlogic.com/downloadJDK/openlogic-openjdk/8u422-b05/openlogic-openjdk-8u422-b05-linux-x64.tar.gz
```

确保所有文件完整且下载成功。

### 编写基础镜像 Dockerfile

```dockerfile
FROM centos:centos7

LABEL maintainer="liboshuai"

COPY ./init/* /usr/local/

RUN cd /usr/local \
 && mv Centos-7.repo /etc/yum.repos.d/ \
 && tar -zxvf openlogic-openjdk-8u422-b05-linux-x64.tar.gz \
 && mv openlogic-openjdk-8u422-b05-linux-x64 openjdk8 \
 && rm -rf openlogic-openjdk-8u422-b05-linux-x64.tar.gz \
 && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
 && cd /etc/yum.repos.d \
 && sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-* \
 && sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-* \
 && yum clean all && yum makecache \
 && yum update -y \
 && yum install -y vim unzip zip epel-release \
 && rpm -Uvh /usr/local/nux-dextop-release-0-5.el7.nux.noarch.rpm \
 && rm -rf /usr/local/nux-dextop-release-0-5.el7.nux.noarch.rpm \
 && yum install -y webkitgtk \
 && yum clean all

WORKDIR /usr/local

ENV JAVA_HOME=/usr/local/openjdk8
ENV CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
ENV PATH=$JAVA_HOME/bin:$PATH

CMD ["java","-version"]
```

### 构建基础镜像

```bash
docker build -t centos7_openjdk8:v1.0 .
```

构建完成后，可以执行 `docker run --rm centos7_openjdk8:v1.0` 检查 Java 版本。

---

## 制作 Kettle 9.2 应用镜像

### 下载及解压 Kettle 资源

进入项目目录并准备 Kettle 安装包和依赖：

```bash
mkdir -p kettle9.2/init && cd kettle9.2

wget -P ./init https://privatefilesbucket-community-edition.s3.us-west-2.amazonaws.com/9.2.0.0-290/ce/client-tools/pdi-ce-9.2.0.0-290.zip
unzip ./init/pdi-ce-9.2.0.0-290.zip

wget -P ./data-integration/lib https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.33/mysql-connector-j-8.0.33.jar
```

确保 MySQL 驱动放置正确，支持数据库连接。

### 配置修改与打包准备

- 修改 `data-integration/spoon.sh` 中的 JVM 参数，加入 `-Dfile.encoding=UTF-8` 以避免编码问题。
- 将 `data-integration` 目录重命名为 `kettle`，便于统一管理。
- 打包该目录：

```bash
mv data-integration kettle
tar -zcvf kettle.tar.gz kettle
```

### 编写 Kettle Dockerfile

```dockerfile
FROM centos7_openjdk8:v1.0

LABEL maintainer="liboshuai"

ADD kettle.tar.gz /usr/local

RUN mkdir -p /root/.kettle

WORKDIR /usr/local/kettle

VOLUME ["/root/.kettle", "/usr/local/kettle/conf", "/usr/local/kettle/logs"]

ENTRYPOINT ["./carte.sh"]
```

### 编写 docker-compose.yml

```yaml
version: '3'

services:
  kettle:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: kettle
    network_mode: host
    volumes:
      - ./repository:/root/.kettle
      - ./conf:/usr/local/kettle/conf
      - ./logs:/usr/local/kettle/logs
    environment:
      - PENTAHO_DI_JAVA_OPTIONS=-Xms8g -Xmx8g
    command: ["./conf/carte-config.xml"]
    extra_hosts:
      - "master:10.0.0.87"
      - "node1:10.0.0.81"
      - "node2:10.0.0.82"
```

`network_mode: host` 方便容器使用宿主网络，`extra_hosts` 映射集群节点主机名与 IP。

---

## 集群部署详细步骤

> 以下操作均在对应的物理主机（master、node1、node2）上执行。

---

### 主节点 master 配置与启动

1. 创建目录结构

```bash
cd kettle9.2
mkdir -p conf logs repository
```

2. 编写 `carte-config.xml` 文件：

```xml
<slave_config>
  <slaveserver>
    <name>master1</name>
    <hostname>master</hostname>
    <port>29000</port>
    <username>kettle</username>
    <password>Kettle@2024</password>
    <master>Y</master>
  </slaveserver>

  <repository>
    <name>mysql</name>
    <username>admin</username>
    <password>admin</password>
  </repository>

  <max_log_lines>10000</max_log_lines>
  <max_log_timeout_minutes>2880</max_log_timeout_minutes>
  <object_timeout_minutes>2880</object_timeout_minutes>
</slave_config>
```

3. 准备 Kettle 资源库配置文件 `repositories.xml`，放置于 `repository` 目录。该文件由 Spoon GUI 导出，确保和集群 repository 配置对应。

4. 构建镜像并启动容器

```bash
docker-compose up -d --build
```

验证容器启动无误后，主节点即完成部署。

---

### 从节点 node1 配置与启动

1. 创建必要目录

```bash
cd kettle9.2
mkdir -p conf logs
```

2. 编写 `carte-config.xml` 文件：

```xml
<slave_config>
  <masters>
    <slaveserver>
      <name>master1</name>
      <hostname>master</hostname>
      <port>29000</port>
      <username>kettle</username>
      <password>Kettle@2024</password>
      <master>Y</master>
    </slaveserver>
  </masters>

  <report_to_masters>Y</report_to_masters>

  <slaveserver>
    <name>node1</name>
    <hostname>node1</hostname>
    <port>29000</port>
    <username>kettle</username>
    <password>Kettle@2024</password>
    <master>N</master>
  </slaveserver>

  <repository>
    <name>mysql</name>
    <username>admin</username>
    <password>admin</password>
  </repository>

  <max_log_lines>10000</max_log_lines>
  <max_log_timeout_minutes>2880</max_log_timeout_minutes>
  <object_timeout_minutes>2880</object_timeout_minutes>
</slave_config>
```

3. 启动容器

```bash
docker-compose up -d --build
```

成功启动后，该从节点将连接主节点注册并汇报状态。

---

### 从节点 node2 配置与启动

重复 node1 的操作，只需调整 `carte-config.xml` 中 `<slaveserver>` 标签中的 `name` 和 `hostname` 为 `node2`：

```xml
<slave_config>
  <masters>
    <slaveserver>
      <name>master1</name>
      <hostname>master</hostname>
      <port>29000</port>
      <username>kettle</username>
      <password>Kettle@2024</password>
      <master>Y</master>
    </slaveserver>
  </masters>

  <report_to_masters>Y</report_to_masters>

  <slaveserver>
    <name>node2</name>
    <hostname>node2</hostname>
    <port>29000</port>
    <username>kettle</username>
    <password>Kettle@2024</password>
    <master>N</master>
  </slaveserver>

  <repository>
    <name>mysql</name>
    <username>admin</username>
    <password>admin</password>
  </repository>

  <max_log_lines>10000</max_log_lines>
  <max_log_timeout_minutes>2880</max_log_timeout_minutes>
  <object_timeout_minutes>2880</object_timeout_minutes>
</slave_config>
```

执行构建与启动命令：

```bash
docker-compose up -d --build
```

---

## 集群管理与后续建议

- **日志存储与监控**  
  通过挂载宿主机目录方式持久化日志，方便集中查看及问题排查。建议引入 ELK 等日志收集工具进行统一分析。

- **安全策略**  
  尽量使用安全加固措施，例如限制访问端口、使用防火墙规则、替换默认密码和启用访问认证。

- **资源配置优化**  
  根据作业负载调整 JVM 参数、容器资源限制。如果集群规模扩大，考虑使用 Kubernetes 管理容器进行弹性调度。

- **自动化部署**  
  利用 CI/CD 工具链实现镜像自动打包和版本发布，缩短上线周期。

---

通过以上步骤，即可实现基于 Docker 的 Kettle 9.2 集群化容器部署环境，支持分布式 ETL 作业高效管理和执行。希望本指南能够帮助您快速搭建稳定、高效的 Kettle 集群体系。

---

# 参考资料

- [Pentaho Data Integration 官方文档](https://help.pentaho.com/)
- [Docker 官方文档](https://docs.docker.com/)
- [Kettle carte-master 配置指南](https://wiki.pentaho.com/display/EAI/Carte+Slave+Server+Configuration)

---

**如需进一步定制开发或集群优化，欢迎咨询专业后端架构服务。**