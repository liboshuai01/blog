---
title: Linux安装Jdk8
tags:
  - 环境搭建
  - Linux
  - Jdk
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250417180441788.png'
toc: true
abbrlink: 42f5694
date: 2022-07-07 07:07:07
---

本篇博客详细介绍了在 Linux 系统中安装 JDK8 的两种方式：自动安装和手动安装。通过提供一键命令的脚本方式，用户可快速完成 JDK8 的部署；同时也指南了从官方下载 OpenJDK 安装包、解压并配置环境变量的完整手动安装流程，适合需要更高自定义配置的开发人员。文章适用于需要搭建 Java 环境的初学者和后端开发者，为后续开发基于 Java 的应用打下坚实基础。安装完成后，通过简单命令检测是否配置成功，确保开发环境无误。

<!-- more -->

## 自动安装

> 注意: 请修改脚本中的 `/your/install/path` 为您所需要的安装路径。

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
    