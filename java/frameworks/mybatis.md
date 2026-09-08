# MyBatis 基础知识结构速记

> 核心：**配置阶段把 SQL 解析成 `MappedStatement`；运行阶段由 `MapperProxy → SqlSession → Executor` 执行 SQL。**

# 1. 基础与核心组件

MyBatis 是 SQL Mapping Framework：SQL 主要由开发者编写，框架负责参数绑定、SQL 执行、结果映射、Mapper 动态代理和缓存。

| 组件 | 作用 |
| --- | --- |
| `Configuration` | 全局运行时配置，保存 Mapper、`MappedStatement`、TypeHandler、插件等 |
| `SqlSessionFactoryBuilder` | 解析配置并构建 `SqlSessionFactory`，主要用于启动阶段 |
| `SqlSessionFactory` | 创建 `SqlSession` |
| `SqlSession` | 数据库会话入口，负责查询、更新、事务和 `getMapper()` |
| `MapperProxy` | Mapper 接口动态代理，把 Mapper 方法转成 MyBatis 调用 |
| `MappedStatement` | 一条映射 SQL 的元数据：SQL、参数映射、结果映射、缓存配置等 |
| `Executor` | SQL 执行器，负责查询、更新、一级缓存等 |
| `StatementHandler` | 创建并操作 JDBC `Statement / PreparedStatement` |
| `ParameterHandler` | 处理 SQL 入参 |
| `ResultSetHandler` | 处理查询结果 |
| `TypeHandler` | Java Type ↔ JDBC Type |

`SqlSession` 不是线程安全对象，不应跨线程共享。

# 2. MyBatis 执行流程

## 初始化阶段

```text
mybatis-config.xml
Mapper XML / Annotation
        ↓
   Configuration
   ├─ MapperRegistry
   ├─ MappedStatement
   ├─ TypeHandler
   └─ Interceptor
        ↓
SqlSessionFactoryBuilder
        ↓
 SqlSessionFactory
```

Mapper XML / 注解中的 SQL 会在初始化时解析为 `MappedStatement` 并保存到 `Configuration`。statementId 通常是：

```text
Mapper 接口全限定名 + 方法名
例如：com.example.UserMapper.selectById
```

> `MappedStatement` 是提前解析好的 SQL 元数据，不是执行过程中“经过的一层执行器”。

## SQL 执行阶段

```java
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
User user = mapper.selectById(1L);
```

`getMapper()` 返回的是 `MapperProxy`，核心流程：

```text
mapper.selectById(1L)
        ↓
    MapperProxy
        ↓
     SqlSession
        │
        ├─ 根据 statementId
        │  从 Configuration 获取 MappedStatement
        ↓
      Executor
 (MappedStatement + 参数)
        ↓
 StatementHandler
        ↓
       JDBC
        ↓
    Database
        ↓
 ResultSetHandler
        ↓
   Java Object
```

核心关系：

```text
MapperProxy → 调用 SqlSession
SqlSession  → 找 MappedStatement，并调用 Executor
Executor    → 使用 MappedStatement 执行 SQL
```

因此不要记成：

```text
MapperProxy → Executor → MappedStatement
```

## Handler 与 TypeHandler

执行参数：

```text
Java 参数
→ ParameterHandler
→ TypeHandler
→ PreparedStatement
→ Database
```

处理结果：

```text
Database
→ ResultSet
→ ResultSetHandler
→ TypeHandler
→ Java Object
```

所以 `TypeHandler` 两边都会参与，而不是只处理查询结果。

# 3. Mapper、Executor 与 SQL 映射

Mapper 通常只有接口：

```java
public interface UserMapper {
    User selectById(Long id);
}
```

MyBatis 通过 `MapperProxy` 动态代理，因此无需手写实现类。

常见 Executor：

| Executor | 特点 |
| --- | --- |
| `SimpleExecutor` | 每次执行创建新的 Statement |
| `ReuseExecutor` | 复用 Statement |
| `BatchExecutor` | 批量执行 INSERT / UPDATE / DELETE |
| `CachingExecutor` | 装饰其他 Executor，处理二级缓存 |

`BaseExecutor` 提供一级缓存、查询/更新模板和事务调用等公共流程。

## `#{}` 与 `${}`

| 写法 | 原理 | 特点 |
| --- | --- | --- |
| `#{}` | 转成 `?`，通过 `PreparedStatement` 参数绑定 | 普通值优先使用，可避免普通参数位置的 SQL 注入 |
| `${}` | 直接字符串替换 | 可用于某些 SQL 结构位置，但有 SQL 注入风险 |

```xml
WHERE id = #{id}
```

变为：

```sql
WHERE id = ?
```

而 `ORDER BY ${column}` 会直接拼接 SQL。

> **值用 `#{}`；确实不能参数化的 SQL 结构位置才谨慎使用 `${}`，并做白名单校验。**

