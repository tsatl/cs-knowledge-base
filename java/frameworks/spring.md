# Spring 基础知识结构速记

> Spring 主线：

```text
IOC
→ 对象交给容器管理

DI
→ 容器给对象注入依赖

Bean 生命周期
→ Bean 从定义、创建、增强到销毁

AOP
→ 通过代理给方法增加公共逻辑

Transaction
→ AOP + TransactionManager

Spring MVC
→ Web 请求分发

Spring Boot
→ 自动配置 + 条件装配
```

---

# Spring IOC

## IOC

IOC：

```text
Inversion of Control
控制反转
```

传统方式：

```text
对象自己创建依赖
```

例如：

```java
UserService service = new UserService(new UserDao());
```

IOC：

```text
对象的创建
依赖关系
生命周期
```

交给：

```text
Spring Container
```

管理。

一句话：

> **IOC = 对象由谁创建、谁管理，从程序自己控制，反转为 Spring 容器控制。**

---

## DI

DI：

```text
Dependency Injection
依赖注入
```

IOC 是：

```text
思想
```

DI 是：

```text
IOC 最主要的实现方式
```

关系：

```text
IOC
 ↓
容器创建 Bean
 ↓
发现 Bean 的依赖
 ↓
DI
 ↓
把依赖注入 Bean
```

例如：

```java
@Service
public class UserService {

    private final UserDao userDao;

    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

`UserDao`：

```text
不是 UserService 自己 new
```

而是：

```text
Spring Container
→ 找到 UserDao Bean
→ 注入 UserService
```

---

## Spring IOC 如何实现

核心流程：

```text
配置 / 注解
   ↓
BeanDefinition
   ↓
BeanFactory / ApplicationContext
   ↓
实例化 Bean
   ↓
依赖注入
   ↓
初始化
   ↓
BeanPostProcessor
   ↓
完整 Bean / Proxy
```

底层可能使用：

```text
反射
工厂模式
代理
BeanPostProcessor
```

但：

```text
IOC ≠ 反射
```

反射只是实现 Bean 创建、属性设置、方法调用等功能的重要技术之一。

---

## BeanFactory

BeanFactory：

```text
Spring IOC 容器的基础接口
```

主要负责：

```text
Bean 创建
Bean 获取
依赖管理
生命周期管理
```

常见方法：

```java
getBean(...)
```

---

## ApplicationContext

ApplicationContext：

```text
BeanFactory 的高级容器体系
```

除了 IOC：

```text
Bean 管理
```

还提供：

```text
事件发布
国际化
资源加载
环境配置
AOP 集成
企业级扩展
```

通常开发中：

```text
直接使用 ApplicationContext
```

---

# Bean

## BeanDefinition

Spring 不会看到一个类就立刻创建对象。

首先把 Bean 的定义信息解析为：

```text
BeanDefinition
```

BeanDefinition 可以理解为：

> **Spring 创建 Bean 的“说明书”。**

里面记录：

```text
Bean Class
Scope
是否 Lazy
构造参数
属性值
初始化方法
销毁方法
依赖关系
```

流程：

```text
@Component / @Bean / XML
        ↓
   BeanDefinition
        ↓
 BeanDefinitionRegistry
        ↓
      Container
        ↓
     创建 Bean
```

---

## Bean 的获取

Spring 中：

```text
Bean Name
      ↓
BeanDefinition
      ↓
创建 / 获取实例
```

单例 Bean 最终通常保存在：

```text
singletonObjects
```

中。

---

# Bean 注入方式

主要：

```text
构造器注入
Setter 注入
字段注入
```

---

## 构造器注入

```java
@Service
public class UserService {

