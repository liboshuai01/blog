---
title: Flink 使用异步 I/O 高效连接 MySQL/Doris
tags:
  - Flink
  - Doris
  - Mysql
categories:
  - 大数据
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262015217.png'
toc: true
abbrlink: cf1c7ac0
date: 2024-01-05 20:14:19
---

在现代大数据应用中，实时数据处理和高效的数据流管理是关键。Apache Flink 作为一款流处理引擎，凭借其强大的实时计算能力和低延迟性，成为构建高效数据处理系统的首选工具。在本篇博文中，我们将深入探讨如何使用 Flink 的异步 I/O 功能，结合 Druid 连接池，来连接 MySQL 或 Doris 数据库，实现高效、可扩展的数据流处理。

<!-- more -->

## 技术背景

- **Apache Flink**：一个开源的流处理框架，支持有状态计算、事件时间处理和容错机制。
- **Druid 连接池**：一个高效的数据库连接池，具有优秀的性能和稳定性，适用于高并发场景。
- **MySQL/Doris**：关系型数据库（MySQL）和分布式 SQL 数据库（Doris），用于存储和管理大规模数据。

## 架构概述

本项目的核心是一个 Flink 流处理应用，主要包括以下几个部分：

1. **数据源**：通过 Flink 的 DataStream 获取规则配置数据流（来自 MySQL/Doris）。
2. **异步 I/O**：使用 `AsyncDataStream` 结合自定义的 `DorisAsyncFunction` 进行异步数据查询，提高吞吐量。
3. **数据处理**：将查询结果封装成 Kafka 事件 DTO，供后续处理使用。
4. **连接池与线程池**：使用 Druid 连接池管理数据库连接，使用线程池管理异步任务的执行。

## 代码详解

### 依赖

```xml
<!-- mysql -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.0.33</version>
</dependency>
<!-- druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.23</version>
</dependency>
```

### 主程序

主程序负责初始化 Flink 环境和数据流，并将数据源与异步 I/O 进行连接。

```java
public static void main(String[] args) {
    // 其他代码省略
    
    // 获取规则配置数据流
    DataStream<RuleCdcDTO> ruleDS = FlinkMysqlConnector.read(env, parameterTool);
    
    // 获取旧状态清理流
    SingleOutputStreamOperator<KafkaEventDTO> clearKafkaEventDtoSO = AsyncDataStream.unorderedWait(
            ruleDS, new DorisAsyncFunction(parameterTool), 10, TimeUnit.SECONDS, 100
    );

    // 其他代码省略
}
```

- **FlinkMysqlConnector.read(env, parameterTool)**：从 MySQL/Doris 数据库读取规则配置数据流。
- **AsyncDataStream.unorderedWait()**：使用 Flink 的异步 I/O 功能，将规则配置流与自定义的 `DorisAsyncFunction` 进行连接，异步查询数据库，设置超时时间为 10 秒，允许最多 100 个并发请求。

### DorisAsyncFunction 类

`DorisAsyncFunction` 是自定义的异步函数，实现了 Flink 的 `RichAsyncFunction`，用于异步查询数据库并处理结果。

