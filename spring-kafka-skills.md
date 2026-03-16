# Spring Kafka Skills — 源码级知识整理

> 基于 `spring-projects/spring-kafka` 源码分析，涵盖核心 API、消息流程与设计原理

---

## 一、核心 API 体系

### 1. KafkaTemplate — 消息发送入口

**核心类:** `spring-kafka/src/main/java/org/springframework/kafka/core/KafkaTemplate.java`

```java
// 基本发送
CompletableFuture<SendResult<K, V>> send(String topic, V data);
CompletableFuture<SendResult<K, V>> send(String topic, K key, V data);
CompletableFuture<SendResult<K, V>> send(String topic, Integer partition, K key, V data);
CompletableFuture<SendResult<K, V>> send(ProducerRecord<K, V> record);
CompletableFuture<SendResult<K, V>> send(Message<?> message);  // Spring Message 抽象

// 事务内发送
<T> T executeInTransaction(OperationsCallback<K, V, T> callback);
```

**发送流程:**
```
send()
  └─ doSend(ProducerRecord)
       ├─ 获取 Producer（事务模式下从 ThreadLocal 取）
       ├─ 应用 ProducerInterceptor
       ├─ producer.send(record, callback)
       └─ callback 触发 ProducerListener.onSuccess/onError
```

**关键配置:**
```java
KafkaTemplate<String, Object> template = new KafkaTemplate<>(producerFactory);
template.setDefaultTopic("my-topic");
template.setProducerListener(new LoggingProducerListener<>());
template.setObservationEnabled(true);  // Micrometer 可观测性
```

---

### 2. ProducerFactory / ConsumerFactory

**核心类:**
- `spring-kafka/.../core/DefaultKafkaProducerFactory.java`
- `spring-kafka/.../core/DefaultKafkaConsumerFactory.java`

**ProducerFactory 关键设计:**
- 默认缓存单个 Producer 实例（线程安全，Kafka Producer 本身线程安全）
- 事务模式：每个事务 ID 对应独立 Producer，存储在 `ThreadLocal<CloseSafeProducer>`
- 支持动态修改配置后调用 `reset()` 重建 Producer

```java
Map<String, Object> configs = new HashMap<>();
configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

DefaultKafkaProducerFactory<String, Object> factory =
    new DefaultKafkaProducerFactory<>(configs);
factory.setTransactionIdPrefix("tx-");  // 开启事务
```

**ConsumerFactory 关键设计:**
- 每次调用 `createConsumer()` 创建新实例（Consumer 非线程安全）
- 支持 `clientIdSuffix` 区分同组多消费者
- 可注册 `Listener` 监听 Consumer 创建/关闭事件

---

## 二、消息监听容器

### 1. 容器层次结构

```
MessageListenerContainer (接口)
  └─ AbstractMessageListenerContainer
       ├─ KafkaMessageListenerContainer      — 单线程，管理单个 Consumer
       └─ ConcurrentMessageListenerContainer — 多线程，内部持有 N 个 KafkaMessageListenerContainer
```

### 2. KafkaMessageListenerContainer 内部 Poll 循环

**核心类:** `spring-kafka/.../listener/KafkaMessageListenerContainer.java`
内部类 `ListenerConsumer` 实现 `Runnable`，在独立线程中运行。

```
ListenerConsumer.run()
  └─ pollAndInvoke()
       ├─ consumer.poll(pollTimeout)          — 拉取消息
       ├─ 过滤（RecordFilterStrategy）
       ├─ invokeListener(records)
       │    ├─ 单条模式: invokeRecordListener()
       │    └─ 批量模式: invokeBatchListener()
       ├─ 处理 acks（根据 AckMode）
       └─ 处理 seek/pause/resume 请求
```

**AckMode 语义:**

| AckMode | 提交时机 |
|---------|---------|
| `RECORD` | 每条记录处理后立即提交 |
| `BATCH` | 每次 poll 批次处理完后提交 |
| `TIME` | 按时间间隔提交 |
| `COUNT` | 按记录数提交 |
| `MANUAL` | 调用 `Acknowledgment.acknowledge()` 后下次 poll 前提交 |
| `MANUAL_IMMEDIATE` | 调用 `Acknowledgment.acknowledge()` 后立即提交 |

### 3. ConcurrentMessageListenerContainer

