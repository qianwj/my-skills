# Spring Boot Skills — 源码级知识整理

## 1. 启动流程 (SpringApplication)

**核心类:** `core/spring-boot/src/main/java/org/springframework/boot/SpringApplication.java`

启动步骤：
1. 检测 WebApplicationType（SERVLET / REACTIVE / NONE）
2. 创建 BootstrapRegistry（早期初始化钩子）
3. 加载 ApplicationContextInitializer 和 ApplicationListener（从 spring.factories）
4. 发布 ApplicationStartingEvent
5. 准备 Environment → 发布 ApplicationEnvironmentPreparedEvent
6. 创建 ApplicationContext（根据 WebApplicationType 选择实现）
7. 刷新 Context（Bean 创建、属性绑定、自动配置）
8. 发布 ApplicationStartedEvent
9. 执行 ApplicationRunner / CommandLineRunner
10. 发布 ApplicationReadyEvent

**事件总线:** `EventPublishingRunListener` 实现 `SpringApplicationRunListener`，将生命周期回调转为 Spring 事件广播。

**扩展点:**
- `SpringApplicationRunListener` — 监听启动各阶段
- `ApplicationContextInitializer` — Context refresh 前回调
- `ApplicationRunner` / `CommandLineRunner` — 启动完成后执行

---

## 2. 自动配置机制 (@EnableAutoConfiguration)

**注册文件:** `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

**加载流程:**
1. `@EnableAutoConfiguration` 触发 `AutoConfigurationImportSelector`
2. 读取 `.imports` 文件中所有自动配置类名
3. 通过 `AutoConfigurationImportFilter`（OnClassCondition 等）过滤不满足条件的类
4. 剩余类通过 `DeferredImportSelector` 在用户配置之后处理，确保用户 Bean 优先

**排序:** `@AutoConfiguration(after = ..., before = ...)` 控制配置类加载顺序，跨模块用 `afterName` 字符串引用。

**典型结构:**
```java
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
@ConditionalOnClass(JpaRepository.class)
@ConditionalOnBean(DataSource.class)
@ConditionalOnMissingBean(JpaRepositoryFactoryBean.class)
public class DataJpaRepositoriesAutoConfiguration { ... }
```

---

## 3. 条件注解 (@Conditional*)

**位置:** `core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition/`

| 注解 | 实现类 | 说明 |
|------|--------|------|
| `@ConditionalOnClass` | `OnClassCondition` | 类路径存在指定类（用 ASM 避免加载） |
| `@ConditionalOnMissingClass` | `OnClassCondition` | 类路径不存在指定类 |
| `@ConditionalOnBean` | `OnBeanCondition` | BeanFactory 中存在指定 Bean |
| `@ConditionalOnMissingBean` | `OnBeanCondition` | BeanFactory 中不存在指定 Bean |
| `@ConditionalOnProperty` | `OnPropertyCondition` | 环境属性满足条件 |
| `@ConditionalOnWebApplication` | `OnWebApplicationCondition` | 是否为 Web 应用（SERVLET/REACTIVE） |
| `@ConditionalOnExpression` | `OnExpressionCondition` | SpEL 表达式为 true |
| `@ConditionalOnJava` | `OnJavaCondition` | Java 版本匹配 |
| `@ConditionalOnResource` | `OnResourceCondition` | 类路径存在指定资源 |

**OnClassCondition 优化:** 多核时将类检查任务拆分到多线程并行执行。

**OnBeanCondition 关键:** 实现 `ConfigurationCondition`，在 `REGISTER_BEAN` 阶段执行，可感知已注册的 Bean；支持 `SearchStrategy`（CURRENT / ANCESTORS / ALL）。

---

## 4. @ConfigurationProperties 绑定

**核心类:**
- `ConfigurationPropertiesBindingPostProcessor` — BeanPostProcessor，触发绑定
- `ConfigurationPropertiesBinder` — 实际绑定逻辑，使用 `Binder` + 验证器

**绑定流程:**
1. `ConfigurationPropertiesBindingPostProcessor.postProcessBeforeInitialization()` 检测 `@ConfigurationProperties`
2. 委托给 `ConfigurationPropertiesBinder.bind()`
3. `Binder` 从 `Environment` 的 `PropertySources` 中读取属性
4. 支持宽松绑定（kebab-case、camelCase、UPPER_CASE 均可）
5. 支持 JSR-303 校验（`@Validated`）

**注册方式:**
- `@EnableConfigurationProperties(MyProps.class)` — 显式注册
- `@ConfigurationPropertiesScan` — 扫描注册
- 直接在 `@Bean` 方法上标注

---

## 5. 模块与 Starter 关系

```
module/spring-boot-redis/          ← 实际代码（AutoConfiguration、支持类）
starter/spring-boot-starter-redis/ ← 仅 POM，依赖 module + 第三方库
```

**规律:** Starter 不含代码，只聚合依赖。真正的自动配置在对应 module 中。

---

## 6. Actuator 端点机制

**位置:** `module/spring-boot-actuator/`

**端点定义:**
```java
@Endpoint(id = "health")
public class HealthEndpoint {
    @ReadOperation
    public HealthComponent health() { ... }

