---
title: 自定义SpringBoot-Starter入门指南
tags:
  - Java
  - SpringBoot
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250424173823338.png'
toc: true
abbrlink: 857d16b8
date: 2022-02-04 17:33:53
---

在现代微服务架构中，统一的请求日志记录是保障系统可观测性和问题排查的重要基础。虽然 Spring Boot 提供了丰富的日志支持，但在多项目中重复配置和开发请求日志功能，难免造成代码冗余和维护成本升高。通过打造自主的自定义 Starter，我们可以将统一请求日志模块封装成独立组件，实现跨项目复用和快速集成。

本文将以「统一请求日志」功能为实战示例，全面讲解如何设计并实现一个生产级的自定义 Spring Boot Starter，覆盖模块设计、自动配置、属性绑定以及与主应用的对接，助你迈入企业级 Starter 开发殿堂。

<!-- more -->

---

## 快速定位项目背景与目标

通常，请求日志中我们希望收集：

- 访问时间戳
- 请求路径与方法
- 请求参数（可选）
- 响应状态码
- 处理时长

理想情况下，所有 Spring Boot 服务只需引入同一个 Starter 并做少量配置，即可启用此请求日志功能，而不需每个项目重复编写 Filter、Interceptor 或 HandlerAspect 等。

---

## 自定义Starter项目结构与依赖配置

创建一个 Maven 项目，命名为 `request-log-springboot-starter`。核心依赖包括：

- `spring-boot-starter-web`（提供 Web 环境）
- `spring-boot-autoconfigure`（支持自动配置）
- `spring-boot-configuration-processor`（生成配置元数据）
- Lombok（简化代码，可根据习惯选择）

示例如下 `pom.xml` 配置：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.18</version>
        <relativePath/>
    </parent>

    <groupId>com.liboshuai.log</groupId>
    <artifactId>request-log-springboot-starter</artifactId>
    <version>1.0</version>
    <packaging>jar</packaging>

    <name>request-log-springboot-starter</name>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>

        <!-- 提供web支持 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 自动配置支持 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
        </dependency>

        <!-- 生成配置元数据 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- 统一日志处理推荐引入（slf4j） -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </dependency>

        <!-- Lombok简化代码 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.30</version>
            <scope>provided</scope>
        </dependency>

    </dependencies>
</project>
```

---

## 编写请求日志配置属性类

提供配置中心，将日志记录相关选项暴露给用户，比如是否启用日志记录、是否打印请求参数：

```java
package com.liboshuai.log;

import org.springframework.boot.context.properties.ConfigurationProperties;
import lombok.Data;

/**
 * 请求日志配置属性
 */
@Data
@ConfigurationProperties(prefix = "request.log")
public class RequestLogProperties {

    /**
     * 是否开启请求日志功能，默认开启
     */
    private boolean enabled = true;

    /**
     * 是否打印请求参数，默认打印
     */
    private boolean printRequestParams = true;
}
```

---

## 编写请求日志过滤器

基于 Spring MVC，实用 Servlet 过滤器实现请求日志打印，计算请求处理耗时，输出结构化日志：

```java
package com.liboshuai.log;

import lombok.extern.slf4j.Slf4j;
import org.springframework.util.StreamUtils;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.util.Enumeration;

/**
 * 请求日志过滤器，打印请求路径、参数、响应状态码和耗时
 */
@Slf4j
public class RequestLogFilter implements Filter {

    private final RequestLogProperties properties;

    public RequestLogFilter(RequestLogProperties properties) {
        this.properties = properties;
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
            throws IOException, ServletException {
        if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)) {
            chain.doFilter(request, response);
            return;
        }

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        long startTime = System.currentTimeMillis();

        // 缓存请求体，便于日志打印（此处仅演示GET请求参数，POST参数可用HttpServletRequestWrapper增强）
        String requestURI = httpRequest.getRequestURI();
        String method = httpRequest.getMethod();

