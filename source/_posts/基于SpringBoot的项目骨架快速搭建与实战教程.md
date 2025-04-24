---
title: 基于SpringBoot的项目骨架快速搭建与实战教程
tags:
  - Java
  - SpringBoot
  - Redis
  - Mysql
  - Knife4j
  - Json
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250424130433239.png'
toc: true
abbrlink: a4d0387e
date: 2023-12-18 12:59:57
---

本文旨在帮助开发者快速搭建一个基于 Spring Boot 的小型项目骨架，涵盖从基础数据库连接配置、Redis 缓存集成、日志系统搭建，到接口设计和前后端数据交互的全流程关键环节。通过详细的配置示例和代码实现，帮助初学者以及有一定经验的开发者快速上手，构建结构清晰、性能稳定且易于维护的后台服务。项目中采用了 MyBatis-Plus 简化数据库操作，Druid 实现高性能连接池管理，Redis 提升系统缓存能力，Knife4j 优化接口文档展示。此外，日志配置支持灵活的日志分级和文件切割，方便生产问题排查。本文内容适合用于学习、参考，乃至作为日常开发的实用模板，为后续功能扩展和二次开发打下坚实基础。无论是个人学习还是团队协作，都将极大提升开发效率和系统质量。

<!-- more -->

项目源码开源地址：
- [Gitee 仓库（mybatisPlus-redis 分支）](https://gitee.com/liboshuai01/springboot-example.git)

## 一、项目环境依赖配置

本项目采用以下技术栈及版本：

- JDK 1.8
- Spring Boot 2.7.6
- MySQL 8.2.0
- Redis 7.0.12
- Knife4j (Swagger UI) 4.3.0

### Maven 依赖引入

pom.xml 配置中重点依赖如下：

```xml
<properties>
    <java.version>1.8</java.version>
    <spring-boot.version>2.7.6</spring-boot.version>
    <mysql-connector-j.version>8.0.33</mysql-connector-j.version>
    <druid.version>1.2.20</druid.version>
    <mybatis-plus.version>3.5.4</mybatis-plus.version>
    <lombok.version>1.18.30</lombok.version>
    <knife4j.version>4.3.0</knife4j.version>
</properties>

<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MySQL Connector -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <version>${mysql-connector-j.version}</version>
    </dependency>

    <!-- Druid 数据源 -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>${druid.version}</version>
    </dependency>

    <!-- Mybatis-Plus ORM -->
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>${mybatis-plus.version}</version>
    </dependency>

    <!-- Redis 支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
    </dependency>

    <!-- Lombok 简化代码 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
    </dependency>

    <!-- Knife4j (Swagger) API 文档 -->
    <dependency>
        <groupId>com.github.xiaoymin</groupId>
        <artifactId>knife4j-openapi3-spring-boot-starter</artifactId>
        <version>${knife4j.version}</version>
    </dependency>

    <!-- 测试依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

此外，编译插件指定了 Java 版本及编码，保证项目编译一致性与可移植性。

## 二、Spring Boot 配置文件细节

合理划分不同环境配置文件，便于开发、测试和生产环境参数分离。

### 基础配置 application.properties

```properties
# 服务器端口及环境
server.port=8080
spring.profiles.active=@profilesActive@

# MyBatis-Plus
mybatis-plus.mapper-locations=classpath:/mapper/*Mapper.xml
mybatis-plus.type-aliases-package=com.liboshuai.springbootexample.entity

# Druid 连接池配置信息（基础性能调优）
spring.datasource.druid.initial-size=5
spring.datasource.druid.min-idle=5
spring.datasource.druid.maxActive=20
spring.datasource.druid.maxWait=60000
spring.datasource.druid.validationQuery=SELECT 1
spring.datasource.druid.testWhileIdle=true
spring.datasource.druid.poolPreparedStatements=true
spring.datasource.druid.maxPoolPreparedStatementPerConnectionSize=20

# Druid 监控功能配置
spring.datasource.druid.filter.slf4j.enabled=true
spring.datasource.druid.filter.wall.enabled=true
spring.datasource.druid.filter.stat.enabled=true
spring.datasource.druid.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
spring.datasource.druid.web-stat-filter.enabled=true
spring.datasource.druid.web-stat-filter.url-pattern=/*
spring.datasource.druid.web-stat-filter.exclusions=*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*
spring.datasource.druid.stat-view-servlet.enabled=true
spring.datasource.druid.stat-view-servlet.url-pattern=/druid/*
spring.datasource.druid.stat-view-servlet.reset-enable=false
spring.datasource.druid.stat-view-servlet.login-username=admin
spring.datasource.druid.stat-view-servlet.login-password=123456

# Swagger / Knife4j 配置
springdoc.swagger-ui.path=/swagger-ui.html
knife4j.enable=true
knife4j.setting.language=zh_cn

# JSON 相关序列化配置
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
spring.jackson.default-property-inclusion=non_null
spring.jackson.serialization.indent_output=true
spring.jackson.serialization.fail_on_empty_beans=false
spring.jackson.deserialization.fail_on_unknown_properties=false
spring.jackson.parser.allow_unquoted_control_chars=true
spring.jackson.parser.allow_single_quotes=true
```

### 开发环境配置 application-dev.properties

```properties
# MySQL 数据库连接
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.username=root
spring.datasource.password=Rongshu@2024
spring.datasource.url=jdbc:mysql://a.sanhans.icu:18306/spring_boot_example?useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true&nullCatalogMeansCurrent=true

# Redis 连接配置
spring.redis.database=0
spring.redis.host=a.sanhans.icu
spring.redis.port=18381
spring.redis.password=admin123456
spring.redis.timeout=5000
spring.redis.lettuce.pool.min-idle=0
spring.redis.lettuce.pool.max-idle=8
spring.redis.lettuce.pool.max-active=8
spring.redis.lettuce.pool.max-wait=-1
```

> 其他环境如 UAT 和 PROD 的配置方式相同，可根据需要替换相应参数。

## 三、日志系统配置

引入了 Logback 日志框架并配置了控制台及文件日志输出，支持按日期切割和日志级别过滤。

`resources/logback-spring.xml` 文件内容精简示例：

```xml
<configuration scan="true" scanPeriod="60 seconds">

    <property name="log.path" value="/opt/projects/springboot-example/logs"/>
    <property name="log.pattern" value="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{20} - [%method,%line] - %msg%n"/>

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder><pattern>${log.pattern}</pattern></encoder>
    </appender>

    <appender name="file_info" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/info.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/info.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder><pattern>${log.pattern}</pattern></encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>INFO</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <appender name="file_error" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${log.path}/error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/error.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>60</maxHistory>
        </rollingPolicy>
        <encoder><pattern>${log.pattern}</pattern></encoder>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <logger name="com.liboshuai.springbootexample" level="info"/>
    <logger name="org.springframework" level="warn"/>

    <root level="info">
        <appender-ref ref="console"/>
        <appender-ref ref="file_info"/>
        <appender-ref ref="file_error"/>
    </root>

</configuration>
```

日志路径和格式均可自定义，确保系统运行时方便排查和监控。

## 四、启动类设计

项目主启动类：

```java
@SpringBootApplication
public class SpringbootExampleApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringbootExampleApplication.class, args);
    }
}
```

采用标准注解自动装配，Spring Boot 默认组件扫描根包为当前类所在包。

## 五、Redis 配置与工具封装

### Redis 配置类

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        Jackson2JsonRedisSerializer<Object> jacksonSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        jacksonSerializer.setObjectMapper(om);

        StringRedisSerializer stringSerializer = new StringRedisSerializer();

        template.setKeySerializer(stringSerializer);
        template.setHashKeySerializer(stringSerializer);

        template.setValueSerializer(jacksonSerializer);
        template.setHashValueSerializer(jacksonSerializer);

        template.afterPropertiesSet();
        return template;
    }
}
```

该配置实现了 RedisTemplate 序列化优化，使用 Jackson 序列化存储对象，键使用字符串序列化，提高可读性和互操作性。

### Redis 工具类（RedisUtil）

封装常用 Redis 操作，涵盖 Key、String、Hash、List、Set、ZSet 各种数据类型，提升业务代码调用简洁度。

示例方法：

```java
@Component
public class RedisUtil {

    private final RedisTemplate<String, Object> redisTemplate;

    public RedisUtil(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    // -------------------key 相关操作---------------------

    /**
     * 删除 key
     *
     * @param key 要删除的 键
     */
    public void delete(String key) {
        redisTemplate.delete(key);
    }

    /**
     * 批量删除 key
     *
     * @param keys 要删除的 键 的集合
     */
    public void delete(Collection<String> keys) {
        redisTemplate.delete(keys);
    }

    /**
     * 是否存在 key
     *
     * @param key 判断该 键 是否存在
     * @return 存在返回 true, 不存在返回 false
     */
    public Boolean hasKey(String key) {
        return redisTemplate.hasKey(key);
    }

    /**
     * 设置过期时间
     *
     * @param key     要设置的 键
     * @param timeout 过期时间
     * @param unit    时间单位
     * @return 设置成功返回 true, 设置失败返回 false
     */
    public Boolean expire(String key, long timeout, TimeUnit unit) {
        return redisTemplate.expire(key, timeout, unit);
    }

    /**
     * 设置在什么时间过期
     *
     * @param key  要设置的 键
     * @param date 过期时间
     * @return 设置成功返回 true, 设置失败返回 false
     */
    public Boolean expireAt(String key, Date date) {
        return redisTemplate.expireAt(key, date);
    }

    /**
     * 查找匹配的 key
     *
     * @param pattern 匹配字符串
     * @return 满足匹配条件的 键 的 Set 集合
     */
    public Set<String> keys(String pattern) {
        return redisTemplate.keys(pattern);
    }

    /**
     * 将当前数据库的 key 移动到给定的数据库 db 当中
     *
     * @param key     要移动的 键
     * @param dbIndex 要移动到的 db 的序号, 从 0 开始
     * @return 移动成功返回 true, 移动失败返回 false
     */
    public Boolean move(String key, int dbIndex) {
        return redisTemplate.move(key, dbIndex);
    }

    /**
     * 移除 key 的过期时间，key 将持久保持
     *
     * @param key 要移除过期时间的 键
     * @return 移除成功返回 true, 并且该 key 将持久存在, 移除失败返回 false
     */
    public Boolean persist(String key) {
        return redisTemplate.persist(key);
    }

    /**
     * 返回 key 的剩余的过期时间
     *
     * @param key  要查询剩余过期时间的 键
     * @param unit 时间的单位
     * @return 剩余的过期时间
     */
    public Long getExpire(String key, TimeUnit unit) {
        return redisTemplate.getExpire(key, unit);
    }

    /**
     * 返回 key 的剩余的过期时间, 默认时间单位: 秒
     *
     * @param key 要查询剩余过期时间的 键
     * @return 剩余的过期时间, 单位: 秒
     */
    public Long getExpire(String key) {
        return redisTemplate.getExpire(key);
    }

    /**
     * 从当前数据库中随机返回一个 key
     *
     * @return 随机获取的 键 的名称
     */
    public String randomKey() {
        return redisTemplate.randomKey();
    }

    /**
     * 修改 key 的名称
     *
     * @param oldKey 修改前的 键 的名称
     * @param newKey 修改后的 键 的名称
     */
    public void rename(String oldKey, String newKey) {
        redisTemplate.rename(oldKey, newKey);
    }

    /**
     * 仅当 newkey 不存在时，将 oldKey 改名为 newkey
     *
     * @param oldKey 修改前的 键 的名称
     * @param newKey 修改后的 键 的名称
     * @return 修改成功返回 true, 修改失败返回 false
     */
    public Boolean renameIfAbsent(String oldKey, String newKey) {
        return redisTemplate.renameIfAbsent(oldKey, newKey);
    }

    /**
     * 返回 key 所储存的值的类型
     *
     * @param key 要查询类型的 键
     * @return key 的数据类型
     */
    public DataType type(String key) {
        return redisTemplate.type(key);
    }

    // -------------------string 相关操作---------------------

    /**
     * 设置指定 key 的值
     *
     * @param key   要设置的 键
     * @param value 要设置的 值
     */
    public void set(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    /**
     * 获取指定 key 的值
     *
     * @param key 键 的名称
     * @return 键 对应的 值
     */
    public Object get(String key) {
        return redisTemplate.opsForValue().get(key);
    }

    /**
     * 返回 key 的 字符串值 中指定位置的 子字符串
     *
     * @param key   要获取值的 键
     * @param start 开始位置, 最小值: 0
     * @param end   结束位置, 最大值: 字符串 - 1, 若为 -1 则是获取整个字符串值
     * @return 指定 键 的 字符串值 的 子字符串
     */
    public String getRange(String key, long start, long end) {
        return redisTemplate.opsForValue().get(key, start, end);
    }

    /**
     * 将给定 key 的值设为 value ，并返回 key 的旧值( old value )
     *
     * @param key   要设置值的 键
     * @param value 新值
     * @return 旧值
     */
    public Object getAndSet(String key, Object value) {
        return redisTemplate.opsForValue().getAndSet(key, value);
    }

    /**
     * 对 key 所储存的字符串值，获取指定偏移量上的位( bit )
     *
     * @param key    键
     * @param offset 偏移量
     * @return 指定偏移量上的 位( 0 / 1)
     */
    public Boolean getBit(String key, long offset) {
        return redisTemplate.opsForValue().getBit(key, offset);
    }

    /**
     * 批量获取 key 的 值
     *
     * @param keys 要获取值的 键 的集合
     * @return key对应的值的集合
     */
    public List<Object> multiGet(Collection<String> keys) {
        return redisTemplate.opsForValue().multiGet(keys);
    }

    /**
     * 设置ASCII码, 字符串'a'的ASCII码是97, 转为二进制是'01100001', 此方法是将二进制第 offset 位值变为 value
     *
     * @param key    要设置的 键
     * @param offset 偏移多少位
     * @param value  值, true 为 1,  false 为 0
     * @return 设置成功返回 true, 设置失败返回 false
     */
    public Boolean setBit(String key, long offset, boolean value) {
        return redisTemplate.opsForValue().setBit(key, offset, value);
    }

    /**
     * 将值 value 关联到 key ，并将 key 的过期时间设为 timeout
     *
     * @param key     键
     * @param value   值
     * @param timeout 过期时间
     * @param unit    时间单位, 天: TimeUnit.DAYS 小时: TimeUnit.HOURS 分钟: TimeUnit.MINUTES
     *                秒: TimeUnit.SECONDS 毫秒: TimeUnit.MILLISECONDS
     */
    public void setEx(String key, Object value, long timeout, TimeUnit unit) {
        redisTemplate.opsForValue().set(key, value, timeout, unit);
    }

    /**
     * 只有在 key 不存在时设置 key 的值
     *
     * @param key   键
     * @param value 值
     * @return 之前已经存在返回 false, 不存在返回 true
     */
    public Boolean setIfAbsent(String key, Object value) {
        return redisTemplate.opsForValue().setIfAbsent(key, value);
    }

    /**
     * 用 value 参数覆写给定 key 所储存的字符串值，从偏移量 offset 开始
     *
     * @param key    键
     * @param value  值
     * @param offset 从指定位置开始覆写
     */
    public void setRange(String key, Object value, long offset) {
        redisTemplate.opsForValue().set(key, value, offset);
    }

    /**
     * 获取字符串的长度
     *
     * @param key 键
     * @return 该 key 对应的 值的长度
     */
    public Long size(String key) {
        return redisTemplate.opsForValue().size(key);
    }

    /**
     * 批量添加 key-value
     *
     * @param maps key-value 的 map 集合
     */
    public void multiSet(Map<String, Object> maps) {
        redisTemplate.opsForValue().multiSet(maps);
    }

    /**
     * 同时设置一个或多个 key-value 对，当且仅当所有给定 key 都不存在才会设置成功
     *
     * @param maps key-value 集合
     * @return 之前已经存在返回 false, 不存在返回 true
     */
    public Boolean multiSetIfAbsent(Map<String, Object> maps) {
        return redisTemplate.opsForValue().multiSetIfAbsent(maps);
    }

    /**
     * 增加(自增长), 负数则为自减
     *
     * @param key       键
     * @param increment 自增量
     * @return 增加后的值
     */
    public Long incrBy(String key, long increment) {
        return redisTemplate.opsForValue().increment(key, increment);
    }

    /**
     * 自增长, 增长量为浮点数
     *
     * @param key       键
     * @param increment 自增量
     * @return 增加后的值
     */
    public Double incrByFloat(String key, double increment) {
        return redisTemplate.opsForValue().increment(key, increment);
    }

    /**
     * 将 value 追加到指定 key 的值的末尾
     *
     * @param key   键
     * @param value 要追加的值
     * @return 追加值后新值的长度
     */
    public Integer append(String key, String value) {
        return redisTemplate.opsForValue().append(key, value);
    }

    // -------------------hash 相关操作-------------------------

    /**
     * 获取存储在哈希表中指定字段的值
     *
     * @param key   键
     * @param field 字段名( 即 map 中的 key )
     * @return 值
     */
    public Object hGet(String key, String field) {
        return redisTemplate.opsForHash().get(key, field);
    }

    /**
     * 获取给定 哈希表 中的所有键值对
     *
     * @param key 哈希表
     * @return 所有的 键值对
     */
    public Map<Object, Object> hGetAll(String key) {
        return redisTemplate.opsForHash().entries(key);
    }

    /**
     * 获取指定 哈希表 中所有给定字段的值
     *
     * @param key    哈希表
     * @param fields 要获取值的字段集合
     * @return 哈希表中所有给定字段的值
     */
    public List<Object> hMultiGet(String key, Collection<Object> fields) {
        return redisTemplate.opsForHash().multiGet(key, fields);
    }

    /**
     * 向指定 哈希表 中存储一个 键值对
     *
     * @param key     哈希表
     * @param hashKey 字段名
     * @param value   值
     */
    public void hPut(String key, String hashKey, Object value) {
        redisTemplate.opsForHash().put(key, hashKey, value);
    }

    /**
     * 向指定 哈希表 中存储多个 键值对
     *
     * @param key  哈希表
     * @param maps 键值对集合
     */
    public void hPutAll(String key, Map<String, Object> maps) {
        redisTemplate.opsForHash().putAll(key, maps);
    }

    /**
     * 仅当 hashKey 不存在时才设置
     *
     * @param key     哈希表
     * @param hashKey 字段名
     * @param value   值
     * @return 设置成功返回 true, 设置失败返回 false
     */
    public Boolean hPutIfAbsent(String key, String hashKey, Object value) {
        return redisTemplate.opsForHash().putIfAbsent(key, hashKey, value);
    }

    /**
     * 删除哈希表中一个或多个字段
     *
     * @param key    哈希表
     * @param fields 要删除的字段集合
     * @return 删除成功的数目
     */
    public Long hDelete(String key, Object... fields) {
        return redisTemplate.opsForHash().delete(key, fields);
    }

    /**
     * 查看哈希表中指定的字段是否存在
     *
     * @param key   要查看的哈希表
     * @param field 字段名
     * @return 存在返回 true, 不存在返回 false
     */
    public boolean hExists(String key, String field) {
        return redisTemplate.opsForHash().hasKey(key, field);
    }

    /**
     * 为哈希表 key 中的指定字段的整数值加上增量 increment
     *
     * @param key       指定的哈希表
     * @param field     字段名
     * @param increment 增加的量
     * @return 增加后的值
     */
    public Long hIncrBy(String key, Object field, long increment) {
        return redisTemplate.opsForHash().increment(key, field, increment);
    }

    /**
     * 为哈希表 key 中的指定字段的整数值加上增量 increment( 浮点型 )
     *
     * @param key   指定的哈希表
     * @param field 字段名
     * @param delta 增加的量( 浮点型 )
     * @return 增加后的值
     */
    public Double hIncrByFloat(String key, Object field, double delta) {
        return redisTemplate.opsForHash().increment(key, field, delta);
    }

    /**
     * 获取所有哈希表中的字段
     *
     * @param key 哈希表
     * @return 所有的 字段
     */
    public Set<Object> hKeys(String key) {
        return redisTemplate.opsForHash().keys(key);
    }

    /**
     * 获取哈希表中字段的数量
     *
     * @param key 哈希表
     * @return 哈希表所有字段的数量
     */
    public Long hSize(String key) {
        return redisTemplate.opsForHash().size(key);
    }

    /**
     * 获取哈希表中所有值
     *
     * @param key 哈希表
     * @return 哈希表中所有的值
     */
    public List<Object> hValues(String key) {
        return redisTemplate.opsForHash().values(key);
    }

    /**
     * 迭代哈希表中的键值对
     *
     * @param key     哈希表
     * @param options 迭代的限制条件, 为 ScanOptions.NONE 则无限制
     * @return 下一个键值对元组的游标
     */
    public Cursor<Entry<Object, Object>> hScan(String key, ScanOptions options) {
        return redisTemplate.opsForHash().scan(key, options);
    }

    // ------------------------list 相关操作----------------------------

    /**
     * 通过索引获取列表中的元素
     *
     * @param key   元素所在的列表
     * @param index 下标, 从 0 开始
     * @return 列表中指定下标的值
     */
    public Object lIndex(String key, long index) {
        return redisTemplate.opsForList().index(key, index);
    }

    /**
     * 获取列表指定范围内的元素
     *
     * @param key   元素所在列表
     * @param start 开始位置, 0 是开始位置
     * @param end   结束位置, -1 返回所有
     * @return 指定索引范围内的元素
     */
    public List<Object> lRange(String key, long start, long end) {
        return redisTemplate.opsForList().range(key, start, end);
    }

    /**
     * 存储在 list 头部( 左边 )
     *
     * @param key   列表
     * @param value 存储的值
     * @return 列表长度
     */
    public Long lLeftPush(String key, Object value) {
        return redisTemplate.opsForList().leftPush(key, value);
    }

    /**
     * 将多个值存入列表中
     *
     * @param key   列表
     * @param value 值, 可以输入多个
     * @return 列表长度
     */
    public Long lLeftPushAll(String key, Object... value) {
        return redisTemplate.opsForList().leftPushAll(key, value);
    }

    /**
     * 将多个值存入列表中
     *
     * @param key   列表
     * @param value 值的集合
     * @return 列表长度
     */
    public Long lLeftPushAll(String key, Collection<Object> value) {
        return redisTemplate.opsForList().leftPushAll(key, value);
    }

    /**
     * 当 list 存在的时候才加入
     *
     * @param key   列表
     * @param value 值
     * @return 列表长度
     */
    public Long lLeftPushIfPresent(String key, Object value) {
        return redisTemplate.opsForList().leftPushIfPresent(key, value);
    }

    /**
     * 如果 pivot 存在,在 pivot 前面添加
     *
     * @param key   列表
     * @param pivot 基准值
     * @param value 要添加的值
     * @return 列表长度
     */
    public Long lLeftPush(String key, Object pivot, Object value) {
        return redisTemplate.opsForList().leftPush(key, pivot, value);
    }

    /**
     * 存储在 list 尾部( 右边 )
     *
     * @param key   列表
     * @param value 值
     * @return 列表长度
     */
    public Long lRightPush(String key, Object value) {
        return redisTemplate.opsForList().rightPush(key, value);
    }

    /**
     * 将多个值存入列表中
     *
     * @param key   列表
     * @param value 值, 可以输入多个
     * @return 列表长度
     */
    public Long lRightPushAll(String key, Object... value) {
        return redisTemplate.opsForList().rightPushAll(key, value);
    }

    /**
     * 将多个值存入列表中
     *
     * @param key   列表
     * @param value 值的集合
     * @return 列表长度
     */
    public Long lRightPushAll(String key, Collection<Object> value) {
        return redisTemplate.opsForList().rightPushAll(key, value);
    }

    /**
     * 为已存在的列表添加值
     *
     * @param key   存在的列表
     * @param value 值
     * @return 列表长度
     */
    public Long lRightPushIfPresent(String key, Object value) {
        return redisTemplate.opsForList().rightPushIfPresent(key, value);
    }

    /**
     * 在 pivot 元素的右边添加值
     *
     * @param key   列表
     * @param pivot 基准值
     * @param value 要添加的值
     * @return 列表长度
     */
    public Long lRightPush(String key, Object pivot, Object value) {
        return redisTemplate.opsForList().rightPush(key, pivot, value);
    }

    /**
     * 通过索引设置列表元素的值
     *
     * @param key   列表
     * @param index 位置
     * @param value 值
     */
    public void lSet(String key, long index, Object value) {
        redisTemplate.opsForList().set(key, index, value);
    }

    /**
     * 移出并获取列表的第一个元素
     *
     * @param key 列表
     * @return 删除的元素
     */
    public Object lLeftPop(String key) {
        return redisTemplate.opsForList().leftPop(key);
    }

    /**
     * 移出并获取列表的第一个元素, 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
     *
     * @param key     列表
     * @param timeout 等待时间
     * @param unit    时间单位
     * @return 删除的元素
     */
    public Object lBLeftPop(String key, long timeout, TimeUnit unit) {
        return redisTemplate.opsForList().leftPop(key, timeout, unit);
    }

    /**
     * 移除并获取列表最后一个元素
     *
     * @param key 列表
     * @return 删除的元素
     */
    public Object lRightPop(String key) {
        return redisTemplate.opsForList().rightPop(key);
    }

    /**
     * 移出并获取列表的最后一个元素, 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
     *
     * @param key     列表
     * @param timeout 等待时间
     * @param unit    时间单位
     * @return 删除的元素
     */
    public Object lBRightPop(String key, long timeout, TimeUnit unit) {
        return redisTemplate.opsForList().rightPop(key, timeout, unit);
    }

    /**
     * 移除列表的最后一个元素，并将该元素添加到另一个列表并返回
     *
     * @param sourceKey      要移除元素的列表
     * @param destinationKey 要添加元素的列表
     * @return 移动的元素
     */
    public Object lRightPopAndLeftPush(String sourceKey, String destinationKey) {
        return redisTemplate.opsForList().rightPopAndLeftPush(sourceKey, destinationKey);
    }

    /**
     * 从列表中弹出一个值，将弹出的元素插入到另外一个列表中并返回它; 如果列表没有元素会阻塞列表直到等待超时或发现可弹出元素为止
     *
     * @param sourceKey      要移除元素的列表
     * @param destinationKey 要添加元素的列表
     * @param timeout        等待时间
     * @param unit           时间单位
     * @return 移动的元素
     */
    public Object lBRightPopAndLeftPush(String sourceKey, String destinationKey, long timeout, TimeUnit unit) {
        return redisTemplate.opsForList().rightPopAndLeftPush(sourceKey, destinationKey, timeout, unit);
    }

    /**
     * 删除集合中值等于 value 的元素
     *
     * @param key   列表
     * @param index index = 0, 删除所有值等于value的元素;
     *              index > 0, 从头部开始删除第一个值等于 value 的元素;
     *              index < 0, 从尾部开始删除第一个值等于 value 的元素;
     * @param value 值
     * @return 列表长度
     */
    public Long lRemove(String key, long index, Object value) {
        return redisTemplate.opsForList().remove(key, index, value);
    }

    /**
     * 裁剪 list
     *
     * @param key   列表
     * @param start 起始位置
     * @param end   结束位置
     * @see <a href="https://redis.io/commands/ltrim">Redis Documentation: LTRIM</a>
     */
    public void lTrim(String key, long start, long end) {
        redisTemplate.opsForList().trim(key, start, end);
    }

    /**
     * 获取列表长度
     *
     * @param key 列表
     * @return 列表长度
     */
    public Long lLen(String key) {
        return redisTemplate.opsForList().size(key);
    }

    // --------------------set 相关操作--------------------------

    /**
     * set 添加元素
     *
     * @param key    集合
     * @param values 值, 可以同时添加多个
     * @return 集合长度
     */
    public Long sAdd(String key, Object... values) {
        return redisTemplate.opsForSet().add(key, values);
    }

    /**
     * set 移除元素
     *
     * @param key    集合
     * @param values 要移除的元素, 可以同时移除多个
     * @return 集合长度
     */
    public Long sRemove(String key, Object... values) {
        return redisTemplate.opsForSet().remove(key, values);
    }

    /**
     * 移除并返回集合的一个随机元素
     *
     * @param key 集合
     * @return 集合中随机一个元素
     */
    public Object sPop(String key) {
        return redisTemplate.opsForSet().pop(key);
    }

    /**
     * 将元素 value 从一个集合移到另一个集合
     *
     * @param key     被移除的集合
     * @param value   要移除的元素
     * @param destKey 移动到的目标集合
     * @return 移动成功返回 true, 移动失败返回 false
     */
    public Boolean sMove(String key, Object value, String destKey) {
        return redisTemplate.opsForSet().move(key, value, destKey);
    }

    /**
     * 获取集合的大小
     *
     * @param key 集合
     * @return 集合长度
     */
    public Long sSize(String key) {
        return redisTemplate.opsForSet().size(key);
    }

    /**
     * 判断集合是否包含 value
     *
     * @param key   集合
     * @param value 元素
     * @return 包含返回 true, 不包含返回 false
     */
    public Boolean sIsMember(String key, Object value) {
        return redisTemplate.opsForSet().isMember(key, value);
    }

    /**
     * 获取两个集合的交集
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @return 两个集合的交集
     */
    public Set<Object> sIntersect(String key, String otherKey) {
        return redisTemplate.opsForSet().intersect(key, otherKey);
    }

    /**
     * 获取 key 集合与多个集合的交集
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @return 多个集合的交集
     */
    public Set<Object> sIntersect(String key, Collection<String> otherKeys) {
        return redisTemplate.opsForSet().intersect(key, otherKeys);
    }

    /**
     * key 集合与 otherKey 集合的交集存储到 destKey 集合中
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @param destKey  用于保存结果的集合
     * @return 新集合的长度
     */
    public Long sIntersectAndStore(String key, String otherKey, String destKey) {
        return redisTemplate.opsForSet().intersectAndStore(key, otherKey,
                destKey);
    }

    /**
     * key 集合与多个集合的交集存储到 destKey 集合中
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @param destKey   用于保存结果的集合
     * @return 新集合的长度
     */
    public Long sIntersectAndStore(String key, Collection<String> otherKeys, String destKey) {
        return redisTemplate.opsForSet().intersectAndStore(key, otherKeys,
                destKey);
    }

    /**
     * 获取两个集合的并集
     *
     * @param key       集合1
     * @param otherKeys 集合2
     * @return 两个集合的并集
     */
    public Set<Object> sUnion(String key, String otherKeys) {
        return redisTemplate.opsForSet().union(key, otherKeys);
    }

    /**
     * 获取 key 集合与多个集合的并集
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @return 多个集合的并集
     */
    public Set<Object> sUnion(String key, Collection<String> otherKeys) {
        return redisTemplate.opsForSet().union(key, otherKeys);
    }

    /**
     * key 集合与 otherKey 集合的并集存储到 destKey 中
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @param destKey  用于保存结果的集合
     * @return 新集合的长度
     */
    public Long sUnionAndStore(String key, String otherKey, String destKey) {
        return redisTemplate.opsForSet().unionAndStore(key, otherKey, destKey);
    }

    /**
     * key 集合与多个集合的并集存储到 destKey 中
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @param destKey   用于保存结果的集合
     * @return 新集合的长度
     */
    public Long sUnionAndStore(String key, Collection<String> otherKeys, String destKey) {
        return redisTemplate.opsForSet().unionAndStore(key, otherKeys, destKey);
    }

    /**
     * 获取两个集合的差集
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @return 集合1 - 集合2 的差集
     */
    public Set<Object> sDifference(String key, String otherKey) {
        return redisTemplate.opsForSet().difference(key, otherKey);
    }

    /**
     * 获取 key 集合与多个集合的差集
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @return 集合1 - 集合2 - 集合3 - ... 集合n 的差集
     */
    public Set<Object> sDifference(String key, Collection<String> otherKeys) {
        return redisTemplate.opsForSet().difference(key, otherKeys);
    }

    /**
     * key 集合与 otherKey 集合的差集存储到 destKey 中
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @param destKey  用于保存结果的集合
     * @return 新集合的长度
     */
    public Long sDifference(String key, String otherKey, String destKey) {
        return redisTemplate.opsForSet().differenceAndStore(key, otherKey,
                destKey);
    }

    /**
     * key 集合与多个集合的差集存储到 destKey 中
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @param destKey   用于保存结果的集合
     * @return 新集合的长度
     */
    public Long sDifference(String key, Collection<String> otherKeys, String destKey) {
        return redisTemplate.opsForSet().differenceAndStore(key, otherKeys,
                destKey);
    }

    /**
     * 获取集合所有元素
     *
     * @param key 集合
     * @return 集合中所有元素
     */
    public Set<Object> setMembers(String key) {
        return redisTemplate.opsForSet().members(key);
    }

    /**
     * 随机获取集合中的一个元素
     *
     * @param key 集合
     * @return 集合中随机一个元素
     */
    public Object sRandomMember(String key) {
        return redisTemplate.opsForSet().randomMember(key);
    }

    /**
     * 随机获取集合中 count 个元素
     *
     * @param key   集合
     * @param count 要获取的元素个数
     * @return count 个随机元素组成的集合
     */
    public List<Object> sRandomMembers(String key, long count) {
        return redisTemplate.opsForSet().randomMembers(key, count);
    }

    /**
     * 随机获取集合中 count 个元素并且去除重复的
     *
     * @param key   集合
     * @param count 要获取的元素个数
     * @return count 个随机元素组成的集合, 并且不包含重复元素
     */
    public Set<Object> sDistinctRandomMembers(String key, long count) {
        return redisTemplate.opsForSet().distinctRandomMembers(key, count);
    }

    /**
     * 迭代集合中的元素
     *
     * @param key     集合
     * @param options 迭代的限制条件, 为 ScanOptions.NONE 则无限制
     * @return 下一个元素的游标
     */
    public Cursor<Object> sScan(String key, ScanOptions options) {
        return redisTemplate.opsForSet().scan(key, options);
    }

    // ------------------zSet 相关操作--------------------------------

    /**
     * 添加元素, 有序集合是按照元素的 score 值由小到大排列
     *
     * @param key   有序集合
     * @param value 元素
     * @param score 分数
     * @return 添加成功返回 true, 添加失败返回 false
     */
    public Boolean zAdd(String key, Object value, double score) {
        return redisTemplate.opsForZSet().add(key, value, score);
    }

    /**
     * 添加多个元素到有序集合中
     *
     * @param key    有序集合
     * @param values 多个元素值
     * @return 有序集合长度
     */
    public Long zAdd(String key, Set<TypedTuple<Object>> values) {
        return redisTemplate.opsForZSet().add(key, values);
    }

    /**
     * 移除有序集合中的值
     *
     * @param key    有序集合
     * @param values 要移除的值, 可以同时移除多个
     * @return 有序集合长度
     */
    public Long zRemove(String key, Object... values) {
        return redisTemplate.opsForZSet().remove(key, values);
    }

    /**
     * 增加元素的 score 值，并返回增加后的值
     *
     * @param key   有序集合
     * @param value 要增加的元素
     * @param delta 增加的分数是多少
     * @return 增加后的分数
     */
    public Double zIncrementScore(String key, Object value, double delta) {
        return redisTemplate.opsForZSet().incrementScore(key, value, delta);
    }

    /**
     * 返回元素在集合的排名,有序集合是按照元素的 score 值由小到大排列
     *
     * @param key   有序集合
     * @param value 值
     * @return 排名, 从小到大顺序, 0 表示第一位
     */
    public Long zRank(String key, Object value) {
        return redisTemplate.opsForZSet().rank(key, value);
    }

    /**
     * 返回元素在集合的排名,按元素的 score 值由大到小排列
     *
     * @param key   有序集合
     * @param value 值
     * @return 排名, 从大到小顺序
     */
    public Long zReverseRank(String key, Object value) {
        return redisTemplate.opsForZSet().reverseRank(key, value);
    }

    /**
     * 获取集合的元素, 从小到大排序
     *
     * @param key   有序集合
     * @param start 开始位置
     * @param end   结束位置, -1 表示从开始位置开始后面的所有元素
     * @return 指定区间的值的集合
     */
    public Set<Object> zRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().range(key, start, end);
    }

    /**
     * 获取集合元素, 并且把 score 值也获取
     *
     * @param key   有序集合
     * @param start 开始位置
     * @param end   结束位置, -1 表示从开始位置开始后面的所有元素
     * @return 指定区间的元素及分数的元组的集合
     */
    public Set<TypedTuple<Object>> zRangeWithScores(String key, long start, long end) {
        return redisTemplate.opsForZSet().rangeWithScores(key, start, end);
    }

    /**
     * 根据 score 值查询集合元素
     *
     * @param key 有序集合
     * @param min 最小值
     * @param max 最大值
     * @return 分数 在最小值与最大值之间的元素集合
     */
    public Set<Object> zRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().rangeByScore(key, min, max);
    }

    /**
     * 根据 score 值查询集合元素及其分数, 并按分数从小到大排序
     *
     * @param key 有序集合
     * @param min 最小值
     * @param max 最大值
     * @return 分数 在最小值与最大值之间的元素与分数的元组的集合
     */
    public Set<TypedTuple<Object>> zRangeByScoreWithScores(String key, double min, double max) {
        return redisTemplate.opsForZSet().rangeByScoreWithScores(key, min, max);
    }

    /**
     * 根据 score 值查询集合元素及其分数, 从小到大排序, 只获取 start 到 end 位置之间的结果
     *
     * @param key   有序集合
     * @param min   最低分数
     * @param max   最高分数
     * @param start 开始位置
     * @param end   结束位置
     * @return 分数在 min 与 max 之间, 位置在 start 与 end 之间的元素与分数的元组的集合
     */
    public Set<TypedTuple<Object>> zRangeByScoreWithScores(String key, double min, double max, long start, long end) {
        return redisTemplate.opsForZSet().rangeByScoreWithScores(key, min, max, start, end);
    }

    /**
     * 获取集合的元素, 从大到小排序
     *
     * @param key   有序集合
     * @param start 开始位置
     * @param end   结束位置
     * @return 按照 分数 倒序的元素集合
     */
    public Set<Object> zReverseRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().reverseRange(key, start, end);
    }

    /**
     * 获取集合的元素, 从大到小排序, 并返回 score 值
     *
     * @param key   有序集合
     * @param start 开始位置
     * @param end   结束位置
     * @return 指定区间的元素及其分数的元组的集合
     */
    public Set<TypedTuple<Object>> zReverseRangeWithScores(String key, long start, long end) {
        return redisTemplate.opsForZSet().reverseRangeWithScores(key, start, end);
    }

    /**
     * 根据 score 值查询集合元素, 从大到小排序
     *
     * @param key 有序集合
     * @param min 分数最小值
     * @param max 分数最大值
     * @return 分数在 min 与 max 之间的元素的集合, 按分数倒序
     */
    public Set<Object> zReverseRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().reverseRangeByScore(key, min, max);
    }

    /**
     * 根据 score 值查询集合元素, 从大到小排序
     *
     * @param key 有序集合
     * @param min 分数最小值
     * @param max 分数最大值
     * @return 分数在 min 与 max 之间的元素与分数的元组的集合, 按分数倒序
     */
    public Set<TypedTuple<Object>> zReverseRangeByScoreWithScores(String key, double min, double max) {
        return redisTemplate.opsForZSet().reverseRangeByScoreWithScores(key, min, max);
    }

    /**
     * 根据 score 值查询集合元素及其分数, 从小到大排序, 只获取 start 到 end 位置之间的结果, 按分数从小到大排序
     *
     * @param key   有序集合
     * @param min   分数最小值
     * @param max   分数最大值
     * @param start 开始位置
     * @param end   结束位置
     * @return 分数在 min 与 max 之间, 位置在 start 与 end 之间的元素与分数的元组的集合, 按分数倒序
     */
    public Set<Object> zReverseRangeByScore(String key, double min, double max, long start, long end) {
        return redisTemplate.opsForZSet().reverseRangeByScore(key, min, max, start, end);
    }

    /**
     * 根据 score 值获取集合元素数量
     *
     * @param key 有序集合
     * @param min 分数最小值
     * @param max 分数最大值
     * @return 分数在最小值与最大值之间的元素数量
     */
    public Long zCount(String key, double min, double max) {
        return redisTemplate.opsForZSet().count(key, min, max);
    }

    /**
     * 获取集合大小( 底层实现还是 zcard )
     *
     * @param key 有序集合
     * @return 集合中的元素数量
     */
    public Long zSize(String key) {
        return redisTemplate.opsForZSet().size(key);
    }

    /**
     * 获取集合大小
     *
     * @param key 有序集合
     * @return 集合中的元素数量
     */
    public Long zZCard(String key) {
        return redisTemplate.opsForZSet().zCard(key);
    }

    /**
     * 获取集合中 value 元素的 score 值
     *
     * @param key   有序集合
     * @param value 元素值
     * @return 该元素值的分数
     */
    public Double zScore(String key, Object value) {
        return redisTemplate.opsForZSet().score(key, value);
    }

    /**
     * 移除指定索引位置的成员
     *
     * @param key   有序集合
     * @param start 开始位置
     * @param end   结束位置
     * @return 移除的元素个数
     */
    public Long zRemoveRange(String key, long start, long end) {
        return redisTemplate.opsForZSet().removeRange(key, start, end);
    }

    /**
     * 根据指定的 score 值的范围来移除成员
     *
     * @param key 有序集合
     * @param min 分数最小值
     * @param max 分数最大值
     * @return 移除的元素个数
     */
    public Long zRemoveRangeByScore(String key, double min, double max) {
        return redisTemplate.opsForZSet().removeRangeByScore(key, min, max);
    }

    /**
     * 获取 key 和 otherKey 的并集并存储在 destKey 中
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @param destKey  用于保存结果的集合
     * @return 新集合的长度
     */
    public Long zUnionAndStore(String key, String otherKey, String destKey) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKey, destKey);
    }

    /**
     * 获取 key 和 otherKeys 的并集并存储在 destKey 中
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @param destKey   用于保存结果的集合
     * @return 新集合的长度
     */
    public Long zUnionAndStore(String key, Collection<String> otherKeys, String destKey) {
        return redisTemplate.opsForZSet().unionAndStore(key, otherKeys, destKey);
    }

    /**
     * 获取 key 和 otherKey 的交集并存储在 destKey 中
     *
     * @param key      集合1
     * @param otherKey 集合2
     * @param destKey  用于保存结果的集合
     * @return 新集合的长度
     */
    public Long zIntersectAndStore(String key, String otherKey, String destKey) {
        return redisTemplate.opsForZSet().intersectAndStore(key, otherKey, destKey);
    }

    /**
     * 获取 key 和 otherKeys 的交集并存储在 destKey 中
     *
     * @param key       集合1
     * @param otherKeys 其余多个集合
     * @param destKey   用于保存结果的集合
     * @return 新集合的长度
     */
    public Long zIntersectAndStore(String key, Collection<String> otherKeys, String destKey) {
        return redisTemplate.opsForZSet().intersectAndStore(key, otherKeys, destKey);
    }

    /**
     * 迭代有序集合
     *
     * @param key 有序集合
     * @param options 迭代限制条件, 为 ScanOptions.NONE 则无限制
     * @return 下一个元素及分数元组的游标
     */
    public Cursor<TypedTuple<Object>> zScan(String key, ScanOptions options) {
        return redisTemplate.opsForZSet().scan(key, options);
    }
}
```

> 具体方法详细丰富，方便满足日常业务场景。

## 六、JSON 处理工具类

借助 Jackson，实现对象与 JSON 字符串的相互转换，支持驼峰与下划线字段互转。

核心代码示例：

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.StringUtils;

import java.io.File;
import java.io.IOException;
import java.text.SimpleDateFormat;
import java.util.List;

/**
 * @Author liboshuai
 * @Date 2023/12/18 18:03
 */
@Slf4j
public class JsonUtil {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    private static final ObjectMapper OBJECT_MAPPER_SNAKE_CASE = new ObjectMapper();
    // 日期格式化
    private static final String STANDARD_FORMAT = "yyyy-MM-dd HH:mm:ss";

    static {
        //对象的所有字段全部列入
        OBJECT_MAPPER.setSerializationInclusion(JsonInclude.Include.ALWAYS);
        //取消默认转换timestamps形式
        OBJECT_MAPPER.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        //忽略空Bean转json的错误
        OBJECT_MAPPER.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        //所有的日期格式都统一为以下的样式，即yyyy-MM-dd HH:mm:ss
        OBJECT_MAPPER.setDateFormat(new SimpleDateFormat(STANDARD_FORMAT));
        //忽略 在json字符串中存在，但是在java对象中不存在对应属性的情况。防止错误
        OBJECT_MAPPER.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    static {
        //对象的所有字段全部列入
        OBJECT_MAPPER_SNAKE_CASE.setSerializationInclusion(JsonInclude.Include.ALWAYS);
        //取消默认转换timestamps形式
        OBJECT_MAPPER_SNAKE_CASE.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        //忽略空Bean转json的错误
        OBJECT_MAPPER_SNAKE_CASE.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        //所有的日期格式都统一为以下的样式，即yyyy-MM-dd HH:mm:ss
        OBJECT_MAPPER_SNAKE_CASE.setDateFormat(new SimpleDateFormat(STANDARD_FORMAT));
        //忽略 在json字符串中存在，但是在java对象中不存在对应属性的情况。防止错误
        OBJECT_MAPPER_SNAKE_CASE.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        //转换为下划线
        OBJECT_MAPPER_SNAKE_CASE.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }

    private JsonUtil() {
    }

    /**
     * 对象转Json格式字符串
     *
     * @param obj 对象
     * @return Json格式字符串
     */
    public static <T> String obj2String(T obj) {
        if (obj == null) {
            return null;
        }
        try {
            return obj instanceof String ? (String) obj : OBJECT_MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("Parse Object to String error : {}", e.getMessage());
            return null;
        }
    }

    /**
     * 对象转file
     * @param fileName
     * @param obj
     */
    public static void obj2File(String fileName,Object obj){
        if (obj == null){
            return;
        }
        try {
            OBJECT_MAPPER.writeValue(new File(fileName),obj);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 对象转Json格式字符串; 属性名从驼峰改为下划线形式
     *
     * @param obj 对象
     * @return Json格式字符串
     */
    public static <T> String obj2StringFieldSnakeCase(T obj) {
        if (obj == null) {
            return null;
        }
        try {
            ObjectMapper objectMapper = OBJECT_MAPPER_SNAKE_CASE;
            return obj instanceof String ? (String) obj : objectMapper.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("Parse Object to String error : {}", e.getMessage());
            return null;
        }
    }

    /**
     * 字符串转换为自定义对象； 属性名从下划线形式改为驼峰
     *
     * @param str   要转换的字符串
     * @param clazz 自定义对象的class对象
     * @return 自定义对象
     */
    public static <T> T string2ObjFieldLowerCamelCase(String str, Class<T> clazz) {
        if (StringUtils.isEmpty(str) || clazz == null) {
            return null;
        }
        try {
            ObjectMapper objectMapper = OBJECT_MAPPER_SNAKE_CASE;
            return clazz.equals(String.class) ? (T) str : objectMapper.readValue(str, clazz);
        } catch (Exception e) {
            log.warn("Parse String to Object error : {}", e.getMessage());
            return null;
        }
    }

    /**
     * 字符串转换为自定义对象(List)； 属性名从下划线形式改为驼峰
     *
     * @param str           要转换的字符串
     * @param typeReference 自定义对象的typeReference List 对象
     * @return 自定义对象
     */
    public static <T> List<T> string2ListFieldLowerCamelCase(String str, TypeReference<List<T>> typeReference) {
        if (StringUtils.isEmpty(str) || typeReference == null) {
            return null;
        }
        try {
            ObjectMapper objectMapper = OBJECT_MAPPER_SNAKE_CASE;
            return objectMapper.readValue(str, typeReference);
        } catch (Exception e) {
            log.warn("Parse String to Object error : {}", e.getMessage());
            return null;
        }
    }

    /**
     * 对象转Json格式字符串(格式化的Json字符串)
     *
     * @param obj 对象
     * @return 美化的Json格式字符串
     */
    public static <T> String obj2StringPretty(T obj) {
        if (obj == null) {
            return null;
        }
        try {
            return obj instanceof String ? (String) obj : OBJECT_MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("Parse Object to String error : {}", e.getMessage());
            return null;
        }
    }

    /**
     * 字符串转换为自定义对象
     *
     * @param str   要转换的字符串
     * @param clazz 自定义对象的class对象
     * @return 自定义对象
     */
    public static <T> T string2Obj(String str, Class<T> clazz) {
        if (StringUtils.isEmpty(str) || clazz == null) {
            return null;
        }
        try {
            return clazz.equals(String.class) ? (T) str : OBJECT_MAPPER.readValue(str, clazz);
        } catch (Exception e) {
            log.warn("Parse String to Object error : {}", e.getMessage());
            return null;
        }
    }

    /**
     * 字符串转换为自定义字段转为list
     * @param str
     * @param typeReference
     * @param <T>
     * @return
     */
    public static <T> T string2Obj(String str, TypeReference<T> typeReference) {
        if (StringUtils.isEmpty(str) || typeReference == null) {
            return null;
        }
        try {
            return (T) (typeReference.getType().equals(String.class) ? str : OBJECT_MAPPER.readValue(str, typeReference));
        } catch (IOException e) {
            log.warn("Parse String to Object error", e);
            return null;
        }
    }

    public static <T> T string2Obj(String str, Class<?> collectionClazz, Class<?>... elementClazzes) {
        JavaType javaType = OBJECT_MAPPER.getTypeFactory().constructParametricType(collectionClazz, elementClazzes);
        try {
            return OBJECT_MAPPER.readValue(str, javaType);
        } catch (IOException e) {
            log.warn("Parse String to Object error : {}" + e.getMessage());
            return null;
        }
    }
}
```

