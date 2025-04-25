---
title: 基于Gitee、Docker、Jenkins、SonarQube和Harbor的最简CI/CD流水线搭建
tags:
  - Linux
  - Docker
  - CI/CD
  - Jenkins
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105925772.png'
toc: true
abbrlink: 8b4c4312
date: 2024-04-16 10:57:37
---

本文最终实现的效果是：只需要在idea中提交代码到git远程仓库后，登陆Jenkins点一下“立即构建”，即可可自动触发CI/CD流水线进行自动部署项目。

博主在网上教程的基础上，进行了进一步封装、简化了CI/CD的部署搭建教程。特点如下：

*   使用的docker、jenkins等组件版本都是最新的。
*   各个组件的镜像版本都是固定版本号的，不会因为后面官方更新镜像，导致教程失效。
*   编写shell脚本，帮助用户可以快速安装、配置各个组件（jenkins的插件、镜像、jenkins中使用docker等都已经帮助用户完成了）
*   没有任何坑（坑我的已经踩过了，跟着我的教程，百分百可以一遍过。

<!-- more -->

![jenkins.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105636758.png)

## 环境准备

至少需要一台 Centos7 服务器，并都需要完成下面的配置要求：

- `docker`、 `docker-compose` 部署并配置完毕
- 新建普通用户`lbs`，并给予`docker`命令执行权限、`sudo`命令免密执行权限

> 环境准备可以参考我下面的博文：
>
> [Centos在线安装指定版本Docker](https://juejin.cn/post/7369023361953841178)
>
> [Centos安装指定版本docker-compose](https://juejin.cn/post/7178879761036279866)
>
> [快速配置docker](https://juejin.cn/spost/7351642274272444426)
>
> [centos系统上新建用户](https://juejin.cn/post/7357917741908787215)
>
> [Centos给普通用户添加sudo命令权限](https://juejin.cn/post/7348309454700068873)

## 组件规划

版本说明：

*   docker: 25.0.5
*   docker-compose: v2.26.1
*   jenkins: latest
*   sonarqube: 10.4.1
*   harbor: v2.10.1
*   centos：7

| 名称      | 节点hostname | 节点IP    |
| --------- | ------------ | --------- |
| harbor    | master       | 10.0.0.87 |
| sonarqube | master       | 10.0.0.87 |
| jenkins   | master       | 10.0.0.87 |

> 说明：
> 1. 除特别说明外，本文的所有操作均在master节点、使用lbs这个非root用户执行（此用户需要有免密执行sudo命令的权限）【必须】
> 2. 命令中出现的IP，均需要替换为自己机器的IP【必须】
> 3. 命令中出现的/home/lbs/software路径，可选择替换为自定义路径【可选】

## 组件搭建

### 部署harbor

> 部署harbor推荐使用root用户进行创建管理，不然会有权限问题

1.  在需要部署`harbor`的路径中下载在线安装包与解压
    ```shell
    mkdir -p /root/software/harbor
    cd /root/software/harbor
    wget https://all-share-public.oss-cn-shanghai.aliyuncs.com/install-harbor/harbor-online-installer-v2.10.1.tgz
    tar -zxvf harbor-online-installer-v2.10.1.tgz
    cd harbor
    ```

2.  配置`harbor.yml`文件
    ```shell
    cp harbor.yml.tmpl harbor.yml
    
    # 只展示变动的地方
    hostname: 10.0.0.87 # 设置为指定部署机器IP 
    http: 
      port: 8023 # 从80修改为指定端口（可选）
    # https: # 注释 https 相关的配置
      # port: 443
      # certificate: /your/certificate/path
      # private_key: /your/private/key/path
    ......
    harbor_admin_password: 12345678 # admin用户指定密码 （可选）
    ......
    data_volume: /root/software/harbor/data # 数据存储目录（可选）
    ......
    log: 
      ......
      local: 
      ......
        location: /root/software/harbor/logs # 日志目录（可选）
    ```

3.  正式安装
    ```shell
    ./prepare
    ./install.sh
    ```

4.  浏览器打开`http://本地IP:8023`，新建镜像仓库，用于后面Jenkins中配置使用

    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105636779.png)

    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105636824.png)

5.  配置Docker镜像仓库为Harbor地址
    ```shell
    vim /etc/docker/daemon.json
    
    # 添加以下内容
    {
      ......
      "insecure-registries":["10.0.0.87:8023"]
      .....
    }
    ```

6.  重启docker
    ```shell
    systemctl daemon-reload 
    systemctl restart docker
    ```

7.  登陆harbor
    ```shell
    docker login 10.0.0.87:8023
    ```
    > 输入账号密码后，提示登陆成功，表示harbor安装适配成功

### 部署sonarqube

1.  在需要部署`sonarqube`的目录中执行下面的命令即可

    ```shell
    mkdir -p /home/lbs/software/sonarqube
    cd /home/lbs/software/sonarqube
    wget https://all-share-public.oss-cn-shanghai.aliyuncs.com/install-sonarqube/sonarqube.tar
    tar -mxvf sonarqube.tar && cd sonarqube && ./install.sh
    ```

2. 浏览器打开`http://IP:8021`，输入默认账号密码`admin/admin`

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105636882.png)

3. 创建token令牌

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105636901.png)
   > 生成的token值要记录下来，下面配置Jenkinsfile流水线时需要使用。

### 部署jenkins

> Jenkins 如果采用在线安装插件的方式，那么最好安装最新的Jenkins，不然在线安装插件会失败（血的教训）。

1.  在需要部署jenkins的目录中执行下面的命令即可

    ```shell
    mkdir -p /home/lbs/software/jenkins
    cd /home/lbs/software/jenkins
    wget https://all-share-public.oss-cn-shanghai.aliyuncs.com/install-jenkins/jenkins.tar
    tar -mxvf jenkins.tar && cd jenkins && ./install.sh
    ```

2.  重启`jenkins`后，浏览器打开`http://IP:8020`，出现Jenkins页面即可

## 配置CI/CD流水线

### 进入Jenkins

1.  浏览器打开jenkins页面`http://IP:8020`
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105636860.png)
2.  执行命令`docker-compose logs -f jenkins`查看jenkins初始密码
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105639111.png)
3.  复制密码到页面，然后点击继续
4.  点击“安装推荐的插件”，等待在线安装十分钟左右（插件镜像安装时，已经通过脚本配置了），然后点击继续即可
    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105641733.png)
    ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105641721.png)