    private final UserDao userDao;

    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

优点：

```text
依赖明确
适合必需依赖
可以使用 final
方便测试
对象创建完成即处于完整状态
```

通常：

```text
优先推荐
```

---

## Setter 注入

```java
@Service
public class UserService {

    private UserDao userDao;

    @Autowired
    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

适合：

```text
可选依赖
需要后续修改的依赖
```

---

## 字段注入

```java
@Autowired
private UserDao userDao;
```

优点：

```text
代码简单
```

缺点：

```text
依赖不够显式
不利于不可变设计
单元测试不方便
```

---

# @Autowired

@Autowired：

```text
Spring 提供
```

主要：

```text
按类型查找 Bean
```

流程：

```text
字段类型
 ↓
Spring Container
 ↓
找到候选 Bean
```

如果：

```text
只有 1 个
→ 直接注入
```

如果：

```text
多个
→ 进一步根据 @Primary / @Qualifier / 名称等判断
```

例如：

```java
@Autowired
@Qualifier("mysqlUserDao")
private UserDao userDao;
```

---

# @Resource

@Resource：

```text
Jakarta / Java 标准注解体系
```

现代 Spring 常见：

```java
jakarta.annotation.Resource
```

主要特点：

```text
默认更偏向按名称匹配
找不到时再进行类型匹配
```

例如：

```java
@Resource
private UserDao userDao;
```

---

## @Autowired vs @Resource

```text
@Autowired
→ Spring
→ 主要按类型

@Resource
→ Jakarta 标准
→ 默认更偏按名称
```

如果有多个同类型 Bean：

```text
@Autowired + @Qualifier
```

是常见解决方式。

---

# Bean Scope

常见：

```text
singleton
prototype
request
session
application
websocket
```

最核心：

```text
singleton
prototype
```

---

## singleton

默认 Scope：

```text
singleton
```

含义：

> **同一个 Spring IOC Container 中，一个 BeanDefinition 通常只有一个共享实例。**

注意：

```text
Spring Singleton
≠
JVM 全局绝对只有一个实例
```

不同容器：

```text
可以有不同实例
```

---

## prototype

```text
每次 getBean
或发生依赖解析
都可以创建新实例
```

Spring 主要负责：

```text
创建
初始化
```

但通常：

```text
不会像 singleton 一样完整管理 prototype Bean 的销毁
```

---

# Bean 线程安全

重要：

```text
singleton
≠
线程安全
```

线程安全取决于：

```text
Bean 内部
有没有共享可变状态
```

---

## 无状态 Bean

例如：

```java
@Service
public class UserService {

