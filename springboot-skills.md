# Spring Boot & Spring Framework Skills — 源码级知识整理

> 基于 Spring Boot (`spring-projects/spring-boot`) 与 Spring Framework (`spring-projects/spring-framework`) 源码分析

---

## 一、Spring Framework 核心机制

### 1. Bean 生命周期

**核心类:**
- `spring-beans/.../support/AbstractBeanFactory.java` — `getBean()` / `doGetBean()`
- `spring-beans/.../support/AbstractAutowireCapableBeanFactory.java` — `createBean()` / `doCreateBean()`

**完整流程:**

```
getBean(name)
  └─ doGetBean()
       ├─ 1. 检查单例缓存（三级缓存）
       ├─ 2. 标记 bean 正在创建
       ├─ 3. 处理 depends-on 依赖
       └─ 4. getSingleton(name, ObjectFactory)
              └─ doCreateBean()
                   ├─ createBeanInstance()     → 实例化（构造器/工厂方法）
                   ├─ addSingletonFactory()    → 三级缓存（解决循环依赖）
                   ├─ populateBean()           → 属性注入（@Autowired）
                   └─ initializeBean()
                        ├─ invokeAwareMethods() → BeanNameAware/BeanFactoryAware
                        ├─ BeanPostProcessor.postProcessBeforeInitialization()
                        ├─ afterPropertiesSet() / 自定义 initMethod
                        └─ BeanPostProcessor.postProcessAfterInitialization()
```

**循环依赖三级缓存:**
- 一级：`singletonObjects` — 完整 bean
- 二级：`earlySingletonObjects` — 早期引用（未初始化完成）
- 三级：`singletonFactories` — ObjectFactory，可生成早期引用（用于 AOP 代理）

---

### 2. ApplicationContext 刷新 (refresh 12 步)

**核心类:** `spring-context/.../support/AbstractApplicationContext.java`

```
refresh()
 1. prepareRefresh()                    — 激活标志、初始化 PropertySources、校验必需属性
 2. obtainFreshBeanFactory()            — 创建/刷新 BeanFactory，加载 BeanDefinitions
 3. prepareBeanFactory()                — 注册基础组件（ClassLoader、Environment、BPP）
 4. postProcessBeanFactory()            — 子类钩子（Web 容器注册 Servlet 相关 Bean）
 5. invokeBeanFactoryPostProcessors()   — 执行 BFPP：ConfigurationClassPostProcessor 处理 @Configuration/@Bean/@Import
 6. registerBeanPostProcessors()        — 注册所有 BPP（按 PriorityOrdered > Ordered > 无序）
 7. initMessageSource()                 — 初始化国际化（messageSource bean）
 8. initApplicationEventMulticaster()   — 初始化事件广播器
 9. onRefresh()                         — 子类钩子（Web 容器：创建嵌入式 WebServer）
10. registerListeners()                 — 注册 ApplicationListener，发布早期事件
11. finishBeanFactoryInitialization()   — 实例化所有非懒加载单例 bean
12. finishRefresh()                     — 启动 Lifecycle beans，发布 ContextRefreshedEvent
```

**关键步骤 5** 由 `ConfigurationClassPostProcessor` 完成：解析 `@Configuration`、`@ComponentScan`、`@Import`、`@Bean`，向 Registry 注册所有 BeanDefinition。

---

### 3. @Configuration 类处理

**核心类:** `spring-context/.../annotation/ConfigurationClassPostProcessor.java`

**处理顺序** (`doProcessConfigurationClass`)：
1. 处理成员内部类（嵌套 `@Configuration`）
2. 处理 `@PropertySource` — 注册属性源
3. 处理 `@ComponentScan` — 扫描组件
4. 处理 `@Import` — 三类：Configuration 类 / ImportSelector / ImportBeanDefinitionRegistrar
5. 处理 `@ImportResource` — 加载 XML BeanDefinition
6. 处理 `@Bean` 方法 — 注册为 RootBeanDefinition（工厂方法模式）
7. 递归处理父类

