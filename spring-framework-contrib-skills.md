# Spring Framework 源码贡献与构建体系 Skills

> 基于 `spring-projects/spring-framework` 源码构建系统与贡献实践整理

---

## 一、Gradle 构建体系

### 1. 根级 build.gradle 结构

```groovy
// 多项目分组方式
ext {
    moduleProjects = subprojects.findAll { it.name.startsWith("spring-") }
    javaProjects   = subprojects.findAll { !it.name.startsWith("framework-") }
}

// 应用约定插件（自定义 buildSrc 插件）
configure([rootProject] + javaProjects) {
    apply plugin: "java"
    apply plugin: "java-test-fixtures"
    apply plugin: 'org.springframework.build.conventions'  // 统一约定入口

    // 所有模块公共测试依赖
    dependencies {
        testImplementation("org.junit.jupiter:junit-jupiter")
        testImplementation("org.junit.platform:junit-platform-suite")
        testImplementation("org.mockito:mockito-core")
        testImplementation("io.mockk:mockk")
        testImplementation("org.assertj:assertj-core")
    }
}
```

**Javadoc 外链列表统一在根级配置：**
```groovy
ext.javadocLinks = [
    "https://docs.oracle.com/en/java/javase/17/docs/api/",
    "https://projectreactor.io/docs/core/release/api/",
    ...
]
```

---

### 2. 统一约定插件 (buildSrc/ConventionsPlugin)

**核心类：** `buildSrc/.../ConventionsPlugin.java`

```java
// ConventionsPlugin 是入口，聚合所有约定
public class ConventionsPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        project.getExtensions().create("springFramework", SpringFrameworkExtension.class);
        new ArchitecturePlugin().apply(project);    // 架构检查
        new CheckstyleConventions().apply(project); // 代码风格
        new JavaConventions().apply(project);       // Java 编译约定
        new KotlinConventions().apply(project);     // Kotlin 编译约定
        new TestConventions().apply(project);       // 测试约定
    }
}
```

---

### 3. Java 编译约定 (JavaConventions)

```java
// 工具链版本配置
private static final JavaLanguageVersion DEFAULT_LANGUAGE_VERSION = JavaLanguageVersion.of(25); // JDK 版本
private static final JavaLanguageVersion DEFAULT_RELEASE_VERSION  = JavaLanguageVersion.of(17); // 字节码目标

// 主源码编译参数 (Werror: 所有警告都报错)
COMPILER_ARGS = [
    "-Xlint:serial", "-Xlint:cast", "-Xlint:classfile", "-Xlint:dep-ann",
    "-Xlint:varargs", "-Xlint:rawtypes", "-Xlint:deprecation", "-Xlint:unchecked",
    "-Werror",    // ← 关键：所有警告提升为编译错误
    "-parameters" // ← 保留参数名（反射需要）
]

// 测试源码编译参数（关闭大部分 Xlint，避免测试过于严格）
TEST_COMPILER_ARGS = [
    "-Xlint:-varargs", "-Xlint:-rawtypes", "-Xlint:-deprecation", "-Xlint:-unchecked"
]
```

**关键点：**
- 主源码启用 `-Werror`，任何 `@SuppressWarnings` 都需要有充分理由
- 测试代码放宽 lint 检查，便于灵活测试
- 所有模块共用同一工具链版本，保证一致性

---

### 4. Checkstyle 约定 (CheckstyleConventions)

```java
// Checkstyle 配置
checkstyle.setToolVersion("13.3.0");
checkstyle.getConfigDirectory().set(project.getRootProject().file("src/checkstyle"));

// 额外添加 Spring JavaFormat 的 checkstyle 检查
checkstyleDependencies.add(
    project.getDependencies().create("io.spring.javaformat:spring-javaformat-checkstyle:" + version)
);
```

**Checkstyle 规则要点（`src/checkstyle/checkstyle.xml`）：**

