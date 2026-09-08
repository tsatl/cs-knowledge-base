# Spring 基础知识结构速记

> 主线：**IOC 与 Bean → AOP → 事务 → Spring MVC → Spring Boot**

# 1. IOC 与 Bean

## IOC / DI

- **IOC**：对象的创建、依赖关系和生命周期交给 Spring 容器管理。
- **DI**：IOC 的主要实现方式，由容器把依赖注入 Bean。

```text
配置 / 注解
   ↓
BeanDefinition
   ↓
Spring Container
   ↓
实例化 Bean
   ↓
DI
   ↓
初始化
   ↓
Bean / Proxy
```

IOC 底层会用到反射、工厂、代理、`BeanPostProcessor` 等机制，但 **IOC ≠ 反射**。

## 容器与 Bean 定义

| 概念 | 作用 |
| --- | --- |
| `BeanFactory` | IOC 容器基础接口，负责 Bean 创建、获取、依赖和生命周期 |
| `ApplicationContext` | BeanFactory 的增强版，增加事件、资源加载、环境配置、AOP 等能力，开发中更常用 |
| `BeanDefinition` | Bean 的“说明书”，记录类型、Scope、Lazy、构造参数、属性、初始化/销毁方法等 |
| `BeanFactoryPostProcessor` | Bean 实例化前修改 `BeanDefinition` 等容器元数据 |
| `BeanPostProcessor` | Bean 实例创建后，在初始化前后处理 Bean；AOP 代理等机制会用到 |

```text
@Component / @Bean / XML
        ↓
   BeanDefinition
        ↓
   Spring Container
        ↓
       Bean
```

单例 Bean 最终通常保存在 `singletonObjects`。

## Bean 创建与管理

### 注入方式

| 方式 | 特点 |
| --- | --- |
| 构造器注入 | 依赖明确，可配合 `final`，适合必需依赖，通常优先 |
| Setter 注入 | 适合可选依赖 |
| 字段注入 | 写法简单，但依赖不显式、测试不方便 |

```java
@Service
public class UserService {
    private final UserDao userDao;

    public UserService(UserDao userDao) {
        this.userDao = userDao;
    }
}
```

依赖注入常见注解：

| 注解 | 作用 |
| --- | --- |
| `@Autowired` | 主要按类型注入 |
| `@Qualifier` | 同类型多个 Bean 时指定候选 |
| `@Resource` | 默认更偏按名称注入 |
| `@Value` | 注入配置值或表达式 |

### Scope 与线程安全

- `singleton`：默认，一个 IOC Container 中一个 `BeanDefinition` 通常共享一个实例。
- `prototype`：每次获取或依赖解析都可以创建新实例；Spring 负责创建和初始化，但通常不继续管理其销毁。
- Web 常见还有 `request`、`session`、`application`。

> `singleton ≠ JVM 全局单例 ≠ 线程安全`

线程安全取决于 Bean 是否存在共享可变状态；Service / DAO 通常设计为无状态 Bean。

### Bean 生命周期

主线：

```text
BeanDefinition
→ 实例化（执行构造方法）
→ 属性注入 / DI
→ Aware 等初始化前回调
→ BeanPostProcessor Before（相关后处理器会触发 @PostConstruct）
→ afterPropertiesSet()
→ init-method
→ BeanPostProcessor After
→ Bean / Proxy
→ 销毁
```

- **实例化**：根据 `BeanDefinition` 创建对象，构造方法在这一阶段执行。
- **属性注入**：完成 `@Autowired`、`@Resource`、Setter 等依赖注入。
- **初始化回调**：常见顺序为 `@PostConstruct → InitializingBean.afterPropertiesSet() → init-method`。
- **BeanPostProcessor After**：可以返回包装后的对象，Spring AOP 代理通常与这一阶段密切相关。
- **销毁**：容器关闭时常见顺序为 `@PreDestroy → DisposableBean.destroy() → destroy-method`。

详细解释见：`spring-bean-lifecycle.md`。

## 循环依赖与三级缓存

循环依赖：

```text
A → B → A
```

