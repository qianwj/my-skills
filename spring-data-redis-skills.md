# Spring Data Redis Skills — 源码级知识整理

> 基于 `spring-projects/spring-data-redis` 源码分析

---

## 一、核心架构分层

```
┌─────────────────────────────────────────────┐
│         RedisTemplate / ReactiveRedisTemplate │  ← 高层 API
├─────────────────────────────────────────────┤
│   ValueOps / ListOps / HashOps / ZSetOps …  │  ← 类型化操作
��─────────────────────────────────────────────┤
│         RedisConnectionFactory              │  ← 连接工厂抽象
│   LettuceConnectionFactory / JedisConnectionFactory │
├─────────────────────────────────────────────┤
│         RedisConnection / RedisCommands     │  ← 命令接口层
├─────────────────────────────────────────────┤
│         Lettuce / Jedis Driver              │  ← 底层驱动
└─────────────────────────────────────────────┘
```

---

## 二、RedisTemplate 执行模型

**核心类:** `core/RedisTemplate.java`

### execute() 完整流程

```
execute(RedisCallback<T> action, boolean exposeConnection, boolean pipeline)
  ├─ RedisConnectionUtils.getConnection(factory)
  │    └─ 检查 TransactionSynchronizationManager（@Transactional 绑定）
  ├─ 若 pipeline=true → connection.openPipeline()
  ├─ 创建连接代理（exposeConnection=false 时防止直接访问）
  ├─ action.doInRedis(connToExpose)   ← 执行回调
  ├─ 若 pipeline → connection.closePipeline() → 反序列化结果列表
  └─ finally: RedisConnectionUtils.releaseConnection()
```

### 序列化配置

| 序列化器字段 | 默认值 | 作用 |
|---|---|---|
| `keySerializer` | `JdkSerializationRedisSerializer` | Key 序列化 |
| `valueSerializer` | `JdkSerializationRedisSerializer` | Value 序列化 |
| `hashKeySerializer` | `JdkSerializationRedisSerializer` | Hash field 序列化 |
| `hashValueSerializer` | `JdkSerializationRedisSerializer` | Hash value 序列化 |
| `stringSerializer` | `StringRedisSerializer` | 内部字符串操作 |

`StringRedisTemplate` 将所有序列化器覆盖为 `StringRedisSerializer`。

### 事务支持

```java
// 开启 Redis 事务支持（绑定连接到当前线程）
template.setEnableTransactionSupport(true);

// 配合 @Transactional 使用：
// MULTI → 执行命令 → EXEC（提交）/ DISCARD（回滚）
```

注意：事务期间所有命令返回 `null`（排队中），结果在 EXEC 后批量返回。

### Pipeline 批量操作

```java
List<Object> results = template.executePipelined((RedisCallback<Object>) conn -> {
    conn.set(key1, val1);
    conn.set(key2, val2);
    return null; // 必须返回 null
});
```

---

## 三、连接层架构

**核心类:** `connection/lettuce/LettuceConnectionFactory.java`

### 生命周期状态机

```
CREATED → STARTING → STARTED → STOPPING → STOPPED → DESTROYED
```

实现 `SmartLifecycle`，随 Spring 容器自动启停。

### 连接模式

| 模式 | 配置类 | 说明 |
|---|---|---|
| Standalone | `RedisStandaloneConfiguration` | 单节点 |
| Sentinel | `RedisSentinelConfiguration` | 哨兵高可用 |
| Cluster | `RedisClusterConfiguration` | 集群分片 |

### 连接复用策略

- 默认：多个 `LettuceConnection` 共享同一底层 Lettuce 连接（线程安全）
- 开启连接池：`LettucePoolingClientConfiguration` + Apache Commons Pool2
- 每个 `LettuceConnection` 包装一个 `StatefulRedisConnection`

### 命令接口层次

