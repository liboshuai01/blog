---
title: Springboot整合Kafka
date: 2023-11-18 15:35:00
tags: [Java,Linux,Kafka,SpringBoot]
categories: [Java]
cover: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250417184826499.png
toc: true
---

在构建生产级Spring Boot应用与Kafka的集成方案时，我发现大多数现有教程都存在明显的局限性。这些资料往往停留在基础Demo层面，缺乏对生产环境关键配置的深入探讨。

经过大量实践验证和官方文档研究，我总结出一套经过生产验证的完整解决方案。本方案不仅涵盖基础配置，更着重解决了以下生产环境痛点：消息可靠性投递保障、消费者重平衡优化、批量处理与手动提交策略、异常处理机制等。

我们特别关注了Kafka在分布式环境下的稳定性表现，通过合理的参数调优避免了常见的数据丢失和重复消费问题。同时，方案中提供的配置类均经过线上环境验证，能够支撑高并发场景下的消息处理需求。

<!-- more -->

## 基础配置

1. 引入依赖
    ```yml
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    ```

2. 编写`kafka`相关配置

    ```yml
    spring:
      kafka:
        producer:
          bootstrap-servers: localhost:9096,localhost:9097,localhost:9098
          # transaction-id-prefix: starlinkRisk-
          retries: 3
          acks: 1
          batch-size: 32768
          buffer-memory: 33554432
        consumer:
          bootstrap-servers: localhost:9096,localhost:9097,localhost:9098
          group-id: slr_connector
          auto-commit-interval: 2000
          auto-offset-reset: latest
          enable-auto-commit: false
          max-poll-records: 3
        properties:
          max:
            poll:
              interval:
                ms: 600000
          session:
            timeout:
              ms: 10000
        listener:
          concurrency: 9
          missing-topics-fatal: false
          poll-timeout: 600000
    ```

3. 编写`kafka`生产者配置类

    ```java
    import com.liboshuai.starlink.slr.connector.common.handler.KafkaSendResultHandler;
    import lombok.extern.slf4j.Slf4j;
    import org.apache.kafka.clients.producer.ProducerConfig;
    import org.apache.kafka.common.serialization.StringSerializer;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.kafka.core.DefaultKafkaProducerFactory;
    import org.springframework.kafka.core.KafkaTemplate;
    import org.springframework.kafka.core.ProducerFactory;
    import org.springframework.kafka.support.serializer.JsonSerializer;

    import javax.annotation.Resource;
    import java.util.HashMap;
    import java.util.Map;

    /**
     * Kafka生产者配置
     */
    @Slf4j
    @Configuration
    public class KafkaProviderConfig {

        @Value("${spring.kafka.producer.bootstrap-servers}")
        private String bootstrapServers;
    //    @Value("${spring.kafka.producer.transaction-id-prefix}")
    //    private String transactionIdPrefix;
        @Value("${spring.kafka.producer.acks}")
        private String acks;
        @Value("${spring.kafka.producer.retries}")
        private String retries;
        @Value("${spring.kafka.producer.batch-size}")
        private String batchSize;
        @Value("${spring.kafka.producer.buffer-memory}")
        private String bufferMemory;

        @Resource
        private KafkaSendResultHandler kafkaSendResultHandler;

        @Bean
        public Map<String, Object> producerConfigs() {
            Map<String, Object> props = new HashMap<>(16);
            props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            //acks=0 ： 生产者在成功写入消息之前不会等待任何来自服务器的响应。
            //acks=1 ： 只要集群的首领节点收到消息，生产者就会收到一个来自服务器成功响应。
            //acks=all ：只有当所有参与复制的节点全部收到消息时，生产者才会收到一个来自服务器的成功响应。
            //开启事务必须设为all
            props.put(ProducerConfig.ACKS_CONFIG, acks);
            //发生错误后，消息重发的次数，开启事务必须大于0
            props.put(ProducerConfig.RETRIES_CONFIG, retries);
            //当多个消息发送到相同分区时,生产者会将消息打包到一起,以减少请求交互. 而不是一条条发送
            //批次的大小可以通过batch.size 参数设置.默认是16KB
            //较小的批次大小有可能降低吞吐量（批次大小为0则完全禁用批处理）。
            //比如说，kafka里的消息5秒钟Batch才凑满了16KB，才能发送出去。那这些消息的延迟就是5秒钟
            //实测batchSize这个参数没有用
            props.put(ProducerConfig.BATCH_SIZE_CONFIG, batchSize);
            //有的时刻消息比较少,过了很久,比如5min也没有凑够16KB,这样延时就很大,所以需要一个参数. 再设置一个时间,到了这个时间,
            //即使数据没达到16KB,也将这个批次发送出去
            props.put(ProducerConfig.LINGER_MS_CONFIG, "5000");
            //生产者内存缓冲区的大小
            props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, bufferMemory);
            //反序列化，和生产者的序列化方式对应
            props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
            return props;
        }

        @Bean
        public ProducerFactory<String, Object> producerFactory() {
            DefaultKafkaProducerFactory<String, Object> factory = new DefaultKafkaProducerFactory<>(producerConfigs());
            //开启事务，会导致 LINGER_MS_CONFIG 配置失效
    //        factory.setTransactionIdPrefix(transactionIdPrefix);
            return factory;
        }

    //    @Bean
    //    public KafkaTransactionManager<String, Object> kafkaTransactionManager(ProducerFactory<String, Object> producerFactory) {
    //        return new KafkaTransactionManager<>(producerFactory);
    //    }

        @Bean
        public KafkaTemplate<String, Object> kafkaTemplate() {
            KafkaTemplate<String, Object> kafkaTemplate = new KafkaTemplate<>(producerFactory());
            kafkaTemplate.setProducerListener(kafkaSendResultHandler);
            return kafkaTemplate;
        }
    }
    ```