| 检查类型 | 规则 |
|---------|------|
| 文件头 | Apache 2.0 License Header（`headerCopyrightPattern: 20\d\d-present`）|
| 换行符 | 文件末尾必须有换行 |
| 注解风格 | `elementStyle=compact`（紧凑格式） |
| 缺失 @Override | 必须加 `@Override` |
| 空块 | 空 catch/if 块必须有注释 |
| 大括号 | 左括号随行，右括号独占一行 |

**Spring JavaFormat 格式化：**
```bash
./gradlew format           # 自动格式化所有源码
./gradlew checkstyleMain   # 只检查不修改
```

**nohttp 检查（防止 HTTP 明文链接）：**
```java
// NoHttpPlugin 检查所有非 build 目录的文件中的 http:// URL
noHttp.setAllowlistFile(project.file("src/nohttp/allowlist.lines"));
// allowlist.lines 中列出允许使用 http 的例外
```

---

### 5. 测试约定 (TestConventions)

```java
// JUnit Platform 配置
test.useJUnitPlatform();
test.include("**/*Tests.class", "**/*Test.class"); // 命名约定

// 系统属性
test.setSystemProperties(Map.of(
    "java.awt.headless", "true",
    "io.netty.leakDetection.level", "paranoid",  // Netty 内存泄漏检测（严格模式）
    "junit.platform.discovery.issue.severity.critical", "INFO"
));

// JVM 参数
test.jvmArgs(
    "--add-opens=java.base/java.lang=ALL-UNNAMED",  // 反射访问
    "--add-opens=java.base/java.util=ALL-UNNAMED",
    "-Xshare:off"  // 关闭 CDS（避免测试中类加载不一致）
);

// CI 环境自动重试失败测试（最多 3 次）
testRetry.getMaxRetries().set(isCi() ? 3 : 0);
testRetry.getFailOnPassedAfterRetry().set(true); // 重试后通过的测试也报告为问题
```

**ByteBuddy 代理配置（用于运行时字节码增强测试）：**
```java
// 在 gradle.properties 中指定: byteBuddyVersion=x.y.z
// TestConventions 会自动将 byte-buddy-agent 添加为 javaagent
test.jvmArgs("-javaagent:" + byteBuddyAgentConfig.getAsPath())
```

---

## 二、Optional 依赖模式

### 6. Maven-style Optional 依赖

**核心类：** `buildSrc/.../optional/OptionalDependenciesPlugin.java`

```java
// 创建 optional 配置：对本项目可见，不传递给下游
Configuration optional = project.getConfigurations().create(OPTIONAL_CONFIGURATION_NAME);
optional.setCanBeConsumed(false);  // 不能被消费
optional.setCanBeResolved(false);  // 不能被解析（仅用于 classpath 扩展）

// optional 加入 compileClasspath 和 runtimeClasspath，但不进入 api/implementation
sourceSets.all(sourceSet -> {
    configurations.getByName(sourceSet.getCompileClasspathConfigurationName()).extendsFrom(optional);
    configurations.getByName(sourceSet.getRuntimeClasspathConfigurationName()).extendsFrom(optional);
});
```

**在模块中使用：**
```groovy
// spring-webmvc.gradle
dependencies {
    optional("com.fasterxml.jackson.core:jackson-databind")  // 存在时启用 Jackson 序列化
    optional("io.projectreactor:reactor-core")               // 存在时启用响应式支持
}
```

**代码中的对应模式（运行时特性检测）：**
```java
// 不直接 import，通过 ClassUtils 检测
private static final boolean jackson2Present =
    ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", ...);

// 或使用 @ConditionalOnClass（Spring Boot 环境）
```

---

## 三、Multi-Release JAR 模式

### 7. 多 Java 版本字节码在同一 JAR 中

**核心类：** `buildSrc/.../multirelease/MultiReleaseExtension.java`

**JAR 结构：**
```
spring-core.jar
├── META-INF/MANIFEST.MF          (Multi-Release: true)
├── org/springframework/...        (Java 17 字节码，基线)
└── META-INF/versions/
    ├── 21/
    │   └── org/springframework/... (Java 21 优化实现)
    └── 24/
        └── org/springframework/... (Java 24 ClassFile API 实现)
```

