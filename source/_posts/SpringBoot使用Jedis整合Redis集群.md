---
title: SpringBoot使用Jedis整合Redis集群
tags:
  - Java
  - SpringBoot
  - Redis
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260127545.png'
toc: true
abbrlink: 45b1597
date: 2024-06-28 01:23:47
---

Spring Boot 默认使用 Lettuce 作为 Redis 客户端，它支持响应式编程且易于使用。但在部分低版本 Spring Boot 中，Lettuce 与 Redis 集群存在版本兼容性问题，导致连接不稳定或无法连接集群。鉴于此，采用 Jedis 客户端作为 Redis 集群的连接方案成为了切实可行的替代方案。

本文详细介绍如何在 Spring Boot 项目中使用 Jedis 整合 Redis 集群，包含依赖配置、属性设置、关键代码实现以及调用示例，为实际项目中使用 Redis 集群提供完整参考。

<!-- more -->

> 相关文章源码地址请参考：[springboot-example 项目](https://gitee.com/liboshuai01/springboot-example) — redis-cluster-jedis 分支

---

## 引入关键依赖

在 Maven 的 `pom.xml` 中，除了引入 Spring Boot Web 和 Redis 相关依赖外，需要注意的是要排除默认的 Lettuce 依赖，改用 Jedis 客户端。相关依赖示例如下：

```xml
<properties>
    <maven.compiler.source>8</maven.compiler.source>
    <maven.compiler.target>8</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <spring.boot.version>2.2.5.RELEASE</spring.boot.version>
    <jedis.version>3.1.0</jedis.version>
    <fastjson.version>1.2.68</fastjson.version>
</properties>

<dependencyManagement>
    <dependencies>
        <!-- 统一依赖管理 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring.boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
        <exclusions>
            <!-- 排除 Lettuce，防止冲突 -->
            <exclusion>
                <groupId>io.lettuce</groupId>
                <artifactId>lettuce-core</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <dependency>
        <groupId>redis.clients</groupId>
        <artifactId>jedis</artifactId>
        <version>${jedis.version}</version>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>${fastjson.version}</version>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

此配置确保项目使用 Jedis 作为底层客户端，避免 Lettuce 与低版本 Spring Boot 造成的兼容性问题。

---

## 优化 Redis 集群配置文件

将 Redis 集群相关配置统一放到 Spring Boot 的配置文件中，例如 `application.properties`，方便集中管理和修改。

```properties
# Redis 集群节点，格式 ip:port，多个节点用逗号隔开
spring.redis.nodes=master:6379,master:6380,node1:6379,node1:6380,node2:6379,node2:6380

# 集群最大重定向次数，避免不断重定向带来性能开销
spring.redis.maxRedirects=3

# 如有密码，填写 Redis 连接密码
spring.redis.password=xxxxxx

# 客户端超时时间，单位毫秒，默认2000
spring.redis.timeout=10000

# Jedis连接池参数
spring.redis.maxIdle=10
spring.redis.maxTotal=300
spring.redis.maxWaitMillis=3000

# 空闲连接逐出时间间隔等连接池参数
spring.redis.minEvictableIdleTimeMillis=300000
spring.redis.numTestsPerEvictionRun=1024
spring.redis.timeBetweenEvictionRunsMillis=30000

# 连接检测设置
spring.redis.testOnBorrow=true
spring.redis.testWhileIdle=true
```

### 说明

- **nodes**：配置 Redis 集群所有节点的地址，可以根据实际环境动态调整。
- **maxRedirects**：集群状态切换时最快重定向请求次数，合理设置可保证连接稳定。
- **密码及超时**：支持基于授权和不同网络环境的定制化配置。
- **连接池参数**：关键提升 Redis 访问性能和稳定性，避免连接资源耗尽。

---

## 编写 Redis 配置类使用 Jedis

通过一个专门的 Spring 配置类完成 Jedis 客户端与 Redis 集群的连接初始化，并定义序列化方式以及事务支持。示例如下：

```java
@Configuration
public class RedisConfig {

    @Autowired
    private RedisProperties redisProperties;

    /**
     * 创建 Jedis 连接池配置，管理连接池相关参数
     */
    @Bean(name = "jedis.pool.config")
    public JedisPoolConfig jedisPoolConfig() {
        JedisPoolConfig poolConfig = new JedisPoolConfig();
        poolConfig.setMaxIdle(redisProperties.getMaxIdle());
        poolConfig.setMaxTotal(redisProperties.getMaxTotal());
        poolConfig.setMaxWaitMillis(redisProperties.getMaxWaitMillis());
        poolConfig.setMinEvictableIdleTimeMillis(redisProperties.getMinEvictableIdleTimeMillis());
        poolConfig.setNumTestsPerEvictionRun(redisProperties.getNumTestsPerEvictionRun());
        poolConfig.setTimeBetweenEvictionRunsMillis(redisProperties.getTimeBetweenEvictionRunsMillis());
        poolConfig.setTestOnBorrow(redisProperties.isTestOnBorrow());
        poolConfig.setTestWhileIdle(redisProperties.isTestWhileIdle());
        return poolConfig;
    }

    /**
     * 配置 Redis 集群节点及授权密码
     */
    @Bean("redis.cluster.config")
    public RedisClusterConfiguration redisClusterConfiguration() {
        String nodes = redisProperties.getNodes();
        if (nodes == null || nodes.trim().isEmpty()) {
            throw new IllegalArgumentException("请配置spring.redis.nodes！至少包含一个Redis集群节点");
        }

        RedisClusterConfiguration clusterConfig = new RedisClusterConfiguration();
        Set<RedisNode> nodeSet = new HashSet<>();
        String[] nodesArray = nodes.split(",");
        for (String node : nodesArray) {
            String[] hostAndPort = node.trim().split(":");
            if (hostAndPort.length != 2) {
                throw new IllegalArgumentException("redis节点配置格式错误，正确格式如：ip:port");
            }
            String host = hostAndPort[0].trim();
            int port = Integer.parseInt(hostAndPort[1].trim());
            nodeSet.add(new RedisNode(host, port));
        }

        clusterConfig.setClusterNodes(nodeSet);
        clusterConfig.setMaxRedirects(redisProperties.getMaxRedirects());
        clusterConfig.setPassword(redisProperties.getPassword());
        return clusterConfig;
    }

    /**
     * 创建 JedisConnectionFactory，集群版
     */
    @Bean(name = "jedis.cluster.factory")
    public JedisConnectionFactory jedisConnectionFactory(
            @Qualifier("redis.cluster.config") RedisClusterConfiguration clusterConfig,
            @Qualifier("jedis.pool.config") JedisPoolConfig poolConfig) {
        return new JedisConnectionFactory(clusterConfig, poolConfig);
    }

    /**
     * 初始化 RedisTemplate 对象，指定序列化器并开启事务支持
     */
    @Bean(name = "redisTemplate")
    @Autowired
    public RedisTemplate<String, Object> redisTemplate(
            @Qualifier("jedis.cluster.factory") JedisConnectionFactory jedisConnectionFactory) {

        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(jedisConnectionFactory);

        // 使用 FastJson 作为值序列化方式，提升读取性能与兼容性
        GenericFastJsonRedisSerializer fastJsonSerializer = new GenericFastJsonRedisSerializer();

        // Key序列化采用字符串序列化
        template.setKeySerializer(new GenericToStringSerializer<>(Object.class));
        template.setHashKeySerializer(new GenericToStringSerializer<>(Object.class));

        // Value序列化采用 FastJson 序列化
        template.setValueSerializer(fastJsonSerializer);
        template.setHashValueSerializer(fastJsonSerializer);

        // 开启事务支持，常用于多命令原子执行
        template.setEnableTransactionSupport(true);

        template.afterPropertiesSet();
        return template;
    }
}
```

### 设计亮点

- **参数集成**：所有配置均从 `RedisProperties` 配置对象注入，配置更灵活。
- **连接池优化**：完整设置 Jedis 连接池，确保连接性能和健康。
- **序列化优化**：采用 `GenericFastJsonRedisSerializer` 代替默认 JDK 序列化，提升存取效率，防止乱码。
- **事务支持**：适应需要多命令事务场景，增强 Redis 操作的完整性保障。

---

## 封装 Redis 参数映射类

该类负责映射 Spring Boot 配置文件中的 `spring.redis` 前缀属性，简化配置读取和管理。

```java
@Data
@Component
@ConfigurationProperties(prefix = "spring.redis")
public class RedisProperties {

    private String nodes;

    private Integer maxRedirects;

    private String password;

    private int timeout;

    private Integer maxIdle;

    private Integer maxTotal;

    private Integer maxWaitMillis;

    private Integer minEvictableIdleTimeMillis;

    private Integer numTestsPerEvictionRun;

    private long timeBetweenEvictionRunsMillis;

    private boolean testOnBorrow;

    private boolean testWhileIdle;
}
```

**备注**：其中使用 Lombok 的 `@Data` 生成 getter/setter，减少样板代码。

---

## 简单测试及验证

通过 Controller 验证 Redis 集群的连接及数据读写功能。示例代码：

```java
@RestController
public class HelloController {

    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    @GetMapping("/setHello")
    public String setHello() {
        redisTemplate.opsForValue().set("hello", "world");
        return "OK!";
    }

    @GetMapping("/getHello")
    public String getHello() {
        Object value = redisTemplate.opsForValue().get("hello");
        return value != null ? value.toString() : "null";
    }
}
```

调用 `/setHello` 接口会把 key 为 `hello` 的字符串写入 Redis，调用 `/getHello` 接口则能正确获取到刚存入的值，验证集群访问正常。

---

## 拓展与优化建议

- **哨兵模式支持**：基于 Jedis，也可以简化扩展为哨兵集群连接。详情可参考官方文档及相关博客教程。
- **单机模式兼容**：单节点 Redis 环境也可通过类似配置实施 Jedis 连接，调整连接工厂即可。
- **连接池监控**：实用时可结合如 Spring Boot Actuator 或 监控平台实时观察连接池状态及性能。
- **序列化方案多样化**：FastJson 外，也可尝试 Jackson、Kryo 等性能更优的序列化方式。
- **安全配置**：确保 Redis 密码安全，必要时增加客户端 IP 白名单与通信加密措施。

---

## 总结

本文通过配置 Maven 依赖、Spring Boot 属性、Java 代码实现，完整演示了如何用 Jedis 高效稳定地连接 Redis 集群，解决了 Lettuce 不兼容的问题。该方案结构清晰、扩展性强，适合生产环境使用。

如果您使用的是低版本 Spring Boot 且面对 Redis 集群连接难题，使用本文方案可以快速上手并应用到实际项目中。

---

**相关阅读**

- [哨兵与单机模式 Jedis 集成示例及源码](https://www.sunofbeach.net/a/1465497779588587522)
- [Spring Data Redis 官方文档](https://docs.spring.io/spring-data/redis/docs/current/reference/html/)

祝您开发顺利！