4. 编写`kafka`消费者配置类

    ```java
    import org.apache.kafka.clients.consumer.ConsumerConfig;
    import org.apache.kafka.common.serialization.StringDeserializer;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
    import org.springframework.kafka.config.KafkaListenerContainerFactory;
    import org.springframework.kafka.core.ConsumerFactory;
    import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
    import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
    import org.springframework.kafka.listener.ContainerProperties;
    import org.springframework.kafka.support.serializer.ErrorHandlingDeserializer;

    import java.util.HashMap;
    import java.util.Map;

    /**
     * Kafka消费者配置
     */
    @Configuration
    public class KafkaConsumerConfig {

        @Value("${spring.kafka.consumer.bootstrap-servers}")
        private String bootstrapServers;
        @Value("${spring.kafka.consumer.group-id}")
        private String groupId;
        @Value("${spring.kafka.consumer.auto-commit-interval}")
        private String autoCommitInterval;
        @Value("${spring.kafka.consumer.enable-auto-commit}")
        private boolean enableAutoCommit;
        @Value("${spring.kafka.properties.session.timeout.ms}")
        private String sessionTimeout;
        @Value("${spring.kafka.properties.max.poll.interval.ms}")
        private String maxPollIntervalTime;
        @Value("${spring.kafka.consumer.max-poll-records}")
        private String maxPollRecords;
        @Value("${spring.kafka.consumer.auto-offset-reset}")
        private String autoOffsetReset;
        @Value("${spring.kafka.listener.concurrency}")
        private Integer concurrency;
        @Value("${spring.kafka.listener.missing-topics-fatal}")
        private boolean missingTopicsFatal;
        @Value("${spring.kafka.listener.poll-timeout}")
        private long pollTimeout;

        @Bean
        public Map<String, Object> consumerConfigs() {
            Map<String, Object> propsMap = new HashMap<>(16);
            propsMap.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            propsMap.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
            //是否自动提交偏移量，默认值是true，为了避免出现重复数据和数据丢失，可以把它设置为false，然后手动提交偏移量
            propsMap.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, enableAutoCommit);
            //自动提交的时间间隔，自动提交开启时生效
            propsMap.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, autoCommitInterval);
            //该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下该作何处理：
            //earliest：当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费分区的记录
            //latest：当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据（在消费者启动之后生成的记录）
            //none：当各分区都存在已提交的offset时，从提交的offset开始消费；只要有一个分区不存在已提交的offset，则抛出异常
            propsMap.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, autoOffsetReset);
            //两次poll之间的最大间隔，默认值为5分钟。如果超过这个间隔会触发reBalance
            propsMap.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, maxPollIntervalTime);
            //这个参数定义了poll方法最多可以拉取多少条消息，默认值为500。如果在拉取消息的时候新消息不足500条，那有多少返回多少；如果超过500条，每次只返回500。
            //这个默认值在有些场景下太大，有些场景很难保证能够在5min内处理完500条消息，
            //如果消费者无法在5分钟内处理完500条消息的话就会触发reBalance,
            //然后这批消息会被分配到另一个消费者中，还是会处理不完，这样这批消息就永远也处理不完。
            //要避免出现上述问题，提前评估好处理一条消息最长需要多少时间，然后覆盖默认的max.poll.records参数
            //注：需要开启BatchListener批量监听才会生效，如果不开启BatchListener则不会出现reBalance情况
            propsMap.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);
            //当broker多久没有收到consumer的心跳请求后就触发reBalance，默认值是10s
            propsMap.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, sessionTimeout);
            // 设置反序列化器为 ErrorHandlingDeserializer，防止药丸信息
            propsMap.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
            propsMap.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
            // 配置 ErrorHandlingDeserializer 的委托反序列化器为 StringDeserializer
            propsMap.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, StringDeserializer.class);
            propsMap.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, StringDeserializer.class);
            return propsMap;
        }

        @Bean
        public ConsumerFactory<Object, Object> consumerFactory() {
            return new DefaultKafkaConsumerFactory<>(consumerConfigs());
        }

        @Bean
        public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<Object, Object>> kafkaListenerContainerFactory() {
            ConcurrentKafkaListenerContainerFactory<Object, Object> factory = new ConcurrentKafkaListenerContainerFactory<>();
            factory.setConsumerFactory(consumerFactory());
            //在侦听器容器中运行的线程数，一般设置为 机器数*分区数
            factory.setConcurrency(concurrency);
            //消费监听接口监听的主题不存在时，默认会报错，所以设置为false忽略错误
            factory.setMissingTopicsFatal(missingTopicsFatal);
            //自动提交关闭，需要设置手动消息确认
            factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
            factory.getContainerProperties().setPollTimeout(pollTimeout);
            //设置为批量监听，需要用List接收（相等于listener.type: batch）
            factory.setBatchListener(true);
            return factory;
        }
    }
    ```

