---
title: 使用Jmeter读取Json文件对Kafka进行压力测试
tags:
  - Linux
  - Jmeter
  - Kafka
categories:
  - 杂货小铺
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425150219154.png'
toc: true
abbrlink: eaa0f0e9
date: 2024-05-17 15:00:49
---

最近因为系统开发需要，要模拟业务系统生产业务数据推送到Kafka中。同时对于生成的业务数据有一定逻辑要求，故采用了先使用代码生成测试业务数据到Json文件中，然后通过Jmeter读取Json文件以一定的并发数推送到Kafka中的方案。

<!-- more -->

## 环境准备

[# 安装JDK8并配置环境变量](https://www.runoob.com/java/java-environment-setup.html)

## `windows`步骤

1. 点击链接[ # 下载 Jmeter](https://dlcdn.apache.org//jmeter/binaries/apache-jmeter-5.6.3.tgz)，并解压到指定路径

2. [# 下载 di-kafkameter](https://github.com/rollno748/di-kafkameter/releases/download/1.0/di-kafkameter-1.0.jar) 到`Jmeter`根目录下的`lib\ext`目录下

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110331.png)

3. 进入到`Jmeter`根目录下的`bin`目录下，双击`jmeter.bat`，进入到`GUI`界面

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110288.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110302.png)

4. 设置为中文

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110324.png)

5. 新增并配置线程组，线程数设置为`100`，永远循环

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110296.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110316.png)

6. 新增并配置`Constant Throughput Timer`，用于控制并发吞吐量一分钟执行18万次，即3000TPS

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110589.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110597.png)

7. 新增并配置`CSV Data Set Config`，用于读取`Json`文件

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110636.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110654.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110659.png)

8. 新增并配置`KafkaProducerConfig`，用于配置`Kafka`集群的信息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110692.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110879.png)

9. 新增并配置`Kafka`生产者取样器，用于向`Kafka`推送消息

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110919.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110911.png)
   > 此处的`${data}`，即为前面步骤7中配置的变量。

10. 新增并配置`查看结果树`，用于测试时查看结果

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110934.png)

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110972.png)

11. 点击运行，并保存配置文件到指定路径

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110988.png)

12. 查看压力测试情况

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110180.png)

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110230.png)

13. 停止压测，并关闭GUI页面，以便使用命令行进行压测（GUI界面会影响压测的性能）

14. 配置`Jmeter`到系统环境变量中

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260110240.png)

15. 打开终端，使用命令行进行压测
    ```bat
    jmeter -n -t D:\APerson\Software\apache-jmeter-5.6.3\bin\查看结果树.jmx -l D:\APerson\Software\apache-jmeter-5.6.3\report\02-result.csv -j D:\APerson\Software\apache-jmeter-5.6.3\report\02-log.log
    ```
    > 1. ` D:\APerson\Software\apache-jmeter-5.6.3\bin\查看结果树.jmx`为第11步骤保存的配置文件
    > 2. `D:\APerson\Software\apache-jmeter-5.6.3\report\02-result.csv` 为测试结果生成路径（文件目录不存在会自动创建）
    > 3. `D:\APerson\Software\apache-jmeter-5.6.3\report\02-log.log` 为测试日志生成路径（文件目录不存在会自动创建）


    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260116446.png)

## `Centos`步骤

1. windows上通过`GUI`界面配置好`xxx.jmx`文件（记得修改Json文件路径为Centos上的路径），并上传到Centos服务器上
2. 打包windows上的Jmeter目录上传到Centos服务器上解压
3. 在Centos系统上配置Jmeter环境变量`vim /etc/profile`，追加内容如下，配置完成后执行`source /etc/profile`
    ```shell
    # Jmeter
    export JMETER_HOME=/home/lbs/software/jmeter
    export CLASSPATH=$JMETER_HOME/lib/ext/ApacheJMeter_core.jar:$JMETER_HOME/lib/jorphan.jar:$CLASSPATH
    export PATH=$JMETER_HOME/bin:$PATH:$HOME/bin
    ```
4. 使用命令行启动压测
    ```shell
    jmeter -n -t /home/lbs/software/jmeter/conf/Kafka压力测试.jmx -l /home/lbs/software/jmeter/result/01-result.csv -j /home/lbs/software/jmeter/result/01-log.log
    ```

## 结语

搞定，收工！