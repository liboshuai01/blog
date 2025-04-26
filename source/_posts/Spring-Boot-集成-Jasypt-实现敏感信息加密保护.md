---
title: Spring Boot 集成 Jasypt 实现敏感信息加密保护
tags:
  - Java
  - SpringBoot
  - Jasypt
categories:
  - Java
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262340389.png'
toc: true
abbrlink: c6d0d078
date: 2025-03-14 23:39:44
---

在数字化时代背景下，互联网渗透生活的方方面面，同时也带来了日益严峻的安全挑战。曾有知名企业工程师因将公司核心代码上传至开源平台导致公司损失惨重，受到法律制裁。这一事件暴露出企业对代码与配置信息保护的薄弱环节。

企业核心系统通常会在配置文件中存储数据库连接信息、第三方接口秘钥等敏感数据。如果以明文存在，一旦泄露，极易被恶意利用，造成严重经济和信誉损失。尤其是中小企业，在安全合规方面常存在疏漏，敏感信息暴露风险更大。

因此，采用有效的敏感信息加密手段不仅是保护知识产权和业务安全的必要举措，也符合行业安全规范和合规要求，能有效降低因信息泄露带来的安全风险。

<!-- more -->

## 哪些类型的敏感信息必须加密

- 数据库用户名和密码
- 缓存、消息队列等服务的认证信息
- 第三方服务的 Access Key、Secret Key
- 其他涉及系统间安全通信的凭证信息

以上信息若在配置文件中以明文形式存在，易成为攻击目标，因此必须采取加密手段保护。

## 敏感信息加密的核心价值

- **防止泄漏风险放大**：即使代码或配置文件泄露，密文因无法被直接识别，降低信息被直接利用的风险。
- **符合安全合规机制要求**：很多安全评审流程要求敏感信息不得以明文暴露，加密处理是审核通过前提。
- **提升开发者安全意识**：引导整个团队对安全问题保持高度警惕，减少人为失误造成的安全隐患。

## 选择 Jasypt 作为加密解决方案的理由

Jasypt（Java Simplified Encryption）是一个简洁、功能强大的Java加密库。其设计宗旨在于：

- 开发者无需深入密码学细节即可轻松实现数据加解密
- 兼具开箱即用的 Spring Boot 集成支持
- 支持多种加密算法和配置选项
- 简化配置文件中敏感配置的加密解密流程