5. 编写`kafka`发送结果回调处理类

    ```java
    import lombok.extern.slf4j.Slf4j;
    import org.apache.kafka.clients.producer.ProducerRecord;
    import org.apache.kafka.clients.producer.RecordMetadata;
    import org.springframework.kafka.support.ProducerListener;
    import org.springframework.stereotype.Component;

    @Slf4j
    @Component("kafkaSendResultHandler")
    public class KafkaSendResultHandler implements ProducerListener<String, Object> {
        @Override
        public void onSuccess(ProducerRecord<String, Object> producerRecord, RecordMetadata recordMetadata) {
            // 记录成功发送的消息信息
            if (recordMetadata != null) {
                log.info("Kafka消息发送成功 - 主题: {}, 分区: {}, 偏移量: {}, 键: {}, 值: {}",
                        producerRecord.topic(),
                        recordMetadata.partition(),
                        recordMetadata.offset(),
                        producerRecord.key(),
                        producerRecord.value());
            } else {
                log.warn("Kafka消息发送成功，但RecordMetadata为null - 键: {}, 值: {}",
                        producerRecord.key(),
                        producerRecord.value());
            }
        }

        @Override
        public void onError(ProducerRecord<String, Object> producerRecord, RecordMetadata recordMetadata, Exception exception) {
            // 记录发送失败的消息信息及异常
            if (recordMetadata != null) {
                log.error("Kafka消息发送失败 - 主题: {}, 分区: {}, 偏移量: {}, 键: {}, 值: {}, 异常: {}",
                        producerRecord.topic(),
                        recordMetadata.partition(),
                        recordMetadata.offset(),
                        producerRecord.key(),
                        producerRecord.value(),
                        exception.getMessage(), exception);
            } else {
                log.error("Kafka消息发送失败 - RecordMetadata为null, 键: {}, 值: {}, 异常: {}",
                        producerRecord.key(),
                        producerRecord.value(),
                        exception.getMessage(), exception);
            }
        }
    }
    ```

