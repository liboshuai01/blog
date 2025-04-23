---
title: Jenkins+Docker实现CI/CD流水线部署SpringBoot项目实战指南
tags:
  - 环境搭建
  - Linux
  - Jenkins
  - Docker
categories:
  - 环境搭建
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232324210.png'
toc: true
abbrlink: 43ff9685
date: 2023-12-06 23:22:26
---

随着企业应用的不断发展，持续集成与持续部署（CI/CD）已成为提升开发效率和保障软件质量的重要手段。

本文针对CentOS 7环境，详细讲解了Docker、Docker-Compose、Harbor、Jenkins的安装和配置过程，并结合实际项目演示了如何通过Jenkins流水线实现代码自动构建、打包镜像、推送到私有仓库以及远程服务器自动部署启动。

通过本篇博文，读者不仅能够掌握基础环境搭建，更能实际应用Jenkins实现完整的自动化交付流程，为后续项目的持续迭代奠定坚实基础。

<!-- more -->

环境准备
---
- Centos 7
- Docker 24.0.7
- Docker-Compose 2.21.0
- Docker-Harbor 2.9.1
- Jenkins 2.434
- Jdk 8
- Maven 3.9.1
- Git 2.39.2

Docker、Docker-Compose安装
---
查看我另外的文章：[# Centos离线安装Docker、Docker-compose](https://juejin.cn/post/7307857584882139174)

Docker-Harbor 安装
---
查看我另外的文章：[# Centos 离线安装 Docker-Harbor](https://juejin.cn/post/7307857584882450470)

Jenkins 安装
---
查看我另外的文章：[# Docker安装Jenkins，并容器化](https://juejin.cn/post/7309352577145995301)


部署配置
---
1. Jenkins 安装插件 `Publish Over SSH`、`Gitee`或者`Gitlab`。

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318035.png)

2. Jenkins 系统配置->全局工具配置，配置Maven、Git、Jdk

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318231.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318056.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318995.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318988.png)

3. Jenkins 系统管理 -> 系统配置，配置 `Publish over SSH`

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318061.png)

4. 新建任务 -> 流水线

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318403.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318375.png)

5. 任务配置

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318434.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318481.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318492.png)

6. 根据前面 `脚本路径` 在项目中创建 `Jenkinsfile` 文件

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318530.png)

   > 添加下面内容作为初始模板

    ```
    pipeline {
        agent any

        stages {
              stage('第一步') {
                  steps {
                      echo "第一步"
                  }
              }

              stage('第二步') {
                    steps {
                        echo "第二步"
                    }
              }

              stage('第三步') {
                    steps {
                        echo "第三步"
                    }
              }

              stage('第四步') {
                    steps {
                        echo "第四步"
                    }
              }
        }
    }
    ```

7. 根据 Jenkins 的流水线语法提示生成`Jenkinsfile`脚本内容

   > 进入`流水线语法`，通过`Jenkins`流水线生成模板生成

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318714.png)

   > 从`Gitee`拉取代码

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318691.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318750.png)

   > Maven构建打包项目

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318798.png)

   > 打包镜像并上传Harbor

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318806.png)

   > 拉取镜像并启动项目

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318785.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318971.png)

8. 最终生成的`Jenkinsfile`文件内容为
    ```
    pipeline {
        agent any
    
        stages {
              stage('从Gitee拉取代码') {
                  steps {
                      checkout scmGit(branches: [[name: '*/master']], extensions: [], userRemoteConfigs: [[credentialsId: '1', url: 'https://gitee.com/liboshuai01/xxl-job.git']])
                  }
              }
    
              stage('Maven构建打包项目') {
                    steps {
                        sh '/opt/module/maven/apache-maven-3.9.2/bin/mvn clean install -DskipTests'
                    }
              }
    
              stage('打包镜像并上传Harbor') {
                    steps {
                        sh '''cd xxl-job-admin
                        docker build -t xxl-job:latest ./
                        docker login -u admin -p Rongshu@2024 10.0.0.38:8004
                        docker tag xxl-job:latest 10.0.0.38:8004/library/xxl-job:latest
                        docker push 10.0.0.38:8004/library/xxl-job:latest
                        docker image prune -f'''
                    }
              }
    
              stage('拉取镜像并启动项目') {
                    steps {
                        sshPublisher(publishers: [sshPublisherDesc(configName: 'root@one', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '''#!/bin/bash
    
                        source /etc/profile
                        cd /opt/projects/xxl-job
                        docker login -u admin -p Rongshu@2024 10.0.0.38:8004
                        docker pull 10.0.0.38:8004/library/xxl-job:latest
                        docker image prune -f
                        docker-compose up -d''', execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                    }
              }
        }
    }
    ```

9. 项目中用到的`Dockerfile`内容如下
    ```
    FROM openjdk:8-jre-slim
    LABEL author="boshuai.li"
    
    ENV PARAMS=""
    
    ENV TZ=PRC
    RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
    
    ADD target/xxl-job-admin-*.jar /app.jar
    
    ENTRYPOINT ["sh","-c","java -jar $JAVA_OPTS /app.jar $PARAMS"]
    ```

10. 项目中用到的`docker-compose.yaml`内容如下
    ```
    version: '3'
    services:
      xxl_job:
        image: 10.0.0.38:8004/library/xxl-job:latest
        container_name: xxl-job
        restart: on-failure:3
        volumes:
          - /opt/projects/xxl-job/logs:/opt/projects/xxl-job/logs
        ports:
          - 8088:8088
    ```

部署验证
---
1. 点击`立即构建`，可以看到下图

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318980.png)

2. 查看`xxl-job`容器服务是否启动

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318113.png)

3. 查看`Harbor`仓库中是否有推送的镜像

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504232318125.png)

## 结语

通过本文的实践操作，我们完整搭建了基于Docker和Jenkins的自动化部署环境，实现了代码从拉取、构建、打包、上传到镜像仓库，再到远程服务器自动更新启动的全流程自动化。

该流程大幅提升了开发与运维协作效率，减少了人为操作风险，保障了应用的快速交付与稳定运行。

希望本文内容能为您的项目持续集成和自动部署提供实用指导，共同推动软件交付能力的升级和优化。