```java
ConcurrentKafkaListenerContainerFactory<String, Object> factory =
    new ConcurrentKafkaListenerContainerFactory<>();
factory.setConsumerFactory(consumerFactory);
factory.setConcurrency(3);  // 创建 3 个 KafkaMessageListenerContainer
factory.getContainerProperties().setAckMode(AckMode.RECORD);
factory.getContainerProperties().setPollTimeout(3000);
```

**并发模型:** 每个内部容器独占一个线程和一个 Kafka Consumer，分配到不同分区。

---

## 三、注解驱动模型

### 1. @KafkaListener 核心属性

**核心类:** `spring-kafka/.../annotation/KafkaListener.java`

```java
@KafkaListener(
    topics = {"topic1", "topic2"},          // 固定 topic
    topicPattern = "events\\..*",           // 正则匹配 topic
    groupId = "my-group",                   // 覆盖工厂默认 groupId
    containerFactory = "myFactory",         // 指定工厂 bean
    concurrency = "3",                      // 覆盖工厂并发数（支持 SpEL）
    errorHandler = "myErrorHandler",        // 指定错误处理器
    batch = "true",                         // 批量模式
    autoStartup = "false"                   // 不自动启动
)
public void listen(ConsumerRecord<String, Object> record, Acknowledgment ack) { ... }
```

**方法参数支持:**
- `ConsumerRecord<K, V>` / `List<ConsumerRecord<K, V>>` — 原始记录
- `@Payload T payload` — 反序列化后的消息体
- `@Header(KafkaHeaders.RECEIVED_TOPIC) String topic` — 提取 Header
- `Acknowledgment ack` — 手动确认句柄
- `Consumer<K, V> consumer` — 原生 Consumer（用于手动 seek）

### 2. 处理流程

```
@KafkaListener 方法
  ↓ KafkaListenerAnnotationBeanPostProcessor (BPP)
  ↓ 注册 MethodKafkaListenerEndpoint 到 KafkaListenerEndpointRegistry
  ↓ KafkaListenerEndpointRegistry.start() 触发所有容器启动
  ↓ 容器内 MessagingMessageListenerAdapter 适配方法调用
  ↓ HandlerAdapter 解析参数（@Payload、@Header 等）
  ↓ 调用实际方法
```

### 3. @EnableKafka

在 `@Configuration` 类上添加，导入 `KafkaListenerConfigurationSelector`，注册 `KafkaListenerAnnotationBeanPostProcessor`。

---

## 四、错误处理

### 1. 错误处理器层次

```
CommonErrorHandler (接口，Spring Kafka 2.8+)
  └─ DefaultErrorHandler            — 带退避重试 + 可恢复
       └─ 内部委托 FailedRecordTracker 跟踪失败记录
```

### 2. DefaultErrorHandler 配置

**核心类:** `spring-kafka/.../listener/DefaultErrorHandler.java`

```java
// 固定退避，最多重试 3 次
DefaultErrorHandler errorHandler = new DefaultErrorHandler(
    new DeadLetterPublishingRecoverer(kafkaTemplate),  // 失败后发送到 DLT
    new FixedBackOff(1000L, 3L)                        // 间隔 1s，最多 3 次
);

// 指定不重试的异常
errorHandler.addNotRetryableExceptions(DeserializationException.class);

// 指定只重试的异常
errorHandler.addRetryableExceptions(TransientDataAccessException.class);
```

**重试流程:**
```
消息处理失败
  └─ DefaultErrorHandler.handleRemaining()
       ├─ 检查是否可重试（异常类型 + 重试次数）
       ├─ 可重试: 等待退避时间，seek 回该 offset，下次 poll 重新消费
       └─ 不可重试/超限: 调用 ConsumerRecordRecoverer（如 DLT 发送）
```

### 3. DeadLetterPublishingRecoverer

```java
DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
    kafkaTemplate,
    (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
);
// DLT 消息自动携带原始 topic、partition、offset、异常信息等 Header
```

---

## 五、非阻塞重试（Retry Topic 模式）

### 1. 核心原理

不阻塞主 topic 消费，将失败消息路由到独立的重试 topic，按退避时间延迟消费。

```
主 topic: my-topic
  └─ 失败 → my-topic-retry-0 (延迟 1s)
              └─ 失败 → my-topic-retry-1 (延迟 2s)
                          └─ 失败 → my-topic-retry-2 (延迟 4s)
                                      └─ 失败 → my-topic-DLT
```