**Configuration 增强（CGLIB 子类化）：**
- `proxyBeanMethods=true`（默认）时，`@Bean` 方法调用会被拦截，保证单例语义
- `proxyBeanMethods=false` 为 Lite 模式，性能更好，但跨 `@Bean` 调用不保证单例

---

### 4. AOP 代理选择

**核心类:** `spring-aop/.../framework/DefaultAopProxyFactory.java`

```java
// 代理类型选择逻辑
if (proxyTargetClass || optimize || 无接口) {
    if (目标是接口 || 已是代理类 || 是 Lambda) → JDK 动态代理
    else → ObjenesisCglibAopProxy（CGLIB 子类化）
} else {
    → JDK 动态代理（有接口时的默认选择）
}
```

| 场景 | 代理类型 |
|------|--------|
| `@EnableAspectJAutoProxy(proxyTargetClass=true)` | CGLIB |
| 目标类无接口 | CGLIB |
| 目标类有接口（默认） | JDK 动态代理 |
| 目标本身是接口/Lambda | JDK 动态代理 |

**CGLIB 使用 Objenesis 绕过构造器调用**，避免副作用。

---

### 5. 事务管理 (@Transactional)

**核心类:** `spring-tx/.../interceptor/TransactionInterceptor.java` → `TransactionAspectSupport`

**执行流程:**
```
@Transactional 方法调用
  └─ TransactionInterceptor.invoke()
       └─ invokeWithinTransaction()
            ├─ 1. 获取事务属性（传播行为、隔离级别、回滚规则）
            ├─ 2. 获取 PlatformTransactionManager
            ├─ 3. getTransaction() — 开启/挂起/加入事务
            ├─ 4. 执行业务方法
            ├─ 5a. commit()  — 正常返回
            └─ 5b. rollback() — 异常匹配回滚规则时
```

**事务上下文:** 通过 `TransactionSynchronizationManager`（ThreadLocal）在当前线程传递。

**Reactive 支持:** 同时支持 `ReactiveTransactionManager`，上下文通过 Reactor Context 传递（非 ThreadLocal）。

---

### 6. 依赖注入 (@Autowired)

**核心类:** `spring-beans/.../annotation/AutowiredAnnotationBeanPostProcessor.java`

**解析算法:**
1. 按类型查找候选 Bean
2. 多个候选时，按 `@Qualifier` 筛选
3. 仍有多个时，使用字段/参数名称匹配 Bean name
4. 多个时取 `@Primary` 标记的
5. 支持 `Optional<T>`、`ObjectProvider<T>`、`List<T>`、`Map<String, T>`

**注入顺序:** 构造器注入 → 字段注入 → Setter 注入（均在 `populateBean()` 阶段）

---

### 7. Environment & PropertySource

**核心类:** `spring-core/.../env/StandardEnvironment.java`

**属性源优先级（从高到低）:**
1. JVM 系统属性（`-Dkey=value`）
2. 系统环境变量
3. 应用属性文件
4. Profile 专属属性文件

**解析算法:** `PropertySourcesPropertyResolver` 遍历有序的 PropertySource 链，取第一个非 null 值，支持占位符嵌套解析（`${other.property}`）。

---

### 8. Web MVC DispatcherServlet

**核心类:** `spring-webmvc/.../DispatcherServlet.java`

**doDispatch 请求流程:**
```
请求进入
  ├─ 1. 检测 Multipart 请求
  ├─ 2. getHandler() — 遍历 HandlerMappings 找到 HandlerExecutionChain
  ├─ 3. applyPreHandle() — 执行拦截器 preHandle（返回 false 则终止）
  ├─ 4. getHandlerAdapter().handle() — 执行 Controller 方法，返回 ModelAndView
  ├─ 5. applyPostHandle() — 执行拦截器 postHandle
  ├─ 6. processDispatchResult()
  │     ├─ 有异常 → HandlerExceptionResolver 处理
  │     └─ 无异常 → render(ModelAndView) → ViewResolver 解析 → View.render()
  └─ 7. afterCompletion() — 拦截器清理（finally）
```