        StringBuilder paramBuilder = new StringBuilder();
        if ("GET".equalsIgnoreCase(method) && properties.isPrintRequestParams()) {
            Enumeration<String> parameterNames = httpRequest.getParameterNames();
            while (parameterNames.hasMoreElements()) {
                String key = parameterNames.nextElement();
                String value = httpRequest.getParameter(key);
                paramBuilder.append(key).append("=").append(value).append("&");
            }
            if (paramBuilder.length() > 0) {
                paramBuilder.deleteCharAt(paramBuilder.length() -1 );
            }
        }

        chain.doFilter(request, response);

        long duration = System.currentTimeMillis() - startTime;
        int status = httpResponse.getStatus();

        log.info("[RequestLog] method={}, uri={}, params={}, status={}, duration={}ms",
                method,
                requestURI,
                properties.isPrintRequestParams() ? paramBuilder.toString() : "N/A",
                status,
                duration);
    }
}
```

说明：

- 过滤器使用 `RequestLogProperties` 控制是否启用及是否打印参数。
- 为保证请求体可读，生产环境中可代替默认 HttpServletRequest 用包装类 `ContentCachingRequestWrapper`。
- 这里只演示 GET 请求参数打印，POST 请求打印可进一步扩展。

---

## 实现自动配置类

自动配置类条件化地注入 `RequestLogFilter` 组件，绑定用户配置，并注册 Filter：

```java
package com.liboshuai.log;

import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 自动配置类，注册请求日志过滤器
 */
@Configuration
@EnableConfigurationProperties(RequestLogProperties.class)
public class RequestLogAutoConfiguration {

    @Bean
    @ConditionalOnProperty(prefix = "request.log", name = "enabled", havingValue = "true", matchIfMissing = true)
    public FilterRegistrationBean<RequestLogFilter> requestLogFilter(RequestLogProperties properties) {
        FilterRegistrationBean<RequestLogFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new RequestLogFilter(properties));

        // 设置过滤路径，这里拦截所有请求
        registrationBean.addUrlPatterns("/*");
        registrationBean.setOrder(Integer.MIN_VALUE); // 尽早执行

        return registrationBean;
    }
}
```

> 自动配置具备条件注解 `@ConditionalOnProperty`，默认启用，可通过配置关闭。

---

## 注册自动配置服务提供者文件

Spring Boot 读取自动配置必须指定实现类，在资源目录下创建：

- 路径为 `resources/META-INF/spring.factories`（Spring Boot 2.x）

写入：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.liboshuai.log.RequestLogAutoConfiguration
```

如果是 Spring Boot 3.x，改为：

`resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，内容为：

```
com.liboshuai.log.RequestLogAutoConfiguration
```

---

## 主应用中集成使用

将该 Starter 打包安装至本地仓库：

```bash
mvn clean install
```

### 添加依赖

在业务应用的`pom.xml`中添加：

```xml
<dependency>
    <groupId>com.liboshuai.log</groupId>
    <artifactId>request-log-springboot-starter</artifactId>
    <version>1.0</version>
</dependency>
```

### 配置启用与定制日志选项

在主应用的 `application.yml`，示例配置：

```yaml
request:
  log:
    enabled: true
    printRequestParams: false
```

这样即可开启请求日志，但关闭请求参数打印。

### 启动应用并测试

运行 Spring Boot 应用，访问任意接口，在控制台及日志中即可见格式化的请求日志输出，例如：

```
[RequestLog] method=GET, uri=/api/users, params=, status=200, duration=18ms
```

---

## 扩展及优化建议

为了满足生产环境需求，可进一步改进：

- 支持更多 HTTP 方法的参数捕获（通过请求包装器支持POST等）
- 支持异步日志写入，避免影响请求响应时间
- 集成日志聚合系统（如 ELK、SkyWalking 等）
- 支持请求唯一标识（traceId）传递，提升链路追踪能力
- 提供自定义日志格式接口和 SPI 扩展
- 结合 Actuator 暴露请求日志开关

---

## 总结

通过实际生产场景中的统一请求日志功能打造自定义 Spring Boot Starter，我们实现了：

- 组件隔离与封装，复用跨项目
- 参数可配置，增强灵活度
- 标准化的自动配置集成
- 降低项目重复开发维护成本

借助 Starter 机制，团队可以构建企业内部通用基础能力，提高研发效率和代码整洁度。希望本示例给你在设计更复杂 Starter 时带来启发！