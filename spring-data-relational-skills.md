# Spring Data Relational 源码级知识整理

> 基于 `spring-projects/spring-data-relational` 源码分析（spring-data-jdbc / spring-data-r2dbc）

---

## 一、项目构建与测试

### 1. 构建体系

**构建工具：** Maven（带 Maven Wrapper），JDK 17+

```bash
# 完整构建（跳过外部数据库集成测试）
./mvnw clean install

# 构建文档（Antora）
./mvnw clean install -Pantora
# 输出：spring-data-jdbc-distribution/target/antora/site/index.html

# 运行单个模块测试
./mvnw test -pl spring-data-jdbc

# 运行单个测试类
./mvnw test -pl spring-data-jdbc -Dtest=JdbcAggregateTemplateIntegrationTests

# 运行单个测试方法
./mvnw test -pl spring-data-jdbc -Dtest=JdbcAggregateTemplateIntegrationTests#methodName

# 指定数据库运行集成测试（通过 Spring Profile）
./mvnw verify -pl spring-data-jdbc -Dspring.profiles.active=postgres
```

**Maven Profile：**

| Profile | 用途 |
|---------|------|
| `no-jacoco` | 跳过代码覆盖率 |
| `ignore-missing-license` | 跳过许可证检查 |
| `jmh` | 运行 JMH 基准测试 |
| `antora` | 生成 Antora 文档站点 |

### 2. 模块结构

```
spring-data-relational-parent (pom.xml 聚合器)
├── spring-data-relational          # 共享核心抽象
├── spring-data-jdbc                # JDBC 实现
├── spring-data-r2dbc               # R2DBC（响应式）实现
└── spring-data-jdbc-distribution   # 文档聚合与发布
```

### 3. 集成测试基础设施

**数据库类型枚举：** `DatabaseType`（在 `spring-data-jdbc` 测试包中）

```java
public enum DatabaseType {
    DB2, HSQL, H2, MARIADB, MYSQL, ORACLE, POSTGRES, SQL_SERVER("mssql");
}
```

**测试注解体系：**

| 注解 | 作用 |
|------|------|
| `@IntegrationTest` | 元注解，装配 Spring 测试上下文 |
| `@EnabledOnDatabase(DatabaseType.HSQL)` | 仅在指定数据库 Profile 下运行 |
| `@EnabledOnFeature` | 仅在数据库支持特定特性时运行 |

**测试配置选择机制：**
- `TestConfiguration` 根据 `spring.profiles.active` 选择 `DataSourceConfiguration`
- 每种数据库有独立配置类（如 `PostgresDataSourceConfiguration`、`HsqlDataSourceConfiguration`）
- 外部数据库（Postgres、MySQL 等）通过 **Testcontainers** 启动

---

## 二、核心设计理念：Aggregate-Oriented ORM

Spring Data Relational 是**聚合根导向**的 ORM，与 JPA 完全不同：

| 特性 | Spring Data Relational | JPA |
|------|----------------------|-----|
| 懒加载 | ❌ 不支持 | ✅ 支持 |
| 二级缓存 | ❌ 不支持 | ✅ 支持 |
| Write-behind | ❌ 不支持 | ✅ 支持 |
| 脏检查 | ❌ 不支持 | ✅ 支持 |
| 聚合根整体持久化 | ✅ 核心设计 | ⚠️ 可选 |

**关键约束：**
- 所有持久化操作针对**完整的聚合根**
- 聚合内的子实体通过**嵌入集合**（`@MappedCollection`）关联
- 跨聚合引用**仅通过 ID**，不通过对象引用
- 每次 `save()` 都会删除旧的子实体并重新插入

---

## 三、spring-data-relational 模块（共享核心）

### 4. 映射模型 (core/mapping)

**核心类层次：**

```
RelationalMappingContext
  ├── 管理 RelationalPersistentEntity（实体元数据）
  │     └── BasicRelationalPersistentEntity
  └── 管理 RelationalPersistentProperty（属性元数据）
        └── BasicRelationalPersistentProperty
```

**AggregatePath — 聚合路径导航：**

`AggregatePath` 表示从聚合根到某个属性的导航路径，是 SQL 生成的核心输入。

```
聚合根 Person
  ├── name (直接属性) → AggregatePath: Person.name
  ├── address (嵌入对象 @Embedded) → AggregatePath: Person.address
  │     └── city → AggregatePath: Person.address.city
  └── orders (集合 @MappedCollection) → AggregatePath: Person.orders
        └── item → AggregatePath: Person.orders.item
```

