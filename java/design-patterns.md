# 设计模式

## 基础概念

**设计模式：** 针对软件设计中反复出现的问题，总结出的可复用设计经验。

作用：

```text
降低耦合
提高复用性
提高可扩展性
提高可维护性
```

设计模式通常分为：

```text
创建型模式
结构型模式
行为型模式
```

---

## 常见设计原则

### 单一职责原则 SRP

一个类：

```text
尽量只负责一类职责
```

避免一个类承担过多功能。

---

### 开闭原则 OCP

```text
对扩展开放
对修改关闭
```

新增功能时尽量：

```text
增加新实现
```

而不是频繁修改原有代码。

---

### 里氏替换原则 LSP

子类对象应该能够：

```text
替换父类对象
```

而不破坏程序原有行为。

---

### 接口隔离原则 ISP

不要让一个类依赖：

```text
它不需要的方法
```

大接口应根据职责：

```text
拆成多个小接口
```

---

### 依赖倒置原则 DIP

高层模块不要直接依赖具体实现，而应该依赖：

```text
抽象
```

即：

```text
面向接口编程
```

---

### 迪米特法则 LoD

一个对象应该：

```text
尽量少地了解其他对象
```

降低对象之间的直接依赖。

---

# 创建型模式

## 单例模式

**定义：**

> 保证一个类只有一个实例，并提供统一的访问入口。

典型场景：

```text
配置对象
资源管理器
线程池管理器
缓存对象
```

Spring Bean 默认通常也是：

```text
singleton 作用域
```

但 Spring 的 singleton 是：

```text
一个 ApplicationContext 中通常只有一个 Bean 实例
```

并不等同于 JVM 全局绝对只有一个对象。

---

### 饿汉式

```java
public class Singleton {

    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {
    }

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

特点：

```text
类加载时创建
线程安全
实现简单
可能提前占用资源
```

---

### 懒汉式

```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

问题：

```text
线程不安全
```

多个线程可能同时创建多个实例。

---

### synchronized 懒汉式

```java
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```

特点：

```text
线程安全
但每次调用都需要获取锁
```

---

### 双重检查锁 DCL

```java
public class Singleton {

    private static volatile Singleton instance;

    private Singleton() {
    }

    public static Singleton getInstance() {

        if (instance == null) {
            synchronized (Singleton.class) {

                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }

        return instance;
    }
}
```

为什么需要两次判断：

```text
第一次
→ 避免已经创建实例后仍然进入 synchronized

第二次
→ 防止多个线程先后进入同步块后重复创建
```

为什么需要 `volatile`：

```text
防止指令重排序
保证其他线程看到完整初始化后的对象
```

---

### 静态内部类

```java
public class Singleton {

    private Singleton() {
    }

    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```

特点：

```text
懒加载
线程安全
代码简单
```

利用：

```text
JVM 类初始化的线程安全机制
```

---

### 枚举单例

```java
public enum Singleton {
    INSTANCE;
}
```

优点：

```text
天然线程安全
防止普通反射创建新实例
天然支持序列化
```

通常是非常稳妥的单例实现方式。

---

## 工厂模式

**核心思想：**

```text
对象创建
从
对象使用
中分离
```

调用者不直接关心具体对象如何创建。

```text
调用者
   ↓
 工厂
   ↓
具体对象
```

---

### 简单工厂

简单工厂严格来说：

```text
不是 GoF 23 种设计模式之一
```

但非常常见。

```java
interface Animal {
    void speak();
}

class Dog implements Animal {
    public void speak() {
        System.out.println("dog");
    }
}

class Cat implements Animal {
    public void speak() {
        System.out.println("cat");
    }
}
```

工厂：

```java
class AnimalFactory {

    public static Animal create(String type) {

        if ("dog".equals(type)) {
            return new Dog();
        }

        if ("cat".equals(type)) {
            return new Cat();
        }

        throw new IllegalArgumentException();
    }
}
```

使用：

```java
Animal animal = AnimalFactory.create("dog");
```

优点：

```text
对象创建集中管理
调用者与具体实现解耦
```

缺点：

```text
新增类型通常需要修改工厂
容易违反开闭原则
```

---

### 工厂方法模式

核心：

```text
一个产品对应一个工厂
```

```java
interface AnimalFactory {
    Animal create();
}

class DogFactory implements AnimalFactory {

    public Animal create() {
        return new Dog();
    }
}
```

新增产品时：

```text
增加产品类
+
增加对应工厂
```

而不是修改原工厂。

---

### 抽象工厂模式

用于创建：

```text
一组相关对象
```

例如：

```text
WindowsButton
WindowsTextBox

MacButton
MacTextBox
```

可以抽象为：

```text
WindowsFactory
MacFactory
```

适合：

```text
产品族
```

场景。

---

### 三种工厂简单区别

