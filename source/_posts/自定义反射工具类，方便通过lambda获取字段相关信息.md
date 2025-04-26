---
title: 自定义反射工具类，方便通过lambda获取字段相关信息
tags:
  - Java
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504261030763.png'
toc: true
abbrlink: 6e0a2623
date: 2024-08-19 10:28:51
---

Java反射是开发中不可或缺的强大工具，它允许我们在运行时动态检查、访问以及操作类、方法和字段。然而，直接使用反射往往导致代码冗长且难以维护。如何简化反射操作，让代码更加优雅且易于使用？本文将介绍一个名为`ReflectUtils`的实用工具类，它巧妙结合Java反射机制和Lambda表达式，为字段的访问和修改提供了简洁、高效的解决方案。

<!-- more -->

# ReflectUtils工具类概述

`ReflectUtils`是一个基于Java反射与Lambda表达式的工具类，专注于简化对于对象字段的获取和设置。它不仅支持快速访问字段名称和值，还具备缓存机制避免重复反射操作，提升运行性能。此外，`ReflectUtils`兼顾了安全性及易用性，借助Lambda表达式清晰定位字段，代码更具可读性和类型安全。

### 设计理念

- 利用Lambda表达式传递字段的getter方法，避免硬编码字段名称
- 通过反射解析Lambda获得字段信息，实现动态字段操作
- 缓存字段映射关系，降低反射开销保证性能
- 提供字段存在性检测和取得类的所有字段的实用方法

# ReflectUtils核心实现详解

```java
@Slf4j
public class ReflectUtils {

    private static final Map<SFunction<?>, Field> FUNCTION_CACHE = new ConcurrentHashMap<>();

    public static <T> String getFieldName(SFunction<T> function) {
        Field field = getField(function);
        return field.getName();
    }

    public static <T> Object getFieldValue(Object obj, SFunction<T> function) {
        try {
            Field field = getField(function);
            return field.get(obj);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Unable to access field value.", e);
        }
    }

    public static <T> void setFieldValue(Object obj, SFunction<T> function, Object value) {
        try {
            Field field = getField(function);
            field.set(obj, value);
        } catch (IllegalAccessException e) {
            throw new RuntimeException("Unable to set field value.", e);
        }
    }

    public static <T> boolean fieldExists(SFunction<T> function) {
        try {
            getField(function);
            return true;
        } catch (RuntimeException e) {
            return false;
        }
    }

    public static List<Field> getAllFields(Class<?> clazz) {
        List<Field> fields = new ArrayList<>();
        while (clazz != null) {
            fields.addAll(Arrays.asList(clazz.getDeclaredFields()));
            clazz = clazz.getSuperclass();
        }
        return fields;
    }

    public static <T> Field getField(SFunction<T> function) {
        return FUNCTION_CACHE.computeIfAbsent(function, ReflectUtils::findField);
    }

    public static <T> Field findField(SFunction<T> function) {
        SerializedLambda serializedLambda = getSerializedLambda(function);
        String implClass = serializedLambda.getImplClass();
        String implMethodName = serializedLambda.getImplMethodName();
        String fieldName = convertToFieldName(implMethodName);
        Field field = getField(fieldName, serializedLambda);

        if (field == null) {
            throw new RuntimeException("No such class 「" + implClass + "」 field 「" + fieldName + "」.");
        }
        field.setAccessible(true);
        return field;
    }

    static Field getField(String fieldName, SerializedLambda serializedLambda) {
        try {
            String declaredClass = serializedLambda.getImplClass().replace("/", ".");
            Class<?> aClass = Class.forName(declaredClass, false, ClassUtils.getDefaultClassLoader());
            return ReflectionUtils.findField(aClass, fieldName);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("get class field exception.", e);
        }
    }

    static String convertToFieldName(String getterMethodName) {
        String prefix;
        if (getterMethodName.startsWith("get")) {
            prefix = "get";
        } else if (getterMethodName.startsWith("is")) {
            prefix = "is";
        } else {
            throw new IllegalArgumentException("invalid getter method: " + getterMethodName);
        }
        return Introspector.decapitalize(getterMethodName.substring(prefix.length()));
    }

    static <T> SerializedLambda getSerializedLambda(SFunction<T> function) {
        try {
            Method method = function.getClass().getDeclaredMethod("writeReplace");
            method.setAccessible(true);
            return (SerializedLambda) method.invoke(function);
        } catch (Exception e) {
            throw new RuntimeException("get SerializedLambda exception.", e);
        }
    }
}
```

