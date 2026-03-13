# My Skills

AI Agent 技能知识库，整理常用开源项目的源码级实践与构建体系。

## 📚 技能列表

### 1. [Hugo + Docsy 文档站点技能](./hugo-docsy-skills.md)

基于 grpc.io 项目整理的 Hugo 静态站点生成器与 Docsy 主题使用技能。

**核心内容：**
- Hugo 项目结构与 URL 映射规则
- Docsy 主题继承机制（git submodule 管理）
- Front Matter 配置（weight、menu、draft、no_list）
- Shortcodes 自定义（alert、blocks、tabs、code）
- 多语言站点配置与 i18n
- Netlify 部署配置与 Hugo 版本管理
- 本地开发与预览构建

---

### 2. [Spring Boot & Spring Framework 技能](./springboot-skills.md)

基于 Spring Boot 与 Spring Framework 源码分析的核心机制整理。

**核心内容：**
- Bean 生命周期（三级缓存、循环依赖）
- ApplicationContext 刷新 12 步
- Configuration 类解析（@Import、@Bean、CGLIB 代理）
- AOP 代理选择（JDK 动态代理 vs CGLIB）
- 事务管理（@Transactional、传播行为、回滚规则）
- Spring Boot 自动配置机制（@Conditional、spring.factories）
- DispatcherServlet 请求处理流程
- Fat JAR 加载机制（JarLauncher、LaunchedClassLoader）
- 关键源码路径速查表

---

### 3. [Spring Framework 源码贡献与构建体系](./spring-framework-contrib-skills.md)

基于 Spring Framework 项目的 Gradle 构建系统与贡献实践整理。

**核心内容：**

**构建体系：**
- 根级 build.gradle 多项目结构与分组
- ConventionsPlugin 统一约定插件架构
- Java 编译约定（工具链、-Werror、-parameters）
- Checkstyle + Spring JavaFormat + nohttp 代码风格检查
- 测试约定（JUnit Platform、ByteBuddy Agent、CI 自动重试）

**关键设计模式：**
- Optional 依赖模式（Maven-style，不传递给下游）
- Multi-Release JAR（Java 17/21/24 多版本字节码同 JAR）
- Shadow JAR / 依赖重打包（Objenesis、JavaPoet 内嵌）
- ArchUnit 架构合规检查（禁止循环依赖、String 操作必须带 Locale）

**贡献规范：**
- Commit 信息格式与 DCO Signed-off-by 要求
- License Header、Javadoc、spring-jcl 日志、Nullability 注解规范
- 测试类命名约定（*Tests.java / *Test.java）

**高级模式：**
- ClassFile API vs ASM 双实现策略（Java 24 运行时自动切换）
- ResourceHandlerUtils 防路径穿越的安全资源加载模式

---

## 🎯 使用场景

这些技能文档适用于：

- **AI Agent 上下文增强** - 为 Claude Code、GitHub Copilot 等工具提供项目特定知识
- **快速上手开源项目** - 理解项目构建体系与核心机制
- **代码审查参考** - 了解项目代码规范与最佳实践
- **贡献者指南** - 掌握提交代码前的必要检查项

## 📝 文档结构

每个技能文档包含：

1. **核心机制** - 关键流程与实现原理
2. **代码示例** - 实际使用场景与代码片段
3. **源码路径** - 快速定位关键类与方法
4. **最佳实践** - 项目约定与注意事项
5. **常见问题** - 典型错误与解决方案

## 🔗 相关资源

- [Spring Framework](https://github.com/spring-projects/spring-framework)
- [Spring Boot](https://github.com/spring-projects/spring-boot)
- [Hugo](https://gohugo.io/)
- [Docsy Theme](https://www.docsy.dev/)
- [grpc.io](https://github.com/grpc/grpc.io)

## 📄 License

本仓库内容基于开源项目源码分析整理，遵循各项目原有 License。

- Spring Framework / Spring Boot: Apache License 2.0
- Hugo: Apache License 2.0
- Docsy: Apache License 2.0