Jasypt 官网项目地址：[http://www.jasypt.org/](http://www.jasypt.org/)

## Spring Boot 集成 Jasypt 的多种方式

### 通用集成（基于自动配置）

针对绝大多数 Spring Boot 项目，可以直接引入 `jasypt-spring-boot-starter` 依赖，无需额外配置，自动完成环境整合。支持对配置文件中带有特定前后缀的密文自动解密。

```xml
<dependency>
  <groupId>com.github.ulisesbocchio</groupId>
  <artifactId>jasypt-spring-boot-starter</artifactId>
  <version>3.0.4</version>
</dependency>
```

### 无自动配置环境下启用加密属性

如果项目未使用 `@SpringBootApplication` 或 `@EnableAutoConfiguration`，需要：

1. 引入 `jasypt-spring-boot` 依赖（非starter）
2. 在配置类添加注解 `@EnableEncryptableProperties`

```java
@Configuration
@EnableEncryptableProperties
public class JasyptConfig {
  // 其他配置
}
```

### 对指定配置文件启用加密支持

无需在整个 Spring 环境开启加密属性的情况下，可以指定应用哪些配置文件支持加解密：

```java
@Configuration
@EncryptablePropertySource("classpath:encrypted.properties")
public class JasyptConfig {
}
```

对于多配置文件支持：

```java
@Configuration
@EncryptablePropertySources({
    @EncryptablePropertySource("classpath:encrypted1.properties"),
    @EncryptablePropertySource("classpath:encrypted2.properties")
})
public class JasyptConfig {
}
```

Jasypt 1.8 版本以后也支持 YAML 配置文件。

## 实战演练：Spring Boot 项目中 Jasypt 集成示例

### 引入依赖

采用自动配置方式，pom中添加：

```xml
<dependency>
  <groupId>com.github.ulisesbocchio</groupId>
  <artifactId>jasypt-spring-boot-starter</artifactId>
  <version>3.0.4</version>
</dependency>
```

### 配置 Jasypt 加密参数

在 `application.yml` 中添加如下配置信息：

```yaml
jasypt:
  encryptor:
    algorithm: PBEWithMD5AndDES
    iv-generator-classname: org.jasypt.iv.NoIvGenerator
    property:
      prefix: IT(
      suffix: )
```

这里指定了加密算法为经典的 `PBEWithMD5AndDES`，同时定义了加密数据的前后缀，方便 Jasypt 自动识别哪些配置需要解密。

### 将敏感信息加密替换到配置文件中

例如，将数据库账号密码替换为密文格式：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/user2?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=Asia/Shanghai
    username: IT(MIJueAfnYWsKa2kiR8Qrrw==)
    password: IT(qH9m5vjj8RYULOASKdhlOw==)
```

注意：加密内容必须以配置的前缀 `IT(` 开始，后缀 `)` 结束。

### 生成密文的多种方案

#### 方案一：使用代码方式加密

示例代码：

```java
public class JasyptEncryptUtils {

    private static final String ALGORITHM = "PBEWithMD5AndDES";
    private static final String PASSWORD = "PEB123@321BEP";

    public static String encrypt(String plaintext) {
        StandardPBEStringEncryptor encryptor = new StandardPBEStringEncryptor();
        EnvironmentStringPBEConfig config = new EnvironmentStringPBEConfig();

        config.setAlgorithm(ALGORITHM);
        config.setPassword(PASSWORD);
        encryptor.setConfig(config);

        return encryptor.encrypt(plaintext);
    }

    public static String decrypt(String ciphertext) {
        StandardPBEStringEncryptor encryptor = new StandardPBEStringEncryptor();
        EnvironmentStringPBEConfig config = new EnvironmentStringPBEConfig();

        config.setAlgorithm(ALGORITHM);
        config.setPassword(PASSWORD);
        encryptor.setConfig(config);

        return encryptor.decrypt(ciphertext);
    }

    public static void main(String[] args) {
        System.out.println(encrypt("root"));
        System.out.println(encrypt("123456"));
    }
}
```

#### 方案二：利用命令行工具加密

在 `jasypt-1.9.3.jar` 包路径下执行：

加密命令：

```bash
java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI input='root' password=PEB123@321BEP algorithm=PBEWithMD5AndDES
```

解密命令：

```bash
java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringDecryptionCLI input='密文内容' password=PEB123@321BEP algorithm=PBEWithMD5AndDES
```

#### 方案三：通过 Maven 插件加密

pom 中添加插件：

```xml
<plugin>
  <groupId>com.github.ulisesbocchio</groupId>
  <artifactId>jasypt-maven-plugin</artifactId>
  <version>3.0.4</version>
</plugin>
```

执行加密：

```bash
mvn jasypt:encrypt-value -Djasypt.encryptor.password="PEB123@321BEP" -Djasypt.plugin.value="需要加密的内容"
```

执行解密：

```bash
mvn jasypt:decrypt-value -Djasypt.encryptor.password="PEB123@321BEP" -Djasypt.plugin.value="密文内容"
```

### 启动参数设置秘钥，确保安全

秘钥不应硬编码或保存在配置文件中，应通过以下方式传递：

- 作为 Java 启动参数：

```bash
java -jar app.jar --jasypt.encryptor.password=PEB123@321BEP
```

- 或作为 JVM 参数：

```bash
java -Djasypt.encryptor.password=PEB123@321BEP -jar app.jar
```

避免秘钥暴露才是真正保障系统安全的关键。

## 常见问题及解决策略

### 算法兼容性引发的启动错误

Jasypt 3.x 版本默认加密算法为 `PBEWITHHMACSHA512ANDAES_256`，此算法要求 JDK 9+ 或安装 JCE 无限制策略文件支持，低版本 JDK 或未安装会导致绑定参数失败。

**解决方案：**

- 降低算法版本，切换为 `PBEWithMD5AndDES`
- 指定无 IV 生成器：

```yaml
jasypt:
  encryptor:
    algorithm: PBEWithMD5AndDES
    iv-generator-classname: org.jasypt.iv.NoIvGenerator
```

- 或者降级 Jasypt 版本到 2.x

### 如何保证秘钥安全

- **切勿将秘钥以明文形式存储在代码或配置中**
- **通过外部传参方式加载秘钥**
- 如有更高安全需求，可自定义加密器逻辑或结合硬件安全模块（HSM）

## 防止敏感信息误上传 Git 的多重手段

- **规范代码提交流程和权限控制**，减少因误操作泄露风险
- 使用 `.gitignore` 文件忽略敏感配置文件，避免被跟踪提交
- 利用敏感信息检测工具（如 `git-secrets`）对提交进行扫描，阻止敏感数据提交
- 严格 Code Review 流程，团队互相监督发现潜在风险

这些手段结合加密技术，形成信息安全多道防线。

## 技术与法律的思考：代码归属权问题

近期发生的 Nginx 创始人被其前东家质疑代码归属事件，激发行业对程序员劳动成果归属的广泛讨论：

- 通常在职期间的开发属于公司资产
- 但开源精神与个人贡献之间需达成平衡与共识
- 合同、企业制度、法律法规均需明确界定权责

这一复杂议题值得团队和企业深入思考，既要尊重知识产权，也要保护开发者权益。

## 结语

安全不仅是一段代码的加密，更是技术人员责任心的体现。Jasypt 为 Spring Boot 项目提供了简单易用的敏感信息加密方案，配合完善的秘钥管理和流程管控，可大幅提升系统安全等级。作为开发者，应始终将安全纳入日常开发习惯，筑牢业务护城河，防止数据泄露风险。