- `DefaultAggregatePath` — 标准实现
- `AggregatePathTraversal` — 路径遍历工具
- `AggregatePathTableUtils` — 路径到表/列名的映射

**命名策略：**
- `NamingStrategy` 接口 → `DefaultNamingStrategy`
- 默认将 camelCase 转为 snake_case（`firstName` → `first_name`）
- `CachingNamingStrategy` 缓存命名结果

**关键注解：**

| 注解 | 作用 |
|------|------|
| `@Table` | 指定表名 |
| `@Column` | 指定列名 |
| `@Id` | 标记主键 |
| `@Embedded` | 嵌入值对象（展开到同一表） |
| `@MappedCollection` | 一对多关系（子表有外键） |
| `@InsertOnlyProperty` | 仅在 INSERT 时写入 |
| `@Sequence` | 指定数据库序列 |

### 5. SQL AST (core/sql)

**抽象语法树，非字符串拼接。**

```
Visitable (接口)
  └── Segment (基类)
       ├── Select / Insert / Update / Delete — 语句类型
       ├── Table / Column / AsteriskFromTable — 表/列引用
       ├── Condition — 条件表达式
       │    ├── Comparison (=, !=, <, >, ...)
       │    ├── In / Between / Like / IsNull
       │    ├── AndCondition / OrCondition / Not
       │    └── NestedCondition
       ├── Join — JOIN 子句
       ├── OrderByField — 排序字段
       ├── LockOptions — 行锁选项
       └── Functions / AnalyticFunction — 函数表达式
```

**构建 SQL 的入口：**
- `StatementBuilder.select()` → `SelectBuilder` → `Select`
- `StatementBuilder.insert()` → `InsertBuilder` → `Insert`
- `StatementBuilder.update()` → `UpdateBuilder` → `Update`
- `StatementBuilder.delete()` → `DeleteBuilder` → `Delete`

**渲染：** `render/` 子包中的 `SqlRenderer` 将 AST 渲染为 SQL 字符串，基于 `Visitor` 模式遍历。

### 6. Dialect 方言体系 (core/dialect)

```
Dialect (接口)
  └── AbstractDialect (抽象基类)
       ├── H2Dialect
       ├── HsqlDbDialect
       ├── PostgresDialect
       ├── MySqlDialect
       ├── MariaDbDialect
       ├── SqlServerDialect
       ├── OracleDialect
       ├── Db2Dialect
       └── AnsiDialect (默认/回退)
```

**Dialect 控制的行为：**

| 方法 | 控制内容 |
|------|---------|
| `limit()` | LIMIT/OFFSET 语法（`LimitClause`） |
| `lock()` | 行锁语法（`LockClause`） |
| `getIdGeneration()` | 主键生成策略（自增 vs 序列） |
| `getArraySupport()` | 数组列支持（`ArrayColumns`） |
| `getInsertRenderContext()` | INSERT 语句渲染上下文 |
| `orderByNullHandling()` | NULL 排序行为 |

### 7. 聚合变更模型 (core/conversion)

**save/delete 操作被拆解为有序的 DbAction 列表：**

```
save(person)
  → RelationalEntityInsertWriter / RelationalEntityUpdateWriter
    → RootAggregateChange<Person>
        ├── DbAction.InsertRoot(person)
        ├── DbAction.Delete(orders)          # 先删旧子实体
        ├── DbAction.Insert(order1)          # 再插入新子实体
        └── DbAction.Insert(order2)
```

**DbAction 类型：**

| DbAction | 说明 |
|----------|------|
| `InsertRoot` | 插入聚合根 |
| `Insert` | 插入子实体 |
| `UpdateRoot` | 更新聚合根 |
| `Update` | 更新子实体 |
| `DeleteRoot` | 删除聚合根 |
| `Delete` | 删除子实体（按类型批量） |
| `DeleteAll` | 删除某类型全部子实体 |
| `AcquireLockRoot` | 获取乐观锁 |

**批量操作：** `BatchingAggregateChange` 将多个相同类型的 `DbAction` 合并为批处理。
- `SaveBatchingAggregateChange` — 批量保存
- `DeleteBatchingAggregateChange` — 批量删除

**WritingContext：** 在遍历聚合对象图时维护当前路径和父级引用关系。

### 8. 事件与回调 (core/mapping/event)

**生命周期事件：**