5. 为了快速，我们直接使用admin账号即可
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105641792.png)
6. 后面一直下一步直到进入首页
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105641529.png)
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105641580.png)
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105641830.png)

### 插件安装

1. 安装`gitee`插件
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105643428.png)

2. 安装`publish over ssh`插件
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105647617.png)

3. 安装`dingTalk`插件

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105647908.png)

3. 插件安装完毕后，重启jenkins
   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105648874.png)

### 配置gitee凭证

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105648989.png)

> 注意配置的`gitee`项目需要在指定的项目路径下创建编写对应的`Dockerfile`文件，下面是其文件路径与内容：

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105648924.png)

```
FROM eclipse-temurin:8-jre

## 创建目录，并使用它作为工作目录
RUN mkdir -p /yudao-server
WORKDIR /yudao-server
## 将后端项目的 Jar 文件，复制到镜像中
COPY ./target/yudao-server.jar app.jar

## 设置 TZ 时区
ENV TZ=Asia/Shanghai
## 设置 JAVA_OPTS 环境变量，可通过 docker run -e "JAVA_OPTS=" 进行覆盖
ENV JAVA_OPTS="-Xms512m -Xmx512m -Djava.security.egd=file:/dev/./urandom"

## 应用参数
ENV ARGS=""

## 暴露后端项目的 48080 端口
EXPOSE 48080

## 启动后端项目
CMD java ${JAVA_OPTS} -jar app.jar $ARGS
```

### 全局工具配置

1. Maven配置文件
   > 配置值：/var/jenkins_home/maven/conf/settings.xml

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105648939.png)

2. Jdk配置
   > 配置值：/var/jenkins_home/jdk

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105650409.png)

3. Maven配置
   > 配置值：/var/jenkins_home/maven

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105650947.png)