**源码目录约定：**
```
spring-core/
└── src/
    ├── main/java/          → 编译为 Java 17 字节码
    ├── main/java21/        → 编译为 Java 21 字节码，打入 META-INF/versions/21/
    └── main/java24/        → 编译为 Java 24 字节码，打入 META-INF/versions/24/
```

**build.gradle 中启用：**
```groovy
plugins {
    id 'org.springframework.build.multiReleaseJar'
}

multiRelease {
    releaseVersions 21, 24  // 声明需要版本专属实现的 Java 版本
}

configurations {
    java21Api.extendsFrom(api)          // java21 可访问主模块的 API
    java24Api.extendsFrom(api)
}
```

**典型场景（spring-core）：**
- `java17`：基线 ASM-based `AnnotationMetadata` 实现
- `java24`：ClassFile API-based `ClassFileAnnotationMetadata`（更快，无需 ASM 依赖）
- 运行时自动选择最优版本，应用代码零感知

---

## 四、架构合规检查 (ArchUnit)

### 8. 自定义架构规则

**核心类：** `buildSrc/.../architecture/ArchitectureRules.java`

```java
// 规则1：禁止循环包依赖
static ArchRule allPackagesShouldBeFreeOfTangles() {
    return SlicesRuleDefinition.slices()
        .assignedFrom(new SpringSlices())
        .should().beFreeOfCycles();
}

// 规则2：禁止使用 String.toLowerCase() 不带 Locale（国际化问题）
static ArchRule noClassesShouldCallStringToLowerCaseWithoutLocale() {
    return ArchRuleDefinition.noClasses()
        .should().callMethod(String.class, "toLowerCase")
        .because("String.toLowerCase(Locale.ROOT) should be used instead");
}

// 规则3：禁止 Java 类导入 Kotlin 专用注解（org.jetbrains.annotations）
static ArchRule javaClassesShouldNotImportKotlinAnnotations() {
    return ArchRuleDefinition.noClasses()
        .that(new DescribedPredicate<>("is not a Kotlin class") {
            public boolean test(JavaClass c) {
                return c.getSourceCodeLocation().getSourceFileName().endsWith(".java");
            }
        })
        .should().dependOnClassesThat().resideInAnyPackage("org.jetbrains.annotations..");
}

// 规则4：禁止导入特定类型（如 org.slf4j.LoggerFactory - 应用 spring-jcl）
static ArchRule classesShouldNotImportForbiddenTypes() {
    return ArchRuleDefinition.noClasses()
        .should().dependOnClassesThat().haveFullyQualifiedName("org.slf4j.LoggerFactory");
}
```

**触发时机：**
```bash
./gradlew checkArchitectureMain   # 检查主源码架构
./gradlew check                   # 包含架构检查的完整验证
```

---

## 五、Shadow JAR / 依赖重打包

### 9. 内嵌第三方库（避免依赖冲突）

**spring-core 将 Objenesis 和 JavaPoet 重打包内嵌：**

```groovy
// spring-core.gradle
def objenesisVersion = "3.5"
configurations { objenesis }

tasks.register('objenesisRepackJar', ShadowJar) {
    archiveBaseName = 'spring-objenesis-repack'
    configurations = [configurations.objenesis]
    relocate('org.objenesis', 'org.springframework.objenesis')  // 重定位包名
}
```

**包名重定位效果：**
- 原始：`org.objenesis.Objenesis` → 重定位为：`org.springframework.objenesis.Objenesis`
- 避免用户自己使用不同版本 Objenesis 时发生冲突
- 源码 JAR 同样处理（`ShadowSource` 任务）

---

## 六、贡献代码规范

### 10. 提交信息规范

**格式：**
```
Fix typo and improve Javadoc for ConfigurationBeanNameGenerator

Closes gh-12345
```

