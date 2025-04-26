---
title: 非SpringBoot环境下Jedis工具操作Redis集群实战指南
tags:
  - Java
  - SpringBoot
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250424173823338.png'
toc: true
abbrlink: c2e80da6
date: 2024-08-20 10:36:39
---

在现代分布式系统中，Redis 作为高性能的内存数据存储解决方案，广泛被用于缓存、会话管理、排行榜等场景。为了满足高可用和扩展需求，Redis 集群成为首选架构。本文将聚焦于非 Spring Boot 应用环境，如何利用 Jedis 客户端来封装一个功能完备且易用的 Redis 工具类，从而简化对 Redis 集群的操作管理。

<!-- more -->

## 引入Jedis依赖

首先确保项目构建文件中正确引入 Jedis 依赖。以下示例基于 Maven：

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.7.1</version>
</dependency>
```

该版本提供了对 Redis 集群的良好支持，同时兼容多种功能调用。

## 构建 Redis 集群工具类

### 设计原则

1. 集中管理 JedisCluster 实例，避免重复创建连接，提升性能。
2. 统一添加命名空间前缀，便于多业务线隔离数据。
3. 提供对 Redis 常用数据结构（String、List、Set、Hash、ZSet）的丰富操作接口。
4. 支持动态配置，便于不同环境灵活调整参数。
5. 简洁易用，减少开发者上手门槛。

### 工具类示例讲解

```java
package com.liboshuai.slr.engine.biz.util;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import redis.clients.jedis.*;

import java.time.Duration;
import java.util.*;

/**
 * Redis 集群操作工具类
 * 支持多种数据类型操作及过期时间管理
 */
@Slf4j
public class RedisUtil {

    private static final JedisCluster jedisCluster;
    private static final String NAMESPACE;

    static {
        // 从参数管理器获取配置（示例中用假设工具类ParameterUtil）
        String clusterNodes = ParameterUtil.getParameters().get("redis.cluster.nodes");
        String[] nodes = clusterNodes.split(",");
        
        Set<HostAndPort> jedisClusterNodes = new HashSet<>();
        for (String node : nodes) {
            String[] hostPort = node.trim().split(":");
            jedisClusterNodes.add(new HostAndPort(hostPort[0], Integer.parseInt(hostPort[1])));
        }

        // 解密密码，假设使用工具类JasyptUtil支持，没密码则为空
        String decryptedPassword = ParameterUtil.getParameters().get("redis.password");
        String password = JasyptUtil.decrypt(decryptedPassword);

        // 命名空间前缀，默认为 starlink_risk_
        NAMESPACE = ParameterUtil.getParameters().get("redis.namespace", "starlink_risk") + ":";

        // 连接参数
        int connectionTimeout = Integer.parseInt(ParameterUtil.getParameters().get("redis.connectionTimeout"));
        int soTimeout = Integer.parseInt(ParameterUtil.getParameters().get("redis.soTimeout"));
        int maxAttempts = Integer.parseInt(ParameterUtil.getParameters().get("redis.maxAttempts"));

        // 连接池配置
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxWait(Duration.ofMillis(Long.parseLong(ParameterUtil.getParameters().get("redis.pool.maxWait"))));
        poolConfig.setTimeBetweenEvictionRuns(Duration.ofMillis(Long.parseLong(ParameterUtil.getParameters().get("redis.pool.timeBetweenEvictionRuns"))));
        poolConfig.setNumTestsPerEvictionRun(Integer.parseInt(ParameterUtil.getParameters().get("redis.pool.numTestsPerEvictionRun")));
        poolConfig.setMaxTotal(Integer.parseInt(ParameterUtil.getParameters().get("redis.pool.maxTotal")));
        poolConfig.setMaxIdle(Integer.parseInt(ParameterUtil.getParameters().get("redis.pool.maxIdle")));
        poolConfig.setMinIdle(Integer.parseInt(ParameterUtil.getParameters().get("redis.pool.minIdle")));
        poolConfig.setTestWhileIdle(true);

        // 初始化JedisCluster
        if (StringUtils.isNotBlank(password)) {
            jedisCluster = new JedisCluster(jedisClusterNodes, connectionTimeout, soTimeout, maxAttempts, password, poolConfig);
        } else {
            jedisCluster = new JedisCluster(jedisClusterNodes, connectionTimeout, soTimeout, maxAttempts, poolConfig);
        }
        log.info("RedisUtil 已初始化，使用命名空间：{}", NAMESPACE);
    }

    // 工具方法 - Key 操作

    public static boolean exists(String key) {
        return jedisCluster.exists(NAMESPACE + key);
    }

    public static void del(String key) {
        jedisCluster.del(NAMESPACE + key);
    }

    public static long ttl(String key) {
        return jedisCluster.ttl(NAMESPACE + key);
    }

    public static void expire(String key, long seconds) {
        jedisCluster.expire(NAMESPACE + key, (int) seconds);
    }

    // String 操作

    public static String getString(String key) {
        return jedisCluster.get(NAMESPACE + key);
    }

    public static void setString(String key, String value) {
        jedisCluster.set(NAMESPACE + key, value);
    }

    public static void setStringWithExpiry(String key, String value, long seconds) {
        jedisCluster.setex(NAMESPACE + key, (int) seconds, value);
    }

    // List 操作

