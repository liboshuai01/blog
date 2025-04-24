---
title: SpringBoot与Jackson高效整合指南（附实用工具类与示例）
tags:
  - Java
  - SpringBoot
  - Json
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250424131512227.png'
toc: true
abbrlink: ccabcaf2
date: 2023-12-18 13:13:33
---

在现代Spring Boot项目中，Jackson作为默认的JSON处理库，承担着Java对象和JSON字符串相互转换的重要职责。本文将结合Spring Boot 2.7.6版本，基于JDK 1.8环境，详细讲解如何优化Jackson的配置，提供一套高效易用的Json工具类，并通过丰富示例辅助理解与应用，帮助您快速上手并提升开发效率。

<!-- more -->

---

## Spring Boot环境及依赖准备

本示范基于以下环境：

- JDK版本：1.8
- Spring Boot版本：2.7.6
- Jackson版本：2.13.4（Spring Boot 2.7默认依赖）

### 依赖引入

Spring Boot Web Starter内置了Jackson依赖，您只需在`pom.xml`中引入如下依赖即可：

```xml
<dependencies>
    <!-- Spring Boot Web Starter，包含Jackson依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- 单元测试依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

注意，无需额外明确添加Jackson依赖，Spring Boot Web Starter自动为您集成。

---

## 优化Jackson核心配置

为了满足日期格式统一、空值处理、允许单引号等需求，可以在Spring Boot的`application.properties`中配置如下属性：

```properties
# 统一日期格式化，方便前后端统一时间表现形式
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss

# 设置时区为东八区（中国标准时间）
spring.jackson.time-zone=GMT+8

# 输出时排除null值字段，减小json体积
spring.jackson.default-property-inclusion=non_null

# 让json输出有格式缩进，便于调试
spring.jackson.serialization.indent_output=true

# 忽略空Bean导致的序列化失败
spring.jackson.serialization.fail_on_empty_beans=false

# 忽略反序列化时未知属性，提升系统容错能力
spring.jackson.deserialization.fail_on_unknown_properties=false