**关键要求：**
- 首行简洁描述（50 字以内，祈使语气）
- DCO 签名（`Signed-off-by: Name <email>`）
- 引用 GitHub Issue（`Closes gh-xxxxx` / `Fixes gh-xxxxx`）
- 提交者使用 `git commit -s` 自动添加 Signed-off-by

**常见提交类型前缀（spring-framework 风格）：**

| 前缀 | 说明 |
|------|------|
| `Fix ...` | Bug 修复 |
| `Add ...` | 新增功能 |
| `Support ...` | 添加对某特性的支持 |
| `Improve ...` | 改进（非 Bug Fix） |
| `Polish ...` / `Polishing` | 代码整理/清理 |
| `Document ...` / `Update docs` | 文档更新 |
| `Upgrade to ...` | 依赖升级 |

---

### 11. 代码规范要点

**Apache 2.0 License Header（必须）：**
```java
/*
 * Copyright 2002-present the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * ...
 */
```

**Javadoc 要求：**
```java
/**
 * 类/方法描述（第一句话是摘要）。
 *
 * <p>更多细节段落。
 *
 * @author Your Name
 * @since 6.x
 * @see RelatedClass
 */
```

**日志规范（用 spring-jcl，不用 SLF4J）：**
```java
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;

protected final Log logger = LogFactory.getLog(getClass());

// 分级日志（避免字符串拼接开销）
if (logger.isDebugEnabled()) {
    logger.debug("Processing " + something);
}
// Java 21+ Text Block 风格
if (logger.isTraceEnabled()) {
    logger.trace("Registering bean definition for @Bean method %s.%s() with bean name '%s'"
        .formatted(className, methodName, beanName));
}
```

**String 操作规范：**
```java
// 必须指定 Locale（架构检查会强制）
str.toLowerCase(Locale.ROOT);    // ✓ 正确
str.toUpperCase(Locale.ROOT);    // ✓ 正确
str.toLowerCase();               // ✗ ArchUnit 报错
```

**Nullability 注解（spring-framework 专用）：**
```java
import org.springframework.lang.Nullable;
// 注意：禁止使用 org.springframework.lang.NonNull（ArchUnit 禁止导入）
// 使用 @Nullable 标记可空，未标注默认非空

public @Nullable String findName(@Nullable String prefix) { ... }
```

---

### 12. 测试类命名约定

```
*Tests.java   → 标准单元测试（Checkstyle include 模式）
*Test.java    → 也包含在内
*IT.java      → 集成测试（可能需要额外配置）
```

**测试结构（JUnit 5 + AssertJ）：**
```java
class FooBarTests {

    @Test
    void shouldDoSomethingWhenCondition() {
        // given
        ...
        // when
        ...
        // then
        assertThat(result).isEqualTo(expected);
    }

    @Test
    void shouldThrowExceptionWhenInvalid() {
        assertThatExceptionOfType(IllegalArgumentException.class)
            .isThrownBy(() -> new FooBar(null))
            .withMessage("...");
    }
}
```

---

## 七、依赖版本管理

### 13. BOM 平台模式 (framework-platform)

```groovy
// 所有子项目通过 framework-platform 统一管理版本
configure(allprojects - project(":framework-platform")) {
    configurations {
        dependencyManagement {
            canBeConsumed = false
            canBeResolved = false
        }
        // 所有 *Classpath 自动继承 dependencyManagement
        matching { it.name.endsWith("Classpath") }.all {
            it.extendsFrom(dependencyManagement)
        }
    }
    dependencies {
        dependencyManagement(enforcedPlatform(":framework-platform"))
    }
}
```

**效果：** 所有模块声明依赖不写版本号，统一从 platform 解析。

```groovy
// 子模块 build.gradle
dependencies {
    implementation("com.fasterxml.jackson.core:jackson-databind")  // 无版本号
    optional("io.projectreactor:reactor-core")                     // 无版本号
}
```

---

## 八、ClassFile API 与 ASM 元数据读取

### 14. 双实现策略（ASM / ClassFile API）