```text
简单工厂
→ 一个工厂根据参数创建不同对象

工厂方法
→ 一个产品对应一个工厂

抽象工厂
→ 一个工厂创建一组相关产品
```

---

## 建造者模式

**Builder Pattern**

核心：

> 将复杂对象的创建过程拆开，逐步设置参数，最后统一创建对象。

例如：

```java
User user = new User.Builder()
        .name("Tom")
        .age(20)
        .email("tom@example.com")
        .build();
```

适合：

```text
构造参数很多
可选参数很多
对象创建过程复杂
```

相比超长构造函数：

```java
new User("Tom", 20, "...", "...", true, ...)
```

Builder：

```text
可读性更高
参数含义更清晰
```

常见：

```text
StringBuilder
Lombok @Builder
各种配置对象
```

---

# 结构型模式

## 代理模式

**定义：**

> 通过代理对象控制对目标对象的访问，并在不修改目标业务代码的情况下增强功能。

结构：

```text
调用者
   ↓
代理对象
   ↓
目标对象
```

可以增加：

```text
日志
事务
权限
缓存
性能统计
```

---

## 静态代理

```java
interface UserService {
    void save();
}
```

目标对象：

```java
class UserServiceImpl implements UserService {

    public void save() {
        System.out.println("save");
    }
}
```

代理：

```java
class UserServiceProxy implements UserService {

    private final UserService target;

    public UserServiceProxy(UserService target) {
        this.target = target;
    }

    public void save() {

        System.out.println("before");

        target.save();

        System.out.println("after");
    }
}
```

缺点：

```text
每个目标类通常需要手写代理类
扩展成本高
```

---

## JDK 动态代理

核心：

```text
基于接口
```

关键类：

```text
Proxy
InvocationHandler
```

```java
UserService proxy =
        (UserService) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                (p, method, args) -> {

                    System.out.println("before");

                    Object result = method.invoke(target, args);

                    System.out.println("after");

                    return result;
                });
```

流程：

```text
接口
 ↓
Proxy
 ↓
InvocationHandler
 ↓
invoke()
 ↓
目标方法
```

---

## CGLIB 动态代理

核心：

```text
基于继承
```

运行时生成：

```text
目标类的子类
```

然后重写目标方法进行增强。

核心组件：

```text
Enhancer
MethodInterceptor
```

概念流程：

```text
目标类
  ↑
  │ extends
CGLIB代理类
  ↓
intercept()
  ↓
增强逻辑
  ↓
原方法
```

限制：

```text
final 类不能被继承
final 方法不能被重写
private 方法不能通过子类重写
```

---

## JDK 与 CGLIB

```text
JDK 动态代理
→ 基于接口

CGLIB
→ 基于继承
→ 不要求目标类实现接口
```

Spring AOP 会根据情况：

```text
使用 JDK 动态代理
或
基于类的代理
```

---

## 装饰器模式

**Decorator Pattern**

定义：

> 在不修改原类的情况下，动态给对象增加功能。

结构：

```text
基础对象
   ↓
装饰器A
   ↓
装饰器B
   ↓
增强后的对象
```

Java IO 是经典应用：

```java
BufferedReader reader =
        new BufferedReader(
                new InputStreamReader(
                        new FileInputStream("a.txt")));
```

可以理解为：

```text
FileInputStream
      ↓
InputStreamReader
      ↓
BufferedReader
```

逐层增加功能。

---

### 装饰器与代理区别

二者结构相似，但侧重点不同：

```text
代理模式
→ 控制对象访问
→ 调用者通常关注代理对象

装饰器模式
→ 动态增强对象能力
→ 可以多层组合
```

例如：

```text
代理
→ 权限、事务、远程访问

装饰器
→ 功能叠加
```

---

## 适配器模式

**Adapter Pattern**

作用：

> 将一个类已有的接口转换成调用者期望的接口。

可以理解为：

```text
原接口
  ↓
适配器
  ↓
目标接口
```

类似：

```text
电源转接头
```

例如：

```java
interface TypeC {
    void connectTypeC();
}

class Usb {
    void connectUsb() {
        System.out.println("USB");
    }
}

class Adapter implements TypeC {

    private final Usb usb;

    Adapter(Usb usb) {
        this.usb = usb;
    }

    public void connectTypeC() {
        usb.connectUsb();
    }
}
```

使用：

```java
TypeC typeC = new Adapter(new Usb());
```

典型思想：

```text
接口不兼容
→ 中间加一层转换
```

Spring MVC 中：

```text
HandlerAdapter
```

就体现了适配器思想。

---

# 行为型模式

## 策略模式

**Strategy Pattern**

定义：

> 将不同算法或业务策略分别封装，并让它们可以相互替换。

典型场景：

```text
大量 if...else
```

例如：