    public User getById(Long id) {
        // 局部变量
        return ...
    }
}
```

没有：

```text
共享可变成员变量
```

通常：

```text
可以被多个线程安全使用
```

Service / DAO 常设计为：

```text
无状态
```

---

## 有状态 Bean

```java
@Service
public class UserService {
    private int count;
}
```

多个线程：

```text
同时修改 count
```

就可能产生：

```text
线程安全问题
```

解决思路：

```text
避免共享可变状态
使用线程安全结构
合理同步
重新设计作用域
```

不是简单：

```text
改 prototype
就一定解决所有并发问题
```

一句话：

> **Spring 只管理 Bean 生命周期，不替你的可变成员变量自动保证线程安全。**

---

# Bean 生命周期

## 主流程

```text
BeanDefinition
   ↓
实例化
   ↓
属性填充 / DI
   ↓
Aware 回调
   ↓
BeanPostProcessor.before
   ↓
初始化方法
   ↓
BeanPostProcessor.after
   ↓
AOP Proxy 等增强
   ↓
Bean 可使用
   ↓
容器关闭
   ↓
销毁
```

---

## 1. 实例化

Spring 根据：

```text
BeanDefinition
```

创建对象。

可能使用：

```text
构造器
反射
工厂方法
```

---

## 2. 属性填充

例如：

```text
@Autowired
@Resource
Setter
```

在这一阶段完成依赖注入。

经典源码学习中常看到：

```text
populateBean()
```

---

## 3. Aware

如果 Bean 实现：

```text
BeanNameAware
BeanFactoryAware
ApplicationContextAware
...
```

Spring 会把容器相关对象回调给 Bean。

---

## 4. BeanPostProcessor Before

```text
postProcessBeforeInitialization()
```

在初始化方法之前处理 Bean。

---

## 5. 初始化

常见顺序理解：

```text
@PostConstruct
   ↓
InitializingBean.afterPropertiesSet()
   ↓
自定义 init-method
```

不要过度依赖具体细节顺序做业务逻辑。

---

## 6. BeanPostProcessor After

```text
postProcessAfterInitialization()
```

AOP：

```text
代理对象生成
```

通常与 BeanPostProcessor 体系密切相关。

---

## 7. 销毁

容器关闭时，singleton Bean 可能执行：

```text
@PreDestroy
   ↓
DisposableBean.destroy()
   ↓
自定义 destroy-method
```

---

# 循环依赖

## 什么是循环依赖

```text
A
→ 依赖 B

B
→ 依赖 A
```

即：

```text
A → B → A
```

---

## 构造器循环依赖

例如：

```java
class A {
    A(B b) {}
}

class B {
    B(A a) {}
}
```

流程：

```text
创建 A
→ 构造器必须先有 B

创建 B
→ 构造器必须先有 A
```

此时：

```text
A、B 都还没有实例化完成
```

没有对象可以提前暴露。

所以经典情况下：

```text
构造器循环依赖
→ 无法通过三级缓存直接解决
```

可能出现：

```text
BeanCurrentlyInCreationException
```

---

## Setter / Field 循环依赖

经典 singleton 场景：

```text
A 已经完成实例化
但还没有完成属性注入
```

此时 Spring 可以：

```text
提前暴露 A 的引用
```

供 B 注入。

---

# 三级缓存

经典 Spring 单例缓存：

```text
一级：
singletonObjects

二级：
earlySingletonObjects

三级：
singletonFactories
```

---

## 一级缓存

```text
singletonObjects
```

保存：

```text
完全初始化完成的 singleton Bean
```

最终正常 Bean 都进入一级缓存。

---

## 二级缓存

```text
earlySingletonObjects
```

保存：

```text
已经生成的 Early Bean Reference
```

也就是：

```text
提前暴露的 Bean / Proxy 引用
```

---

## 三级缓存

```text
singletonFactories
```

保存：

```text
ObjectFactory
```

它可以：

```text
按需生成 Early Reference
```

为什么不直接只用二级缓存：

```text
因为某些 Bean
需要考虑 AOP Proxy
```

三级缓存可以：

```text
延迟决定
到底暴露原对象还是代理相关引用
```

---

## A ↔ B 经典流程

```text
创建 A
   ↓
A 实例化完成
   ↓
A 的 ObjectFactory 放三级缓存
   ↓
A 需要 B
   ↓
创建 B
   ↓
B 需要 A
   ↓
一级缓存没有 A
   ↓
二级缓存没有 A
   ↓
三级缓存找到 A Factory
   ↓
生成 A Early Reference
   ↓
放入二级缓存
   ↓
B 注入 A
   ↓
B 初始化完成
   ↓
B 放一级缓存
   ↓
A 注入 B
   ↓
A 初始化完成
   ↓
A 放一级缓存
```

---

## 当前工程如何看循环依赖

三级缓存主要用于：

```text
理解 Spring IOC 内部机制
```

工程实践：

```text
优先避免循环依赖
```

解决思路：

```text
重新划分职责
提取公共依赖
事件机制
@Lazy
ObjectProvider
Setter 注入
```

Spring Boot 新版本默认通常：

```text
不鼓励允许循环引用
```

不要把：

```text
三级缓存
```

当作正常架构设计手段。

---

# Spring AOP

## AOP

AOP：

```text
Aspect-Oriented Programming
面向切面编程
```

解决：

```text
多个业务方法
都需要相同公共逻辑
```

例如：

```text
日志
事务
权限
性能监控
缓存
审计
```

如果直接写业务：

```text
业务代码
+
日志
+
事务
+
权限
```

会造成：

```text
重复
耦合
```

AOP：

```text
公共逻辑
抽成 Aspect
```

---

## 核心概念

### Aspect

```text
切面
```

公共逻辑的模块。

---

### Join Point

```text
连接点
```

可以被增强的位置。

Spring AOP 中最常关注：

```text
方法执行
```

---

### Pointcut

```text
切点
```

决定：

```text
哪些 Join Point
需要增强
```

---

### Advice

```text
通知
```

决定：

```text
什么时候增强
+
增强什么
```

常见：

```text
@Before
@After
@AfterReturning
@AfterThrowing
@Around
```

---

## AOP 执行流程

```text
调用者
   ↓
Proxy
   ↓
Interceptor / Advice
   ↓
Target Method
   ↓
返回 Proxy
   ↓
调用者
```

---

# JDK 动态代理

适合：

```text
目标对象实现接口
```

核心：

```text
Proxy
+
InvocationHandler
```

代理对象：

```text
实现相同接口
```

流程：

```text
接口方法调用
   ↓
InvocationHandler.invoke()
   ↓
增强逻辑
   ↓
目标方法
```

---

# CGLIB 代理

如果需要基于类代理：

```text
生成目标类的子类
```

通过：

```text
Override 方法
```

插入增强逻辑。

限制：

```text
final class
→ 不能继承

final method
→ 不能 override

private method
→ 不能 override
```

因此这些方法：

```text
无法通过普通 CGLIB 子类代理方式增强
```

---

## JDK vs CGLIB

```text
JDK Proxy
→ 基于接口

CGLIB
→ 基于继承
```

Spring 会根据：

```text
配置
目标类型
代理方式
```

选择合适机制。

不要简单记：

```text
有接口永远 JDK
没接口永远 CGLIB
```

因为代理策略还可以人为配置。

---

# 自调用问题

假设：

```java
public void methodA() {
    this.methodB();
}

@Transactional
public void methodB() {
}
```

外部调用：

```text
Caller
 ↓
Proxy
 ↓
methodA
```

但：

```text
methodA
 ↓
this.methodB()
```

是：

```text
目标对象内部直接调用
```

没有重新经过：

```text
Proxy
```

所以基于 Spring Proxy 的 AOP：

```text
methodB 增强可能不会重新触发
```

这就是：

```text
Self Invocation
自调用问题
```

---

# Spring 事务

## 事务管理方式

Spring 支持：

```text
编程式事务
声明式事务
```

---

## 编程式事务

常见：

```text
TransactionTemplate
```

特点：

```text
事务逻辑
写在业务代码里
```

优点：

```text
控制精细
```

缺点：

```text
有代码侵入
```

---

## 声明式事务

常见：

```java
@Transactional
```

特点：

```text
基于 AOP
```

事务逻辑与业务代码分离。

---

# 声明式事务原理

核心：

```text
@Transactional
只是事务元数据
```

Spring：

```text
读取事务属性
   ↓
为 Bean 创建 AOP Proxy
   ↓
方法调用进入 Proxy
   ↓
TransactionInterceptor
   ↓
TransactionManager
   ↓
开启 / 加入事务
   ↓
调用目标方法
   ↓
提交 / 回滚
```

可以记：

```text
@Transactional
+
AOP Proxy
+
TransactionInterceptor
+
TransactionManager
```

---

## TransactionManager

Spring 事务管理核心抽象：

```text
TransactionManager
```

传统 JDBC 常见：

```text
DataSourceTransactionManager
```

它负责：

```text
开启事务
提交
回滚
挂起
恢复
```

---

# @Transactional 默认行为

常见默认：

```text
Propagation
→ REQUIRED

Isolation
→ DEFAULT

readOnly
→ false
```

回滚：

```text
RuntimeException
Error
→ 默认回滚

Checked Exception
→ 默认不回滚
```

---

# rollbackFor

如果希望 Checked Exception 也回滚：

```java
@Transactional(rollbackFor = Exception.class)
```

表示：

```text
Exception
及其子类
满足条件时回滚
```

---

# 事务失效 / 不符合预期

## 1. Self Invocation

```java
this.methodB();
```

没有经过 Proxy：

```text
@Transactional
可能不会触发
```

这是最核心场景之一。

---

## 2. 异常被自己 catch

```java
@Transactional
public void save() {
    try {
        ...
    } catch (Exception e) {
        // 吃掉异常
    }
}
```

代理看到：

```text
方法正常返回
```

于是：

```text
可能提交事务
```

解决：

```text
继续向外抛
```

或显式：

```text
标记 rollback-only
```

---

## 3. Checked Exception

```java
@Transactional
public void save() throws IOException {
    ...
}
```

默认：

```text
Checked Exception
不一定触发回滚
```

解决：

```java
@Transactional(rollbackFor = Exception.class)
```

---

## 4. 非 Spring Bean

```java
UserService service = new UserService();
```

对象不是：

```text
Spring Container
```

创建和管理。

因此：

```text
没有 Spring AOP Proxy
→ 声明式事务不生效
```

---

## 5. private 方法

Proxy 模式下：

```text
private method
```

不能作为普通代理拦截入口。

所以：

```text
@Transactional
不应依赖 private 方法
```

---

## 6. 非 public 方法的版本区别

不要死记：

```text
@Transactional 只能 public
```

当前 Spring 6+：

```text
Class-based Proxy
→ protected
→ package-private
也可以被事务代理处理
```

但是：

```text
Interface-based Proxy
→ 事务方法必须是接口中的 public 方法
```

同时：

```text
Self Invocation
仍然绕过 Proxy
```

所以日常建议：

```text
事务边界优先放在 public Service 方法
```

最清晰。

---

## 7. final

CGLIB：

```text
通过子类 override
```

因此：

```text
final class
final method
```

无法以普通 CGLIB 子类代理方式增强。

---

## 8. 新线程

本地事务通常：

```text
绑定当前线程
```

因此：

```text
父线程事务
不会自动传递给新线程
```

注意：

```text
新线程通过另一个 Spring Proxy
调用 @Transactional 方法
```

可以：

```text
开启它自己的事务
```

但不是：

```text
自动加入原线程事务
```

---

## 9. 数据库不支持事务

例如：

```text
MySQL MyISAM
```

本身不支持事务。

Spring：

```text
无法凭空提供数据库事务能力
```

---

# 事务传播行为

传播行为解决：

> **一个事务方法调用另一个事务方法时，新的方法应该加入、创建、挂起还是拒绝事务。**

七种：

| 类型 | 行为 |
| --- | --- |
| REQUIRED | 有事务就加入，没有就新建 |
| SUPPORTS | 有事务就加入，没有就非事务执行 |
| MANDATORY | 必须有事务，否则抛异常 |
| REQUIRES_NEW | 永远新建事务，原事务挂起 |
| NOT_SUPPORTED | 非事务执行，有事务则挂起 |
| NEVER | 非事务执行，有事务则抛异常 |
| NESTED | 有事务则嵌套执行，没有则类似 REQUIRED |

---

## REQUIRED

默认：

```text
当前有事务
→ 加入

当前无事务
→ 创建
```

---

## REQUIRES_NEW

```text
当前有事务
→ 挂起

新建一个独立事务
```

内外事务：

```text
提交 / 回滚相对独立
```

---

## NESTED

当前存在事务：

```text
在嵌套事务中执行
```

通常依赖：

```text
Savepoint
```

实际能力取决于：

```text
TransactionManager
数据库驱动
底层资源
```

---

# Spring MVC

## 请求流程

核心：

```text
Client
   ↓
Filter
   ↓
DispatcherServlet
   ↓
HandlerMapping
   ↓
HandlerInterceptor
   ↓
Controller
   ↓
Service
   ↓
返回结果
   ↓
HandlerInterceptor
   ↓
DispatcherServlet
   ↓
Filter
   ↓
Client
```

---

# Filter vs Interceptor

## Filter

Filter：

```text
Servlet 规范
```

作用位置：

```text
Servlet 外层
```

可以处理：

```text
编码
跨域
日志
鉴权
包装 Request / Response
```

---

## Interceptor

Interceptor：

```text
Spring MVC
HandlerInterceptor
```

作用于：

```text
Controller Handler
```

常见方法：

```text
preHandle
postHandle
afterCompletion
```

适合：

```text
登录校验
Controller 日志
权限判断
耗时统计
```

---

## 执行关系

```text
Request
 ↓
Filter
 ↓
DispatcherServlet
 ↓
Interceptor.preHandle
 ↓
Controller
 ↓
Interceptor.postHandle
 ↓
Interceptor.afterCompletion
 ↓
Filter
 ↓
Response
```

一句话：

```text
Filter
→ Servlet 层

Interceptor
→ Spring MVC 层
```

---

# Spring Boot

## @SpringBootApplication

核心：

```java
@SpringBootApplication
```

可以重点理解成：

```text
@SpringBootConfiguration
+
@ComponentScan
+
@EnableAutoConfiguration
```

---

## @SpringBootConfiguration

本质：

```text
@Configuration
```

表示：

```text
这是 Spring Boot 主配置类
```

---

## @ComponentScan

默认：

```text
扫描启动类所在包
以及子包
```

所以启动类通常放在：

```text
项目较上层包
```

---

## @EnableAutoConfiguration

Spring Boot 自动配置核心入口。

作用：

```text
根据 Classpath
已有 Bean
配置属性
运行环境
```

自动创建合适的 Bean。

---

# 自动配置原理

主线：

```text
@SpringBootApplication
   ↓
@EnableAutoConfiguration
   ↓
加载 AutoConfiguration Candidates
   ↓
@Conditional...
   ↓
条件成立
   ↓
创建配置中的 Bean
```

---

## AutoConfiguration.imports

现代 Spring Boot 自定义自动配置候选类通常声明在：

```text
META-INF/spring/
org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

里面列出：

```text
AutoConfiguration Class
```

---

# 条件装配

自动配置不是：

```text
所有配置类全部无条件生效
```

而是大量使用：

```text
@Conditional...
```

---

## @ConditionalOnClass

```text
Classpath 中存在指定类
→ 条件成立
```

例如：

```text
存在 DataSource 类
```

才考虑数据库相关自动配置。

---

## @ConditionalOnMissingBean

```text
容器里没有用户自己定义的 Bean
→ 自动配置才创建默认 Bean
```

这就是：

> **Spring Boot 自动配置“用户配置优先”的重要机制。**

---

## @ConditionalOnProperty

根据：

```text
配置文件属性
```

决定是否装配。

例如：

```text
feature.enabled=true
```

---

## 自动配置一句话

> **Spring Boot 自动配置 = 候选配置类 + 条件判断 + 默认 Bean。**

---

# Spring 常见注解

## IOC

```text
@Component
@Service
@Repository
@Controller
@Configuration
@Bean
```

---

### @Component

通用组件。

---

### @Service

业务层组件。

本质：

```text
@Component
语义化封装
```

---

### @Repository

DAO / Repository 组件。

---

### @Controller

Spring MVC Controller。

---

### @RestController

相当于常见组合：

```text
@Controller
+
@ResponseBody
```

---

### @Configuration

配置类。

---

### @Bean

把方法返回值：

```text
注册为 Spring Bean
```

---

# DI 注解

```text
@Autowired
@Qualifier
@Resource
@Value
```

---

## @Qualifier

同类型多个 Bean 时：

```text
指定 Bean
```

---

## @Value

注入：

```text
配置值
表达式
```

例如：

```java
@Value("${server.port}")
private int port;
```

---

# AOP / Transaction

```text
@Aspect
@Pointcut
@Before
@After
@Around
@Transactional
@EnableTransactionManagement
```

---

# Spring MVC 常见注解

```text
@RequestMapping
@GetMapping
@PostMapping
@PutMapping
@DeleteMapping

@RequestParam
@PathVariable
@RequestBody
@RequestHeader

@ResponseBody
@RestController
```

---

## @RequestParam

参数：

```text
Query Parameter / Form Parameter
```

例如：

```text
?id=1
```

---

## @PathVariable

路径变量：

```text
/users/{id}
```

---

## @RequestBody

读取：

```text
HTTP Request Body
```

常用于：

```text
JSON → Java Object
```

---

# Spring Boot 常见注解

```text
@SpringBootApplication
@EnableAutoConfiguration
@ConfigurationProperties
@ConditionalOnClass
@ConditionalOnMissingBean
@ConditionalOnProperty
```

---

# 速记

## IOC / DI

```text
IOC
→ 对象交给容器

DI
→ 容器把依赖塞给对象
```

---

## Bean

```text
@Component / @Bean
      ↓
BeanDefinition
      ↓
实例化
      ↓
DI
      ↓
初始化
      ↓
BeanPostProcessor
      ↓
Proxy / Bean
```

---

## Scope

```text
singleton
→ 一个 IOC Container 内共享一个实例

prototype
→ 多次创建
```

记：

```text
singleton
≠
线程安全
```

---

## 生命周期

```text
实例化
 ↓
属性注入
 ↓
Aware
 ↓
BPP Before
 ↓
初始化
 ↓
BPP After
 ↓
使用
 ↓
销毁
```

---

## 三级缓存

```text
一级 singletonObjects
→ 完整 Bean

二级 earlySingletonObjects
→ Early Reference

三级 singletonFactories
→ ObjectFactory
```

解决的是：

```text
经典 singleton
Setter / Field 循环依赖
```

不是：

```text
所有循环依赖
```

---

## AOP

```text
Caller
 ↓
Proxy
 ↓
Advice / Interceptor
 ↓
Target
```

```text
JDK
→ 接口代理

CGLIB
→ 子类代理
```

---

## 事务

```text
@Transactional
   ↓
Proxy
   ↓
TransactionInterceptor
   ↓
TransactionManager
   ↓
Target Method
```

默认：

```text
REQUIRED

RuntimeException / Error
→ Rollback

Checked Exception
→ 默认不 Rollback
```

---

## 常见事务失效

```text
this 自调用
异常被 catch
Checked Exception 未配置 rollbackFor
对象不是 Spring Bean
private 方法
CGLIB final 限制
跨线程不会继承原事务
数据库本身不支持事务
```

---

## Spring Boot

```text
@SpringBootApplication
├── @SpringBootConfiguration
├── @ComponentScan
└── @EnableAutoConfiguration
```

自动配置：

```text
AutoConfiguration
+
@Conditional
+
默认 Bean
```

---

# 一句话总结

```text
IOC
→ 管对象

DI
→ 注依赖

Bean 生命周期
→ 管对象从出生到销毁

AOP
→ 代理增强

Transaction
→ AOP 管事务边界

MVC
→ 管 Web 请求

Boot
→ 自动装配 Spring
```