**HandlerMapping 类型:**
- `RequestMappingHandlerMapping` — `@RequestMapping` 注解
- `BeanNameUrlHandlerMapping` — Bean name 作为 URL
- `RouterFunctionMapping` — WebFlux 函数式路由

---

### 9. SpEL

**核心类:** `spring-expression/.../spel/standard/SpelExpressionParser.java`

**架构:** Parser → AST（抽象语法树）→ Interpreter / Compiled（字节码）

**支持语法:** 属性访问 `person.name`、方法调用 `list.size()`、集合投影 `list.![name]`、条件 `a ?: b`、Null安全 `person?.address`、Bean 引用 `@beanName`、类型操作 `T(java.lang.Math).PI`

---

## 二、Spring Boot 核心机制

### 10. 启动流程 (SpringApplication)

**核心类:** `core/spring-boot/.../SpringApplication.java`

```
SpringApplication.run()
  ├─ 1. 检测 WebApplicationType
  ├─ 2. 创建 BootstrapRegistry
  ├─ 3. 加载 ApplicationContextInitializer / ApplicationListener
  ├─ 4. 发布 ApplicationStartingEvent
  ├─ 5. 准备 Environment → ApplicationEnvironmentPreparedEvent
  ├─ 6. 创建 ApplicationContext（根据 WebApplicationType）
  ├─ 7. context.refresh()（触发上方 12 步）
  ├─ 8. 发布 ApplicationStartedEvent
  ├─ 9. 执行 ApplicationRunner / CommandLineRunner
  └─ 10. 发布 ApplicationReadyEvent
```

**事件总线:** `EventPublishingRunListener` 将生命周期回调转为 Spring 事件广播。

---

### 11. 自动配置机制

**注册文件:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`（分散在各 module，共 310 个配置类）

**加载流程:**
1. `@EnableAutoConfiguration` → `AutoConfigurationImportSelector`（实现 `DeferredImportSelector`）
2. 读取所有 `.imports` 文件
3. 通过 `AutoConfigurationImportFilter` 快速过滤（不加载类，仅检查元数据）
4. 在用户 `@Configuration` 之后处理，保证用户 Bean 优先

**自动配置类计数（按模块）:**

| 模块 | 数量 |
|------|------|
| micrometer（指标/Tracing）| 39 |
| data（JPA/Redis/MongoDB等）| 31 |
| actuate（所有 Actuator 端点）| 24 |
| security（OAuth2/SAML）| 20 |
| graphql | 12 |
| autoconfigure（核心）| 12 |
| webflux | 9 |
| jdbc | 9 |

> 核心 `spring-boot-autoconfigure` 模块仅含 12 个，其余全部分散到各 module 的独立 imports 文件。

---

### 12. 条件注解 (@Conditional*)

**位置:** `core/spring-boot-autoconfigure/.../condition/`

| 注解 | 实现 | 说明 |
|------|------|------|
| `@ConditionalOnClass` | `OnClassCondition` | 类路径检查（ASM，不加载类）|
| `@ConditionalOnMissingBean` | `OnBeanCondition` | BeanFactory 中无指定 Bean |
| `@ConditionalOnBean` | `OnBeanCondition` | BeanFactory 中有指定 Bean |
| `@ConditionalOnProperty` | `OnPropertyCondition` | 环境属性满足条件 |
| `@ConditionalOnWebApplication` | `OnWebApplicationCondition` | 是否 Web 应用 |

- `OnClassCondition` 多核时并行执行类检查
- `OnBeanCondition` 实现 `ConfigurationCondition`，在 `REGISTER_BEAN` 阶段执行，可感知已注册 Bean，支持 `SearchStrategy`（CURRENT / ANCESTORS / ALL）

---

### 13. @ConfigurationProperties 绑定

```
ConfigurationPropertiesBindingPostProcessor (BeanPostProcessor)
  └─ ConfigurationPropertiesBinder.bind()
       └─ Binder（从 Environment 的 PropertySources 读取）
            ├─ 宽松绑定（kebab-case = camelCase = UPPER_CASE）
            ├─ 支持嵌套对象、List、Map
            ├─ 支持构造器绑定（@ConstructorBinding）
            └─ 支持 JSR-303 校验（@Validated）
