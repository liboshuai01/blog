---
title: SpringBoot整合ELK日志
tags:
  - Java
  - SpringBoot
  - ELK
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191609739.png'
toc: true
abbrlink: 7444ad67
date: 2023-02-27 16:08:38
---

在现代微服务架构中，日志的集中管理和可视化分析对于问题排查与系统监控至关重要。本文详尽介绍了如何在 Spring Boot 项目中整合
ELK（Elasticsearch、Logstash、Kibana）日志系统。通过引入 Logstash Encoder 依赖并配置 logback 文件，实现了应用日志以 JSON 格式输出到
Logstash，再传送至 Elasticsearch 中进行存储与搜索。文章还提供了 Kibana 的详细配置步骤，包括索引模板与模式的创建，使我们可以直观地在
Web 页面上查看日志信息。同时，还补充了如何按日志级别将信息分别归档为 info、warn、error 等索引，增强了日志分类管理能力，并通过
Spring Profile 实现不同环境下的日志输出控制，提高了日志系统在生产环境中的实用性和可维护性。

<!-- more -->

logback日志配置
---

1. 引入logstash依赖
    ```
    <dependency>
      <groupId>net.logstash.logback</groupId>
      <artifactId>logstash-logback-encoder</artifactId>
      <version>7.0.1</version>
    </dependency>
    ```

2. 配置logback配置文件
    ```
    <?xml version="1.0" encoding="UTF-8"?>
    <configuration debug="false">
    
        <!-- 控制台输出 -->
        <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
            <!--控制台只打印info以上级别的日志-->
            <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
                <level>info</level>
            </filter>
            <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
                <!-- 1格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
                <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
                <charset>utf-8</charset>
            </encoder>
        </appender>
    
        <!--logstash配置-->
        <appender name="logstash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
            <destination>liboshuai.com:14560</destination>
            <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
                <level>INFO</level>
            </filter>
            <!-- 日志输出编码 -->
            <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
                <providers>
                    <timestamp>
                        <timeZone>UTC</timeZone>
                    </timestamp>
                    <pattern>
                        <pattern>
                            {
                            "logLevel": "%level",
                            "serviceName": "${springAppName:-}",
                            "pid": "${PID:-}",
                            "thread": "%thread",
                            "class": "%logger{40}",
                            "rest": "%message"
                            }
                        </pattern>
                    </pattern>
                </providers>
            </encoder>
            <!--<encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>-->
        </appender>
    
    
        <!-- 日志输出级别 -->
        <root level="DEBUG">
            <appender-ref ref="stdout" />
            <appender-ref ref="logstash" />
        </root>
    </configuration>
    ```

