---
title: 如何在 Maven 项目中将本地依赖库打包到最终的 JAR 中
tags:
  - Maven
categories:
  - 杂货小铺
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262035185.png'
toc: true
abbrlink: eed0bc59
date: 2025-01-06 20:17:20
---

在现代后端开发中，构建高效且可扩展的 Web 应用程序通常依赖于多种第三方库和内部依赖。这些依赖可以来自公共仓库，也可能是公司内部自研的库或尚未发布到公共仓库的 JAR 包。本文将详细介绍如何在 Maven 项目中处理本地依赖库，并确保这些依赖能够正确地打包到最终的可执行 JAR 文件中。本文不仅以 Doris 连接器 (`flink-doris-connector`) 作为示例，还涵盖了处理其他本地依赖库的通用方法。

<!-- more -->


## 为什么需要打包本地依赖库？

通常，依赖库可以通过 Maven 中央仓库或其他公共仓库轻松获取和管理。然而，有时我们需要使用一些未发布到公共仓库的本地 JAR 包，例如：

- 公司内部开发的库
- 第三方提供但未上传到 Maven 仓库的库
- 特殊版本或定制版的库

直接引用本地依赖库可能会引发一些问题，尤其是在构建和部署过程中。为了确保项目的可移植性和一致性，必须将这些本地依赖正确地打包到最终的 JAR 文件中。

## 常见问题

### 使用 `system` 作用域

在 Maven 中，可以使用 `system` 作用域来引用本地 JAR 包。然而，这种方法有几个显著的缺点：

1. **不可移植性**：`system` 作用域依赖的路径是硬编码的，其他开发人员在不同的环境中可能无法找到该路径。
2. **打包问题**：使用 `system` 作用域的依赖默认不会包含在最终打包的 JAR 文件中，导致运行时缺少必要的依赖。

### 依赖管理的最佳实践

为了避免上述问题，推荐的做法是将本地依赖库安装到 Maven 本地仓库中，并使用常规的依赖管理机制进行引用。这样，可以确保依赖库的一致性和可移植性，同时也方便后续的依赖管理和版本控制。

## 解决方案：将本地依赖库打包到最终 JAR

以下是详细的步骤，展示如何在 Maven 项目中包含本地依赖库并将其打包到最终的 JAR 文件中。

### 步骤 1：将本地 JAR 安装到 Maven 本地仓库

首先，需要将本地的 JAR 包安装到 Maven 的本地仓库中。假设我们有一个本地的 `flink-doris-connector` JAR 文件位于项目的 `libs` 目录下。

打开终端，执行以下命令：

```bash
mvn install:install-file \
  -DgroupId=org.apache.doris \
  -DartifactId=flink-connector-doris_2.12 \
  -Dversion=1.14_2.12-1.1.1 \
  -Dpackaging=jar \
  -Dfile=libs/flink-doris-connector-1.14_2.12-1.1.1.jar
```

**参数说明：**

- `-DgroupId`：依赖的组织 ID，通常与包名相对应。
- `-DartifactId`：依赖的模块名。
- `-Dversion`：依赖的版本号。
- `-Dpackaging`：依赖的打包类型，通常为 `jar`。
- `-Dfile`：本地 JAR 文件的路径。

通过上述命令，将本地的 JAR 包安装到 Maven 本地仓库中，使其能够像其他依赖一样被 Maven 管理。

### 步骤 2：修改 `pom.xml` 文件中的依赖配置

安装完成后，需要在项目的 `pom.xml` 文件中引用该依赖。移除之前使用 `system` 作用域的配置，并改为默认的 `compile` 作用域。

#### 原始依赖配置（使用 `system` 作用域）

```xml
<!-- Doris Connector -->
<dependency>
    <groupId>org.apache.doris</groupId>
    <artifactId>flink-connector-doris_${scala.binary.version}</artifactId>
    <version>1.14_2.12-1.1.1</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/libs/flink-doris-connector-1.14_2.12-1.1.1.jar</systemPath>
</dependency>
```

#### 修改后的依赖配置

```xml
<!-- Doris Connector -->
<dependency>
    <groupId>org.apache.doris</groupId>
    <artifactId>flink-connector-doris_${scala.binary.version}</artifactId>
    <version>1.14_2.12-1.1.1</version>
</dependency>
```

