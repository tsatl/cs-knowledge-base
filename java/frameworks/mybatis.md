# MyBatis 基础知识结构速记
> MyBatis 主线：

```text
配置
 ↓
SqlSessionFactory
 ↓
SqlSession
 ↓
MapperProxy
 ↓
Executor
 ↓
MappedStatement
 ↓
StatementHandler
 ↓
JDBC
 ↓
ResultSetHandler
 ↓
Java Object
```

MyBatis 的核心：把 Java 方法、映射到 SQL、把 Java 参数、映射到 JDBC 参数、把 ResultSet、映射成 Java Object

# MyBatis 基础
## MyBatis 解决什么问题
直接 JDBC：加载驱动、获取 Connection、创建 PreparedStatement、设置参数、执行 SQL、处理 ResultSet、关闭资源

大量代码重复。

MyBatis：SQL → 自己控制；JDBC 模板代码；参数映射；结果映射；Mapper Proxy；缓存

由框架完成。

## MyBatis vs ORM
MyBatis 常被称为：半自动 ORM / SQL Mapping Framework

特点：SQL 自己写、映射框架帮你做

相比完全自动 ORM：SQL 可控性更高

# 核心组件
主要：

```text
Configuration
SqlSessionFactory
SqlSession
MapperProxy
Executor
MappedStatement
StatementHandler
ParameterHandler
ResultSetHandler
TypeHandler
```

## Configuration
Configuration：MyBatis 全局配置对象

里面维护：

```text
环境配置
Mapper Registry
MappedStatement
TypeHandler
Interceptor
Cache
Alias
Settings
```

可以理解：

> **Configuration = MyBatis 的运行时总配置中心。**

## SqlSessionFactoryBuilder
负责：读取 MyBatis 配置、构建 SqlSessionFactory

典型：`Builder`

使用完成后通常：无需长期保存

## SqlSessionFactory
作用：创建 SqlSession

通常一个应用环境：维护一个 SqlSessionFactory

## SqlSession
SqlSession：MyBatis 与数据库交互的会话对象

提供：

```text
selectOne
selectList
insert
update
delete
commit
rollback
getMapper
```

注意：SqlSession、不是线程安全对象

不要跨线程共享。

## Mapper
通常定义：

```java
public interface UserMapper {
    User selectById(Long id);
}
```

我们没有自己写实现类。

MyBatis：运行时生成 Mapper Proxy

## MapperProxy
调用：userMapper.selectById(1L);

实际上：

```text
Mapper Proxy
   ↓
识别 Mapper Method
   ↓
找到 MappedStatement
   ↓
调用 SqlSession
   ↓
Executor
```

所以：Mapper Interface、不需要手写实现类

依赖：动态代理

# MyBatis 执行流程
完整主线：

```text
mybatis-config.xml
Mapper XML / Annotation
      ↓
Configuration
      ↓
SqlSessionFactoryBuilder
      ↓
SqlSessionFactory
      ↓
SqlSession
      ↓
getMapper()
      ↓
MapperProxy
      ↓
Executor
      ↓
MappedStatement
      ↓
StatementHandler
      ↓
ParameterHandler
      ↓
JDBC PreparedStatement
      ↓
Database
      ↓
ResultSet
      ↓
ResultSetHandler
      ↓
TypeHandler
      ↓
Java Object
```

## 1. 加载配置
读取：mybatis-config.xml、Mapper XML、Mapper Annotation

解析后存进：`Configuration`

## 2. 构建 SqlSessionFactory
```text
Configuration
   ↓
SqlSessionFactoryBuilder
   ↓
SqlSessionFactory
```

## 3. 创建 SqlSession
```text
SqlSessionFactory
   ↓
openSession()
   ↓
SqlSession
```

SqlSession 内部持有 / 使用：`Executor`

## 4. 获取 Mapper
```java
UserMapper mapper = sqlSession.getMapper(UserMapper.class);
```

实际上创建：`MapperProxy`

## 5. 找到 MappedStatement
每条 Mapper SQL：`namespace + statement id`

最终对应：`MappedStatement`

例如：`com.example.UserMapper.selectById`

MappedStatement 封装：SQL 类型、SQL Source、参数映射、结果映射、Cache、Statement 配置

## 6. Executor
Executor：SQL 执行器

负责：查询、更新、一级缓存、事务相关调用、批处理

常见：SimpleExecutor、ReuseExecutor、BatchExecutor、CachingExecutor