    @WriteOperation
    public void configure(...) { ... }
}
```

**操作注解:** `@ReadOperation`（GET）、`@WriteOperation`（POST）、`@DeleteOperation`（DELETE）

**暴露方式:** 端点通过 `@EndpointWebExtension` 或 `@EndpointJmxExtension` 适配到 Web/JMX。

---

## 7. 测试切片 (Test Slices)

**位置:** `core/spring-boot-test-autoconfigure/`

**机制:** 每个切片注解（如 `@WebMvcTest`）对应：
1. 一个 `TypeExcludeFilter`（如 `WebMvcTypeExcludeFilter`）— 只加载相关 Bean 类型（Controller、Converter、Filter 等）
2. 一个 `.imports` 文件 — 只激活相关自动配置子集

**WebMvcTypeExcludeFilter 包含的类型:**
- `@Controller`, `@ControllerAdvice`, `@JsonComponent`
- `Converter`, `GenericConverter`, `Filter`, `HandlerInterceptor`
- `WebMvcConfigurer`, `WebMvcRegistrations`, `HandlerMethodArgumentResolver`

**DataJpaTest 类似:** 只加载 JPA 相关配置，默认使用内嵌数据库，事务自动回滚。

---

## 8. Fat JAR 加载机制

**Manifest 结构:**
```
Main-Class: org.springframework.boot.loader.launch.JarLauncher
Start-Class: com.example.MyApplication
```

**JAR 结构:**
```
app.jar
├── BOOT-INF/classes/     ← 应用类
├── BOOT-INF/lib/         ← 依赖 JAR（嵌套）
└── org/springframework/boot/loader/  ← Loader 类（直接在根）
```

**LaunchedClassLoader:** 自定义 ClassLoader，支持从嵌套 JAR 中加载类，解决标准 ClassLoader 不支持嵌套 JAR 的问题。

---

## 9. ApplicationContext 层次结构

- `SpringApplicationBuilder` 支持构建父子 Context
- 典型场景：Web 层 Context（子）+ 业务层 Context（父）
- 子 Context 可访问父 Context 的 Bean，反之不行
- `setParent()` / `parent()` 方法设置层次

---

## 10. 自动配置全景（310 个配置类，分布在 90 个 imports 文件）

按模块分组（配置类数量）：

| 模块 | 数量 | 说明 |
|------|------|------|
| micrometer | 39 | 指标、Prometheus、OTLP、Tracing |
| data | 31 | JPA、MongoDB、Redis、Cassandra、Neo4j 等 |
| actuate | 24 | 所有 Actuator 端点 |
| security | 20 | Web 安全、OAuth2、SAML |
| graphql | 12 | GraphQL over HTTP/WebSocket/RSocket |
| autoconfigure | 12 | 核心：AOP、SSL、Task、JMX 等 |
| webflux | 9 | Reactive Web |
| jdbc | 9 | DataSource、JdbcTemplate、事务 |
| r2dbc | 7 | Reactive 数据库 |
| http | 7 | HTTP 客户端/服务端基础 |
| health | 7 | 健康检查贡献者 |
| webmvc | 6 | Servlet Web MVC |
| tomcat/jetty | 5/5 | 嵌入式服务器 |
| batch | 4 | Spring Batch |
| session | 4 | Session（Redis/JDBC） |
| kafka | 2 | Kafka 消息 |
| flyway/liquibase | 2/2 | 数据库迁移 |

**核心模块（core/spring-boot-autoconfigure）只有 12 个配置类**，其余全部分散在各 module 的独立 imports 文件中。这是 Spring Boot 5.x 的模块化重构结果——每个 module 自包含其自动配置注册。

---

## 11. 关键源码路径速查

| 功能 | 路径 |
|------|------|
| SpringApplication | `core/spring-boot/src/main/java/org/springframework/boot/SpringApplication.java` |
| 自动配置列表 | `core/spring-boot-autoconfigure/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 条件注解 | `core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition/` |
| ConfigurationProperties | `core/spring-boot/src/main/java/org/springframework/boot/context/properties/` |
| WebMvc 自动配置 | `module/spring-boot-webmvc/src/main/java/org/springframework/boot/webmvc/autoconfigure/WebMvcAutoConfiguration.java` |
| JPA 自动配置 | `module/spring-boot-data-jpa/src/main/java/org/springframework/boot/data/jpa/autoconfigure/` |
| Actuator 端点 | `module/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/` |
| 测试切片 | `core/spring-boot-test-autoconfigure/src/main/java/org/springframework/boot/test/autoconfigure/` |
| Fat JAR Loader | `loader/spring-boot-loader/src/main/java/org/springframework/boot/loader/` |
