---
title: Flink本地idea与yarn集群配置log4j2日志
tags:
  - Idea
  - Flink
  - Yarn
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261104577.png'
toc: true
abbrlink: 3429625b
date: 2024-08-24 11:03:08
---

在现代分布式系统中，日志记录是确保应用程序稳定性和可维护性的关键部分。Apache Flink作为一款强大的流处理框架，提供了灵活的日志管理功能。本指南将带您了解如何在Flink项目中配置和使用Log4j2，以便在本地和YARN环境中有效地记录日志。通过正确的配置，您可以轻松管理日志输出，监控应用程序的运行状态，并在出现问题时快速定位故障。

<!-- more -->

## 环境

| 名称 | 版本 |
| --- | --- |
| centos | 7.9 |
| jdk | 1.8 |
| flink | 1.14.2 |
| hadoop | 2.6.0-cdh5.14.2 |


## 配置

无论是本地运行，还是yarn运行，都需要在项目`pom.xml`文件中添加如下依赖：

```xml
<!-- log4j2依赖 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-api</artifactId>
    <version>2.16.0</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.16.0</version>
    <scope>provided</scope>
</dependency>
<!-- log4j2和slf4j桥接依赖 -->
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-slf4j-impl</artifactId>
    <version>2.16.0</version>
    <scope>provided</scope>
</dependency>
<!-- slf4j依赖 -->
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
</dependency>
```

### `idea`本地环境

项目`resources`目录下创建`log4j2.xml`文件，内容如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration monitorInterval="5">
    <Properties>
        <property name="LOG_PATTERN" value="%date{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n" />
        <property name="LOG_LEVEL" value="INFO" />
    </Properties>

    <appenders>
        <console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <ThresholdFilter level="${LOG_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
        </console>
    </appenders>

    <loggers>
        <root level="${LOG_LEVEL}">
            <appender-ref ref="Console"/>
        </root>
    </loggers>

</configuration>
```

### `yarn`运行环境

flink根目录下面的`lib`下已经包含了`log4j2`相关的依赖，所以如果我们要使用log4j2的话，就必须保证我们自己打的jar包中没有log的相关依赖，不然会出现各种奇怪的问题。这点很重要，

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <artifactSet>
            <excludes>
                <exclude>org.slf4j:*</exclude>
                <exclude>log4j:*</exclude>
                <exclude>ch.qos.logback:*</exclude>
            </excludes>
        </artifactSet>
    </configuration>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                        </excludes>
                    </filter>
                </filters>
            </configuration>
        </execution>
    </executions>
</plugin>
```

提交`flink`任务到yarn环境运行后，项目中配置的`log4j2.xml`文件的内容就无效了，而是使用的yarn集群对应节点中的flink目录下`conf/log4j.properties`文件。
低版本的flink默认的日志配置文件不是滚动的，所以日志文件很大的话，会占用较多的资源，我们需要修改为滚动日志。但我使用的`1.14.2`版本官方已经默认配置了滚动更新了,就不需要进行额外配置。

```xml
# Allows this configuration to be modified at runtime. The file will be checked every 30 seconds.
monitorInterval=30

# This affects logging for both user code and Flink
rootLogger.level = INFO
rootLogger.appenderRef.file.ref = MainAppender

# Uncomment this if you want to _only_ change Flink's logging
#logger.flink.name = org.apache.flink
#logger.flink.level = INFO

# The following lines keep the log level of common libraries/connectors on
# log level INFO. The root logger does not override this. You have to manually
# change the log levels here.
logger.akka.name = akka
logger.akka.level = INFO
logger.kafka.name= org.apache.kafka
logger.kafka.level = INFO
logger.hadoop.name = org.apache.hadoop
logger.hadoop.level = INFO
logger.zookeeper.name = org.apache.zookeeper
logger.zookeeper.level = INFO
logger.shaded_zookeeper.name = org.apache.flink.shaded.zookeeper3
logger.shaded_zookeeper.level = INFO

# Log all infos in the given file
appender.main.name = MainAppender
appender.main.type = RollingFile
appender.main.append = true
appender.main.fileName = ${sys:log.file}
appender.main.filePattern = ${sys:log.file}.%i
appender.main.layout.type = PatternLayout
appender.main.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
appender.main.policies.type = Policies
appender.main.policies.size.type = SizeBasedTriggeringPolicy
appender.main.policies.size.size = 100MB
appender.main.policies.startup.type = OnStartupTriggeringPolicy
appender.main.strategy.type = DefaultRolloverStrategy
appender.main.strategy.max = ${env:MAX_LOG_FILE_NUMBER:-10}

# Suppress the irrelevant (wrong) warnings from the Netty channel handler
logger.netty.name = org.jboss.netty.channel.DefaultChannelPipeline
logger.netty.level = OFF
```

重新提交任务滚动日志即可生效（如果不需要滚动日志，则不需要进行多余的配置）。

生成的日志文件存放在任务运行节点上的`hadoop`根目录下的`logs/userlogs`下的应用ID以及下面的各个容器目录下。

## 结语

通过本指南，您已经掌握了在Apache Flink项目中配置Log4j2的基本步骤。无论是在本地开发环境还是在YARN集群中运行，合理的日志配置将大大提高您的应用程序的可观测性和可调试性。希望您能够根据项目需求灵活调整日志设置，以确保最佳的性能和资源利用率。随着对日志管理的深入理解，您将能够更有效地监控和维护您的流处理应用程序。

> 参考文章：https://www.cnblogs.com/upupfeng/p/14489664.html