```
RedisCommands
  ├─ RedisKeyCommands        — DEL, EXISTS, EXPIRE, SCAN …
  ├─ RedisStringCommands     — GET, SET, MGET, INCR …
  ├─ RedisListCommands       — LPUSH, RPOP, LRANGE …
  ├─ RedisSetCommands        — SADD, SMEMBERS, SINTER …
  ├─ RedisZSetCommands       — ZADD, ZRANGE, ZRANGEBYSCORE …
  ├─ RedisHashCommands       — HSET, HGET, HMGET …
  ├─ RedisGeoCommands        — GEOADD, GEODIST, GEORADIUS …
  ├─ RedisHyperLogLogCommands — PFADD, PFCOUNT …
  ├─ RedisStreamCommands     — XADD, XREAD, XACK …
  ├─ RedisTxCommands         — MULTI, EXEC, DISCARD …
  └─ RedisPubSubCommands     — SUBSCRIBE, PUBLISH …
```

---

## 四、Reactive 支持

**核心类:** `core/ReactiveRedisTemplate.java`

### 与阻塞模板的关键差异

| 维度 | RedisTemplate | ReactiveRedisTemplate |
|---|---|---|
| 返回类型 | `T` / `List<T>` | `Mono<T>` / `Flux<T>` |
| 序列化配置 | 多个独立 Serializer | `RedisSerializationContext<K,V>` |
| 连接工厂 | `RedisConnectionFactory` | `ReactiveRedisConnectionFactory` |
| 驱动支持 | Lettuce + Jedis | 仅 Lettuce |

### ReactiveRedisTemplate 执行流程

```
execute(ReactiveRedisCallback<T> action)
  └─ factory.getReactiveConnection()
       └─ Mono.usingWhen(
            connectionMono,
            conn → action.doInRedis(conn),   // 执行操作
            conn → conn.closeLater()          // 释放连接
          )
```

### 序列化上下文构建

```java
RedisSerializationContext<String, MyObject> ctx =
    RedisSerializationContext.<String, MyObject>newSerializationContext()
        .key(StringRedisSerializer.UTF_8)
        .value(new Jackson2JsonRedisSerializer<>(MyObject.class))
        .hashKey(StringRedisSerializer.UTF_8)
        .hashValue(new Jackson2JsonRedisSerializer<>(MyObject.class))
        .build();
```

---

## 五、Repository 层

**核心类:** `repository/configuration/EnableRedisRepositories.java`  
`core/convert/MappingRedisConverter.java`

### 存储模型

每个实体在 Redis 中存储为：
```
# Hash（主数据）
<keyspace>:<id>          → HSET 存储所有字段

# 索引（@Indexed 字段）
<keyspace>:<field>:<value>  → SET 存储匹配的 id 集合

# TTL（@TimeToLive 字段）
<keyspace>:<id>          → EXPIRE 设置过期时间
```

### 关键注解

```java
@RedisHash("user")          // 指定 keyspace，可设 timeToLive
public class User {
    @Id String id;           // 主键，自动生成 UUID
    @Indexed String email;   // 创建二级索引，支持 findByEmail()
    @TimeToLive long ttl;    // 控制该实体的 TTL（秒）
}
```

### MappingRedisConverter 转换流程

```
Java Object → RedisData (Map<String, Object>)
  ├─ 基本类型 → 直接存为字符串
  ├─ 嵌套对象 → 展平为 "field.nestedField" 格式
  ├─ 集合 → "field.[0]", "field.[1]" 格式
  └─ 引用对象 → 存储引用 id，懒加载

RedisData → Java Object（反向转换）
```

---

## 六、缓存实现

**核心类:** `cache/RedisCacheManager.java`, `cache/RedisCache.java`

### 配置方式

```java
@Bean
RedisCacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .serializeKeysWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new StringRedisSerializer()))
        .serializeValuesWith(RedisSerializationContext.SerializationPair
            .fromSerializer(new GenericJackson2JsonRedisSerializer()))
        .disableCachingNullValues();

    return RedisCacheManager.builder(factory)
        .cacheDefaults(config)
        .withCacheConfiguration("users", config.entryTtl(Duration.ofHours(1)))
        .build();
}
```

