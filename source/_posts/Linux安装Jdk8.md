---
title: Linux安装Jdk8
date: 2020-07-07 07:07:07
tags: [环境搭建,Linux,Jdk]
categories: [环境搭建]
cover: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250417180441788.png
toc: true
---

## 自动安装

```shell
curl -sfL https://lbs-install.oss-cn-shanghai.aliyuncs.com/jdk/install_jdk8.sh | sh -s -- /your/install/path
```

## 手动安装

1. 下载jdk8安装包

    ```
    wget https://builds.openlogic.com/downloadJDK/openlogic-openjdk/8u442-b06/openlogic-openjdk-8u442-b06-linux-x64.tar.gz
    ```

2. 解压jdk8安装包

    ```shell
    tar -zxvf openlogic-openjdk-8u442-b06-linux-x64.tar.gz -C /home/lbs/software/jdk8
    ```

3. 配置环境变量

    ```shell
    tee -a ~/.bash_profile <<'EOF'
    
    # Java
    export JAVA_HOME=/home/lbs/software/jdk8
    export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
    export PATH=$JAVA_HOME/bin:$PATH
    EOF
    ```

4. 重新加载环境变量

    ```shell
    source ~/.bash_profile
    ```
   
5. 查看java版本

    ```shell
    java -version
    ```
    