# 允许解析非标准的JSON字符和单引号
spring.jackson.parser.allow_unquoted_control_chars=true
spring.jackson.parser.allow_single_quotes=true
```

以上配置覆盖了大部分业务常用场景，使Jackson的行为更符合实际使用需求。

---

## 实用Jackson工具类设计

为了避免重复编写JSON转换代码，建议自定义一个通用的`JsonUtil`工具类，封装常用的序列化与反序列化操作，并支持特定的命名策略转换。

### 设计思路

- 内置两个ObjectMapper实例：
    - 默认驼峰命名策略，用于绝大多数情况
    - 下划线命名策略，用于和数据库字段、第三方接口数据对接
- 提供对象与字符串之间的相互转换，多重重载方便调用
- 支持漂亮格式化输出，提升阅读体验
- 增加对文件写入的支持，方便调试和导出

### 工具类代码完整版

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

@Slf4j
public class JsonUtil {

    private static final ObjectMapper DEFAULT_MAPPER = new ObjectMapper();
    private static final ObjectMapper SNAKE_CASE_MAPPER = new ObjectMapper();

    private static final String DATE_FORMAT = "yyyy-MM-dd HH:mm:ss";

    static {
        // 通用配置
        configureMapper(DEFAULT_MAPPER);

        // 下划线转换配置
        configureMapper(SNAKE_CASE_MAPPER);
        SNAKE_CASE_MAPPER.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
    }

    private static void configureMapper(ObjectMapper mapper) {
        mapper.setSerializationInclusion(JsonInclude.Include.ALWAYS);
        mapper.configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
        mapper.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        mapper.setDateFormat(new SimpleDateFormat(DATE_FORMAT));
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }

    private JsonUtil() {
        // 私有化构造器，防止实例化
    }

    /**
     * Java对象转JSON字符串（默认驼峰）
     */
    public static <T> String obj2String(T obj) {
        if (obj == null) return null;
        try {
            return obj instanceof String ? (String) obj : DEFAULT_MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("转换对象为JSON字符串失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * Java对象转JSON文件（默认驼峰）
     */
    public static void obj2File(String fileName, Object obj) {
        if (obj == null || fileName == null || fileName.trim().isEmpty()) return;
        try {
            DEFAULT_MAPPER.writeValue(new File(fileName), obj);
        } catch (IOException e) {
            log.error("写入对象到文件失败: {}", e.getMessage());
        }
    }

    /**
     * Java对象转JSON字符串（字段名驼峰转下划线）
     */
    public static <T> String obj2StringFieldSnakeCase(T obj) {
        if (obj == null) return null;
        try {
            return obj instanceof String ? (String) obj : SNAKE_CASE_MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("转换对象为JSON字符串失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * JSON字符串转Java对象（字段名下划线转驼峰）
     */
    public static <T> T string2ObjFieldLowerCamelCase(String str, Class<T> clazz) {
        if (!StringUtils.hasText(str) || clazz == null) return null;
        try {
            return clazz.equals(String.class) ? (T) str : SNAKE_CASE_MAPPER.readValue(str, clazz);
        } catch (IOException e) {
            log.warn("JSON字符串转对象失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * JSON字符串转Java对象集合（字段名下划线转驼峰）
     */
    public static <T> List<T> string2ListFieldLowerCamelCase(String str, TypeReference<List<T>> typeReference) {
        if (!StringUtils.hasText(str) || typeReference == null) return null;
        try {
            return SNAKE_CASE_MAPPER.readValue(str, typeReference);
        } catch (IOException e) {
            log.warn("JSON字符串转集合失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * Java对象转漂亮格式化JSON字符串
     */
    public static <T> String obj2StringPretty(T obj) {
        if (obj == null) return null;
        try {
            return obj instanceof String ? (String) obj : DEFAULT_MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            log.warn("转换对象为漂亮格式JSON失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * JSON字符串转Java对象（默认驼峰）
     */
    public static <T> T string2Obj(String str, Class<T> clazz) {
        if (!StringUtils.hasText(str) || clazz == null) return null;
        try {
            return clazz.equals(String.class) ? (T) str : DEFAULT_MAPPER.readValue(str, clazz);
        } catch (IOException e) {
            log.warn("JSON字符串转对象失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * JSON字符串转Java对象（支持TypeReference，方便泛型）
     */
    public static <T> T string2Obj(String str, TypeReference<T> typeReference) {
        if (!StringUtils.hasText(str) || typeReference == null) return null;
        try {
            if (typeReference.getType().equals(String.class)) {
                return (T) str;
            }
            return DEFAULT_MAPPER.readValue(str, typeReference);
        } catch (IOException e) {
            log.warn("JSON字符串转对象失败: {}", e.getMessage());
            return null;
        }
    }

    /**
     * JSON字符串转带泛型的Java对象（集合+泛型元素）
     */
    public static <T> T string2Obj(String str, Class<?> collectionClazz, Class<?>... elementClazzes) {
        if (!StringUtils.hasText(str) || collectionClazz == null) return null;
        JavaType javaType = DEFAULT_MAPPER.getTypeFactory().constructParametricType(collectionClazz, elementClazzes);
        try {
            return DEFAULT_MAPPER.readValue(str, javaType);
        } catch (IOException e) {
            log.warn("JSON字符串转泛型对象失败: {}", e.getMessage());
            return null;
        }
    }
}
```

---

## 实战示例：User实体与转换演示

为了更清晰展现工具类的使用，我们定义一个简单的用户实体类：

```java
import lombok.Data;
import java.util.List;

@Data
public class User {
    private String username;
    private Integer age;
    private List<String> info;
    private Long userId;
}
```

---

### 示例一：Java对象转JSON字符串

```java
@Test
void testObjToString() {
    User user = new User();
    user.setUsername("clllb");
    user.setAge(24);
    user.setUserId(1L);
    user.setInfo(Arrays.asList("有一百万", "发大财"));

    String json = JsonUtil.obj2String(user);
    System.out.println(json);
}
```

**输出结果：**