### Key 生成规则

默认格式：`<cacheName>::<key>`  
例：`users::42`

### 缓存写入流程（RedisCache.put）

```
put(key, value)
  └─ cacheWriter.put(name, serializeKey(key), serializeValue(value), ttl)
       └─ connection.set(key, value, Expiration.from(ttl), SetOption.upsert())
```

---

## 七、Pub/Sub 消息监听

**核心类:** `listener/RedisMessageListenerContainer.java`

### 架构特点

- 单连接复用：所有订阅共享一个 Redis 连接（不同于普通命令连接）
- 实现 `SmartLifecycle`，随容器自动启停
- 消息分发通过 `TaskExecutor`（默认 `SimpleAsyncTaskExecutor`）

### 注册监听器

```java
container.addMessageListener(listener, new ChannelTopic("orders"));
container.addMessageListener(listener, new PatternTopic("order.*"));
```

### 自动重连

内置重连机制，支持配置退避策略：
```java
container.setRecoveryInterval(5000); // 重连间隔 ms
```

---

## 八、Redis Streams

**核心类:** `stream/StreamMessageListenerContainer.java`

### 消费模式

| 模式 | 方法 | 说明 |
|---|---|---|
| 独立消费 | `receive(StreamOffset, StreamListener)` | 无消费组 |
| 消费组（手动 ACK） | `receive(Consumer, StreamOffset, StreamListener)` | 需手动 XACK |
| 消费组（自动 ACK） | `receiveAutoAck(Consumer, StreamOffset, StreamListener)` | 处理后自动 ACK |

### Offset 类型

```java
StreamOffset.create("stream-key", ReadOffset.latest())       // 只读新消息
StreamOffset.create("stream-key", ReadOffset.lastConsumed()) // 消费组上次位置
StreamOffset.create("stream-key", ReadOffset.from("0-0"))    // 从头读
```

### Record 类型

```java
// Map 类型
MapRecord<String, String, String> record = StreamRecords.mapBacked(map).withStreamKey("key");

// 对象类型（需 HashMapper）
ObjectRecord<String, MyEvent> record = StreamRecords.objectBacked(event).withStreamKey("key");
```

---

## 九、序列化器速查

| 序列化器 | 适用场景 | 注意 |
|---|---|---|
| `StringRedisSerializer` | String key/value | 最轻量 |
| `JdkSerializationRedisSerializer` | 任意 Serializable 对象 | 默认，不跨语言 |
| `Jackson2JsonRedisSerializer<T>` | 强类型 JSON | 需指定类型 |
| `GenericJackson2JsonRedisSerializer` | 多态 JSON | 存储类型信息 |
| `OxmSerializer` | XML | 需 Spring OXM |
| `ByteArrayRedisSerializer` | 原始字节 | 无转换 |

---

## 十、关键源码路径速查

| 功能 | 路径 |
|---|---|
| 核心模板 | `core/RedisTemplate.java` |
| 响应式模板 | `core/ReactiveRedisTemplate.java` |
| 连接工厂 | `connection/lettuce/LettuceConnectionFactory.java` |
| 命令接口总入口 | `connection/RedisCommands.java` |
| String 命令接口 | `connection/RedisStringCommands.java` |
| Repository 注解 | `repository/configuration/EnableRedisRepositories.java` |
| 实体映射转换器 | `core/convert/MappingRedisConverter.java` |
| 缓存管理器 | `cache/RedisCacheManager.java` |
| 消息监听容器 | `listener/RedisMessageListenerContainer.java` |
| Stream 监听容器 | `stream/StreamMessageListenerContainer.java` |
| 序列化器目录 | `serializer/` |
| GraalVM AOT 支持 | `aot/RedisRuntimeHints.java` |