## ResultMap / 自动映射

`ResultMap` 用于数据库列到 Java 属性的复杂映射：

```xml
<resultMap id="userMap" type="User">
    <id property="id" column="user_id"/>
    <result property="name" column="user_name"/>
</resultMap>
```

开启 `mapUnderscoreToCamelCase` 后可自动完成 `user_name → userName`。

# 4. 延迟加载与缓存

## 延迟加载

```text
查询 User
→ 暂不查询 Orders
→ 调用 user.getOrders()
→ 再执行 Orders SQL
```

可用 `fetchType="lazy"` / `fetchType="eager"` 控制；全局 `lazyLoadingEnabled` 默认 `false`，当前默认延迟加载代理为 `JAVASSIST`。

延迟加载可能产生 N+1：

```text
查询 N 个 User → 1 次 SQL
逐个访问 Orders → N 次 SQL
总计 1 + N 次
```

因此应根据场景选择 JOIN、批量查询或 Lazy Loading。

## 一级缓存

一级缓存是 **SqlSession 级本地缓存**，默认 `localCacheScope=SESSION`。

```text
第一次相同查询 → 一级 miss → DB → 放入一级缓存
第二次相同查询 → 一级 hit
```

`update`、`commit`、`rollback`、`close`、`clearCache()` 等会清理或影响本地缓存。可以设为 `STATEMENT`，使其只在单条语句执行期间使用。

注意：一级缓存属于 `SqlSession`，不是线程级缓存；SESSION 模式可能返回缓存中的同一对象引用。

## 二级缓存

二级缓存是 **Mapper Namespace 级缓存**，可跨 `SqlSession` 共享，通常需要在 Mapper 中显式配置：

```xml
<cache/>
```

查询关系：

```text
CachingExecutor
→ 二级缓存
   ↓ miss
→ Executor
→ 一级缓存
   ↓ miss
→ DB
```

即：

```text
二级 → 一级 → DB
```

同一 Namespace 下写操作通常会刷新对应二级缓存。跨 Namespace、分布式环境和实时性要求可能带来一致性问题，因此实际项目也常用 Redis、Caffeine 或业务层缓存。

# 5. 动态 SQL 与插件

| 标签 | 作用 |
| --- | --- |
| `if` | 条件 SQL |
| `choose / when / otherwise` | 多分支条件 |
| `where` | 自动处理 WHERE 和前置 AND/OR |
| `set` | 动态 UPDATE，处理多余逗号 |
| `trim` | 自定义前后缀 |
| `foreach` | IN、批量操作 |
| `sql / include` | SQL 片段复用 |

```xml
<foreach collection="ids" item="id"
         open="(" separator="," close=")">
    #{id}
</foreach>
```

MyBatis `Interceptor` 可以拦截 `Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`，常用于分页、SQL 日志、性能监控、数据权限等。

# 6. MyBatis 与 Spring

Spring / Spring Boot 项目通常使用 MyBatis-Spring：

```java
@Mapper
public interface UserMapper {
}
```

或使用 `@MapperScan("com.example.mapper")` 批量扫描。

Spring 负责 DataSource、事务和 Mapper Bean，MyBatis 负责 SQL Mapping。`SqlSessionTemplate` 将 MyBatis `SqlSession` 与 Spring 事务体系结合，因此业务代码通常不需要手动 `openSession()` / `close()`。

# 7. 设计模式

| 模式 | MyBatis 中的体现 |
| --- | --- |
| Builder | `SqlSessionFactoryBuilder` |
| Factory | `SqlSessionFactory` |
| Proxy | `MapperProxy` |
| Decorator | Cache、`CachingExecutor` |
| Strategy | `TypeHandler`、不同 Executor |
| Template Method | `BaseExecutor` |

# 8. 速记

```text
【初始化】
配置 / Mapper
→ Configuration
→ 解析并保存 MappedStatement
→ SqlSessionFactoryBuilder
→ SqlSessionFactory
```

```text
【执行】
SqlSessionFactory
→ SqlSession
→ getMapper()
→ MapperProxy
→ SqlSession
→ 找 MappedStatement
→ Executor
→ StatementHandler
→ JDBC
→ ResultSetHandler
→ Java Object
```

```text
【参数 / 结果】
参数：Java → ParameterHandler → TypeHandler → JDBC
结果：JDBC → ResultSetHandler → TypeHandler → Java
```

```text
【缓存】
一级缓存 → SqlSession 级，默认 SESSION
二级缓存 → Mapper Namespace 级，通常需显式配置
查询顺序 → 二级 → 一级 → DB
```

一句话：

> **MapperProxy 把 Mapper 方法交给 SqlSession，SqlSession 根据 statementId 找到 MappedStatement，再委托 Executor 执行；MappedStatement 是 SQL 元数据，不是独立执行器。**
