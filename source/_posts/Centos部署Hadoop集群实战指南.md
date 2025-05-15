---
title: Centos部署Hadoop集群实战指南
tags:
  - Linux
  - Hadoop
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408113518754.png'
toc: true
abbrlink: '62974551'
date: 2023-10-10 15:30:00
---

这篇文章详细记录了如何在三台CentOS7服务器上搭建一个稳定运行的Hadoop集群。

首先，从环境准备开始，作者列举了完成集群部署所需的基础配置，包括关闭防火墙、配置免密登录、时钟同步以及安装JDK环境等操作，确保每台服务器都满足集群运行的条件。

接着，根据集群角色分配规则，作者对Hadoop的核心配置文件进行了详细的修改，包括`core-site.xml`、`hdfs-site.xml`、`mapred-site.xml`和`yarn-site.xml`。通过这些配置，实现了对集群内各节点的分工，如NameNode、ResourceManager以及DataNode和NodeManager等。

之后，文章介绍了Hadoop安装包的分发以及环境变量的配置，并说明了启动集群时需要进行的第一次格式化`hdfs`操作。随后，提供了具体的启动步骤，并引导用户通过Java进程检查和Web管理页面验证集群是否正常运行。

最后，文章还对常见问题进行了分析，如NameNode未启动的故障处理，结合日志排查进行了详细的指导，为集群问题提供了解决思路。无论是搭建、管理还是故障排查，本篇文章都提供了完整的流程与实用性技巧，是一份极其重要的参考文档。

<!-- more -->

## 环境要求

需要有三台 Centos7 服务器，并都需要完成下面的配置要求：
- 关闭防火墙
- 新建普通用户`me`
- 阿里云时钟同步服务器
- 配置免密登陆
- 关闭`selinux`
- 配置`xsync`、`xcall`同步脚本
- 配置`jdk8`环境
- 确保端口`8020,50090,50070,10020,19888,8088`端口没有被占用