```

---

### 14. 测试切片 (Test Slices)

每个切片注解 = `TypeExcludeFilter`（只加载相关组件）+ `.imports` 文件（只激活相关自动配置）

**@WebMvcTest 加载范围:** `@Controller` / `@ControllerAdvice` / `Converter` / `Filter` / `HandlerInterceptor` / `WebMvcConfigurer`

**@DataJpaTest:** 只加载 JPA 配置，内嵌数据库，事务自动回滚。

---

### 15. Fat JAR 加载

```
MANIFEST.MF:
  Main-Class: org.springframework.boot.loader.launch.JarLauncher
  Start-Class: com.example.MyApplication

JAR 结构:
  BOOT-INF/classes/    ← 应用类
  BOOT-INF/lib/        ← 依赖 JARs（嵌套）
  org/springframework/boot/loader/  ← Loader 类
```

`LaunchedClassLoader` 自定义 ClassLoader，支持从嵌套 JAR 加载类。

---

## 三、Spring Data Redis 核心机制

### 16. RedisTemplate 核心操作

**核心类:** `spring-data-redis/src/main/java/org/springframework/data/redis/core/RedisTemplate.java`

**工作原理:**
- Redis 数据访问的中心抽象，自动序列化/反序列化
- 使用 `RedisCallback` 接口处理底层连接
- 自动管理连接生命周期（获取 → 执行 → 释放）
- 线程安全（初始化后）

**执行模式:**
```
execute(RedisCallback)
  └─ getConnection()
       └─ doInRedis(connection)
            └─ releaseConnection()
```

**操作类型:**

| 操作 | 接口 | 典型方法 |
|------|------|---------|
| 字符串 | `ValueOperations<K,V>` | set, get, append, increment |
| 列表 | `ListOperations<K,V>` | push, pop, range, trim |
| 集合 | `SetOperations<K,V>` | add, remove, union, intersect |
| 有序集合 | `ZSetOperations<K,V>` | zadd, zrange, zrank |
| 哈希 | `HashOperations<K,HK,HV>` | hset, hget, hgetall |
| 地理位置 | `GeoOperations<K,V>` | geoadd, geodist, georadius |
| HyperLogLog | `HyperLogLogOperations<K,V>` | pfadd, pfcount |
| Stream | `StreamOperations<K,HK,HV>` | xadd, xread, xgroup |

**Bound Operations（绑定键操作）:**
```java
BoundValueOperations<K,V> ops = redisTemplate.boundValueOps("mykey");
ops.set("value");
ops.increment();
```

---

### 17. 连接管理

**工厂接口:** `RedisConnectionFactory`

**Lettuce 实现:** `LettuceConnectionFactory`（基于 Netty 的异步客户端）

**支持的配置模式:**
- `RedisStandaloneConfiguration` — 单实例
- `RedisStaticMasterReplicaConfiguration` — 主从复制
- `RedisSentinelConfiguration` — Sentinel 高可用
- `RedisClusterConfiguration` — 集群模式
- `RedisSocketConfiguration` — Unix Socket

**连接共享:** 默认多个 `LettuceConnection` 共享一个线程安全的原生连接，提升性��。

**连接池:** 可选配置 `LettucePoolingClientConfiguration` 使用 Apache Commons Pool2。

---

### 18. 序列化策略

**核心接口:** `RedisSerializer<T>`

| 序列化器 | 用途 | 场景 |
|---------|------|------|
| `StringRedisSerializer` | UTF-8 字符串 | 键/值为字符串 |
| `JdkSerializationRedisSerializer` | Java 对象序列化 | 默认，任何 Serializable |
| `Jackson2JsonRedisSerializer` | Jackson JSON（类型化） | 特定类型 JSON |
| `GenericJackson2JsonRedisSerializer` | Jackson JSON（通用） | 多态类型 + 类型信息 |
| `ByteArrayRedisSerializer` | 字节数组透传 | 原始二进制 |
| `GenericToStringSerializer` | toString/构造器 | 自定义字符串转换 |

**配置示例:**
```java
RedisTemplate<String, Object> template = new RedisTemplate<>();
template.setKeySerializer(new StringRedisSerializer());
template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
template.setHashKeySerializer(new StringRedisSerializer());
template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
```

---

### 19. Spring Cache 集成

**核心类:** `RedisCache` / `RedisCacheManager`

**工作流程:**
```
@Cacheable("users")
  └─ CacheInterceptor
       └─ RedisCacheManager.getCache("users")
            └─ RedisCache.get(key)
                 ├─ 命中 → 反序列化返回
                 └─ 未命中 → 执行方法 → put(key, result)
