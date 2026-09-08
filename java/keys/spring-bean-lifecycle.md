# Spring Bean 生命周期

Spring Bean 生命周期可以按四部分理解：

```text
创建 → 初始化 → 使用 → 销毁
```

完整流程：

```text
BeanDefinition
   ↓
实例化（执行构造方法）
   ↓
属性注入 / DI
   ↓
Aware
   ↓
BeanPostProcessor Before
   ↓
@PostConstruct
   ↓
InitializingBean.afterPropertiesSet()
   ↓
init-method
   ↓
BeanPostProcessor After
   ↓
Bean / Proxy
   ↓
容器关闭
   ↓
@PreDestroy
   ↓
DisposableBean.destroy()
   ↓
destroy-method
```

## 1. BeanDefinition

Spring 会先把 `@Component`、`@Bean`、XML 等配置解析成 `BeanDefinition`。

`BeanDefinition` 可以理解为 Bean 的“说明书”，记录 Bean 类型、Scope、构造参数、属性、初始化方法、销毁方法等。

> 此时 Bean 对象还没有真正创建。

## 2. 实例化

Spring 根据 `BeanDefinition` 创建 Bean 对象，**构造方法就在这一阶段执行**。

```text
BeanDefinition
→ 选择构造器
→ 执行构造方法
→ 创建 Bean 对象
```

如果使用字段注入：

```java
@Autowired
private UserDao userDao;
```

构造方法执行时，`userDao` 通常还没有完成注入。

如果使用构造器注入：

```java
public UserService(UserDao userDao) {
    this.userDao = userDao;
}
```

依赖会作为构造参数传入。

## 3. 属性注入 / DI

Bean 创建后，Spring 给它注入依赖和属性。

常见方式：

- `@Autowired`
- `@Resource`
- Setter
- `@Value`

例如：

```text
UserService 已创建
      ↓
找到 UserDao Bean
      ↓
注入 UserService.userDao
```

因此：

```text
实例化 = 创建对象
DI     = 给对象装配依赖
```

## 4. Aware

如果 Bean 实现了 `Aware` 相关接口，Spring 会把容器信息回调给 Bean。

| 接口 | 作用 |
| --- | --- |
| `BeanNameAware` | 获取 Bean 名称 |
| `BeanFactoryAware` | 获取 `BeanFactory` |
| `ApplicationContextAware` | 获取 `ApplicationContext` |

可以理解为：

> DI 是注入业务依赖，Aware 是让 Bean 感知 Spring 容器本身的信息。

## 5. BeanPostProcessor Before

Spring 在 Bean 正式初始化之前，会执行：

```java
postProcessBeforeInitialization()
```

它是 Spring 提供的重要扩展点，可以在初始化前统一处理 Bean。

## 6. @PostConstruct

`@PostConstruct` 标记的方法会在依赖注入完成后执行一次，常用于初始化。

```java
@PostConstruct
public void init() {
    // 初始化缓存、加载数据等
}
```

此时 Bean 的依赖通常已经完成注入。

## 7. afterPropertiesSet()

如果 Bean 实现：

```java
InitializingBean
```

Spring 会调用：

```java
afterPropertiesSet()
```

例如：

```java
public class UserService implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        // 初始化逻辑
    }
}
```

它和 `@PostConstruct` 都属于初始化回调，只是实现方式不同。

## 8. init-method

也可以自定义初始化方法：

```java
@Bean(initMethod = "init")
public UserService userService() {
    return new UserService();
}
```

Spring 会调用：

```java
userService.init();
```

常见初始化顺序：

```text
@PostConstruct
→ afterPropertiesSet()
→ init-method
```

## 9. BeanPostProcessor After

初始化完成后，Spring 会执行：

```java
postProcessAfterInitialization()
```

Spring AOP 代理的创建通常与 `BeanPostProcessor` 体系密切相关。

因此最终容器中的对象可能是：

```text
原始 Bean
```

也可能是：

```text
Proxy
 ↓
原始 Bean
```

所以：

> 实例化得到的对象，不一定就是最终 `getBean()` 得到的对象。

## 10. Bean / Proxy 使用

初始化完成后，Bean 就可以正常提供给其他对象使用。

普通 Bean：

```text
Container → UserService
```

存在 AOP 时：

```text
Container → Proxy → UserService
```

例如事务方法：

```text
调用者
 ↓
Proxy
 ↓
事务增强
 ↓
目标方法
```

## 11. @PreDestroy

容器关闭、Bean 准备销毁时，会执行 `@PreDestroy` 标记的方法。

```java
@PreDestroy
public void cleanup() {
    // 释放资源
}
```

常用于关闭连接、停止线程、释放资源。

## 12. DisposableBean.destroy()

如果 Bean 实现：

```java
DisposableBean
```

Spring 会调用：

```java
destroy()
```

它和 `InitializingBean.afterPropertiesSet()` 相对应：

```text
InitializingBean
→ afterPropertiesSet()
→ 初始化

DisposableBean
→ destroy()
→ 销毁
```

## 13. destroy-method

也可以指定自定义销毁方法：

```java
@Bean(destroyMethod = "close")
public UserService userService() {
    return new UserService();
}
```

容器关闭时会调用：

```java
userService.close();
```

销毁阶段常见顺序：

```text
@PreDestroy
→ DisposableBean.destroy()
→ destroy-method
```

# 速记

```text
① 创建
BeanDefinition
→ 实例化（构造方法）
→ 属性注入 / DI

② 初始化
Aware
→ BPP Before
→ @PostConstruct
→ afterPropertiesSet()
→ init-method
→ BPP After

③ 使用
Bean / Proxy

④ 销毁
@PreDestroy
→ destroy()
→ destroy-method
```

重点区分：

```text
构造方法
→ 属于实例化

@Autowired / @Resource
→ 属于属性注入

@PostConstruct / afterPropertiesSet / init-method
→ 属于初始化

@PreDestroy / destroy / destroy-method
→ 属于销毁
```