3. 配置kibana

   首先，进入索引管理页面，过程如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191611004.png)

   在索引管理页面点击 “索引管理”，之后点击索引模板选项卡，此时页面中会出现“创建模板” 按钮，如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191611505.png)

   进入创建模板页面后，输入模板名称和索引模式，名称可以随便写，索引模式则需要与之前 Logstash
   实例中配置的索引有一定的关系。如下图所示，这里笔者就配置了模板名称为 pulasr-logs，而索引模式则为 pulars-*，这样就可以匹配到
   Elastic Search 中的索引了，比如 pulars-2023.10.24、pulars-2024.02.14、pulars-2025.02.07 这些索引都会被当前配置的索引模式匹配进来。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191611286.png)

   接着，就可以点击页面下方的 “下一步” 按钮即可，后续几个页面也可以不用刻意配置，直接点击下一步即可，最后点击 “创建模板”
   按钮完成该步骤。

   然后，需要再配置索引模式，这一步主要是为了对前一个步骤中创建的索引模板做匹配。

   首先，在索引管理页面点击 “索引模式” 进入 Kibana 的索引模式页面，之后点击 “创建索引模式” 按钮，如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191612150.png)

   在索引模式创建页面，填入名称，即上一个步骤中创建的索引模板名称。如果没有输入错误，此时页面右侧就会出现这个名称所能匹配到的索引源，这里展示的索引就是
   Logstash 实例输出给 Elastic Search 实例的数据，Kibana 会根据 “pulars-*” 去 Elastic Search
   实例中搜索并进行匹配，匹配成功的会出现在页面右侧。当然，如果 Elastic Search
   实例中有多个索引与这个名称匹配，则这些索引都会出现页面右侧的列表中。之后，选择时间戳字段，最后点击 “创建索引模式”
   按钮就完成创建了，如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191612563.png)

   当然，在这个步骤中，如果 Kibana 实例通过读取 Elastic Search 实例中的索引，发现并没有匹配到索引，是无法完成创建的。而恰好，刚刚我们已经启动了
   spring-boot-elk 项目，日志会通过 Logstash 实例打入到 Elastic Search 实例中。这时，Elastic Search 实例中是创建了一个名称为
   pulars-xxxx.xx.xx 的索引的，比如 pulars-2023.01.10，spring-boot-elk 项目输出到日志都会被保存在这个索引中。所以，为了测试流程顺利，在配置索引模式前，先使用程序通过
   Logstash 实例向 Elastic Search 实例输出一些日志，这样，索引也创建了，日志数据也有了，索引模式也能够顺利地添加了。

   > PS：只需要保证 logback 配置文件中 logstash 的 ip 和端口正确，并且顺利启动 spring-boot-elk 项目就可以了，Spring Boot
   项目的启动日志也会被 Logstash 实例收集并且输出给 Elastic Search 实例的。对的，只需要启动一下 spring-boot-elk 项目就可以。

   索引模式创建完成后的页面如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191612664.png)

4. 通过kibana查看日志

   好的，代码也写好了，Kibana 配置也完成了。接下来就去查看一下日志数据吧，日志浏览页面如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191612044.png)

   当然，如果在浏览器地址栏中访问 / elk-logs 地址，输出的三条不同级别的测试日志也会实时的出现在 Kibana 页面中，如下图所示。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191612747.png)

   日志信息在 Kibana 网页中已经能够正常显示了，日志中心的搭建和基础整合就完成了。

日志配置补充
---
如果我们需要区别日志级别并且不希望dev环境的日志也推送到elk平台，可以按照下面的步骤来设置logstash和kibana。

1. 修改log_to_es.conf配置文件为下面内容。
    ```
    input{
      tcp {
        mode => "server"
        host => "0.0.0.0"
        port => 4560
        codec => json_lines
        type=> "infoLogs"
      }
      tcp {
        mode => "server"
        host => "0.0.0.0"
        port => 4561
        codec => json_lines
        type=> "warnLogs"
      }
      tcp {
        mode => "server"
        host => "0.0.0.0"
        port => 4562
        codec => json_lines
        type=> "errorLogs"
      }
    }
    output{
      if[type] == "infoLogs"{
        elasticsearch{
          hosts=>["127.0.0.1:8200"]
          user => "elastic"
          password => "Rongshu@2023"
          index => "pulsar-info-%{+YYYY.MM.dd}"
        }
      }
      if[type] == "warnLogs"{
        elasticsearch{
          hosts=>["127.0.0.1:8200"]
          user => "elastic"
          password => "Rongshu@2023"
          index => "pulsar-warn-%{+YYYY.MM.dd}"
        }         
      }
      if[type] == "errorLogs"{
        elasticsearch{
          hosts=>["127.0.0.1:8200"]
          user => "elastic"
          password => "Rongshu@2023"
          index => "pulsar-error-%{+YYYY.MM.dd}"
        }         
      }
    }
    ```