> 环境准备可以参考我下面的博文：
>
> [centos防火墙常用命令](https://juejin.cn/post/7178874541744062522)
>
> [centos新建普通用户](https://juejin.cn/post/7357917741908787215)
>
> [centos系统时间同步](https://juejin.cn/post/7357917741908656143)
>
> [centos配置免密登录](https://juejin.cn/post/7277395904217939968)
>
> [centos关闭SElinux](https://juejin.cn/post/7322518787424305162)
>
> [centos配置xsync和xcall同步脚本](https://juejin.cn/post/7295962144750813221)
>
> [centos安装jdk8](https://juejin.cn/post/7173667982051606558)

## 集群规则

> 说明：
> 1. 除特别说明外，本文的所有操作均在master节点、使用me这个非root用户执行
> 2. 命令中出现的IP域名，均需要替换为自己集群中的IP域名【必须】
> 3. 命令中出现的`/home/lbs/software`路径，可选择替换为自定义路径【可选】

<table>
    <thead>
        <tr>
            <th>服务器 IP域名</th>
            <th>master</th>
            <th>node1</th>
            <th>node2</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>HDFS</td>
            <td>NameNode</td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td>HDFS</td>
            <td>SecondaryNameNode</td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td>HDFS</td>
            <td>DataNode</td>
            <td>DataNode</td>
            <td>DataNode</td>
        </tr>
        <tr>
            <td>YARN</td>
            <td>ResourceManager</td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td>YARN</td>
            <td>NodeManager</td>
            <td>NodeManager</td>
            <td>NodeManager</td>
        </tr>
        <tr>
            <td>历史日志服务器</td>
            <td>JobHistoryServer</td>
            <td></td>
            <td></td>
        </tr>
    </tbody>
</table>

## 安装集群

1. 从百度云下载资源包`hadoop-2.6.0-cdh5.14.2_after_compile.tar.gz`，并上传到`master`服务器
    ```shell
    https://pan.baidu.com/s/1nSUqi50p5u0skUAn4A-i-A?pwd=z8d7
    ```

2. 解压`hadoop-2.6.0-cdh5.14.2_after_compile.tar.gz`到指定安装路径，并重命名目录为`hadoop`
    ```shell
    tar -zxvf hadoop-2.6.0-cdh5.14.2_after_compile.tar.gz -C /home/lbs/software
    mv /home/lbs/software/hadoop-2.6.0-cdh5.14.2 /home/lbs/software/hadoop
    ```

3. 进入到`/home/lbs/software/hadoop`目录，执行下面命令检查环境配置是否符合要求
   ```shell
   [me@master hadoop]$ ./bin/hadoop checknative
   
    24/04/15 17:09:57 INFO bzip2.Bzip2Factory: Successfully loaded & initialized native-bzip2 library system-native
    24/04/15 17:09:57 INFO zlib.ZlibFactory: Successfully loaded & initialized native-zlib library
    Native library checking:
    hadoop:  true /home/lbs/software/hadoop/lib/native/libhadoop.so.1.0.0
    zlib:    true /lib64/libz.so.1
    snappy:  true /lib64/libsnappy.so.1
    lz4:     true revision:10301
    bzip2:   true /lib64/libbz2.so.1
    openssl: true /lib64/libcrypto.so
   ```
   > 如果有`false`项，则需要使用`yum`进行安装，直到全部为`true`

4. 配置`hadoop-env.sh`文件
    ```shell
    vim /home/lbs/software/hadoop/etc/hadoop/hadoop-env.sh
    
    # 注释掉之前的内容，新增实际的jdk路径
    # export JAVA_HOME=${JAVA_HOME}
    export JAVA_HOME=/app/jdk
    ```

5. 配置`core-site.xml`
    ```shell
    echo '<?xml version="1.0" encoding="UTF-8"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://master:8020</value>
        </property>
        <property>
            <name>hadoop.tmp.dir</name>
            <value>/home/lbs/software/hadoop/hadoopDatas/tempDatas</value>
        </property>
        <!-- 缓冲区大小，实际工作中根据服务器性能动态调整 -->
        <property>
            <name>io.file.buffer.size</name>
            <value>4096</value>
        </property>
        <!-- 开启hdfs的垃圾桶机制，删除掉的数据可以从垃圾桶中回收，单位分钟 -->
        <property>
            <name>fs.trash.interval</name>
            <value>10080</value>
        </property>
    </configuration>' > /home/lbs/software/hadoop/etc/hadoop/core-site.xml
    ```

6. 配置`hdfs-site.xml`
    ```shell
    echo '<?xml version="1.0" encoding="UTF-8"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <!-- NameNode存储元数据信息的路径，实际工作中，一般先确定磁盘的挂载目录，然后多个目录用，进行分割   -->
        <property>
            <name>dfs.namenode.secondary.http-address</name>
            <value>master:50090</value>
        </property>
        <property>
            <name>dfs.namenode.http-address</name>
            <value>master:50070</value>
        </property>
        <property>
            <name>dfs.namenode.name.dir</name>
            <value>file:///home/lbs/software/hadoop/hadoopDatas/namenodeDatas</value>
        </property>
        <!-- 定义dataNode数据存储的节点位置，实际工作中，一般先确定磁盘的挂载目录，然后多个目录用，进行分割 -->
        <property>
            <name>dfs.datanode.data.dir</name>
            <value>file:///home/lbs/software/hadoop/hadoopDatas/datanodeDatas</value>
        </property>
        <property>
            <name>dfs.namenode.edits.dir</name>
            <value>file:///home/lbs/software/hadoop/hadoopDatas/dfs/nn/edits</value>
        </property>
        <property>
            <name>dfs.namenode.checkpoint.dir</name>
            <value>file:///home/lbs/software/hadoop/hadoopDatas/dfs/snn/name</value>
        </property>
        <property>
            <name>dfs.namenode.checkpoint.edits.dir</name>
            <value>file:///home/lbs/software/hadoop/hadoopDatas/dfs/nn/snn/edits</value>
        </property>
        <property>
            <name>dfs.replication</name>
            <value>3</value>
        </property>
        <property>
            <name>dfs.permissions</name>
            <value>false</value>
        </property>
        <property>
            <name>dfs.blocksize</name>
            <value>134217728</value>
        </property>
        <!--hdfs web端ui取消ip限制-->
        <property>
            <name>dfs.namenode.http-address</name>
            <value>0.0.0.0:50070</value>
        </property>
    </configuration>' > /home/lbs/software/hadoop/etc/hadoop/hdfs-site.xml
    ```
7. 配置`mapred-site.xml`
    ```shell
    echo '<?xml version="1.0"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
        <property>
            <name>mapreduce.job.ubertask.enable</name>
            <value>true</value>
        </property>
        <property>
            <name>mapreduce.jobhistory.address</name>
            <value>master:10020</value>
        </property>
        <property>
            <name>mapreduce.jobhistory.webapp.address</name>
            <value>0.0.0.0:19888</value>
        </property>
    </configuration>' > /home/lbs/software/hadoop/etc/hadoop/mapred-site.xml
    ```

8. 配置`yarn-site.xml`
    ```
    <?xml version="1.0"?>
    <configuration>
        <!-- 节点管理器可用的 CPU 核心总数 -->
        <property>
            <name>yarn.nodemanager.resource.cpu-vcores</name>
            <value>12</value>
        </property>
    
        <!-- 调度器可以分配给一个容器的最大 vcore 数 -->
        <property>
            <name>yarn.scheduler.maximum-allocation-vcores</name>
            <value>4</value>
        </property>
    
        <!-- 调度器可以分配给一个容器的最小 vcore 数 -->
        <property>
            <name>yarn.scheduler.minimum-allocation-vcores</name>
            <value>1</value>
        </property>
    
        <!-- 节点管理器可用的内存总量（MB） -->
        <property>
            <name>yarn.nodemanager.resource.memory-mb</name>
            <value>34000</value>
        </property>
    
        <!-- 调度器可以分配给一个容器的最大内存量（MB） -->
        <property>
            <name>yarn.scheduler.maximum-allocation-mb</name>
            <value>34000</value>
        </property>
    
        <!-- 调度器可以分配给一个容器的最小内存量（MB） -->
        <property>
            <name>yarn.scheduler.minimum-allocation-mb</name>
            <value>1024</value>
        </property>
    
        <!-- 资源管理器的主机名 -->
        <property>
            <name>yarn.resourcemanager.hostname</name>
            <value>one</value>
        </property>
    
        <!-- 节点管理器的辅助服务（例如 MapReduce Shuffle） -->
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
        </property>
    
        <!-- 资源管理器 Web 界面的访问地址 -->
        <property>
            <name>yarn.resourcemanager.webapp.address</name>
            <value>0.0.0.0:8088</value>
        </property>
    </configuration>
    ```

9. 配置`slaves`
    ```shell
    echo 'master
    node1
    node2' > /home/lbs/software/hadoop/etc/hadoop/slaves
    ```

10. 创建所需目录
    ```shell
    mkdir -p /home/lbs/software/hadoop/hadoopDatas/tempDatas
    mkdir -p /home/lbs/software/hadoop/hadoopDatas/namenodeDatas
    mkdir -p /home/lbs/software/hadoop/hadoopDatas/datanodeDatas 
    mkdir -p /home/lbs/software/hadoop/hadoopDatas/dfs/nn/edits
    mkdir -p /home/lbs/software/hadoop/hadoopDatas/dfs/snn/name
    mkdir -p /home/lbs/software/hadoop/hadoopDatas/dfs/nn/snn/edits
    ```

11. 分发`/home/lbs/software/hadoop`文件夹到集群中的其他两台机器
    ```shell
    xsync /home/lbs/software/hadoop
    ```
12. 分别登录到集群中三台机器，都配置`hadoop`到环境变量中
    ```shell
    sudo vim /etc/profile
    # Hadoop
    export HADOOP_HOME=/home/lbs/software/hadoop
    export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
    
    # 最后重新加载环境变量
    source /etc/profile
    ```

## 启动集群

1. 首先格式化hdfs（仅第一次启动时需要！后续启动请勿重复执行）
    ```shell
    hdfs namenode -format
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202503181059046.jpg)

2. 依次启动`hdfs`、`yarn`、`historyserver`
   > 注意：下面的三条命令逐一执行，而不是一下全部执行
    ```shell
    start-dfs.sh 
    start-yarn.sh 
    mr-jobhistory-daemon.sh start historyserver
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202503181059523.jpg)

## 集群验证

1. 首先验证各节点Java进程情况
    ```shell
    [me@master bin]$ xcall jps
    ================current host is master=================
    --> execute command "jps"
    25985 JobHistoryServer
    24419 DataNode
    26261 Jps
    24219 NameNode
    25292 ResourceManager
    25437 NodeManager
    24750 SecondaryNameNode
    Command executed successfully on master
    ================current host is node1=================
    --> execute command "jps"
    3545 DataNode
    3690 NodeManager
    3948 Jps
    Command executed successfully on node1
    ================current host is node2=================
    --> execute command "jps"
    3609 DataNode
    3754 NodeManager
    4013 Jps
    Command executed successfully on node2
    All commands executed successfully!
    ```
   > 如果发现`namenode`进程没有出现，可以到文章最后查看解决方法
2. 查看`web`管理页面

   | 名称 | 地址 |
               | --- | --- |
   | hdfs | http://master:50070 |
   | yarn | http://master:8088 |
   | jobHistory | http://master:19888 |

## 关闭集群

> 关闭服务器之前最好提前手动关闭`hadoop`集群，防止出现集群异常的情况

```shell
mr-jobhistory-daemon.sh stop historyserver
stop-yarn.sh 
stop-dfs.sh 
```

## 问题解决

**启动集群后发现`namenode`进程没有启动，怎么办？**

1. 首先停止集群
    ```shell
    mr-jobhistory-daemon.sh stop historyserver
    stop-yarn.sh 
    stop-dfs.sh 
    ```
2. 清理配置在`core-site.xml`中`<name>hadoop.tmp.dir</name>`配置项所指向的目录下的所有内容（不需要删除目录本身，只需要删除下面的内容即可）
3. 重新执行初始化`hdfs`的命令`hadoop namenode -format`
4. 最后再次启动hadoop集群
    ```shell
    start-dfs.sh 
    start-yarn.sh 
    mr-jobhistory-daemon.sh start historyserver
    ```
5. 如果还是没有解决问题，需要进入到`hadoop`根目录的`logs`目录下，查看`hadoop-admin-namenode-master.log`日志。是否是端口被占用了，还是其他问题。