- **构造器循环依赖**：A 尚未实例化完成就需要 B，没有可提前暴露的 A，三级缓存无法直接解决。
- **Setter / Field 循环依赖**：经典 singleton 场景可通过“提前暴露引用”解决。

| 缓存 | 内容 |
| --- | --- |
| `singletonObjects` | 一级缓存：完整初始化完成的 Bean |
| `earlySingletonObjects` | 二级缓存：已经生成的 Early Reference |
| `singletonFactories` | 三级缓存：用于按需生成 Early Reference 的 `ObjectFactory` |

完整流程：

```text
创建 A
 ↓
A 实例化完成
 ↓
A 的 ObjectFactory 放入三级缓存
 ↓
A 需要 B → 创建 B
 ↓
B 需要 A
 ↓
查一级缓存：没有 A
 ↓
查二级缓存：没有 A
 ↓
查三级缓存：找到 A Factory
 ↓
Factory 生成 A Early Reference
 ↓
A Early Reference 放入二级缓存
同时移除 A 的三级 Factory
 ↓
B 注入 A Early Reference
 ↓
B 初始化完成 → 放入一级缓存
 ↓
A 注入 B
 ↓
A 初始化完成 → 放入一级缓存
 ↓
清理 A 的二级 / 三级缓存
```

核心关系：

```text
三级 ObjectFactory
→ 第一次需要提前引用时生成 Early Reference
→ 放入二级缓存
→ Bean 完整初始化后进入一级缓存
```

解决普通循环依赖的核心是**提前暴露引用**；第三级 `ObjectFactory` 还提供了按需生成 Early Reference 的机会，存在 AOP 时可参与生成提前代理引用。

注意：这套机制只针对特定的 singleton 循环依赖。现代 Spring Boot 默认 `spring.main.allow-circular-references=false`，工程上仍应优先消除循环依赖。

# 2. Spring AOP

## AOP 原理

AOP 用于把日志、事务、权限、监控、缓存等公共逻辑从业务代码中抽离。

| 概念 | 含义 |
| --- | --- |
| Aspect | 切面，公共逻辑 |
| Join Point | 可被增强的位置，Spring AOP 主要关注方法 |
| Pointcut | 决定哪些方法需要增强 |
| Advice | 决定何时增强、增强什么 |

常见 Advice：`@Before`、`@After`、`@AfterReturning`、`@AfterThrowing`、`@Around`。

```text
Caller
  ↓
Proxy
  ↓
Advice / Interceptor
  ↓
Target Method
```

## 动态代理

| 方式 | 原理 | 限制 |
| --- | --- | --- |
| JDK Proxy | 基于接口，`Proxy + InvocationHandler` | 依赖接口代理能力 |
| CGLIB | 生成目标类子类并重写方法 | `final class` 不能继承，`final/private` 方法不能普通重写增强 |

不要死记“有接口一定 JDK、没接口一定 CGLIB”，实际还受配置和代理策略影响。

## 自调用问题

```java
public void methodA() {
    this.methodB();
}

@Transactional
public void methodB() {}
```

外部调用：

```text
Caller → Proxy → methodA
```

但 `this.methodB()` 是对象内部直接调用，没有重新经过 Proxy，因此在常见的 **proxy 模式** 下，对 `methodB()` 的 AOP / 事务增强不会被再次触发。

# 3. Spring 事务

> Spring 声明式事务本质上建立在 AOP 代理之上。

## 事务原理

Spring 支持：

- **编程式事务**：如 `TransactionTemplate`。
- **声明式事务**：常用 `@Transactional`，基于 AOP。

```text
@Transactional
   ↓
Proxy
   ↓
TransactionInterceptor
   ↓
TransactionManager
   ↓
开启 / 加入事务
   ↓
Target Method
   ↓
提交 / 回滚
```

`TransactionManager` 负责开启、提交、回滚、挂起、恢复事务。

## @Transactional

默认行为：

