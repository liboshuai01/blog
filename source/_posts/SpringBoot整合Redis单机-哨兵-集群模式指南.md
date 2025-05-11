---
title: SpringBoot整合Redis单机/哨兵/集群模式指南
tags:
  - Java
  - SpringBoot
  - Redis
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504270006831.png'
toc: true
abbrlink: 31c79302
date: 2023-07-21 00:04:11
---

本文旨在为后端开发者提供一份在 Spring Boot 项目中整合 Redis 的实践指南，涵盖单节点、主从、哨兵及集群四种常见部署模式。我们将探讨每种模式的配置方法、适用场景及关键注意事项，帮助您根据实际需求选择并实施最合适的 Redis 解决方案。

**核心技术栈:** Spring Boot, Redis, Lettuce (Spring Boot 2.x 默认 Redis 客户端)

<!-- more -->

> 项目仓库：[github](https://github.com/liboshuai01/lbs-demo/tree/master/springboot-redis) [gitee](https://gitee.com/liboshuai01/lbs-demo/tree/master/springboot-redis)

## 通用配置与基础

在深入不同模式之前，我们需要引入必要的 Maven 依赖并配置基础的 `RedisTemplate`，以便在 Spring Boot 应用中与 Redis 进行交互。

### Maven 依赖引入

确保您的 `pom.xml` 文件包含以下依赖：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-parent</artifactId>
    <version>2.2.6.RELEASE</version>
    <!-- 建议使用更新的 Spring Boot 版本以获得更好的特性和支持 -->
</parent>

<dependencies>
    <!-- Spring Web Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Data Redis Starter (包含 Lettuce 客户端) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Lettuce 连接池依赖 -->
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
    </dependency>

    <!-- Lombok (可选，简化代码) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```
*   `spring-boot-starter-data-redis`: 核心依赖，提供了 Spring Data Redis 的抽象和 Lettuce 客户端。
*   `commons-pool2`: Lettuce 客户端需要此依赖来实现连接池。

### Redis 配置类 (`RedisConfig`)

为了更友好地处理 Redis 数据的序列化（特别是对象存储），我们通常会自定义 `RedisTemplate` 的序列化器。默认的 JDK 序列化可读性差且可能存在安全风险。推荐使用 JSON 格式。

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    @SuppressWarnings("all") // 压制所有警告，实际项目中应谨慎使用
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory); // 设置连接工厂

        // 使用 Jackson2JsonRedisSerializer 来序列化和反序列化 redis 的 value 值
        Jackson2JsonRedisSerializer<Object> jacksonSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        // 指定要序列化的域，field,get和set,以及修饰符范围，ANY是都有包括private和public
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        // 指定序列化输入的类型，类必须是非final修饰的，final修饰的类，比如String,Integer等会跑出异常
        // om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL); // 在较新的 Jackson 版本中可能已被弃用或推荐其他方式
        om.activateDefaultTyping(om.getPolymorphicTypeValidator(), ObjectMapper.DefaultTyping.NON_FINAL); // 更推荐的方式
        jacksonSerializer.setObjectMapper(om);

        // String 的序列化
        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        // key 采用 String 的序列化方式
        template.setKeySerializer(stringSerializer);
        // hash 的 key 也采用 String 的序列化方式
        template.setHashKeySerializer(stringSerializer);
        // value 序列化方式采用 jackson
        template.setValueSerializer(jacksonSerializer);
        // hash 的 value 序列化方式采用 jackson
        template.setHashValueSerializer(jacksonSerializer);

        // 开启事务支持 (可选，根据业务需求决定)
        // template.setEnableTransactionSupport(true);

        template.afterPropertiesSet(); // 应用配置
        return template;
    }
}
```
*   **序列化**：配置了 Key 使用 `StringRedisSerializer`，Value 和 HashValue 使用 `Jackson2JsonRedisSerializer`。这使得 Redis 中存储的数据更易读，且方便不同语言的应用共享数据。
*   **`ObjectMapper` 配置**：允许 Jackson 序列化所有字段，并启用了默认类型信息（`NON_FINAL`），这对于存储多态类型对象非常重要，但需注意安全风险和兼容性。

### Redis 工具类 (`RedisUtil`) (可选)

原文提供了一个非常详尽的 `RedisUtil` 类，封装了 `RedisTemplate` 的各种操作。这对于简化业务代码调用非常有帮助。

```java
// RedisUtil.java 内容... (同原文，此处省略以保持篇幅简洁)
// 建议将 RedisUtil 放在项目代码中或参考原文 Gitee 链接获取完整代码
// https://gitee.com/liboshuai01/spring-redis-test/blob/master/src/main/java/com/example/demo/config/RedisUtil.java
```
*   **使用建议**：虽然 `RedisUtil` 提供了便利，但在某些场景下，直接使用 `RedisTemplate` 或其提供的 `opsForXXX()` 方法（如 `opsForValue()`, `opsForHash()` 等）能提供更清晰的语义和更强的类型检查。根据团队习惯和项目复杂度决定是否使用此类工具类。

---

## 单节点模式 (Standalone)

最简单的模式，适用于开发、测试或数据量、并发量不高的场景。

### 模式介绍

*   单个 Redis 实例提供服务。
*   简单易部署。
*   无高可用性，节点宕机服务即中断。

### `application.yaml` 配置

```yaml
spring:
  redis:
    host: 127.0.0.1       # Redis 服务器地址
    port: 6379           # Redis 服务器端口
    password: your_password # Redis 访问密码 (如果没有设置，则无需此行或留空)
    database: 0          # 使用的 Redis 数据库索引 (默认 0)
    timeout: 5000ms      # 连接超时时间 (单位：毫秒)，建议使用 'ms' 后缀明确单位

    lettuce:
      pool:
        min-idle: 0         # 连接池中的最小空闲连接
        max-idle: 8         # 连接池中的最大空闲连接
        max-active: 8       # 连接池最大连接数
        max-wait: -1ms      # 连接池最大阻塞等待时间 (-1 表示无限等待)，建议使用 '-1ms' 或 '-1'
```

### 整合说明

只需上述 `application.yaml` 配置，配合前面定义的 `RedisConfig`，Spring Boot 即可自动配置好连接单节点 Redis 的 `RedisTemplate`。随后可以通过注入 `RedisTemplate` 或 `RedisUtil` 来使用 Redis。

---

## 主从模式 (Master-Slave) / 读写分离

### 模式介绍

*   **数据冗余**：Master 的数据会实时同步到 Slave 节点。
*   **读写分离**：Master 负责写操作和部分读操作，Slave 负责读操作，分摊读压力。
*   **基础高可用**：Master 宕机后，无法进行写操作，但 Slave 仍可提供读服务。需手动或借助其他机制（如 Sentinel）进行 Master 切换。

### Spring Boot 整合方式

Spring Boot 的标准自动配置**并不直接支持**针对 Master-Slave 模式的读写分离优化。`spring-boot-starter-data-redis` 默认会将所有操作（读和写）都发送到配置的节点（通常是 Master）。

**常见处理方式：**

1.  **简单配置 (不实现读写分离)**：
    *   将 `application.yaml` 配置指向 Master 节点（如同单节点模式）。
    *   应用所有读写请求都发往 Master。Slave 仅作为数据备份和手动故障转移的备用。
    *   这是最简单的整合方式，但未利用 Slave 的读能力。

2.  **利用 Sentinel 模式 (推荐)**：
    *   Sentinel 模式是官方推荐的、基于主从模式实现自动故障转移和高可用的方案。它能动态发现 Master 和 Slave，并提供服务发现能力。详见下一章节。

3.  **手动配置 `LettuceConnectionFactory` (复杂)**：
    *   开发者可以不依赖自动配置，手动创建 `LettuceConnectionFactory` 或 `JedisConnectionFactory` Bean。
    *   在 Bean 配置中，可以设置 `ReadFrom` 策略（如 `ReadFrom.REPLICA_PREFERRED`），让 Lettuce 客户端优先从 Slave 读取数据。
    *   这种方式配置相对复杂，需要对 Spring Data Redis 和 Lettuce/Jedis 客户端有更深入的理解。

4.  **使用数据库中间件 / 代理**：
    *   如 Twemproxy、Codis 或 Redis 官方的 Redis Proxy (仍在发展中)。这些中间件可以代理客户端请求，实现路由、读写分离和分片等功能，对应用层透明。

**结论**：对于需要利用主从模式进行读写分离和高可用的场景，直接升级到 **Sentinel 模式** 是 Spring Boot 中更标准、更推荐的做法。如果仅需数据备份，按单节点配置指向 Master 即可。

---

## 哨兵模式 (Sentinel)

### 模式介绍

Sentinel 是 Redis 官方推荐的高可用性 (High Availability, HA) 解决方案。它基于主从模式，并增加了自动监控、通知和故障转移能力。

*   **监控 (Monitoring)**：Sentinel 持续监控 Master 和 Slave 节点是否正常工作。
*   **通知 (Notification)**：当被监控的 Redis 实例出现问题时，Sentinel 可以通知管理员或其他应用程序。
*   **自动故障转移 (Automatic Failover)**：当 Master 节点宕机时，Sentinel 会从 Slave 节点中选举出一个新的 Master，并更新其他 Slave 指向新 Master，确保服务持续可用（写操作）。
*   **配置提供者 (Configuration Provider)**：客户端连接 Sentinel 询问当前 Master 的地址，而不是直接连接固定的 Redis 地址。

### 哨兵模式搭建

部署 Sentinel 需要启动一个或多个 Sentinel 进程（推荐至少 3 个，奇数个），它们互相监控，并共同监控 Redis 主从集群。具体搭建过程请参考 Redis 官方文档或其他详细教程。

### `application.yaml` 配置

```yaml
spring:
  redis:
    password: your_password # Redis 主从节点的密码 (如果设置了)
    timeout: 5000ms      # 连接超时时间

    sentinel:
      master: mymaster     # Redis 主节点名称 (在 sentinel.conf 中定义的名字)
      nodes:             # Sentinel 节点的地址列表 (IP:Port)
        - 47.116.114.234:44801
        - 139.224.249.155:44801
        - 47.116.113.67:44801
        # 只需要提供部分 Sentinel 节点即可，客户端会自动发现其他 Sentinel

    lettuce:
      pool:
        min-idle: 0
        max-idle: 8
        max-active: 8
        max-wait: -1ms # 或 -1
      # lettuce 支持从 sentinel 节点读取数据，可配置读写分离策略
      # lettuce:
      #   sentinel:
      #     read-from: REPLICA_PREFERRED # 例如：优先从副本读
```
*   **`sentinel.master`**: 必须与 Redis Sentinel 配置文件 (`sentinel.conf`) 中 `sentinel monitor <master-name> ...` 定义的 `master-name` 一致。
*   **`sentinel.nodes`**: 提供一个或多个 Sentinel 实例的地址。Spring Boot 会连接这些 Sentinel 来获取当前 Master 的信息。

### 整合说明

配置好 `application.yaml` 后，Spring Boot 会自动配置 `LettuceConnectionFactory` 连接到 Sentinel 集群。应用像单节点一样注入和使用 `RedisTemplate` 或 `RedisUtil` 即可。Lettuce 客户端会自动处理 Master 地址的发现和故障转移。

**读写分离**：Lettuce 配合 Sentinel 可以更好地支持读写分离。可以通过 `spring.redis.lettuce.sentinel.read-from` 属性（或在手动配置`LettuceClientConfigurationBuilderCustomizer` Bean）指定读取策略，如 `MASTER` (仅主), `MASTER_PREFERRED` (主优先), `REPLICA` (仅从), `REPLICA_PREFERRED` (从优先)。

---

## 集群模式 (Cluster)

### 模式介绍

Redis Cluster 是 Redis 官方提供的分布式解决方案，用于解决单机 Redis 的容量瓶颈和提供更高的可用性与扩展性。

*   **数据分片 (Sharding)**：数据自动分布在多个 Redis Master 节点上（通过哈希槽 Slot）。
*   **高可用 (HA)**：每个 Master 节点可以有一个或多个 Slave 节点。当 Master 宕机时，其 Slave 会被自动提升为新的 Master，接管哈希槽。
*   **去中心化架构**：节点间通过 Gossip 协议互相通信，了解集群状态，客户端可以连接任意节点，会被自动重定向到正确的节点。
*   **线性扩展**：可以通过添加新的 Master 节点来扩展集群的容量和性能。

### Cluster 模式搭建

搭建 Redis Cluster 需要至少 3 个 Master 节点（官方推荐），通常每个 Master 配备一个 Slave 以实现高可用。搭建过程涉及节点配置、启动和使用 `redis-cli --cluster create` 命令创建集群。具体请参考官方文档或相关教程。

### `application.yaml` 配置

```yaml
spring:
  redis:
    password: your_cluster_password # Cluster 中所有节点的密码 (如果设置了)
    timeout: 5000ms              # 连接超时时间

    cluster:
      nodes:                     # Cluster 节点的地址列表 (IP:Port)
        - 47.116.114.234:54801
        - 47.116.114.234:54802
        - 139.224.249.155:54801
        # ... 其他节点 ...
        # 只需要提供部分集群节点即可，客户端会自动发现所有节点和槽分布

      max-redirects: 3           # 处理 MOVED/ASK 重定向的最大次数

    lettuce:
      pool:
        min-idle: 0
        max-idle: 8
        max-active: 8
        max-wait: -1ms # 或 -1
```
*   **`cluster.nodes`**: 提供集群中部分节点的地址列表。客户端会连接这些节点获取整个集群的拓扑结构（节点和槽位映射）。
*   **`cluster.max-redirects`**: 当客户端访问的 Key 不在当前连接的节点上时，节点会返回 MOVED 或 ASK 重定向指令，此参数限制了重定向的最大次数，防止无限重定向。

### 整合说明

配置好 `application.yaml` 后，Spring Boot 会自动配置 `LettuceConnectionFactory` 以集群模式连接。应用层使用 `RedisTemplate` 或 `RedisUtil` 的方式与单节点或 Sentinel 模式完全一致。Lettuce 客户端会自动处理 Key 的路由和节点故障转移。

**注意**：Redis Cluster 不支持涉及多个 Key 且这些 Key 不在同一个 Slot 的原子操作（如 `MSET`, `MGET`，除非使用 Hash Tags `{}` 强制 Key 在同一 Slot）。也不支持多数据库 (`select` 命令无效，默认只有 database 0)。

---

## 进阶探讨与最佳实践

*   **选择合适的模式**：
    *   **开发/测试/简单应用**: 单节点
    *   **需要高可用，数据量不大**: Sentinel
    *   **数据量大，需要水平扩展和高可用**: Cluster
*   **连接池调优**：`max-active`, `max-idle`, `min-idle`, `max-wait` 等参数需要根据应用的并发量和响应时间要求进行调整。监控连接池状态非常重要。
*   **序列化选择**：JSON (如 Jackson) 是通用且易读的选择。String 适合纯文本数据。避免使用 JDK 默认序列化。考虑 Protobuf、Kryo 等更高效的序列化库，但可能增加复杂性。
*   **客户端选择 (Lettuce vs Jedis)**：Spring Boot 2.x 默认 Lettuce。Lettuce 基于 Netty，是线程安全的异步客户端，性能通常更好，尤其是在高并发场景。Jedis 是老牌同步客户端，简单易用，但在需要共享连接时需配合连接池使用。
*   **超时设置**：合理设置 `timeout` (连接超时) 和命令执行超时（可能需要在更底层的客户端配置中设置）以避免长时间阻塞。
*   **监控**：使用 Redis 监控工具（如 RedisInsight, Prometheus+Grafana with Redis exporter）或 APM 工具监控 Redis 性能、内存使用、连接数等指标。

---

## 总结

Spring Boot 提供了强大的自动配置能力，使得整合 Redis 的各种模式变得相对简单。理解不同模式的特点和适用场景，正确配置 `application.yaml` 并选择合适的序列化方式，是成功在 Spring Boot 应用中使用 Redis 的关键。对于需要高可用和/或可扩展性的生产环境，Sentinel 和 Cluster 模式是更优的选择。记得根据实际负载情况调整连接池和超时参数，并进行充分的监控。