2. 修改logback配置文件为下面内容。
    ```
   <?xml version="1.0" encoding="UTF-8"?>
   <configuration debug="false">

       <!-- 控制台输出 -->
       <appender name="stdout" class="ch.qos.logback.core.ConsoleAppender">
           <!--控制台只打印info以上级别的日志-->
           <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
               <level>info</level>
           </filter>
           <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
               <!-- 1格式化输出：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度%msg：日志消息，%n是换行符-->
               <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
               <charset>utf-8</charset>
           </encoder>
       </appender>

       <!--logstash配置info级别的日志-->
       <appender name="logstash_info" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
           <destination>liboshuai.com:14560</destination>
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>info</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
           <!-- 日志输出编码 -->
           <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
               <providers>
                   <timestamp>
                       <timeZone>UTC</timeZone>
                   </timestamp>
                   <pattern>
                       <pattern>
                           {
                           "logLevel": "%level",
                           "serviceName": "${springAppName:-}",
                           "pid": "${PID:-}",
                           "thread": "%thread",
                           "class": "%logger{40}",
                           "rest": "%message"
                           }
                       </pattern>
                   </pattern>
               </providers>
           </encoder>
           <!--<encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>-->
       </appender>

       <!--logstash配置warn级别的日志-->
       <appender name="logstash_warn" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
           <destination>liboshuai.com:14561</destination>
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>warn</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
           <!-- 日志输出编码 -->
           <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
               <providers>
                   <timestamp>
                       <timeZone>UTC</timeZone>
                   </timestamp>
                   <pattern>
                       <pattern>
                           {
                           "logLevel": "%level",
                           "serviceName": "${springAppName:-}",
                           "pid": "${PID:-}",
                           "thread": "%thread",
                           "class": "%logger{40}",
                           "rest": "%message"
                           }
                       </pattern>
                   </pattern>
               </providers>
           </encoder>
           <!--<encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>-->
       </appender>

       <!--logstash配置error级别的日志-->
       <appender name="logstash_error" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
           <destination>liboshuai.com:14562</destination>
           <filter class="ch.qos.logback.classic.filter.LevelFilter">
               <level>error</level>
               <onMatch>ACCEPT</onMatch>
               <onMismatch>DENY</onMismatch>
           </filter>
           <!-- 日志输出编码 -->
           <encoder charset="UTF-8" class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
               <providers>
                   <timestamp>
                       <timeZone>UTC</timeZone>
                   </timestamp>
                   <pattern>
                       <pattern>
                           {
                           "logLevel": "%level",
                           "serviceName": "${springAppName:-}",
                           "pid": "${PID:-}",
                           "thread": "%thread",
                           "class": "%logger{40}",
                           "rest": "%message"
                           }
                       </pattern>
                   </pattern>
               </providers>
           </encoder>
           <!--<encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>-->
       </appender>

       <!-- dev配置 -->
       <springProfile name="dev">
           <!-- root -->
           <root level="DEBUG">
               <appender-ref ref="stdout" />
           </root>
       </springProfile>

       <!-- prod配置 -->
       <springProfile name="prod">
           <!-- root -->
           <root level="INFO">
               <appender-ref ref="logstash_info"/>
               <appender-ref ref="logstash_warn"/>
               <appender-ref ref="logstash_error"/>
           </root>
       </springProfile>
   </configuration>
    ```
3. 按照上面的方式，分别配置三个`pulsar-info-*`、 `pulsar-warn-*`、 `pulsar-error-*`的索引管理和索引模板。最后成效如下图：

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191614826.png)

4. 更好的查看信息

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191614314.png)

结语
---
日志不仅承载着系统的运行历史，更是保障服务稳定与业务连续的重要基础。通过本文所述 Spring Boot 与 ELK
的整合流程，不仅可以有效降低日志分析与排查的门槛，还能提升整个系统的运维效率与智能化水平。无论是项目开发期还是部署上线后，通过
Kibana 对日志实时观察与分析，都能快速定位问题、优化性能。此外，分环境输出和分级别索引也为复杂业务系统带来了灵活的日志管理能力。未来可进一步结合
AlertManager 或 ElastAlert 实现告警系统，构建更完善的可观测性平台，助力系统稳定演进。