```json
{"username":"clllb","age":24,"info":["有一百万","发大财"],"userId":1}
```

---

### 示例二：Java对象转JSON字符串（字段名驼峰转下划线）

```java
@Test
void testObjToSnakeCaseString() {
    User user = new User();
    user.setUsername("clllb");
    user.setAge(24);
    user.setUserId(11L);
    user.setInfo(Arrays.asList("有一百万", "发大财"));

    String json = JsonUtil.obj2StringFieldSnakeCase(user);
    System.out.println(json);
}
```

**输出结果：**

```json
{"username":"clllb","age":24,"info":["有一百万","发大财"],"user_id":11}
```

---

### 示例三：Java对象转JSON文件

```java
@Test
void testObjToFile() {
    User user = new User();
    user.setUsername("clllb");
    user.setAge(24);
    user.setUserId(1L);
    user.setInfo(Arrays.asList("有一百万", "发大财"));

    String fileName = "user.json";
    JsonUtil.obj2File(fileName, user);
    System.out.println("JSON文件已生成：" + fileName);
}
```

执行后会在当前项目根目录生成`user.json`文件，内容与`obj2String()`输出一致。

---

### 示例四：JSON字符串转Java对象（驼峰字段）

```java
@Test
void testStringToObj() {
    String json = "{\"username\":\"clllb\",\"age\":24,\"info\":[\"有一百万\",\"发大财\"],\"userId\":11}";
    User user = JsonUtil.string2Obj(json, User.class);
    System.out.println(user);
}
```

**输出结果：**

```text
User(username=clllb, age=24, info=[有一百万, 发大财], userId=11)
```

---

### 示例五：JSON字符串转Java对象集合（驼峰字段）

```java
@Test
void testStringToListObj() {
    String json = "[" +
            "{\"username\":\"clllb\",\"age\":24,\"info\":[\"有一百万\",\"发大财\"],\"userId\":11}," +
            "{\"username\":\"陈老老老板\",\"age\":25,\"info\":[\"有一千万\",\"发大大财\"],\"userId\":12}" +
            "]";
    List<User> users = JsonUtil.string2Obj(json, new TypeReference<List<User>>() {});
    users.forEach(System.out::println);
}
```

**输出结果：**

```
User(username=clllb, age=24, info=[有一百万, 发大财], userId=11)
User(username=陈老老老板, age=25, info=[有一千万, 发大大财], userId=12)
```

---

### 示例六：JSON字符串转Java对象集合（下划线字段转驼峰）

```java
@Test
void testStringToListObjSnakeCase() {
    String json = "[" +
            "{\"username\":\"clllb\",\"age\":24,\"info\":[\"有一百万\",\"发大财\"],\"user_id\":11}," +
            "{\"username\":\"陈老老老板\",\"age\":25,\"info\":[\"有一千万\",\"发大大财\"],\"user_id\":12}" +
            "]";
    List<User> users = JsonUtil.string2ListFieldLowerCamelCase(json, new TypeReference<List<User>>() {});
    users.forEach(System.out::println);
}
```

**输出结果：**

```
User(username=clllb, age=24, info=[有一百万, 发大财], userId=11)
User(username=陈老老老板, age=25, info=[有一千万, 发大大财], userId=12)
```

---

## 总结与提升建议

本文全面展示了如何在Spring Boot项目中集成并优化Jackson，涵盖依赖引入、核心配置、自定义工具类及丰富实用示例。通过这些实践，您可以：

- 统一JSON的格式和时间处理规则
- 灵活支持驼峰和下划线两种命名转化
- 简化JSON与Java对象的互转代码
- 支持文件读写，方便数据持久化和调试

后续可根据业务进一步扩展，如：

- 支持自定义序列化/反序列化器（实现复杂对象转换）
- 集成Jackson Visibilty控制实现安全过滤敏感字段
- 结合Spring MVC自定义消息转换器配置，以适应特殊请求场景

Jackson是日常后端开发的重要工具，掌握并灵活应用将大幅提升开发效率。希望本文能帮助您打造更健壮、易维护的JSON处理体系！