| 属性 | 默认 |
| --- | --- |
| 传播行为 | `REQUIRED` |
| 隔离级别 | `DEFAULT` |
| `readOnly` | `false` |
| RuntimeException / Error | 默认回滚 |
| Checked Exception | 默认不回滚 |

Checked Exception 也希望回滚：

```java
@Transactional(rollbackFor = Exception.class)
```

## 事务传播与失效

### 传播行为

| 类型 | 行为 |
| --- | --- |
| `REQUIRED` | 有事务就加入，没有就新建 |
| `SUPPORTS` | 有事务就加入，没有就非事务执行 |
| `MANDATORY` | 必须有事务，否则异常 |
| `REQUIRES_NEW` | 新建事务，原事务挂起 |
| `NOT_SUPPORTED` | 非事务执行，有事务则挂起 |
| `NEVER` | 非事务执行，有事务则异常 |
| `NESTED` | 有事务则嵌套执行，没有则类似 REQUIRED |

重点记：

```text
REQUIRED
→ 默认
→ 有就加入，没有就新建

REQUIRES_NEW
→ 永远新建
→ 原事务挂起

NESTED
→ 常通过 Savepoint 实现
```

### 常见事务失效 / 不符合预期

| 场景 | 原因 |
| --- | --- |
| `this` 自调用 | 没经过 Proxy |
| 异常被自己 catch | 代理可能看到正常返回 |
| Checked Exception 未配 `rollbackFor` | 默认不一定回滚 |
| 对象不是 Spring Bean | 没有 Spring AOP Proxy |
| `private` 方法 | 不能作为普通代理拦截入口 |
| CGLIB `final` 限制 | 不能通过子类重写增强 |
| 新线程 | 命令式事务通常绑定当前线程，不会自动传播到新线程 |
| 数据库不支持事务 | Spring 无法凭空提供事务能力 |

方法可见性要区分代理方式：Spring 6+ 的类代理可支持 `protected` / package-private 事务方法；JDK 接口代理要求事务方法是接口中的 `public` 方法。日常事务边界放在 `public Service` 方法最清晰。

# 4. Spring MVC

## 请求流程

Spring MVC 核心链路：

```text
Client
  ↓
Filter
  ↓
DispatcherServlet
  ↓
HandlerMapping
  ↓
找到 Handler + Interceptor
  ↓
HandlerAdapter
  ↓
Controller
  ↓
返回值处理
  ├─ HttpMessageConverter（JSON / @ResponseBody）
  └─ ViewResolver / View（页面）
  ↓
Response
```

`DispatcherServlet` 是前端控制器；`HandlerMapping` 负责找到处理器，`HandlerAdapter` 负责以统一方式调用对应 Handler。Controller 中通常再调用 Service 完成业务逻辑。

异常处理时还可能进入 `HandlerExceptionResolver`。

## Filter / Interceptor

| 类型 | 所属 | 常见用途 |
| --- | --- | --- |
| Filter | Servlet 规范 | 编码、跨域、日志、鉴权、包装 Request/Response |
| Interceptor | Spring MVC | Controller 前后，登录校验、权限、日志、耗时统计 |

Interceptor 三个核心回调：

```text
preHandle()
→ Handler 执行前

postHandle()
→ Handler 执行后

afterCompletion()
→ 整个请求完成后
```

安全鉴权通常优先使用 Spring Security / Filter 链，而不是只依赖 MVC Interceptor。

# 5. Spring Boot

## 启动与自动配置

`@SpringBootApplication` 重点理解为：

```text
@SpringBootApplication
├── @SpringBootConfiguration
├── @ComponentScan
└── @EnableAutoConfiguration
```

- `@SpringBootConfiguration`：Spring Boot 主配置类，本质基于 `@Configuration`。
- `@ComponentScan`：默认扫描启动类所在包及其子包。
- `@EnableAutoConfiguration`：开启自动配置。

```text
@SpringBootApplication
   ↓
@EnableAutoConfiguration
   ↓
加载自动配置候选类
   ↓
@Conditional...
   ↓
条件成立
   ↓
注册默认 Bean
```

现代 Spring Boot 的自动配置类通常使用 `@AutoConfiguration`，候选类记录在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。

