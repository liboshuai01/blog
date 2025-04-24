---
title: 基于SpringBoot的项目骨架快速搭建与实战教程
date: 2023-12-18 12:59:57
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

    public RedisUtil(RedisTemplate<String, Object> redisTemplate) { this.redisTemplate = redisTemplate; }

    public void set(String key, Object value) { redisTemplate.opsForValue().set(key, value); }

    public Object get(String key) { return redisTemplate.opsForValue().get(key); }

    public void delete(String key) { redisTemplate.delete(key); }

    // 其他类型方法省略......
}
```

> 具体方法详细丰富，方便满足日常业务场景。

## 六、JSON 处理工具类

借助 Jackson，实现对象与 JSON 字符串的相互转换，支持驼峰与下划线字段互转。

核心代码示例：

```java
@Slf4j
public class JsonUtil {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();
    private static final ObjectMapper OBJECT_MAPPER_SNAKE_CASE = new ObjectMapper();
    private static final String STANDARD_FORMAT = "yyyy-MM-dd HH:mm:ss";

    static {
        OBJECT_MAPPER.setSerializationInclusion(JsonInclude.Include.ALWAYS);
        OBJECT_MAPPER.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        OBJECT_MAPPER.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        OBJECT_MAPPER.setDateFormat(new SimpleDateFormat(STANDARD_FORMAT));
        OBJECT_MAPPER.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

        OBJECT_MAPPER_SNAKE_CASE.setSerializationInclusion(JsonInclude.Include.ALWAYS);
        OBJECT_MAPPER_SNAKE_CASE.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        OBJECT_MAPPER_SNAKE_CASE.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        OBJECT_MAPPER_SNAKE_CASE.setDateFormat(new SimpleDateFormat(STANDARD_FORMAT));
        OBJECT_MAPPER_SNAKE_CASE.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        OBJECT_MAPPER_SNAKE_CASE.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }

    public static <T> String obj2String(T obj) {
        if (obj == null) return null;
        try {
            return obj instanceof String ? (String) obj : OBJECT_MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("转JSON失败：{}", e.getMessage());
            return null;
        }
    }

    public static <T> T string2Obj(String str, Class<T> clazz) {
        if (StringUtils.isEmpty(str) || clazz == null) return null;
        try {
            return clazz.equals(String.class) ? (T) str : OBJECT_MAPPER.readValue(str, clazz);
        } catch (Exception e) {
            log.warn("转对象失败：{}", e.getMessage());
            return null;
        }
    }

    // 支持下划线转驼峰，格式化输出等方法略
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