## BaseExecutor
BaseExecutor：Executor 基础抽象实现

负责通用流程：一级缓存、query/update 模板、事务调用

具体执行细节：交给子类

这体现：模板方法思想

## CachingExecutor
CachingExecutor：对其他 Executor 进行包装

主要：处理二级缓存

流程：

```text
二级缓存
 ↓ miss
委托内部 Executor
 ↓
一级缓存 / DB
```

## 7. StatementHandler
负责：创建和操作 JDBC Statement

包括：prepare、parameterize、query、update

常见：`PreparedStatement`

最终由这里进入 JDBC。

## 8. ParameterHandler
负责：把 Java 参数、设置进 PreparedStatement

例如：

```sql
SELECT *
FROM user
WHERE id = ?
```

ParameterHandler：把 1L、绑定到 ?

## 9. ResultSetHandler
负责：处理 JDBC ResultSet

转换：数据库行 → Java Object

## 10. TypeHandler
负责：Java Type、↔、JDBC Type

例如：String、↔ VARCHAR、Integer、↔ INTEGER、LocalDateTime、↔ TIMESTAMP

# 参数映射
## #{}
例如：

```xml
SELECT *
FROM user
WHERE id = #{id}
```

MyBatis 处理后：

```sql
SELECT *
FROM user
WHERE id = ?
```

然后：PreparedStatement + 参数绑定

特点：参数和值分离、安全、可以防止普通参数位置的 SQL Injection

普通参数：优先使用 #{}

## ${}
例如：`ORDER BY ${column}`

MyBatis 会：直接字符串替换

不会变成：`?`

特点：灵活、但有 SQL Injection 风险

如果值来自用户：必须严格白名单校验

## #{} vs ${}
```text
#{}
→ PreparedStatement
→ ?
→ 参数绑定
→ 安全

${}
→ 字符串拼接
→ 原样替换
→ 有注入风险
```

一句话：

> **值用 `#{}`，确实无法参数化的 SQL 结构位置才谨慎考虑 `${}`。**

# ResultMap
ResultMap：控制 ResultSet、如何映射成 Java Object

适合：列名与属性名不同、复杂对象、一对一、一对多、嵌套映射

例如：

```xml
<resultMap id="userMap" type="User">
    <id property="id" column="user_id"/>
    <result property="name" column="user_name"/>
</resultMap>
```

# 自动映射
简单场景：column name、与、property name、匹配

MyBatis 可以自动映射。

如果数据库：`user_name`

Java：`userName`

可配置：`mapUnderscoreToCamelCase`

完成：下划线 → 驼峰

# 延迟加载
## 定义
立即加载：查询 User、同时查询 Orders

延迟加载：查询 User、暂时不查 Orders、真正调用：、user.getOrders()、时、再执行 Orders SQL

## fetchType
关联映射中可以设置：`fetchType="lazy"`

用于：延迟加载

或：`fetchType="eager"`

立即加载。

## 全局配置
`lazyLoadingEnabled`

控制全局 Lazy Loading。

当前 MyBatis：默认 false

具体 association / collection：`fetchType`

可以覆盖全局策略。

# 延迟加载原理
核心：

```text
查询主对象
   ↓
关联属性暂不加载
   ↓
MyBatis 返回带延迟加载能力的代理对象
   ↓
调用 getOrders()
   ↓
Proxy 拦截
   ↓
ResultLoader
   ↓
执行关联 SQL
   ↓
填充属性
   ↓
返回结果
```

## Lazy Loading Proxy
不要简单记：一定使用 JDK Proxy

当前 MyBatis 默认：`Javassist ProxyFactory`

用于生成：支持延迟加载的对象代理

历史 CGLIB ProxyFactory：已经被标记为 deprecated

# N+1 问题
Lazy Loading 容易出现：

```text
先查 100 个 User
   ↓
遍历每个 User
   ↓
每个 User 再查一次 Orders
```

结果：1 次主查询 + N 次子查询

即：`N + 1`

所以：延迟加载、不是永远更好

需要根据：访问模式、数据量、查询复杂度

决定：JOIN、批量查询、Lazy Loading

# MyBatis 缓存
MyBatis 有：一级缓存、二级缓存

# 一级缓存
一级缓存：Local Cache、SqlSession 级

不是：线程级缓存

每创建一个 SqlSession：都会创建本地缓存

默认：`localCacheScope = SESSION`

## 一级缓存流程
第一次：