推荐结合项目业务扩展，如支持写文件、List 类型转换、Pretty 格式化等。

## 七、设计数据库实体及数据访问层

### 用户实体类 UserEntity

```java
@Data
@TableName("user")
public class UserEntity {

    @Schema(description = "主键id", defaultValue = "1")
    private Long id;

    @Schema(description = "用户名称", defaultValue = "admin", nullable = true)
    private String userName;

    @Schema(description = "用户密码", defaultValue = "admin123", nullable = true)
    private String password;

}
```

采用 MyBatis-Plus 注解指定表名，使用 Swagger @Schema 注解增强 API 文档。

### Mapper 接口 UserMapper

```java
@Mapper
public interface UserMapper extends BaseMapper<UserEntity> {}
```

继承 MyBatis-Plus 提供的 BaseMapper，免去大量 CRUD 编码。

### 业务接口 UserService

```java
public interface UserService {
    List<UserEntity> getUserList();
}
```

定义用户相关业务方法，解耦业务逻辑。

### 业务实现 UserServiceImpl

```java
@Service
public class UserServiceImpl implements UserService {

    @Resource
    private UserMapper userMapper;

    public List<UserEntity> getUserList() {
        return userMapper.selectList(null);
    }
}
```

实现业务接口，调用底层 Mapper 实现数据库操作。