**注意**：省略了 `<scope>` 和 `<systemPath>` 元素，默认作用域为 `compile`，这样 Maven 会自动处理该依赖。

### 步骤 3：配置 Maven Shade 插件以打包依赖

为了将所有的依赖（包括新添加的本地依赖）打包到最终的可执行 JAR 中，可以使用 Maven Shade 插件。确保在 `pom.xml` 中正确配置该插件。

> 打包插件可以使用其他插件，下面的只是一个示范而已。打包插件的配置内容不需要进行更改。

#### 示例 `pom.xml` 配置

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" 
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.liboshuai.starlink</groupId>
        <artifactId>slr-engine</artifactId>
        <version>${revision}</version>
    </parent>

    <artifactId>slr-engine-biz</artifactId>
    <packaging>jar</packaging>
    <name>${artifactId}</name>

    <properties>
        <scala.binary.version>2.12</scala.binary.version>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    </properties>

    <dependencies>
        <!-- Doris Connector -->
        <dependency>
            <groupId>org.apache.doris</groupId>
            <artifactId>flink-connector-doris_${scala.binary.version}</artifactId>
            <version>1.14_2.12-1.1.1</version>
        </dependency>
        <!-- 其他依赖 -->
    </dependencies>

    <build>
        <finalName>${project.artifactId}-${project.version}</finalName>
        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <includes>
                    <include>flink-${profilesActive}.properties</include>
                    <include>flink.properties</include>
                    <include>**/*.xml</include>
                </includes>
                <filtering>true</filtering>
            </resource>
        </resources>

        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.4.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <artifactSet>
                                <excludes>
                                    <exclude>org.apache.flink:flink-shaded-force-shading</exclude>
                                    <exclude>com.google.code.findbugs:jsr305</exclude>
                                    <exclude>org.slf4j:*</exclude>
                                    <exclude>log4j:*</exclude>
                                    <exclude>ch.qos.logback:*</exclude>
                                    <exclude>org.apache.hadoop:*</exclude>
                                </excludes>
                            </artifactSet>
                            <filters>
                                <filter>
                                    <!-- 防止签名文件导致的安全异常 -->
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers combine.children="append">
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer" />
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <!-- 其他插件 -->
        </plugins>
    </build>

</project>
```

**关键点说明：**

- **`artifactSet` 中的 `excludes`**：排除不需要打包的依赖，防止冲突或冗余。
- **`filters`**：过滤掉 `META-INF` 中的签名文件，避免运行时的安全异常。
- **`transformers`**：配置资源转换器，例如 `ServicesResourceTransformer`，用于处理 `META-INF/services` 文件。

### 步骤 4：重新构建项目

完成上述配置后，执行以下命令重新构建项目：

```bash
mvn clean package
```

此命令将：

1. 清理之前的构建产物。
2. 编译项目源代码。
3. 使用 Maven Shade 插件将所有依赖（包括本地依赖）打包到最终的 JAR 文件中。

构建成功后，您将在 `target` 目录下找到包含所有依赖的可执行 JAR 文件。

## 总结

在 Maven 项目中正确处理本地依赖库，对于确保项目的可移植性和一致性至关重要。通过将本地 JAR 包安装到 Maven 本地仓库，并使用常规的依赖管理方式引用它们，可以避免 `system` 作用域带来的诸多问题。同时，结合 Maven Shade 插件，可以轻松地将所有依赖打包到最终的 JAR 文件中，简化部署和分发流程。

**最佳实践提示：**

- **避免使用 `system` 作用域**：尽量使用 Maven 的依赖管理机制，而非 `system` 作用域，以提高项目的可维护性和可移植性。
- **版本管理**：确保本地依赖库的版本与项目中引用的一致，避免因版本不匹配导致的兼容性问题。
- **自动化构建**：在 CI/CD 流程中，确保本地依赖库的安装步骤被正确执行，避免构建失败或缺少依赖。

通过遵循上述步骤和最佳实践，您可以有效地管理和打包本地依赖库，构建出高质量且稳定的后端应用程序。