---
title: SpringBoot+Shiro同服务器多项目Cookie冲突解决方案
abbrlink: f4c35ddb
tags:
  - Java
  - SpringBoot
  - Shiro
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250422142718117.png'
toc: true
date: 2025-04-22 14:28:59
---

在企业级应用中，基于 Spring Boot 和 Apache Shiro 构建的安全架构非常常见。当同一台服务器上部署多个独立的 Spring Boot + Shiro 项目时，往往会遇到一个棘手的问题：用户在同一浏览器中登录多个项目时，后登录的会话会覆盖先前的会话，导致只能保持一个有效登录状态。究其原因，主要是多个项目共享了相同的 Session Cookie 名称，使浏览器无法区分不同应用的会话标识，进而引发登录冲突。本文将从问题梳理入手，深入分析 Session Cookie 共享带来的困扰，并结合 Shiro 的 Session 管理机制，详细讲解如何通过自定义 SessionId Cookie 名称，有效实现同服务器多项目环境下的会话隔离，保证用户多项目同时登录的无缝体验。

<!-- more -->

## 背景问题

在基于 Spring Boot 和 Shiro 构建的多个项目中，部署在同一台服务器但使用不同端口时，遇到了一个比较棘手的问题：

- **多项目部署，端口不同；**
- **使用同一浏览器登录多个项目时，后登录的会挤掉先登录的，导致只能唯一登录一个项目；**

通过排查发现，导致登录会话冲突的主要原因是：**多个项目的 Shiro SessionId Cookie 名称相同，浏览器发送 Cookie 时发生覆盖，从而导致会话串联问题。**

---

## 问题原因分析

Shiro 作为安全框架，会维护用户的会话信息，关键是通过 **SessionId Cookie** 来标识当前会话。默认情况下，Shiro 使用默认的 Cookie 名称，比如 `JSESSIONID` 或者其默认的 Session Cookie 名称。

多个不同端口的项目在同一域名（如 `localhost`）下共享 Cookie 名称，导致浏览器发送 Cookie 时，后启动的服务覆盖了前一个服务的 SessionId，使得前一个服务的会话失效。

简而言之：

- Cookie 名称相同 → 浏览器只存一份 Cookie → 后登录的覆盖先登录的 → 会话串号 → 登录被挤出。

---

## 解决思路

### 核心：给不同的项目设置不同的 **SessionId Cookie 名称**

通过修改 Shiro 相关配置，确保不同项目的 SessionId Cookie 名称不会发生冲突，从根本上解决会话覆盖问题。

---

## 方案一：修改 Spring Boot 自带 Session Cookie 名称

在 `application.properties` 中设置：

```properties
server.servlet.session.cookie.name=项目专属SessionCookieName
```

- 例如，项目 A 设置为 `PROJECT_A_SESSIONID`
- 项目 B 设置为 `PROJECT_B_SESSIONID`

**问题：**

- 该配置针对 Spring Boot 内置的 Servlet 容器管理的 Session（如 HttpSession）。
- 但如果项目中整合了 Shiro 自定义 Session 管理，Shiro 会重写默认 Cookie，导致该配置无效。
- 实践中发现，这种方式往往不能彻底解决 Shiro Session 的冲突问题。

---

## 方案二：Shiro Java 配置自定义 SessionId Cookie 名称

建议针对项目中 Shiro 的 `SessionManager` 手动定制 SessionId Cookie，具体步骤如下：

### 1. 定义 SessionId Cookie 名称

```java
private SimpleCookie sessionIdCookie() {
    // 创建 SimpleCookie，设置唯一名称，确保不同项目名称不同
    SimpleCookie cookie = new SimpleCookie("MY_SESSION_ID");
    // 可设置作用域、是否HttpOnly、路径等安全属性
    cookie.setHttpOnly(true);
    cookie.setPath("/");
    return cookie;
}
```

### 2. 自定义 `DefaultWebSessionManager`

```java
@Bean
public DefaultWebSessionManager sessionManager() {
    DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();

    // 设置会话超时时间，单位毫秒
    sessionManager.setGlobalSessionTimeout(30 * 60 * 1000); // 30分钟

    // 配置sessionDAO，如果有自定义，绑定进来
    sessionManager.setSessionDAO(sessionDAO());

    // 启用并注入自定义 Cookie 名称
    sessionManager.setSessionIdCookieEnabled(true);
    sessionManager.setSessionIdCookie(sessionIdCookie());

    // 可以添加 session 监听器
    Collection<SessionListener> listeners = new ArrayList<>();
    listeners.add(new BDSessionListener());
    sessionManager.setSessionListeners(listeners);

    return sessionManager;
}
```

### 3. 关键：

- 为每个项目定制不同的 SessionId Cookie 名称，比如：

    - 项目 A：`MY_SESSION_ID_A`
    - 项目 B：`MY_SESSION_ID_B`

- 浏览器持有同域下多个不同名称的 SessionId Cookie，不会相互覆盖。

---

## 代码示例总结

```java
@Configuration
public class ShiroConfig {

    @Bean
    public DefaultWebSessionManager sessionManager() {
        DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
        sessionManager.setGlobalSessionTimeout(30 * 60 * 1000);

        sessionManager.setSessionDAO(sessionDAO());
        sessionManager.setSessionIdCookieEnabled(true);
        sessionManager.setSessionIdCookie(sessionIdCookie());

        Collection<SessionListener> listeners = new ArrayList<>();
        listeners.add(new BDSessionListener());
        sessionManager.setSessionListeners(listeners);

        return sessionManager;
    }

    private SimpleCookie sessionIdCookie() {
        SimpleCookie cookie = new SimpleCookie("MY_SESSION_ID_A"); // 项目A专属Cookie名
        cookie.setHttpOnly(true);
        cookie.setPath("/");
        return cookie;
    }
    
    // ... 其他Bean配置 ...
}
```

> **注意：** 每个项目都要定义不同的 Cookie 名称，防止浏览器 Cookie 冲突。

---

## 扩展点和优化建议

### 1. 配置参数化

将 Cookie 名称参数放在 `application.properties` 中，通过配置文件读取：

```properties
shiro.session.cookie.name=MY_SESSION_ID_A
```

然后用 `@Value` 读取配置，提升项目灵活性。

### 2. Cookie安全设置

请根据实际业务需求配置：

- `cookie.setHttpOnly(true);` 防止JavaScript读取
- 设置 `cookie.setSecure(true);` 仅 HTTPS 访问
- 设置合理的 `cookie.setPath("/")`，域名和路径隔离

### 3. 区分不同子域名部署

如果不同项目部署在不同子域建议利用 Cookie 的域名特性进行隔离：

```java
cookie.setDomain("projectA.example.com");
```

从根本上降低 Cookie 冲突可能性。

---

## 总结

- Shiro Session 在不同项目部署情况下要避免 SessionId Cookie 名冲突。
- Spring Boot 自带的 Session Cookie 配置无法解决 Shiro Session 冲突问题。
- 最优解是通过 Shiro `DefaultWebSessionManager` 手动配置 SessionId Cookie 的名称，使之在不同项目中唯一。
- 结合安全实践合理配置 Cookie 属性，保障安全性和用户体验。