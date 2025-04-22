---
title: Java8中更优雅的记录代码运行时间
tags:
  - Java
  - Jdk8
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504191643018.png'
toc: true
abbrlink: 484517b9
date: 2023-04-19 16:32:25
---

在日常后端开发中，性能优化是一项核心任务。我们经常需要测量某段代码的执行耗时，例如查询耗时、接口响应时间、批处理任务处理时间等。在 Java 中，传统的做法可能是使用 System.currentTimeMillis()：

```java
long start = System.currentTimeMillis();
// 业务逻辑
long end = System.currentTimeMillis();
System.out.println("执行耗时: " + (end - start) + "ms");
```

虽然这非常直接，但在 Java 8 引入 java.time 包之后，我们可以使用更现代、更语义化的方式 —— Instant 和 Duration 来实现这一目标。

本文将带你深入了解 Java 8 中几种记录代码运行时间的优雅方式，并附上实用工具类与建议，提高你的代码可读性与复用性。

<!-- more -->

## Java 8 简单实现方式

Java 8 中的 java.time.Instant 表示一个时间点，Duration 表示两个时间点之间的时长：

```java
import java.time.Duration;
import java.time.Instant;

public class SampleTimer {
    public static void main(String[] args) {
        Instant start = Instant.now();

        // 模拟业务逻辑
        executeBusinessLogic();

        Instant end = Instant.now();
        long timeElapsed = Duration.between(start, end).toMillis();
        System.out.println("执行耗时: " + timeElapsed + " ms");
    }

    private static void executeBusinessLogic() {
        try {
            Thread.sleep(500); // 模拟耗时操作
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

相比 System.currentTimeMillis()，这种写法更具可读性和表达力，并且跨平台表现一致（System.currentTimeMillis 可能受系统时间调整影响）。

## 封装计时工具类（更通用 & 可复用）

为了更好地管理运行时间测量代码，我们可以封装一个小型工具类，例如 TimerUtils：

```java
import java.time.Duration;
import java.time.Instant;
import java.util.function.Supplier;

public class TimerUtils {

    public static <T> T measure(String taskName, Supplier<T> supplier) {
        Instant start = Instant.now();
        try {
            return supplier.get();
        } finally {
            Instant end = Instant.now();
            long duration = Duration.between(start, end).toMillis();
            System.out.printf("【任务: %s】耗时: %d ms%n", taskName, duration);
        }
    }

    public static void measure(String taskName, Runnable runnable) {
        Instant start = Instant.now();
        try {
            runnable.run();
        } finally {
            Instant end = Instant.now();
            long duration = Duration.between(start, end).toMillis();
            System.out.printf("【任务: %s】耗时: %d ms%n", taskName, duration);
        }
    }
}
```

使用方式如下：

```java
public class TimerUtilsTest {
    public static void main(String[] args) {
        // 没有返回值
        TimerUtils.measure("执行任务A", () -> {
            try {
                Thread.sleep(200);
            } catch (InterruptedException ignored) {}
        });

        // 有返回值
        String result = TimerUtils.measure("执行任务B", () -> {
            try {
                Thread.sleep(300);
            } catch (InterruptedException ignored) {}
            return "任务完成";
        });

        System.out.println("任务B返回结果：" + result);
    }
}
```

这种封装方式可以很好地提升代码的可读性、复用性，尤其适合在 Spring Boot 项目中进行日志输出分析。

## 进阶使用：结合日志框架输出

实际项目中，推荐的做法是将耗时统计信息输出到日志中，便于统一采集和排查性能瓶颈。结合 SLF4J + Logback，可以这样集成：

```java
private static final Logger logger = LoggerFactory.getLogger(MyService.class);

public void process() {
    Instant start = Instant.now();

    try {
        // 执行业务逻辑
    } finally {
        long elapsed = Duration.between(start, Instant.now()).toMillis();
        logger.info("处理接口耗时: {} ms", elapsed);
    }
}
```

比 System.out.println 更专业，也更便于后续 ELK/Prometheus 等监控系统采集。

## 使用 AOP 自动记录方法执行时间（Spring Boot 推荐）

性能日志可以通过 Spring AOP 自动记录，无需在每个方法中手动添加计时代码：

1. 自定义注解：

    ```java
    @Target(ElementType.METHOD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Timed {
        String value() default "";
    }
    ```

2. 编写切面类：

    ```java
    @Aspect
    @Component
    public class TimingAspect {
        private static final Logger logger = LoggerFactory.getLogger(TimingAspect.class);
    
        @Around("@annotation(timed)")
        public Object around(ProceedingJoinPoint pjp, Timed timed) throws Throwable {
            Instant start = Instant.now();
            Object result = pjp.proceed();
            long elapsed = Duration.between(start, Instant.now()).toMillis();
    
            String methodName = pjp.getSignature().toShortString();
            logger.info("方法 [{}] 执行耗时: {} ms", methodName, elapsed);
            return result;
        }
    }
    ```

    使用示例：
    
    ```java
    @Timed
    public void handleRequest() {
        // 逻辑代码
    }
    ```
    
    优势：
    
    - 解耦业务逻辑与日志统计
      - 透明统一统计性能数据
      - 对 Controller、Service 层尤为实用


## 其他替代品推荐

除了 Java 原生类，还有一些开源工具类也支持高效计时：

- Guava 的 Stopwatch：

  ```java
  Stopwatch stopwatch = Stopwatch.createStarted();
  // 执行代码
  stopwatch.stop();
  System.out.println("耗时: " + stopwatch.elapsed(TimeUnit.MILLISECONDS) + "ms");
  ```

- Micrometer 和 Spring Boot Actuator 提供的 @Timed 注解和 Metrics 接入更强大采集工具，如 Prometheus、Grafana。

---

## 总结

在注重性能的后端开发中，精准、高效地记录代码执行时间是不可或缺的一步。本文从 Java 8 原生的 Instant/Duration 入手，展示了编写更优雅、规范、可扩展的代码计时方式。

| 方法           | 优雅度 | 可复用性 | 推荐度     |
|----------------|--------|----------|------------|
| System.currentTimeMillis | 一般    | 低        | ❌  |
| Instant + Duration       | 良好    | 中        | ✅  |
| 工具类封装              | 高      | 高        | ✅✅ |
| AOP 自动化记录          | 极高    | 极高       | ✅✅✅ |

希望这些方法能在你的日常开发中帮到你，更好掌握性能优化核心工具。