## 通过Lambda表达式关联字段

`ReflectUtils`依赖于一个自定义的函数式接口`SFunction`（继承了`Function`接口），用于传递字段的getter方法引用，比如`User::getName`。反射内部会通过调用`writeReplace`方法反序列化Lambda，拿到`SerializedLambda`，从而解析类名和方法名。进而将getter方法名转换成字段名，实现精准字段操作。

## 字段缓存机制

每次解析Lambda获取字段信息都会缓存到`FUNCTION_CACHE`中，避免重复反射查找，有效提升访问性能，保障在高频调用场景中的效率。

## 字段名转换逻辑

根据Java Bean规范，getter方法通常以 `get` 或 `is` 开头。`convertToFieldName`方法截取前缀后，将字段首字母转换为小写，得到标准字段名。

## 获取字段对象

`findField`借助Spring的`ReflectionUtils.findField`方法在指定Class中查找字段。它不仅获取当前类，还支持访问父类字段，扩展性强。

# 基本功能介绍

### 获取字段名称

调用`getFieldName`，传入getter方法的Lambda表达式，即可轻松获得对应字段的名称。

```java
String fieldName = ReflectUtils.getFieldName(User::getName);
```

### 获取字段值

通过`getFieldValue`方法传入对象实例与Lambda，可动态读取该字段当前值，无需显式调用getter。

```java
String name = (String) ReflectUtils.getFieldValue(user, User::getName);
```

### 设置字段值

`setFieldValue`实现字段的动态赋值，支持修改private属性，简化了传统繁琐的反射写法。

```java
ReflectUtils.setFieldValue(user, User::getName, "Alice");
```

### 字段存在性校验

`fieldExists`检测字段是否真实存在，避免操作不存在字段导致异常。

```java
boolean exists = ReflectUtils.fieldExists(User::getName);
```

### 查询类所有字段

通过`getAllFields`方法可以遍历获取类及其父类所有字段，适合需要统一处理字段的场景。

```java
List<Field> fields = ReflectUtils.getAllFields(User.class);
```

# 使用示例解析

```java
public class User {
    private String name;
    private int age;

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
```

```java
User user = new User();

// 设置字段值
ReflectUtils.setFieldValue(user, User::getName, "Alice");
ReflectUtils.setFieldValue(user, User::getAge, 30);

// 获取字段值
String name = (String) ReflectUtils.getFieldValue(user, User::getName);
int age = (int) ReflectUtils.getFieldValue(user, User::getAge);

// 输出结果
System.out.println("Name: " + name); // Name: Alice
System.out.println("Age: " + age);   // Age: 30
```

以上示例展示了无需调用getter/setter或硬编码字段名，仅凭方法引用即可灵活访问与修改字段，提升代码安全性和可维护性。

# 小结与展望

`ReflectUtils`巧妙融合了Java反射与Lambda表达式，打造出简洁且高效的字段操作利器。它解决了反射使用复杂、字段名硬编码的痛点，使字段访问更安全并提升开发体验。基于该工具类，你可以轻松实现动态查询、自动绑定、序列化等场景下对字段的灵活处理。

如果你正在构建需要广泛字段访问与操作的Java应用，不妨试试将`ReflectUtils`引入到项目中，借助Lambda表达式的优势，写出更加优美且健壮的代码！

未来可以考虑在工具类基础上扩展支持方法调用、注解解析或复杂类型字段操作，进一步提升反射工具的通用性和适用范围。

# 附录：SFunction接口示例

```java
@FunctionalInterface
public interface SFunction<T> extends Function<T, ?>, Serializable {
}
```

实现`Serializable`接口是`ReflectUtils`获取`SerializedLambda`的关键所在，必须确保Lambda表达式可序列化。

---

通过深入理解`ReflectUtils`背后的反射与Lambda机制，相信你已具备将这类工具灵活运用在Java项目中，从而有效提升代码质量与开发效率的能力。