### 2. @RetryableTopic 注解

**核心类:** `spring-kafka/.../retrytopic/RetryableTopic.java`

```java
@RetryableTopic(
    attempts = "4",                          // 总尝试次数（含首次）
    backoff = @Backoff(delay = 1000, multiplier = 2),
    dltStrategy = DltStrategy.FAIL_ON_ERROR, // DLT 处理策略
    autoCreateTopics = "true",               // 自动创建重试 topic
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_DELAY_VALUE
)
@KafkaListener(topics = "my-topic")
public void listen(String message) { ... }

@DltHandler
public void handleDlt(String message, @Header KafkaHeaders.RECEIVED_TOPIC) String topic) { ... }
```

### 3. 编程式配置

```java
@Bean
RetryTopicConfiguration retryTopicConfig(KafkaTemplate<String, Object> template) {
    return RetryTopicConfigurationBuilder
        .newInstance()
        .maxAttempts(4)
        .fixedBackOff(1000L)
        .includeTopic("my-topic")
        .dltSuffix("-dead-letter")
        .create(template);
}
```

---

## 六、请求-回复模式

### 1. ReplyingKafkaTemplate

**核心类:** `spring-kafka/.../requestreply/ReplyingKafkaTemplate.java`

```java
// 发送请求并等待回复
RequestReplyFuture<String, String, String> future =
    replyingTemplate.sendAndReceive(
        new ProducerRecord<>("request-topic", "key", "payload"),
        Duration.ofSeconds(10)
    );

SendResult<String, String> sendResult = future.getSendFuture().get();
ConsumerRecord<String, String> reply = future.get();
```

**工作原理:**
```
sendAndReceive()
  ├─ 生成 correlationId，写入 KafkaHeaders.CORRELATION_ID Header
  ├─ 发送到 request-topic
  ├─ 在 reply-topic 上监听（内部 KafkaMessageListenerContainer）
  └─ 收到回复时按 correlationId 匹配，完成 CompletableFuture
```

**服务端处理:**
```java
@KafkaListener(topics = "request-topic")
@SendTo("reply-topic")  // 返回值自动发送到回复 topic，携带原始 correlationId
public String handleRequest(String request) {
    return "response: " + request;
}
```

---

## 七、事务支持

### 1. KafkaTransactionManager

**核心类:** `spring-kafka/.../transaction/KafkaTransactionManager.java`

```java
@Bean
KafkaTransactionManager<String, Object> kafkaTransactionManager(
        ProducerFactory<String, Object> producerFactory) {
    return new KafkaTransactionManager<>(producerFactory);
}
```

**事务语义:**
- 生产者事务：`@Transactional` 方法内所有 `send()` 原子提交/回滚
- 消费-生产事务（Exactly-Once）：消费 offset 提交与消息发送在同一事务内

```java
@Transactional("kafkaTransactionManager")
public void processAndForward(ConsumerRecord<String, String> record) {
    String result = process(record.value());
    kafkaTemplate.send("output-topic", result);  // 与 offset 提交同一事务
}
```

### 2. Exactly-Once 语义配置

```java
// 消费者端
configs.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

// 生产者端
factory.setTransactionIdPrefix("eos-");
configs.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
```

---

## 八、Kafka Streams 集成

### 1. StreamsBuilderFactoryBean

**核心类:** `spring-kafka/.../streams/StreamsBuilderFactoryBean.java`

```java
@Bean
StreamsBuilderFactoryBean streamsBuilderFactoryBean(KafkaStreamsConfiguration config) {
    StreamsBuilderFactoryBean factory = new StreamsBuilderFactoryBean(config);
    factory.setAutoStartup(true);
    factory.setStateListener((newState, oldState) ->
        log.info("Streams state: {} -> {}", oldState, newState));
    return factory;
}

@Bean
KafkaStreams kafkaStreams(StreamsBuilder streamsBuilder) {
    // streamsBuilder 由 StreamsBuilderFactoryBean 提供
    KStream<String, String> stream = streamsBuilder.stream("input-topic");
    stream.filter((k, v) -> v != null)
          .mapValues(String::toUpperCase)
          .to("output-topic");
    return streamsBuilder.build();  // 实际由 factory bean 管理生命周期
}
```

