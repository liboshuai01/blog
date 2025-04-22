---
title: SpringBoot+Maven多环境配置
tags:
  - Java
  - SpringBoot
  - Maven
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504180935601.png'
toc: true
abbrlink: 92bf4e1a
date: 2022-12-02 08:01:00
---

本篇文章深入讲解了在 Spring Boot 项目中，如何通过 Maven 多环境配置，实现开发（dev）、验证（uat）等不同环境下的资源文件管理与构建流程。

文章首先介绍了 pom.xml 中 profiles、build 资源过滤与属性占位的核心设置，详述 profilesActive 参数如何与 application.yml、application-dev.yml、application-uat.yml 等文件关联；

然后借助 IDEA 或 mvn -Pdev、-Puat 等命令行参数，演示了在本地和打包阶段灵活切换环境的操作。

通过一个简单的 DemoController 示例，说明了环境变量对 @Value 注入的影响。该方案让团队能够统一管理多套配置，避免手动拷贝与冲突，大幅提升构建与部署效率，保证环境隔离和一致性，帮助后端开发人员在不同阶段快速切换与验证配置。

<!-- more -->

> 项目源码：[github](https://github.com/liboshuai01/lbs-demo/tree/master/maven-multi-env), [gitee](https://gitee.com/liboshuai01/lbs-demo/tree/master/maven-multi-env)

## 配置步骤

1. 创建maven项目

2. 配置pom.xml
    > 关注`profiles`与`build`配置块

    ```xml
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>com.liboshuai.demo</groupId>
            <artifactId>lbs-demo</artifactId>
            <version>1.0</version>
        </parent>
    
        <artifactId>maven-multi-env</artifactId>
        <packaging>jar</packaging>
    
        <name>maven-multi-env</name>
    
        <!-- 关注这里 -->
        <profiles>
            <profile>
                <id>dev</id>
                <properties>
                    <!-- 环境标识，需要与配置文件的名称相对应 -->
                    <profilesActive>dev</profilesActive>
                </properties>
                <activation>
                    <!-- 默认环境 -->
                    <activeByDefault>true</activeByDefault>
                </activation>
            </profile>
            <profile>
                <id>uat</id>
                <properties>
                    <profilesActive>uat</profilesActive>
                </properties>
            </profile>
        </profiles>
    
        <properties>
            <maven.compiler.source>8</maven.compiler.source>
            <maven.compiler.target>8</maven.compiler.target>
            <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
            <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    
            <spring-boot.version>2.7.18</spring-boot.version>
        </properties>
    
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
                <version>${spring-boot.version}</version>
            </dependency>
        </dependencies>
    
        <!-- 关注这里 -->
        <build>
            <finalName>${project.artifactId}-${project.version}</finalName>
            <resources>
                <resource>
                    <directory>src/main/resources</directory>
                    <excludes>
                        <exclude>application-${profilesActive}.yml</exclude>
                        <exclude>application.yml</exclude>
                        <exclude>mapper/**/*.xml</exclude>
                    </excludes>
                    <!-- 关闭过滤 -->
                    <filtering>false</filtering>
                </resource>
                <resource>
                    <directory>src/main/resources</directory>
                    <!-- 引入所有 匹配文件进行过滤 -->
                    <includes>
                        <include>application-${profilesActive}.yml</include>
                        <include>application.yml</include>
                        <include>mapper/**/*.xml</include>
                    </includes>
                    <!-- 启用过滤 即该资源中的变量将会被过滤器中的值替换 -->
                    <filtering>true</filtering>
                </resource>
            </resources>
            <plugins>
                <!-- 打包 -->
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <configuration>
                        <!-- 指定主main类（防止多个main类问题）-->
                        <mainClass>com.liboshuai.demo.DemoApplication</mainClass>
                    </configuration>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal> <!-- 将引入的 jar 打入其中 -->
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </project>
    ```
   
3. 创建配置文件

    application.yml配置文件内容
    ```yml
    server:
      port: 30001
    
    spring:
      profiles:
        active: @profilesActive@
    ```
   
    application-dev.yml配置文件内容
    ```yml
    demo:
      value: "我是dev环境"
    ```
    
    application-uat.yml配置文件内容
    ```yml
    demo:
      value: "我是uat环境"
    ```
   
4. 编写代码
    
    DemoApplication.java代码
    ```java
    package com.liboshuai.demo;
    
    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    
    @SpringBootApplication
    public class DemoApplication {
        public static void main(String[] args) {
            SpringApplication.run(DemoApplication.class, args);
        }
    }
    ```
    
    DemoController.java代码
    ```java
    package com.liboshuai.demo.controller;
    
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RestController;
    
    @RestController
    @RequestMapping("/demo")
    public class DemoController {
    
        @Value("${demo.value}")
        private String value;
    
        @RequestMapping("/getValue")
        public String getValue(){
            return "不同环境获取到的value值: " + value;
        }
    }
    ```

5. idea选择对应环境进行启动

    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504180947482.png)
    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504180948853.png)

6. 如果是进行maven打包，则需要通过`-Pdev`参数进行指定环境

    ```shell
    mvn clean install -Dmaven.test.skip=true -Pdev
    ```

## 结果验证

1. 访问`http://localhost:30001/demo/getValue` ，返回结果为

    ```txt
    不同环境获取到的value值: 我是dev环境
    ```
   
2. 在idea中切换环境为uat，再次访问`http://localhost:30001/demo/getValue` ，返回结果为

    ```txt
    不同环境获取到的value值: 我是uat环境
    ```

## 结语

在项目开发和运维中，多环境配置管理是常见且关键的需求，能够帮助团队快速在开发、测试、预生产、生产等不同阶段进行环境切换与验证。

本文所示的 Maven 多环境配置方案，通过 profiles 与资源过滤的巧妙组合，实现了配置与代码分离、打包与运行时的自动化切换。

开发者只需在 pom.xml 中定义好 profiles 和对应的占位符，在 application-xxx.yml 中维护环境属性，然后通过 IDEA 环境选择或 mvn -P 指定环境，即可完成一键打包与部署。

此外，该方法可灵活拓展至更多环境（如 prod、staging）及更多资源文件（如日志配置、数据源配置等），并可无缝集成至 CI/CD 流程，实现自动化构建、测试与发布，从而进一步提升团队协作效率和系统可靠性。