6. 编写`kafka`消费异常处理类

    ```java
    import lombok.extern.slf4j.Slf4j;
    import org.apache.kafka.clients.consumer.Consumer;
    import org.springframework.kafka.listener.KafkaListenerErrorHandler;
    import org.springframework.kafka.listener.ListenerExecutionFailedException;
    import org.springframework.messaging.Message;
    import org.springframework.stereotype.Component;

    @Slf4j
    @Component("kafkaConsumerExceptionHandler")
    public class KafkaConsumerExceptionHandler implements KafkaListenerErrorHandler {

        /**
         * 处理错误，不带 Consumer 对象
         */
        @Override
        public Object handleError(Message<?> message, ListenerExecutionFailedException e) {
            log.error("kafka消费消息时发生错误。消息内容: {}, 错误信息: {}", message, e.getMessage(), e);
            // 可以根据需要选择是否抛出异常
            // 例如：return null; 表示忽略错误
            // throw e;    // 抛出异常以触发重试机制
            return null;
        }

        /**
         * 处理错误，带 Consumer 对象
         */
        @Override
        public Object handleError(Message<?> message, ListenerExecutionFailedException exception, Consumer<?, ?> consumer) {
            log.error("kafka消费消息时发生错误。消息内容: {}, 错误信息: {}", message, exception.getMessage(), exception);
            // 可以根据需要选择处理方式，例如手动提交偏移量，或其他操作
            // 这里仅记录日志并返回 null
            return null;
        }
    }
    ```

## 生产者示例

```java
import com.liboshuai.starlink.slr.engine.api.dto.KafkaEventDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;
import org.springframework.util.CollectionUtils;

import javax.annotation.Resource;
import java.util.List;

@Slf4j
@Component
public class KafkaEventProvider {

    @Resource
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Value("${slr-connector.kafka.provider_topic}")
    private String providerTopic;

    /**
     * 批量上送事件信息到kafka
     */
    public void batchSend(List<KafkaEventDTO> kafkaEventDTOList) {
        if (CollectionUtils.isEmpty(kafkaEventDTOList)) {
            return;
        }
        kafkaEventDTOList.forEach(eventUploadDTO -> kafkaTemplate.send(providerTopic, eventUploadDTO));
    }
}
```

## 消费者实例

```java
package com.liboshuai.starlink.slr.connector.dao.kafka.consumer;

import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Component;

import java.util.List;

@Slf4j
@Component
public class KafkaEventListener {

    @KafkaListener(
            topics = "${slr-connector.kafka.consumer_topic}",
            groupId = "${spring.kafka.consumer.group-id}",
            containerFactory = "kafkaListenerContainerFactory",
            errorHandler = "kafkaConsumerExceptionHandler"
    )
    public void onAlert(List<ConsumerRecord<String, String>> consumerRecordList, Acknowledgment ack) {
        for (ConsumerRecord<String, String> record : consumerRecordList) {
            // 打印消费的详细信息
            log.info("Consumed message - Topic: {}, Partition: {}, Offset: {}, Key: {}, Value: {}",
                    record.topic(),
                    record.partition(),
                    record.offset(),
                    record.key(),
                    record.value());
        }
        // 手动提交偏移量
        ack.acknowledge();
    }
}
```

## 结语

本文重点解决了Spring Boot与Kafka集成在生产环境中的核心配置问题，为开发者提供了可直接落地的解决方案。虽然示例部分相对简洁，但已经包含了生产环境最关键的配置项和实现模式。

在实际应用中，开发者可以根据业务需求进一步扩展：例如实现消息轨迹追踪、搭建监控告警系统、设计消息补偿机制等。值得强调的是，Kafka的配置优化是一个持续的过程，需要根据实际业务量级和性能指标进行动态调整。

建议在生产部署前进行充分的压力测试，特别是关注max.poll.records和session.timeout.ms等关键参数的调优。后续我们将继续分享Kafka事务消息、Exactly-Once语义等高级特性的实现方案，帮助构建更加健壮的消息处理系统。