```
save() 流程：
  BeforeConvertEvent / BeforeConvertCallback
    → BeforeSaveEvent / BeforeSaveCallback
      → [执行 SQL]
        → AfterSaveEvent / AfterSaveCallback

delete() 流程：
  BeforeDeleteEvent / BeforeDeleteCallback
    → [执行 SQL]
      → AfterDeleteEvent / AfterDeleteCallback

load() 流程：
  [执行 SQL]
    → AfterConvertEvent / AfterConvertCallback
```

**Event vs Callback：**
- Event（`ApplicationEvent`）— 只读观察，不能修改实体
- Callback — 可以返回修改后的实体（如设置审计字段）

---

## 四、spring-data-jdbc 模块

### 9. JdbcAggregateTemplate — 主入口

**核心类：** `core/JdbcAggregateTemplate.java`

实现 `JdbcAggregateOperations` 接口，编排整个持久化流程：

```
template.save(entity)
  ├── 1. 触发 BeforeConvertCallback
  ├── 2. 判断 isNew → InsertWriter / UpdateWriter 生成 AggregateChange
  ├── 3. 触发 BeforeSaveCallback
  ├── 4. AggregateChangeExecutor 执行 AggregateChange
  │     └── 遍历 DbAction → DataAccessStrategy 逐条执行
  └── 5. 触发 AfterSaveCallback
```

### 10. DataAccessStrategy — 数据访问策略

```
DataAccessStrategy (接口)
  ├── DefaultDataAccessStrategy — 标准实现（N+1 查询）
  ├── SingleQueryDataAccessStrategy — 单 SQL JOIN 查询
  ├── SingleQueryFallbackDataAccessStrategy — 单查询 + 回退
  ├── CascadingDataAccessStrategy — 级联多策略
  └── DelegatingDataAccessStrategy — 委托模式
```

**DefaultDataAccessStrategy：**
- 使用 `SqlGenerator` 生成 SQL
- 通过 `NamedParameterJdbcTemplate` 执行
- 读取时先查聚合根，再递归查子实体（N+1）

**SingleQueryDataAccessStrategy：**
- 使用 `SingleQuerySqlGenerator`（在 `spring-data-relational` 的 `sqlgeneration` 包中）
- 通过 JOIN 一次查询加载整个聚合
- 由 `RowDocumentResultSetExtractor` 将扁平结果集重组为聚合对象图

### 11. SqlGenerator — SQL 生成

**核心类：** `core/convert/SqlGenerator.java`

为每个实体类型生成 CRUD SQL：

| 方法 | 生成的 SQL |
|------|-----------|
| `getInsert()` | `INSERT INTO ... VALUES (...)` |
| `getUpdate()` | `UPDATE ... SET ... WHERE id = :id` |
| `getDeleteById()` | `DELETE FROM ... WHERE id = :id` |
| `getFindOne()` | `SELECT ... FROM ... WHERE id = :id` |
| `getFindAll()` | `SELECT ... FROM ...` |
| `getExists()` | `SELECT COUNT(*) FROM ... WHERE id = :id` |

- `SqlGeneratorSource` — 缓存每个实体类型的 `SqlGenerator` 实例
- 使用 `SqlContext` 维护当前表/列的上下文

### 12. 类型转换

**核心类：** `core/convert/MappingJdbcConverter.java`

```
JdbcConverter (接口)
  └── MappingJdbcConverter
       ├── 读取：ResultSet → Entity（通过 EntityRowMapper）
       └── 写入：Entity → SQL 参数
```

**EntityRowMapper：** 标准 Spring `RowMapper<T>` 实现，委托给 `MappingJdbcConverter`。

**QueryMapper：** 将 Spring Data 的 `Query`/`Criteria` 对象转换为 SQL AST 中的 `Condition`。

### 13. Repository 支持

```
JdbcRepositoryFactory
  └── 创建 Repository 代理
       ├── SimpleJdbcRepository — CRUD 方法实现
       └── JdbcQueryLookupStrategy — 查询方法解析
            ├── @Query 注解 → StringBasedJdbcQuery
            └── 方法名推导 → PartTreeJdbcQuery
```

**配置入口：** `@EnableJdbcRepositories` → `JdbcRepositoriesRegistrar` → `JdbcRepositoryConfigExtension`

### 14. MyBatis 集成

**包：** `mybatis/`

- `MyBatisDataAccessStrategy` — 实现 `DataAccessStrategy`，委托给 MyBatis `SqlSession`
- 可与 `DefaultDataAccessStrategy` 通过 `CascadingDataAccessStrategy` 组合使用