**场景：** `spring-core` 需要读取类文件的注解元数据，同时支持 Java 17（ASM）和 Java 24（ClassFile API）。

**目录结构：**
```
spring-core/src/main/java/
  org/springframework/core/type/classreading/
    SimpleAnnotationMetadata.java         (ASM-based, Java 17+)
    SimpleAnnotationMetadataReadingVisitor.java
spring-core/src/main/java24/
  org/springframework/core/type/classreading/
    ClassFileAnnotationMetadata.java      (ClassFile API, Java 24+)
    ClassFileAnnotationDelegate.java
    MetadataReaderFactoryDelegate.java
```

**关键设计点（ClassFile 实现）：**
```java
// 获取嵌套类的外围类名 —— 需同时检查 NestHost 和 InnerClasses
// NestHost: Java11+ 字节码（编译器生成）
// InnerClasses: 更通用（Kotlin 也支持）
String enclosingClassName = classModel.findAttribute(Attributes.nestHost())
    .map(attr -> attr.nestHost().asInternalName())
    .orElseGet(() -> resolveFromInnerClasses(classModel));  // fallback
```

**MetadataReaderFactory 策略选择（委托模式）：**
```java
// MetadataReaderFactoryDelegate —— 运行时判断使用哪个实现
// Java 24+ → ClassFileMetadataReaderFactory
// Java 17-23 → SimpleMetadataReaderFactory (ASM)
```

---

## 九、安全资源处理模式

### 15. ResourceHandlerUtils 防路径穿越

**场景：** ScriptTemplateView 加载模板资源时，需防止路径穿越攻击（`../../../etc/passwd`）。

```java
// 正确的资源加载模式
protected @Nullable Resource getResource(String location) {
    String normalizedLocation = ResourceHandlerUtils.normalizeInputPath(location);

    // 1. 过滤危险路径（.. 穿越、特殊字符等）
    if (ResourceHandlerUtils.shouldIgnoreInputPath(normalizedLocation)) {
        return null;
    }

    for (String path : this.resourceLoaderPaths) {
        Resource resource = context.getResource(path + normalizedLocation);
        try {
            // 2. 确认资源在允许的根目录下（防符号链接逃逸）
            if (resource.exists() &&
                ResourceHandlerUtils.isResourceUnderLocation(context.getResource(path), resource)) {
                return resource;
            }
        }
        catch (IOException ex) {
            // 3. 分级日志（debug 级别）
            if (logger.isTraceEnabled()) {
                logger.trace("Skip location due to error", ex);
            } else if (logger.isDebugEnabled()) {
                logger.debug("Skip location [" + normalizedLocation + "]: " + ex.getMessage());
            }
        }
    }
    return null;
}
```

**要点：**
- `normalizeInputPath` 规范化路径（去掉 `..`、多余 `/`）
- `shouldIgnoreInputPath` 过滤危险输入（空路径、特殊字符）
- `isResourceUnderLocation` 确保资源在声明的根路径下，防止符号链接绕过

---

## 十、关键路径速查

| 功能 | 路径 |
|------|------|
| 根 Gradle 构建 | `build.gradle` |
| Gradle 属性 | `gradle.properties` |
| 构建约定插件 | `buildSrc/src/main/java/org/springframework/build/` |
| Java 编译约定 | `.../build/JavaConventions.java` |
| Checkstyle 约定 | `.../build/CheckstyleConventions.java` |
| 测试约定 | `.../build/TestConventions.java` |
| 架构规则 | `.../build/architecture/ArchitectureRules.java` |
| Optional 依赖插件 | `.../build/optional/OptionalDependenciesPlugin.java` |
| Multi-Release 扩展 | `.../build/multirelease/MultiReleaseExtension.java` |
| Checkstyle 配置 | `src/checkstyle/checkstyle.xml` |
| nohttp 白名单 | `src/nohttp/allowlist.lines` |
| spring-core 构建 | `spring-core/spring-core.gradle` |
| spring-core Java24 源码 | `spring-core/src/main/java24/` |