## 八、控制层接口设计

示范一个简单的用户列表查询接口，集成 Redis 缓存实现：

```java
@Slf4j
@RestController
@Tag(name = "用户管理")
public class UserController {

    @Resource
    private UserService userService;

    @Resource
    private RedisUtil redisUtil;

    @GetMapping("/userList")
    @Operation(summary = "获取用户列表")
    public List<UserEntity> userList() {
        log.info("调用 [获取用户列表] 接口");

        // 从 MySQL 中查询
        List<UserEntity> userList = userService.getUserList();

        // 缓存到 Redis，采用 JSON 字符串形式存储
        redisUtil.set("userList", JsonUtil.obj2String(userList));

        // 从 Redis 中读取内容，转换回 List<UserEntity> 返回
        Object redisVal = redisUtil.get("userList");
        if (redisVal != null) {
            return JsonUtil.string2Obj(redisVal.toString(), new TypeReference<List<UserEntity>>() {});
        }

        // 万一 Redis 失败，退回原始数据
        return userList;
    }
}
```

这样做保证了接口性能优化的启用，并保持数据一致性。

## 九、数据库初始化语句

运行以下脚本，创建并初始化数据库及表数据：

```sql
CREATE DATABASE IF NOT EXISTS spring_boot_example DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;
USE spring_boot_example;

CREATE TABLE `user` (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  `user_name` varchar(255) NOT NULL COMMENT '用户名称',
  `password` varchar(255) NOT NULL COMMENT '用户密码',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_bin;

INSERT INTO `user` (`id`, `user_name`, `password`) VALUES 
  (1, 'admin', 'admin123'),
  (2, 'root', 'root123');
```

