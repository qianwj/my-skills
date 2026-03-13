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

## 三、关键源码路径速查

| 功能 | 路径 |
|------|------|
| Bean 生命周期 | `spring-beans/src/main/java/org/springframework/beans/factory/support/AbstractAutowireCapableBeanFactory.java` |
| Context 刷新 | `spring-context/src/main/java/org/springframework/context/support/AbstractApplicationContext.java` |
| Configuration 解析 | `spring-context/src/main/java/org/springframework/context/annotation/ConfigurationClassPostProcessor.java` |
| AOP 代理选择 | `spring-aop/src/main/java/org/springframework/aop/framework/DefaultAopProxyFactory.java` |
| 事务拦截器 | `spring-tx/src/main/java/org/springframework/transaction/interceptor/TransactionInterceptor.java` |
| @Autowired 注入 | `spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java` |
| DispatcherServlet | `spring-webmvc/src/main/java/org/springframework/web/servlet/DispatcherServlet.java` |
| SpringApplication | `core/spring-boot/src/main/java/org/springframework/boot/SpringApplication.java` |
| 自动配置选择器 | `core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/AutoConfigurationImportSelector.java` |
| 条件注解 | `core/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition/` |
| ConfigurationProperties | `core/spring-boot/src/main/java/org/springframework/boot/context/properties/` |
| Fat JAR Loader | `loader/spring-boot-loader/src/main/java/org/springframework/boot/loader/` |