## 条件装配

| 注解 | 条件 |
| --- | --- |
| `@ConditionalOnClass` | Classpath 中存在指定类 |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean，体现“用户配置优先” |
| `@ConditionalOnProperty` | 配置属性满足条件 |

> **Spring Boot 自动配置 = 自动配置候选类 + 条件判断 + 默认 Bean；用户自己定义 Bean 后，很多默认配置会自动 back off。**

# 6. 常见注解速查

| 分类 | 注解 | 作用 |
| --- | --- | --- |
| IOC | `@Component` | 通用组件 |
| IOC | `@Service` | 业务层组件 |
| IOC | `@Repository` | DAO / Repository 组件 |
| IOC | `@Controller` | MVC Controller |
| IOC | `@RestController` | `@Controller + @ResponseBody` |
| IOC | `@Configuration` | 配置类 |
| IOC | `@Bean` | 方法返回值注册为 Bean |
| DI | `@Autowired` | 主要按类型注入 |
| DI | `@Qualifier` | 指定同类型 Bean |
| DI | `@Primary` | 多个同类型 Bean 中指定默认优先候选 |
| DI | `@Resource` | 默认更偏按名称注入 |
| DI | `@Value` | 注入配置值 |
| 生命周期 | `@PostConstruct` | 依赖注入后执行初始化方法 |
| 生命周期 | `@PreDestroy` | Bean 销毁前执行清理方法 |
| AOP | `@Aspect` | 定义切面 |
| AOP | `@Pointcut` | 定义切点 |
| AOP | `@Before` / `@After` / `@Around` | 定义 Advice |
| 事务 | `@Transactional` | 声明事务 |
| 事务 | `@EnableTransactionManagement` | 开启注解驱动事务管理 |
| MVC | `@RequestMapping` | 通用请求映射 |
| MVC | `@GetMapping` / `@PostMapping` | GET / POST 请求映射 |
| MVC | `@PutMapping` / `@DeleteMapping` | PUT / DELETE 请求映射 |
| MVC | `@RequestParam` | 获取 Query / Form 参数 |
| MVC | `@PathVariable` | 获取路径变量 |
| MVC | `@RequestBody` | 请求体 → Java 对象 |
| MVC | `@RequestHeader` | 获取请求头 |
| MVC | `@ResponseBody` | 返回值写入响应体 |
| MVC | `@ExceptionHandler` | 处理 Controller 异常 |
| MVC | `@ControllerAdvice` | 全局 Controller 增强 / 异常处理 |
| Boot | `@SpringBootApplication` | Boot 启动入口组合注解 |
| Boot | `@EnableAutoConfiguration` | 开启自动配置 |
| Boot | `@ConfigurationProperties` | 配置属性绑定到 Java 对象 |
| Boot | `@ConditionalOnClass` | 类存在时装配 |
| Boot | `@ConditionalOnMissingBean` | Bean 不存在时装配 |
| Boot | `@ConditionalOnProperty` | 配置满足条件时装配 |

# 7. 速记

```text
IOC
→ 容器管对象

DI
→ 容器注依赖

Bean
→ BeanDefinition → 实例化 → DI → 初始化 → Bean / Proxy

Bean 生命周期
→ 实例化 → 属性注入 → 初始化 → 销毁

三级缓存
→ 一级：完整 Bean
→ 二级：已生成 Early Reference
→ 三级：ObjectFactory
→ 三级生成 Early Reference 后转入二级，Bean 完成后进入一级

AOP
→ Caller → Proxy → Advice → Target
→ JDK：接口代理
→ CGLIB：子类代理

事务
→ @Transactional → Proxy → TransactionInterceptor → TransactionManager
→ 默认 REQUIRED
→ RuntimeException / Error 默认回滚
→ Checked Exception 默认不回滚

MVC
→ Filter → DispatcherServlet → HandlerMapping → HandlerAdapter → Controller

Boot
→ @SpringBootApplication
→ @EnableAutoConfiguration
→ AutoConfiguration + @Conditional + 默认 Bean
```