## 十、接口调试及效果展示

启动服务后访问：http://localhost:8080/userList

预期返回：

```json
[
  {
    "id": 1,
    "userName": "admin",
    "password": "admin123"
  },
  {
    "id": 2,
    "userName": "root",
    "password": "root123"
  }
]
```

此接口调用流程：

- 通过 `UserService` 从数据库查询用户列表
- 使用 `RedisUtil` 将结果缓存 Redis，存储为 JSON 字符串
- 读取缓存并反序列化返回给前端

## 结语

本文通过一个工程项目示例，详细讲解了从依赖管理、配置文件分层、日志管理，到数据库访问、Redis 集成以及 API 层设计的全流程搭建方案。结合 MyBatis-Plus 简化数据库操作、Redis 缓存优化访问性能、Knife4j 优化接口文档展示，构建一个干净、易维护、可扩展的小型 Spring Boot 项目骨架，适用于多数企业级微服务初学者和开发者作为模板参考。

扩展建议：

- 集成 Spring Security 进行安全控制
- 配置全局异常处理和响应格式统一
- 引入 swagger 接口自动化生成与测试
- 扩展分布式缓存设计和多数据源支持
- 引入单元测试和 CI/CD 自动构建流程

持续关注并实践，能有效提升团队的开发效率与系统的稳定性。祝您项目开发顺利！