```

**配置选项:**
- `RedisCacheConfiguration` — TTL、键前缀、序列化器
- `enableStatistics()` — 启用缓存统计
- `disableCachingNullValues()` — 禁止缓存 null
- `entryTtl(Duration)` — 设置过期时间

**锁机制:** 使用 `RedisCacheWriter` 的 `lock()` 防止缓存击穿（多线程同时加载同一键）。

---

### 20. Pipeline & Transaction

**Pipeline（管道）:**
```java
List<Object> results = redisTemplate.executePipelined(
    (RedisCallback<Object>) connection -> {
        connection.set("key1".getBytes(), "value1".getBytes());
        connection.get("key2".getBytes());
        return null; // 返回值被忽略
    }
);
```
- 批量发送命令，减少网络往返
- 返回所有命令的结果列表

**Transaction（事务）:**
```java
redisTemplate.execute(new SessionCallback<List<Object>>() {
    public List<Object> execute(RedisOperations operations) {
        operations.multi();
        operations.opsForValue().set("key1", "value1");
        operations.opsForValue().increment("counter");
        return operations.exec(); // 原子执行
    }
});
```
- `MULTI` / `EXEC` 包裹的命令原子执行
- 不支持回滚（Redis 特性）

---

### 21. Pub/Sub 消息监听

**核心类:** `RedisMessageListenerContainer`

**配置示例:**
```java
@Bean
RedisMessageListenerContainer container(RedisConnectionFactory factory) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(factory);
    container.addMessageListener(new MessageListener() {
        public void onMessage(Message message, byte[] pattern) {
            System.out.println("Received: " + message);
        }
    }, new ChannelTopic("news"));
    return container;
}
```

**监听器类型:**
- `MessageListener` — 原始字节消息
- `MessageListenerAdapter` — 自动反序列化 + 方法委托

**Topic 类型:**
- `ChannelTopic` — 精确频道匹配
- `PatternTopic` — 模式匹配（如 `news.*`）

---

### 22. Redis Repository

**核心类:** `RedisKeyValueAdapter` / `RedisRepositoryFactoryBean`

**使用方式:**
```java
@RedisHash("users")
public class User {
    @Id String id;
    @Indexed String email;
    String name;
}

interface UserRepository extends CrudRepository<User, String> {
    List<User> findByEmail(String email);
}
```

**索引机制:**
- `@Indexed` 字段会创建 Redis Set 作为二级索引
- 查询时通过索引 Set 快速定位主键，再获取完整对象

**存储结构:**
```
users:{id}           → Hash（对象字段）
users:{id}:idx       → Set（索引引用）
users:email:{value}  → Set（email 索引，存储 id）
```

**TTL 支持:** `@TimeToLive` 注解或 `@RedisHash(timeToLive=60)`

---

### 23. Redis Streams

**核心接口:** `StreamOperations<K, HK, HV>`

**生产消息:**
```java
StreamRecords.newRecord()
    .ofObject(new SensorData(...))
    .withStreamKey("sensor-stream");