**生命周期:** `StreamsBuilderFactoryBean` 实现 `SmartLifecycle`，在 Spring 容器启动时自动启动 `KafkaStreams`，关闭时优雅停止。

---

## 九、测试支持

### 1. @EmbeddedKafka

**核心类:** `spring-kafka-test/.../test/EmbeddedKafkaBroker.java`

```java
@SpringBootTest
@EmbeddedKafka(
    partitions = 3,
    topics = {"topic1", "topic2"},
    brokerProperties = {
        "log.dir=/tmp/kafka-test",
        "auto.create.topics.enable=false"
    }
)
class MyKafkaTest {

    @Autowired
    private EmbeddedKafkaBroker embeddedKafka;

    @Value("${spring.embedded.kafka.brokers}")
    private String brokerAddresses;
}
```

### 2. KafkaTestUtils

```java
// 创建测试用 Consumer 并订阅 topic
Consumer<String, String> consumer = KafkaTestUtils.consumer(embeddedKafka, "test-group");
embeddedKafka.consumeFromAnEmbeddedTopic(consumer, "my-topic");

// 发送消息
Map<String, Object> producerProps = KafkaTestUtils.producerProps(embeddedKafka);
Producer<String, String> producer = new KafkaProducer<>(producerProps);
producer.send(new ProducerRecord<>("my-topic", "key", "value"));

// 消费并断言
ConsumerRecord<String, String> record = KafkaTestUtils.getSingleRecord(consumer, "my-topic");
assertThat(record.value()).isEqualTo("value");
```

### 3. 集成测试最佳实践

```java
@SpringBootTest
@EmbeddedKafka
class IntegrationTest {

    // 覆盖 bootstrap servers 指向嵌入式 Kafka
    @TestConfiguration
    static class Config {
        @Bean
        ConsumerFactory<String, String> consumerFactory(EmbeddedKafkaBroker broker) {
            Map<String, Object> props = KafkaTestUtils.consumerProps("test", "true", broker);
            return new DefaultKafkaConsumerFactory<>(props);
        }
    }
}
```

---

## 十、关键设计模式

| 模式 | 应用场景 |
|------|---------|
| 工厂模式 | `ProducerFactory`、`ConsumerFactory`、`KafkaListenerContainerFactory` |
| 模板方法 | `AbstractMessageListenerContainer`、`AbstractKafkaListenerContainerFactory` |
| 策略模式 | `CommonErrorHandler`、`RecordFilterStrategy`、`ConsumerRecordRecoverer` |
| 适配器模式 | `MessagingMessageListenerAdapter` — 桥接 `@KafkaListener` 方法与 `MessageListener` 接口 |
| 观察者模式 | `ProducerListener`、容器生命周期事件（`ListenerContainerIdleEvent` 等） |
| 装饰器模式 | 错误处理链、拦截器链（`ProducerInterceptor`、`ConsumerInterceptor`） |
| 生命周期模式 | 所有容器和 `StreamsBuilderFactoryBean` 实现 `SmartLifecycle` |

---

## 十一、源码路径速查

| 功能 | 关键类路径 |
|------|-----------|
| 消息发送 | `core/KafkaTemplate.java` |
| 生产者工厂 | `core/DefaultKafkaProducerFactory.java` |
| 消费者工厂 | `core/DefaultKafkaConsumerFactory.java` |
| 单线程容器 | `listener/KafkaMessageListenerContainer.java` |
| 并发容器 | `listener/ConcurrentMessageListenerContainer.java` |
| 注解处理器 | `annotation/KafkaListenerAnnotationBeanPostProcessor.java` |
| 容器工厂 | `config/ConcurrentKafkaListenerContainerFactory.java` |
| 容器注册表 | `config/KafkaListenerEndpointRegistry.java` |
| 错误处理 | `listener/DefaultErrorHandler.java` |
| DLT 恢复 | `listener/DeadLetterPublishingRecoverer.java` |
| 重试 Topic | `retrytopic/RetryTopicConfigurationBuilder.java` |
| 请求回复 | `requestreply/ReplyingKafkaTemplate.java` |
| 事务管理 | `transaction/KafkaTransactionManager.java` |
| Streams 集成 | `streams/StreamsBuilderFactoryBean.java` |
| 嵌入式 Kafka | `test/EmbeddedKafkaBroker.java`（spring-kafka-test 模块）|

所有类均在 `org.springframework.kafka.*` 包下。
