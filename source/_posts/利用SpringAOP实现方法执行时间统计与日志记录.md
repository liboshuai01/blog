---
title: 利用SpringAOP实现方法执行时间统计与日志记录
tags:
  - Java
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504270008533.png'
toc: true
abbrlink: d87c6abc
date: 2024-08-19 10:32:14
---

在现代应用程序开发中，性能监控和日志记录是确保应用程序高效运行和便于调试的关键因素。本文将介绍如何使用Spring AOP（面向切面编程）结合自定义注解，实现对方法执行时间的统计和详细的日志记录。

<!-- more -->

## 背景介绍

Spring AOP是一种强大的编程范式，可以在不修改原始代码的情况下增强功能。通过定义切面（Aspect），我们可以在方法执行的各个阶段插入自定义逻辑。本例中，我们将使用AOP来记录方法的执行时间和相关信息。

## 1. 引入依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
</dependency>

<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

## 2. 自定义注解

首先，我们定义一个自定义注解`@TakeTime`，用于标记需要统计执行时间的方法。

```java
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TakeTime {
}
```

- **@Documented**: 使注解包含在Javadoc中。
- **@Target**: 指定注解适用的位置，这里是方法。
- **@Retention**: 指定注解的生命周期，在运行时可通过反射获取。

## 3. 定义切面类

接下来，创建一个切面类`TakeTimeAspect`，使用`@Aspect`和`@Component`注解，让Spring管理其生命周期。

```java
@Slf4j
@Aspect
public class TakeTimeAspect {
    //统计请求的处理时间
    TransmittableThreadLocal<Long> startTime = new TransmittableThreadLocal<>();
    TransmittableThreadLocal<Long> endTime = new TransmittableThreadLocal<>();

    /**
     * 带有@TakeTime注解的方法
     */
    @Pointcut("@annotation(com.liboshuai.starlink.slr.framework.takeTime.core.aop.TakeTime)")
    public void TakeTime() {

    }

    @Before("TakeTime()")
    public void doBefore(JoinPoint joinPoint) {
        // 获取方法的名称
        String methodName = joinPoint.getSignature().getName();
        // 获取方法入参
        Object[] param = joinPoint.getArgs();
        StringBuilder sb = new StringBuilder();
        for (Object o : param) {
            sb.append(o).append(";");
        }
        //接收到请求，记录请求内容
        String requestUrl = null;
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (Objects.nonNull(attributes)) {
            HttpServletRequest httpServletRequest = attributes.getRequest();
            requestUrl = httpServletRequest.getRequestURL().toString();
        }
        startTime.set(System.currentTimeMillis());
        log.info("==================================================");
        log.info("方法名称: [{}] ", methodName);
        log.info("方法参数: {}", sb);
        log.info("请求URL: [{}]", requestUrl);
        log.info("开始时间: [{}]", DateUtil.format(new Date(startTime.get()), DatePattern.NORM_DATETIME_MS_PATTERN));
        log.info("--------------------------------------------------");
    }

    @AfterReturning(returning = "ret", pointcut = "TakeTime()")
    public void doAfterReturning(JoinPoint joinPoint, Object ret) {
        endTime.set(System.currentTimeMillis());
        long duration = endTime.get() - startTime.get();
        log.info("--------------------------------------------------");
        log.info("方法名称: [{}]", joinPoint.getSignature().getName());
        log.info("方法返回值: {}", JsonUtils.toJsonString(ret));
        log.info("执行耗时: {} ms / {} seconds / {} minute",
                duration,
                String.format("%.2f", duration / 1000.0),
                String.format("%.2f", duration / 1000.0 / 60.0));
        log.info("结束时间: [{}]", DateUtil.format(new Date(endTime.get()), DatePattern.NORM_DATETIME_MS_PATTERN));
        log.info("==================================================");
    }
}
```

## 4. 代码解析

- **切点定义**: 使用`@Pointcut`定义切点，标识所有使用`@TakeTime`注解的方法。
- **前置通知**: 使用`@Before`记录方法开始执行的时间、名称、参数和请求URL。
- **后置通知**: 使用`@AfterReturning`计算并记录方法执行时间和返回值。
- **ThreadLocal**: 用于存储每个线程独立的开始和结束时间，避免线程间数据干扰。

## 5. 优化建议

- **异常处理**: 考虑在切面中添加对异常的处理，以便在方法抛出异常时也能记录日志。
- **参数序列化**: 对方法参数进行简化处理，避免过多日志输出。
- **日志级别控制**: 在生产环境中，合理设置日志级别以减少性能影响。

## 6. 结论

通过Spring AOP和自定义注解的结合，我们可以轻松实现对方法执行时间的统计和日志记录。这种方式不仅提高了代码的可读性和可维护性，也为性能优化和问题排查提供了有力支持。