```text
SELECT
   ↓
一级缓存 miss
   ↓
Database
   ↓
Result
   ↓
放一级缓存
```

第二次相同：同一 SqlSession + 相同 Statement + 相同参数

可能：直接命中一级缓存

## 一级缓存作用域
默认：`SESSION`

可以配置：`STATEMENT`

STATEMENT：缓存只在单条 Statement 执行期间生效

一级缓存：无法完全关闭

因为 MyBatis 内部某些机制也依赖 Local Cache。

## 一级缓存清理
常见：update、commit、rollback、close、clearCache()

会清理 / 影响 Local Cache。

## 一级缓存注意
SESSION 模式下：缓存中保存的是对象引用

再次查询：可能拿到相同对象引用

所以：不要随意修改缓存返回对象

否则：可能影响同 Session 后续查询结果

# 二级缓存
二级缓存：Mapper Namespace 级

不是简单：整个 SqlSessionFactory 一个大 Map

多个 SqlSession：只要属于同一个 Namespace

就可能共享对应二级缓存。

## 二级缓存默认状态
默认：只有一级本地缓存直接可用

Mapper 二级缓存需要显式配置，例如：`<cache/>`

## 二级缓存流程
整体：

```text
Mapper Query
   ↓
CachingExecutor
   ↓
二级缓存
   │
   ├── hit
   │   ↓
   │ 返回
   │
   └── miss
       ↓
内部 Executor
       ↓
一级缓存
       │
       ├── hit
       │   ↓
       │ 返回
       │
       └── miss
           ↓
        Database
```

可以记：

```text
二级
 ↓
一级
 ↓
数据库
```

## 二级缓存写入时机
二级缓存具有事务语义。

查询结果：不是一查询出来、就立刻对所有 Session 可见

通常需要：Session commit / close 等事务边界

之后才进入可共享状态。

## 缓存清理
同一 Mapper Namespace 下：INSERT、UPDATE、DELETE

默认可能：刷新对应二级缓存

防止：长期读取旧数据

## 二级缓存序列化
不要简单记：所有二级缓存对象、必须 Serializable

默认 read-write 语义下：缓存通常需要通过序列化复制对象

此时对象：需要支持 Serializable

如果配置：`readOnly=true`

则语义不同：可以共享对象引用

调用方：不应该修改返回对象

# PerpetualCache
MyBatis 默认基础 Cache 实现：`PerpetualCache`

内部核心：`HashMap`

其他缓存能力通过：`Decorator`

层层包装。

例如：

```text
PerpetualCache
      ↓
LruCache
      ↓
SerializedCache
      ↓
SynchronizedCache
      ↓
TransactionalCache
```

具体组合取决于：配置和运行过程

# MyBatis 设计模式
## 工厂模式
`SqlSessionFactory`

负责：创建 SqlSession

## Builder
`SqlSessionFactoryBuilder`

负责：解析配置、构建 SqlSessionFactory

## 代理模式
Mapper：只有接口、没有实现类

MyBatis：`MapperProxy`

生成动态代理。

流程：

```text
Mapper Method
   ↓
MapperProxy
   ↓
SqlSession
   ↓
Executor
```

## 装饰器模式
MyBatis Cache：一个 Cache、包装另一个 Cache

例如：

```text
PerpetualCache
   ↓
LruCache
   ↓
SerializedCache
   ↓
SynchronizedCache
```

每层：增加一种功能

这是：`Decorator Pattern`

比简单说：责任链

更典型。

## 策略模式
例如：`TypeHandler`

不同 Java / JDBC 类型：使用不同 TypeHandler

也可以把不同 Executor：Simple、Reuse、Batch

理解为策略选择。

## 模板方法模式
BaseExecutor：定义查询 / 更新总体流程

具体细节：由子类实现

例如：doQuery()、doUpdate()

体现：`Template Method`

# Executor
## SimpleExecutor
每次 SQL：创建新的 Statement

使用简单直接。

## ReuseExecutor
会：复用 Statement

减少重复创建开销。

## BatchExecutor
主要：批量执行更新

适合大量：INSERT、UPDATE、DELETE

## CachingExecutor
主要：二级缓存

本质：装饰其他 Executor

# MyBatis 插件
MyBatis 提供：`Interceptor`

可以拦截特定核心组件的方法。

典型：Executor、StatementHandler、ParameterHandler、ResultSetHandler

可用于：分页、SQL 日志、性能监控、数据权限