```java
@Slf4j
public class DorisAsyncFunction extends RichAsyncFunction<RuleCdcDTO, KafkaEventDTO> {
    // 成员变量声明
    private final ParameterTool parameterTool;
    private DruidDataSource druidDataSource;
    private ExecutorService executorService;

    // 构造函数
    public DorisAsyncFunction(ParameterTool parameterTool) {
        this.parameterTool = parameterTool;
    }

    @Override
    public void open(Configuration parameters) {
        // 初始化数据库连接池
        String host = parameterTool.get(ParameterConstants.DORIS_FE_HOST);
        String queryPort = parameterTool.get(ParameterConstants.DORIS_FE_PORT_QUERY);
        String feNodes = host + DefaultConstants.COLON + queryPort;
        String username = parameterTool.get(ParameterConstants.DORIS_USERNAME);
        String password = parameterTool.get(ParameterConstants.DORIS_PASSWORD);
        String database = parameterTool.get(ParameterConstants.DORIS_DATABASE);
        
        druidDataSource = new DruidDataSource();
        druidDataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        druidDataSource.setUsername(username);
        druidDataSource.setPassword(password);
        String url = String.format("jdbc:mysql://%s/%s?useUnicode=true&characterEncoding=utf8&serverTimezone=UTC", feNodes, database);
        druidDataSource.setUrl(url);
        
        // Druid 连接池配置
        druidDataSource.setInitialSize(5);
        druidDataSource.setMinIdle(5);
        druidDataSource.setMaxActive(20);
        druidDataSource.setMaxWait(2000); // 连接超时时间，单位毫秒
        druidDataSource.setTimeBetweenEvictionRunsMillis(60000);
        druidDataSource.setValidationQuery("SELECT 1");
        druidDataSource.setTestWhileIdle(true);
        druidDataSource.setTestOnBorrow(false);
        druidDataSource.setTestOnReturn(false);

        // 初始化线程池
        executorService = new ThreadPoolExecutor(
                5,
                15,
                1,
                TimeUnit.MINUTES,
                new LinkedBlockingDeque<>(100),
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }

    @Override
    public void asyncInvoke(RuleCdcDTO ruleCdcDTO, ResultFuture<KafkaEventDTO> resultFuture) {
        String op = ruleCdcDTO.getOp();
        // 如果操作不是 DELETE，直接返回空结果
        if (!Envelope.Operation.DELETE.code().equals(op)) {
            resultFuture.complete(Collections.emptyList());
            return;
        }
        String tableName = parameterTool.get(ParameterConstants.DORIS_TABLE_KEY);
        // 提交异步任务
        executorService.submit(() -> {
            RuleJsonDTO ruleCdcDTOBefore = ruleCdcDTO.getBefore();
            String ruleJsonBefore = ruleCdcDTOBefore.getRuleJson();
            RuleInfoDTO ruleInfoDTOBefore = JsonUtil.parseObject(ruleJsonBefore, RuleInfoDTO.class);
            if (Objects.isNull(ruleInfoDTOBefore)) {
                throw new BusinessException("Mysql Cdc 规则流 ruleCdcDTOBefore 必须非空");
            }
            List<KafkaEventDTO> kafkaEventDTOList = new ArrayList<>();
            try (DruidPooledConnection connection = druidDataSource.getConnection();
                 PreparedStatement preparedStatement = connection.prepareStatement(
                         String.format("SELECT TARGET_VALUE FROM %s WHERE RULE_CODE = ? and RULE_VERSION = ? and TARGET_FIELD = ?", tableName)
                 )) {
                // 设置 SQL 参数
                preparedStatement.setLong(1, ruleInfoDTOBefore.getRuleCode());
                preparedStatement.setLong(2, ruleInfoDTOBefore.getRuleVersion());
                preparedStatement.setString(3, ruleInfoDTOBefore.getTargetField());
                // 执行查询
                try (ResultSet resultSet = preparedStatement.executeQuery()) {
                    while (resultSet.next()) {
                        String targetValue = resultSet.getString("TARGET_VALUE");
                        RuleKeyHistoryDTO ruleKeyHistoryDTO = RuleKeyHistoryDTO.builder()
                                .ruleCode(ruleInfoDTOBefore.getRuleCode())
                                .ruleVersion(ruleInfoDTOBefore.getRuleVersion())
                                .targetField(ruleInfoDTOBefore.getTargetField())
                                .targetValue(targetValue)
                                .build();
                        KafkaEventDTO kafkaEventDTO = KafkaEventDTO.builder()
                                .targetField(ruleInfoDTOBefore.getTargetField())
                                .targetValue(targetValue)
                                .ruleKeyHistoryDTO(ruleKeyHistoryDTO)
                                .build();
                        kafkaEventDTOList.add(kafkaEventDTO);
                    }
                }
                // 提交处理结果
                log.warn("构建的数据清洗流：{}", kafkaEventDTOList);
                resultFuture.complete(kafkaEventDTOList);
            } catch (Exception e) {
                log.error("Error processing asyncInvoke for RuleInfoDTO: {}", ruleCdcDTO, e);
                // 根据需求选择提交异常
                resultFuture.completeExceptionally(e);
            }
        });
    }

    @Override
    public void close() {
        // 关闭连接池
        if (druidDataSource != null) {
            druidDataSource.close();
        }

        // 关闭线程池
        if (executorService != null) {
            executorService.shutdown();
            try {
                if (!executorService.awaitTermination(30, TimeUnit.SECONDS)) {
                    executorService.shutdownNow();
                }
            } catch (InterruptedException e) {
                executorService.shutdownNow();
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

### 初始化

在 `open` 方法中，初始化了 Druid 连接池和线程池：

- **DruidDataSource**：配置了数据库连接的基本参数，包括数据库 URL、用户名、密码，以及连接池的各种参数（如初始连接数、最大活动连接数、连接超时时间等）。
- **线程池（ExecutorService）**：使用 `ThreadPoolExecutor` 创建了一个可扩展的线程池，核心线程数为 5，最大线程数为 15，线程空闲时间为 1 分钟，任务队列为 `LinkedBlockingDeque`，并设置了拒绝策略为 `CallerRunsPolicy`。

### 异步调用

`asyncInvoke` 方法是 Flink 异步 I/O 的核心部分：

1. **操作类型判断**：首先判断当前操作类型是否为 `DELETE`，如果不是，则直接返回空结果。
2. **提交异步任务**：使用 `executorService.submit` 提交一个异步任务，该任务负责从数据库查询相关数据并处理结果。
3. **数据库查询**：
    - 从 `ruleCdcDTO` 中提取查询参数，包括 `ruleCode`、`ruleVersion` 和 `targetField`。
    - 使用 Druid 连接池获取数据库连接，准备并执行 SQL 查询。
    - 遍历结果集，将查询到的 `TARGET_VALUE` 封装成 `KafkaEventDTO` 对象，并加入结果列表。
4. **结果提交**：将处理好的结果列表通过 `resultFuture.complete` 提交给 Flink 进行后续处理。
5. **异常处理**：在查询和处理过程中如果出现异常，记录日志并通过 `resultFuture.completeExceptionally` 提交异常，确保 Flink 能够正确处理错误。

### 资源清理

`close` 方法负责释放资源：

- **关闭 Druid 连接池**：释放所有数据库连接资源。
- **关闭线程池**：优雅地关闭线程池，等待正在执行的任务完成，如果超时则强制关闭。

## 关键技术点分析

### 异步 I/O 在 Flink 中的应用

在 Flink 处理中，异步 I/O 可以显著提高吞吐量，尤其是在需要外部系统（如数据库）查询的场景下。通过 `AsyncDataStream.unorderedWait`，Flink 可以并行地处理多个异步请求，不必等待前一个请求完成，从而充分利用系统资源，降低延迟。

### Druid 连接池的优势

Druid 连接池以其高性能和稳定性著称，适用于高并发访问的场景。相比于传统的 JDBC 连接池，Druid 提供了更强的监控能力和更丰富的连接池参数配置，能够更好地适应复杂的生产环境。

### 线程池的配置与管理

合理配置线程池对于异步 I/O 的性能至关重要：

- **核心线程数与最大线程数**：根据实际的查询负载，设置合理的核心和最大线程数，确保在高并发时有足够的线程处理请求，同时避免过多线程导致资源浪费。
- **任务队列**：使用 `LinkedBlockingDeque` 作为任务队列，可以有效缓冲短时高峰的请求。
- **拒绝策略**：采用 `CallerRunsPolicy`，在队列满时由调用线程执行任务，避免任务丢失，同时能够减缓请求压力。

## 错误处理与容错

在异步 I/O 的过程中，可能会遇到各种异常，如数据库连接失败、SQL 执行错误等。良好的错误处理机制不仅能提升系统的稳定性，还能提供有价值的故障排查信息。

在 `asyncInvoke` 方法中，通过 `try-catch` 块捕捉可能的异常，并记录详细的错误日志。同时，通过 `resultFuture.completeExceptionally` 将异常信息提交给 Flink，确保 Flink 的容错机制能够正确触发，如重启任务或转移到备用节点。

## 性能优化建议

1. **连接池配置优化**：根据数据库的性能和请求的并发量，调整 Druid 连接池的参数，如 `maxActive`、`initialSize` 等，确保有足够的连接数应对高并发请求。
2. **线程池调优**：根据实际的 CPU 和内存资源，调整线程池的大小，避免线程过多导致的上下文切换开销或线程过少导致的请求积压。
3. **批量查询**：在可能的情况下，采用批量查询策略，减少数据库的访问次数，提高查询效率。
4. **缓存机制**：对于频繁访问的数据，可以引入缓存机制，减少对数据库的查询压力。
5. **监控与报警**：通过监控工具实时监控连接池和线程池的使用情况，设置合理的报警阈值，及时响应系统异常。

## 总结

通过本文的详细解析，我们了解了如何在 Apache Flink 项目中，结合异步 I/O、Druid 连接池和线程池，构建一个高效、可扩展的数据处理系统。合理的资源管理和错误处理机制是确保系统稳定运行的关键。此外，性能优化建议为实际应用提供了实用的指导。希望本篇博文能为您在大数据实时处理领域的项目开发提供有价值的参考与借鉴。

> 参考：[# Java版本Flink异步IO连接MySQL](https://blog.csdn.net/weixin_44785708/article/details/129601555)