    public static void lpush(String key, String... values) {
        jedisCluster.lpush(NAMESPACE + key, values);
    }

    public static List<String> lrange(String key, long start, long end) {
        return jedisCluster.lrange(NAMESPACE + key, start, end);
    }

    public static void lpushWithExpiry(String key, long seconds, String... values) {
        lpush(key, values);
        expire(key, seconds);
    }

    // Set 操作

    public static void sadd(String key, String... members) {
        jedisCluster.sadd(NAMESPACE + key, members);
    }

    public static Set<String> smembers(String key) {
        return jedisCluster.smembers(NAMESPACE + key);
    }

    public static void saddWithExpiry(String key, long seconds, String... members) {
        sadd(key, members);
        expire(key, seconds);
    }

    // Hash 操作

    public static void hset(String key, String field, String value) {
        jedisCluster.hset(NAMESPACE + key, field, value);
    }

    public static String hget(String key, String field) {
        return jedisCluster.hget(NAMESPACE + key, field);
    }

    public static Map<String, String> hgetAll(String key) {
        return jedisCluster.hgetAll(NAMESPACE + key);
    }

    public static void hsetWithExpiry(String key, long seconds, String field, String value) {
        hset(key, field, value);
        expire(key, seconds);
    }

    public static void hdel(String key, String... fields) {
        jedisCluster.hdel(NAMESPACE + key, fields);
    }

    // ZSet 操作

    public static void zadd(String key, double score, String member) {
        jedisCluster.zadd(NAMESPACE + key, score, member);
    }

    public static Set<String> zrange(String key, long start, long end) {
        return jedisCluster.zrange(NAMESPACE + key, start, end);
    }

    public static void zaddWithExpiry(String key, long seconds, double score, String member) {
        zadd(key, score, member);
        expire(key, seconds);
    }

    // 关闭资源（可选）
    public static void close() {
        try {
            jedisCluster.close();
            log.info("JedisCluster 资源已关闭");
        } catch (Exception e) {
            log.error("关闭 JedisCluster 异常", e);
        }
    }
}
```

### 关键点解析

- **命名空间管理**：通过添加固定前缀避免命名冲突，方便区分业务数据。
- **配置解耦**：所有配置均来自统一参数管理（如配置文件或配置中心），便于灵活调整。
- **安全管理**：密码采用加密存储及解密处理，保障敏感信息安全。
- **丰富数据结构支持**：覆盖String、List、Set、Hash、ZSet，满足常用业务需求。
- **超时及连接池配置**：合理配置连接池参数，保证连接健康和性能稳定。
- **日志输出**：初始化及关键操作记录日志，便于排查问题。

## 配置示例

在项目配置文件中添加 Redis 集群相关参数：

```properties
redis.cluster.nodes=one:6379,one:6380,two:6379,two:6380,three:6379,three:6380
redis.password=LBS(BJMQFVCQ5pkShzdex82tlFtxbLzHs5xl)
redis.namespace=starlink_risk_test
redis.connectionTimeout=3000
redis.soTimeout=3000
redis.maxAttempts=3
redis.pool.maxWait=3000
redis.pool.timeBetweenEvictionRuns=30000
redis.pool.numTestsPerEvictionRun=-1
redis.pool.maxTotal=200
redis.pool.maxIdle=200
redis.pool.minIdle=50
```

具体参数说明：

- **redis.cluster.nodes**： Redis集群所有节点地址，多个以逗号分隔。
- **redis.password**：Redis访问密码（加密存储）。
- **redis.namespace**：键名前缀，用于逻辑隔离。
- 其余均为连接和线程池相关的性能调优参数。

## 使用示例

```java
public class RedisDemo {

    public static void main(String[] args) {
        // 写入字符串
        RedisUtil.setString("user:1001", "张三");

        // 读取字符串
        String user = RedisUtil.getString("user:1001");
        System.out.println("用户信息：" + user);

        // 操作Hash对象
        RedisUtil.hset("user:1001:profile", "age", "30");
        String age = RedisUtil.hget("user:1001:profile", "age");
        System.out.println("用户年龄：" + age);

        // 操作有序集合
        RedisUtil.zadd("rankings", 1000, "user:1001");
        Set<String> topRank = RedisUtil.zrange("rankings", 0, 10);
        System.out.println("排行榜前十：" + topRank);
    }
}
```

## 扩展建议与注意事项

- **异常处理**：生产环境中，建议对方法调用做更细腻的异常捕获和处理，保证系统稳定。
- **连接关闭**：JedisCluster 连接是线程安全的，应当单例使用且在系统关闭时释放资源。
- **序列化支持**：本文示例均使用字符串操作，如果项目中存储复杂对象，可结合 JSON 序列化方案封装数据转换。
- **性能监控**：结合监控工具持续观察 Redis Cluster 状态，优化参数配置。
- **安全配置**：Redis 集群涉及权限管理时，可考虑 ACL 方案和访问控制。

## 总结

使用 Jedis 操作 Redis 集群在非 Spring Boot 环境同样高效且灵活。通过封装统一的工具类，不仅简化操作，还能将配置与业务解耦，使 Redis 成为业务加速的利器。合理配置连接池与超时参数，将显著提升系统稳定性和响应速度。希望本文示例能够助你快速搭建健壮的 Redis 集群交互层，为业务发展保驾护航。