# Mapper XML 常见标签
```text
select
insert
update
delete
resultMap
sql
include
if
choose
when
otherwise
where
set
trim
foreach
```

## if
```xml
<if test="name != null">
    AND name = #{name}
</if>
```

用于动态 SQL。

## where
自动处理：WHERE、AND / OR

前缀问题。

## set
用于：动态 UPDATE

自动处理多余逗号。

## foreach
用于：IN、批量操作

例如：

```xml
WHERE id IN
<foreach collection="ids"
         item="id"
         open="("
         separator=","
         close=")">
    #{id}
</foreach>
```

# MyBatis 与 Spring
实际 Spring Boot 项目常使用：`MyBatis-Spring`

Spring：管理 DataSource、管理 Transaction、管理 Mapper Bean

MyBatis：负责 SQL Mapping

## Mapper Bean
常见：

```java
@Mapper
public interface UserMapper {
}
```

或者：@MapperScan("com.example.mapper")

批量扫描 Mapper。

## SqlSessionTemplate
MyBatis-Spring 中常见：`SqlSessionTemplate`

它负责：把 MyBatis SqlSession、与 Spring 事务体系结合

因此业务代码通常：不需要自己 openSession / close

# 常见问题
## 为什么 Mapper 接口没有实现类也能运行
因为：MyBatis → 为 Mapper Interface 创建 MapperProxy

方法调用被代理：

```text
Mapper Method
 ↓
MappedStatement
 ↓
Executor
 ↓
SQL
```

## 为什么一级缓存不是线程级
因为一级缓存：属于 SqlSession

不是：`ThreadLocal Cache`

虽然很多框架使用时：一个请求 / 事务、对应一个会话上下文

但概念上：SqlSession Scope、≠ Thread Scope

## 为什么修改数据后缓存会清理
如果：UPDATE 后、仍保留旧 SELECT Cache

就会：读取脏旧数据

因此写操作会：触发对应缓存刷新策略

## 二级缓存为什么谨慎使用
风险：跨 Namespace 更新、缓存一致性、分布式环境、数据实时性

例如：UserMapper、缓存 User、OrderMapper、更新了与 User 相关数据

如果缓存边界没设计好：可能出现数据不一致

因此现代项目：不一定依赖 MyBatis 二级缓存

可能使用：Redis、Caffeine、业务层缓存

统一管理。

# 速记
## 执行流程
```text
Configuration
 ↓
SqlSessionFactory
 ↓
SqlSession
 ↓
MapperProxy
 ↓
Executor
 ↓
MappedStatement
 ↓
StatementHandler
 ↓
ParameterHandler
 ↓
JDBC
 ↓
ResultSetHandler
 ↓
TypeHandler
 ↓
Object
```

## Mapper Proxy
```text
Mapper Interface
→ 无实现类

getMapper()
→ MapperProxy

调用方法
→ 找 MappedStatement
→ 执行 SQL
```

## #{} / ${}
```text
#{}
→ ?
→ PreparedStatement
→ 参数绑定
→ 安全
```

${} → 字符串替换，SQL Injection 风险

## Lazy Loading
```text
主对象先查
 ↓
关联属性未加载
 ↓
访问 Getter
 ↓
代理触发 SQL
 ↓
加载关联数据
```

当前默认代理：`Javassist`

## 一级缓存
SqlSession 级、默认 SESSION

清理：update、commit、rollback、close、clearCache

## 二级缓存
Mapper Namespace 级

需要：`<cache/>`

查询：

```text
二级
 ↓ miss
一级
 ↓ miss
DB
```

## 设计模式
```text
Builder
→ SqlSessionFactoryBuilder

Factory
→ SqlSessionFactory

Proxy
→ MapperProxy

Decorator
→ Cache

Strategy
→ TypeHandler / Executor

Template Method
→ BaseExecutor
```

# 一句话总结
```text
SqlSessionFactory
→ 创建会话

SqlSession
→ 数据库会话入口

MapperProxy
→ 把接口方法转成 MyBatis 调用

MappedStatement
→ 一条 SQL 的完整描述

Executor
→ 执行 SQL + 一级缓存

CachingExecutor
→ 二级缓存

StatementHandler
→ 操作 JDBC Statement

ParameterHandler
→ Java 参数 → JDBC

ResultSetHandler
→ ResultSet → Object

TypeHandler
→ Java Type ↔ JDBC Type
```