4. 钉钉配置
   > 推荐具体使用指南查看[dingtalk-plugin文档](https://jenkinsci.github.io/dingtalk-plugin/)
   >
   > 如果需要支持『飞书』『企业微信』等通知方式，请查看[Lark Notice文档](https://721806280.github.io/lark-notice-plugin-doc/)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105651331.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105651414.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105651616.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105651647.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105651804.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105652269.png)


    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260116680.png)
    
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260116287.png)
    
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260117426.png)

### 配置Publish over SSH

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105652899.png)

### 配置流水线任务

1. 新建流水线任务

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105653971.png)

2. 基本的任务配置，如下

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105654382.png)

3. 配置参数化构建

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105654371.png)

4. 配置流水线【重点】

   找到“流水线”配置部分，将下面的Jenkinsfile文件内容粘贴进去，然后进行自身适配的修改即可。

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105654643.png)

   Jenkinsfile文件内容如下，只需要根据自身环境替换`environment`里面的变量值即可。

    ```Jenkinsfile
    pipeline {
        agent any
   
        environment {
            PROJECT_NAME="starlinkrisk-admin" // 项目名称
            PRJECT_CODE_PARENT_PATH="./starlinkRisk-admin" // 项目父模块代码路径
            PRJECT_CODE_SUB_PATH="./starlinkRisk-admin/yudao-server" // 项目子模块代码路径（如果没有子模块，则删除此变量，下面引用此变量的地方均修改为$PRJECT_CODE_PARENT_PATH）
            SERVER_PATH="/home/lbs/project/starlinkRisk/starlinkRisk-admin" // 项目部署服务器路径
            SERVER_CREDENTIALS_ID="lbs@master" // 项目部署服务器认证ID
   
            GIT_CREDENTIALS_ID="gitee-liboshuai01" // git仓库的认证凭证ID
            GIT_URL='https://gitee.com/liboshuai01/starlink-risk.git' // git仓库的url地址
   
            SONARQUEB_URL="http://10.0.0.87:8021" // sonarqube的url地址
            SONARQUBE_TOKEN="sqa_83b933dd1c448cb495a47aed25e9c6128bcac916" // sonarqube的token令牌
   
            HARBOR_USER="admin" // harbor镜像仓库的用户名
            HARBOR_PASSWORD="Rongshu@2024" // harbor镜像仓库的用户密码
            HARBOR_URL="10.0.0.87:8023" // harbor仓库的url地址
            HARBOR_REPO="starlinkrisk" // harbor仓库项目名称
   
            DING_TALK="Jenkins-DingDing" // 钉钉机器人ID
   
        }
   
        stages {
   
            stage('从Git仓库拉取代码') {
              steps {
                  checkout scmGit(branches: [[name: env.ENV_PROFILE]], extensions: [], userRemoteConfigs: [[credentialsId: env.GIT_CREDENTIALS_ID, url: env.GIT_URL]])
              }
            }
   
            stage('Maven编译构建项目') {
                steps {
                    sh 'cd $PRJECT_CODE_PARENT_PATH && /var/jenkins_home/maven/bin/mvn clean install -DskipTests -P$ENV_PROFILE'
                }
            }
   
            stage('检测代码质量') {
                steps {
                    sh """
                         /var/jenkins_home/sonar-scanner/bin/sonar-scanner \
                        -Dsonar.projectKey=$PROJECT_NAME-$ENV_PROFILE \
                        -Dsonar.projectName=$PROJECT_NAME-$ENV_PROFILE \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.sources=$PRJECT_CODE_PARENT_PATH \
                        -Dsonar.java.binaries=$PRJECT_CODE_SUB_PATH/target/ \
                        -Dsonar.host.url=$SONARQUEB_URL \
                        -Dsonar.login=$SONARQUBE_TOKEN
                       """
                }
            }
   
            stage('构建镜像到Harbor仓库') {
                steps {
                    sh '''cd $PRJECT_CODE_SUB_PATH
                        docker build -t $PROJECT_NAME:$ENV_PROFILE .
                        docker login -u $HARBOR_USER -p $HARBOR_PASSWORD $HARBOR_URL
                        docker tag $PROJECT_NAME:$ENV_PROFILE $HARBOR_URL/$HARBOR_REPO/$PROJECT_NAME:$ENV_PROFILE
                        docker push $HARBOR_URL/$HARBOR_REPO/$PROJECT_NAME:$ENV_PROFILE
                        docker image prune -f
                        '''
                }
            }
   
            stage('目标服务器拉取镜像并运行') {
                steps {
                    sshPublisher(publishers: [sshPublisherDesc(configName: env.SERVER_CREDENTIALS_ID, transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "cd $SERVER_PATH && ./deploy.sh $HARBOR_URL $HARBOR_USER $HARBOR_PASSWORD $HARBOR_REPO $PROJECT_NAME $ENV_PROFILE", execTimeout: 120000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
                }
            }
        }
        post {
            success {
                dingtalk (
                    robot: '${DING_TALK}',
                    type:'MARKDOWN',
                    title: "${PROJECT_NAME}项目构建成功",
                    text: [
                      "### ${PROJECT_NAME}项目构建成功",
                      "---",
                      "- 分支: $ENV_PROFILE",
                      "- 持续时间: ${currentBuild.durationString}",
                      "- 执行人：${currentBuild.buildCauses.shortDescription}"
                    ],
                    at: ['17613013712']
                )
            }
            failure {
                dingtalk (
                    robot: '${DING_TALK}',
                    type:'MARKDOWN',
                    title: "${PROJECT_NAME}项目构建失败",
                    text: [
                      "### ${PROJECT_NAME}项目构建失败",
                      "---",
                      "- 分支: $ENV_PROFILE",
                      "- 持续时间: ${currentBuild.durationString}",
                      "- 执行人：${currentBuild.buildCauses.shortDescription}"
                    ],
                    at: ['17613013712']
                )
            }
            unstable {
                dingtalk (
                    robot: '${DING_TALK}',
                    type:'MARKDOWN',
                    title: "${PROJECT_NAME}项目构建异常",
                    text: [
                      "### ${PROJECT_NAME}项目构建异常",
                      "---",
                      "- 分支: $ENV_PROFILE",
                      "- 持续时间: ${currentBuild.durationString}",
                      "- 执行人：${currentBuild.buildCauses.shortDescription}"
                    ],
                    at: ['17613013712']
                )
            }
        }
    }
    ```