streamOps.add(record);
```

**消费消息（Consumer Group）:**
```java
StreamReadOptions options = StreamReadOptions.empty()
    .block(Duration.ofSeconds(2))
    .count(10);

List<MapRecord<String, Object, Object>> messages = streamOps.read(
    Consumer.from("group1", "consumer1"),
    options,
    StreamOffset.create("sensor-stream", ReadOffset.lastConsumed())
);

// 确认消息
streamOps.acknowledge("group1", record);
```

**特性:**
- 支持 Consumer Group（多消费者负载均衡）
- 支持 Pending 消息（未确认消息重新投递）
- 支持范围查询（按 ID 或时间）

---

### 24. Lua 脚本执行

**核心类:** `DefaultScriptExecutor` / `RedisScript<T>`

**使用方式:**
```java
RedisScript<Long> script = RedisScript.of(
    "return redis.call('incr', KEYS[1])",
    Long.class
);

Long result = redisTemplate.execute(
    script,
    Collections.singletonList("counter")
);
```

**脚本缓存:** 使用 `SCRIPT LOAD` + `EVALSHA` 减少网络传输。

**原子性保证:** Lua 脚本在 Redis 中原子执行，适合复杂的原子操作（如限流、分布式锁）。

---

### 25. ReactiveRedisTemplate

**核心类:** `ReactiveRedisTemplate`

**与同步版本的区别:**
- 返回 `Mono<T>` / `Flux<T>`（Project Reactor）
- 基于 `ReactiveRedisConnection`（非阻塞 I/O）
- 禁止 null 值（Reactive Streams 规范）

**使用示例:**
```java
ReactiveRedisTemplate<String, String> template = ...;

Mono<Boolean> result = template.opsForValue()
    .set("key", "value")
    .then(template.expire("key", Duration.ofMinutes(5)));
```

**Reactive 操作:**
- `ReactiveValueOperations`
- `ReactiveListOperations`
- `ReactiveSetOperations`
- `ReactiveZSetOperations`
- `ReactiveHashOperations`
- `ReactiveStreamOperations`

**脚本执行:** `DefaultReactiveScriptExecutor` 返回 `Mono<T>`。

---

## 四、关键源码路径速查

| 功能 | 路径 |
|------|------|
| Bean 生命周期 | `spring-beans/.../support/AbstractAutowireCapableBeanFactory.java` |
| Context 刷新 | `spring-context/.../support/AbstractApplicationContext.java` |
| Configuration 解析 | `spring-context/.../annotation/ConfigurationClassPostProcessor.java` |
| AOP 代理选择 | `spring-aop/.../framework/DefaultAopProxyFactory.java` |
| 事务拦截器 | `spring-tx/.../interceptor/TransactionInterceptor.java` |
| @Autowired 注入 | `spring-beans/.../annotation/AutowiredAnnotationBeanPostProcessor.java` |
| DispatcherServlet | `spring-webmvc/.../web/servlet/DispatcherServlet.java` |
| SpringApplication | `spring-boot/core/spring-boot/.../SpringApplication.java` |
| 自动配置选择器 | `spring-boot/core/spring-boot-autoconfigure/.../AutoConfigurationImportSelector.java` |
| 条件注解 | `spring-boot/core/spring-boot-autoconfigure/.../condition/` |
| ConfigurationProperties | `spring-boot/core/spring-boot/.../context/properties/` |
| Fat JAR Loader | `spring-boot/loader/spring-boot-loader/.../loader/` |
| RedisTemplate | `spring-data-redis/src/main/java/.../redis/core/RedisTemplate.java` |
| LettuceConnectionFactory | `spring-data-redis/src/main/java/.../redis/connection/lettuce/LettuceConnectionFactory.java` |
| RedisCache | `spring-data-redis/src/main/java/.../redis/cache/RedisCache.java` |
| RedisMessageListenerContainer | `spring-data-redis/src/main/java/.../redis/listener/RedisMessageListenerContainer.java` |
| RedisKeyValueAdapter | `spring-data-redis/src/main/java/.../redis/core/RedisKeyValueAdapter.java` |