```java
if ("wechat".equals(type)) {
    ...
} else if ("alipay".equals(type)) {
    ...
} else if ("card".equals(type)) {
    ...
}
```

可以抽象：

```java
interface PayStrategy {
    void pay();
}
```

实现：

```java
class WechatPay implements PayStrategy {

    public void pay() {
        System.out.println("微信支付");
    }
}

class AliPay implements PayStrategy {

    public void pay() {
        System.out.println("支付宝支付");
    }
}
```

使用：

```java
class PayContext {

    private final PayStrategy strategy;

    PayContext(PayStrategy strategy) {
        this.strategy = strategy;
    }

    void pay() {
        strategy.pay();
    }
}
```

优点：

```text
消除大量分支
新增策略方便
符合开闭原则
```

---

## 模板方法模式

**Template Method**

定义：

> 父类定义算法整体流程，把部分具体步骤交给子类实现。

例如：

```java
abstract class AbstractTask {

    public final void execute() {
        before();
        doTask();
        after();
    }

    protected void before() {
        System.out.println("before");
    }

    protected abstract void doTask();

    protected void after() {
        System.out.println("after");
    }
}
```

子类：

```java
class TaskA extends AbstractTask {

    protected void doTask() {
        System.out.println("Task A");
    }
}
```

核心：

```text
父类
→ 定义流程骨架

子类
→ 实现变化步骤
```

常用于：

```text
固定流程
部分步骤可变
```

---

## 观察者模式

**Observer Pattern**

定义：

> 一个对象状态发生变化时，通知依赖它的多个观察者。

结构：

```text
被观察者
   ↓
状态变化
   ↓
通知
 ┌─┼─┐
 ↓ ↓ ↓
A  B  C
```

典型场景：

```text
事件监听
消息通知
GUI 事件
Spring 事件机制
```

例如：

```text
订单创建
   ↓
发布事件
   ↓
积分服务
短信服务
日志服务
```

这样：

```text
订单模块
```

不需要直接依赖所有后续处理模块。

---

### 观察者与发布订阅

二者思想非常接近，但常见区别：

```text
观察者模式
→ 被观察者通常直接维护观察者列表

发布订阅
→ 发布者和订阅者之间通常存在事件中心 / 消息中间件
```

因此：

```text
发布订阅
```

可以看作更解耦的一种事件通信方式。

---

## 责任链模式

**Chain of Responsibility**

定义：

> 将多个处理器连接成链，请求沿链传递，每个处理器决定是否处理以及是否继续传递。

结构：

```text
Request
   ↓
Handler A
   ↓
Handler B
   ↓
Handler C
```

例如：

```java
abstract class Handler {

    protected Handler next;

    public void setNext(Handler next) {
        this.next = next;
    }

    public abstract void handle(Request request);
}
```

常见场景：

```text
过滤器链
拦截器链
权限校验
审批流程
Spring Security Filter Chain
```

优点：

```text
处理逻辑解耦
可以灵活增加、删除、调整处理节点
```

---

# 高频对比

## 工厂模式 vs 建造者模式

```text
工厂模式
→ 关注创建哪一种对象

建造者模式
→ 关注一个复杂对象如何一步步创建
```

---

## 代理模式 vs 装饰器模式

```text
代理
→ 控制访问
→ 权限、事务、远程代理

装饰器
→ 增强功能
→ 可以多层叠加
```

---

## 策略模式 vs 模板方法模式

```text
策略模式
→ 不同算法封装成不同对象
→ 组合关系

模板方法
→ 父类固定流程，子类改变部分步骤
→ 继承关系
```

---

## 观察者模式 vs 发布订阅

```text
观察者
→ 被观察者通常直接通知观察者

发布订阅
→ 通常通过事件中心 / 消息中间件解耦
```

---

# Java 高频设计模式

Java / Spring 面试优先掌握：

```text
单例模式
工厂模式
代理模式
装饰器模式
适配器模式
策略模式
模板方法模式
观察者模式
责任链模式
```

对应常见应用：

```text
单例
→ Spring Bean 默认 singleton

工厂
→ BeanFactory、FactoryBean

代理
→ Spring AOP、事务

装饰器
→ Java IO

适配器
→ Spring MVC HandlerAdapter

策略
→ Comparator、业务策略选择

模板方法
→ 各类固定流程模板

观察者
→ Spring ApplicationEvent

责任链
→ Filter、Interceptor、Spring Security
```

---

# 速记

```text
创建型
├── 单例：一个实例
├── 工厂：统一创建对象
└── 建造者：分步骤创建复杂对象

结构型
├── 代理：控制访问 / 增强
├── 装饰器：动态叠加功能
└── 适配器：接口转换

行为型
├── 策略：算法可替换
├── 模板方法：固定骨架，步骤可变
├── 观察者：状态变化自动通知
└── 责任链：请求沿处理链传递
```