### 配置项目部署服务器

登录到`SERVER_CREDENTIALS_ID`对应服务器，然后在`SERVER_PATH`路径下编写`deploy.sh`（记得赋予权限）与`docker-compose.yaml`两个文件，下面是我提供的文件示例模版（需根据自身项目修改）。

> `deploy.sh`的内容如下：
```shell
#!/bin/bash

HARBOR_URL=$1
HARBOR_USER=$2
HARBOR_PASSWORD=$3
HARBOR_REPO=$4
PROJECT_NAME=$5
ENV_PROFILE=$6

docker login -u ${HARBOR_USER} -p ${HARBOR_PASSWORD} ${HARBOR_URL}
docker pull ${HARBOR_URL}/${HARBOR_REPO}/${PROJECT_NAME}:${ENV_PROFILE}
docker tag ${HARBOR_URL}/${HARBOR_REPO}/${PROJECT_NAME}:${ENV_PROFILE} ${PROJECT_NAME}:${ENV_PROFILE}

docker-compose up -d ${PROJECT_NAME}-${ENV_PROFILE}

docker image prune -f
```

> `docker-compose.yaml`的内容如下：
```yaml
version: '3'
services:
  starlinkRisk-admin-dev:
    image: starlinkrisk-admin:dev
    container_name: starlinkrisk-admin-dev
    volumes:
      - logs_dev:/root/logs
      - /etc/localtime:/etc/localtime:ro
    environment:
      TZ: Asia/Shanghai
    ports:
      - "48080:48080"
  starlinkRisk-admin-main:
    image: starlinkrisk-admin:main
    container_name: starlinkrisk-admin-main
    volumes:
      - logs_main:/root/logs
      - /etc/localtime:/etc/localtime:ro
    environment:
      TZ: Asia/Shanghai
    ports:
      - "48081:48080"

volumes:
  logs_dev:
    driver: local
  logs_main:
    driver: local
```

## 验证流水线

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105654964.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105655331.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105656219.png)

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425105656709.png)

> 本文结束