---

## 五、spring-data-r2dbc 模块

### 15. R2dbcEntityTemplate — 响应式主入口

镜像 `JdbcAggregateTemplate` 的设计，但返回 `Mono<T>` / `Flux<T>`：

```java
Mono<Person> saved = template.insert(person);
Flux<Person> people = template.select(Person.class).all();
Mono<Person> one = template.selectOne(query, Person.class);
```

### 16. R2DBC 与 JDBC 差异

| 方面 | JDBC | R2DBC |
|------|------|-------|
| 返回类型 | 阻塞式 `T` / `List<T>` | 响应式 `Mono<T>` / `Flux<T>` |
| 模板类 | `JdbcAggregateTemplate` | `R2dbcEntityTemplate` |
| 数据访问 | `NamedParameterJdbcTemplate` | `DatabaseClient` |
| 行映射 | `EntityRowMapper`（RowMapper） | `EntityRowMapper`（BiFunction） |
| MyBatis | ✅ 支持 | ❌ 不支持 |

**共享部分（spring-data-relational）：**
- SQL AST 和渲染
- Dialect 方言体系
- 映射模型（`RelationalMappingContext`）
- 聚合变更模型（`AggregateChange`）
- 事件/回调机制
- 命名策略

---

## 六、SQL 生成全链路示例

以 `save(person)` 为例，完整的 SQL 生成路径：

```
1. JdbcAggregateTemplate.save(person)
2. RelationalEntityInsertWriter 生成 AggregateChange
     └── DbAction.InsertRoot(person)
3. AggregateChangeExecutor 遍历 DbAction
4. DefaultDataAccessStrategy.insert(person, Person.class)
5. SqlGenerator.getInsert() — 使用 SQL AST 构建
     ├── 从 RelationalPersistentEntity 获取表名、列名
     ├── StatementBuilder.insert(Table.create("person"))
     │     .columns(Column.create("first_name"), Column.create("last_name"))
     │     .values(...)
     │     .build()
     └── SqlRenderer.render(insert) → "INSERT INTO person (first_name, last_name) VALUES (:first_name, :last_name)"
6. NamedParameterJdbcTemplate.update(sql, parameterSource)
```

---

## 七、关键源码路径速查

| 功能 | 路径 |
|------|------|
| 映射上下文 | `spring-data-relational/src/main/java/.../relational/core/mapping/RelationalMappingContext.java` |
| 聚合路径 | `spring-data-relational/src/main/java/.../relational/core/mapping/AggregatePath.java` |
| SQL AST 入口 | `spring-data-relational/src/main/java/.../relational/core/sql/StatementBuilder.java` |
| SQL 渲染 | `spring-data-relational/src/main/java/.../relational/core/sql/render/SqlRenderer.java` |
| Dialect 接口 | `spring-data-relational/src/main/java/.../relational/core/dialect/Dialect.java` |
| 聚合变更 | `spring-data-relational/src/main/java/.../relational/core/conversion/AggregateChange.java` |
| DbAction | `spring-data-relational/src/main/java/.../relational/core/conversion/DbAction.java` |
| 生命周期事件 | `spring-data-relational/src/main/java/.../relational/core/mapping/event/` |
| 单查询 SQL 生成 | `spring-data-relational/src/main/java/.../relational/core/sqlgeneration/SingleQuerySqlGenerator.java` |
| JdbcAggregateTemplate | `spring-data-jdbc/src/main/java/.../jdbc/core/JdbcAggregateTemplate.java` |
| DataAccessStrategy | `spring-data-jdbc/src/main/java/.../jdbc/core/convert/DataAccessStrategy.java` |
| JDBC SqlGenerator | `spring-data-jdbc/src/main/java/.../jdbc/core/convert/SqlGenerator.java` |
| MappingJdbcConverter | `spring-data-jdbc/src/main/java/.../jdbc/core/convert/MappingJdbcConverter.java` |
| Repository 工厂 | `spring-data-jdbc/src/main/java/.../jdbc/repository/support/JdbcRepositoryFactory.java` |
| R2dbcEntityTemplate | `spring-data-r2dbc/src/main/java/.../r2dbc/core/R2dbcEntityTemplate.java` |
| 测试注解 | `spring-data-jdbc/src/test/java/.../jdbc/testing/IntegrationTest.java` |
| 数据库类型 | `spring-data-jdbc/src/test/java/.../jdbc/testing/DatabaseType.java` |
