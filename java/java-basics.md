# Java 基础

## 基础概念

**Java 的特点：** 平台无关性、面向对象、自动内存管理、多线程。

**JVM、JRE、JDK：**
- JVM：负责执行 Java 字节码。
- JRE = JVM + Java 运行类库。
- JDK = JRE + 编译、调试等开发工具。

**Java 程序运行过程：**

```text
.java
  ↓ javac
.class 字节码
  ↓
JVM 执行
```

---

## 数据类型

**8 种基本数据类型：**

```text
byte(1B)、short(2B)、int(4B)、long(8B)
float(4B)、double(8B)、char(2B)
boolean（JLS 未规定固定大小）
```

整数默认：

```text
int
```

小数默认：

```text
double
```

**基本类型与引用类型：**

```text
基本类型变量 → 直接保存值
引用类型变量 → 保存对象引用
```

**类型转换：**

```text
小范围 → 大范围
自动转换

大范围 → 小范围
强制转换
可能发生溢出或精度损失
```

### instanceof

`instanceof` 用于判断对象是否属于某个类型或其子类型。

```java
Animal animal = new Dog();

animal instanceof Animal; // true
animal instanceof Dog;    // true
```

```java
null instanceof String; // false
```

`instanceof` 只能用于引用类型判断，基本类型不能直接使用：

```java
int i = 1;

// 编译错误
// i instanceof Object;
```

---

### 包装类

```text
byte    → Byte
short   → Short
int     → Integer
long    → Long
float   → Float
double  → Double
char    → Character
boolean → Boolean
```

**装箱与拆箱：**

```text
基本类型 → 包装类：装箱
包装类 → 基本类型：拆箱
```

Java 5 后支持自动装箱和自动拆箱：

```java
Integer a = 10; // 自动装箱
int b = a;      // 自动拆箱
```

本质类似：

```java
Integer a = Integer.valueOf(10);
int b = a.intValue();
```

### Integer 缓存

`Integer.valueOf()` 默认缓存：

```text
-128 ~ 127
```

因此该范围内自动装箱通常会复用对象：

```java
Integer a = 127;
Integer b = 127;

a == b; // true

Integer c = 128;
Integer d = 128;

c == d; // false
```

包装类比较数值优先使用：

```java
equals()
```

不要依赖：

```java
==
```

---

### BigDecimal

`float`、`double` 使用二进制浮点数表示，某些十进制小数不能被精确表示。

涉及：

```text
金额
高精度计算
```

通常使用：

```text
BigDecimal
```

推荐：

```java
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = BigDecimal.valueOf(0.1);
```

不推荐：

```java
BigDecimal a = new BigDecimal(0.1);
```

数值比较通常使用：

```java
a.compareTo(b) == 0
```

`equals()` 还会比较 `scale`：

```java
new BigDecimal("1.0")
        .equals(new BigDecimal("1.00")); // false
```

---

## 方法

### 值传递

**Java 只有值传递。**

```text
基本类型
→ 传递值的副本

引用类型
→ 传递引用值的副本
```

Java 不存在真正意义上的引用传递。

---

### 静态方法与实例方法

**静态方法：**

```text
属于类
只能直接访问静态成员
不能直接使用实例成员、this、super
```

推荐：

```java
ClassName.method();
```

**实例方法：**

```text
属于对象
可以访问实例成员和静态成员
```

---

### 重载与重写

**重载 Overload：**

```text
同一个类
方法名相同
参数列表不同
编译期多态
```

**重写 Override：**

```text
父子类
方法名相同
参数列表相同
运行期多态
```

| 对比 | 重载 | 重写 |
| --- | --- | --- |
| 位置 | 同一个类 | 父子类 |
| 参数列表 | 必须不同 | 必须相同 |
| 发生时期 | 编译期 | 运行期 |

重写时：

```text
子类方法访问权限不能比父类更严格
```

返回值可以：

```text
相同
或使用协变返回类型
```

---

## 访问修饰符

| 修饰符 | 本类 | 同包 | 不同包子类 | 其他包 |
| --- | --- | --- | --- | --- |
| `public` | ✓ | ✓ | ✓ | ✓ |
| `protected` | ✓ | ✓ | ✓ | × |
| 默认 | ✓ | ✓ | × | × |
| `private` | ✓ | × | × | × |

这里的默认访问权限表示：

```text
没有显式写访问修饰符
```

速记：

```text
public
→ 到处都能访问

protected
→ 本类 + 同包 + 子类

default
→ 本类 + 同包

private
→ 仅本类
```

---

## 关键字

### final

`final` 修饰类：

```text
不能被继承
```

`final` 修饰方法：

```text
不能被子类重写
```

`final` 修饰变量：

```text
只能赋值一次
```

修饰引用时：

```text
引用不能重新指向其他对象
但对象内部状态仍可能改变
```

```java
final Person p = new Person();

p.name = "Tom";      // 可以
// p = new Person(); // 不可以
```

因此：

```text
final 类 ≠ 不可变类
```

例如 String 的不可变性由：

```text
final
内部状态封装
不暴露修改内部状态的方法
等机制
```

共同保证。

---

### static

`static` 将成员与：

```text
类
```

关联，而不是与具体对象关联。

**静态变量：**

```text
属于类
所有对象共享一份
```

**静态方法：**

```text
属于类
推荐使用 类名.方法() 调用
```

**静态代码块：**

```text
类初始化阶段执行
通常只执行一次
早于对象构造方法
```

---

### 类加载与 static 变量赋值

JVM 类生命周期简化：

```text
加载
 ↓
链接
 ├─ 验证
 ├─ 准备
 └─ 解析
 ↓
初始化
```

**准备阶段：**

普通 `static` 变量：

```text
分配内存
赋类型默认值
```

例如：

```java
static int num = 10;
```

准备阶段：

```text
num = 0
```

**初始化阶段：**

执行：

```text
static 显式赋值
static 代码块
```

因此：

```text
num = 10
```

静态代码块属于：

```text
初始化阶段
```

而不是加载阶段。

**static final 编译期常量：**

```java
static final int NUM = 100;
```

如果属于编译期常量并带有 `ConstantValue` 属性：

```text
准备阶段可能直接赋常量值
```

---

### this

`this`：

```text
指向当前对象
```

可以：

```text
访问当前对象成员
通过 this() 调用本类其他构造方法
```

---

### super

`super`：

```text
表示当前对象中的父类部分
```

可以：

```text
访问父类成员
通过 super() 调用父类构造方法
```

---

## Java 8 新特性

### Lambda

用于简化函数式接口的匿名内部类写法：

```java
Runnable r = () -> System.out.println("hello");
```

---

### 函数式接口

只有一个：

```text
抽象方法
```

的接口。

可以使用：

```java
@FunctionalInterface
```

标记。

常见：

```text
Consumer
Supplier
Function
Predicate
```

---

### 方法引用

```java
list.forEach(x -> System.out.println(x));
```

可以简化为：

```java
list.forEach(System.out::println);
```

常见形式：

```text
类名::静态方法
对象::实例方法
类名::实例方法
类名::new
```

---

### Stream API

用于对集合等数据进行：

```text
声明式处理
```

常见操作：

```text
filter
map
sorted
distinct
collect
reduce
forEach
```

```java
list.stream()
    .filter(x -> x > 2)
    .map(x -> x * 2)
    .forEach(System.out::println);
```

常见分类：

```text
中间操作
→ filter、map、sorted

终止操作
→ collect、reduce、forEach
```

Stream 本身通常：

```text
不修改原集合
```

---

### Optional

用于显式表达：

```text
值可能为空
```

可以减少部分直接操作 `null` 带来的空指针风险。

通常更适合作为：

```text
方法返回值
```

不应滥用。

---

### 接口 default / static 方法

Java 8 接口可以定义：

```text
default 方法
static 方法
```

```java
interface A {

    default void test() {
        System.out.println("default");
    }

    static void hello() {
        System.out.println("hello");
    }
}
```

---

### 新日期时间 API

Java 8 引入：

```text
java.time
```

常见类：

```text
LocalDate
LocalTime
LocalDateTime
Instant
Duration
Period
DateTimeFormatter
```

相比传统：

```text
Date
Calendar
```

API 更清晰，也更符合不可变对象设计。

---

# 面向对象

## 封装、继承与多态

### 封装

将：

```text
数据
+
操作数据的方法
```

封装在类内部。

通过：

```text
访问修饰符
```

控制外部访问。

作用：

```text
隐藏实现细节
降低耦合
提高可维护性
```

---

### 继承

子类通过：

```java
extends
```

复用父类成员和行为。

Java 类：

```text
只支持单继承
```

接口：

```text
可以多实现
```

---

### 多态

父类引用可以指向子类对象：

```java
Animal animal = new Dog();
```

调用被重写的方法：

```java
animal.sound();
```

运行期根据：

```text
对象真实类型
```

决定执行哪个实现。

可以记为：

```text
父类引用指向子类对象
+
方法重写
+
动态绑定
```

其中：

```text
编译时类型：Animal
运行时类型：Dog
```

---

### 向上转型

子类对象转为父类引用：

```java
Animal animal = new Dog();
```

自动完成。

常用于：

```text
多态
```

---

### 向下转型

父类引用转为子类引用：

```java
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;
}
```

需要：

```text
强制类型转换
```

通常先使用：

```text
instanceof
```

判断。

错误转换可能抛出：

```text
ClassCastException
```

---

## 构造方法

构造方法：

```text
方法名与类名相同
没有返回值类型
创建对象时调用
```

例如：

```java
class User {

    User() {
    }
}
```

如果类中：

```text
没有显式定义任何构造方法
```

编译器会提供：

```text
默认无参构造方法
```

一旦显式定义构造方法：

```text
默认无参构造不会再自动生成
```

例如：

```java
class User {

    User(String name) {
    }
}
```

此时：

```java
// new User(); // 编译错误
```

---

### 父子类构造顺序

子类构造方法会直接或间接调用：

```text
父类构造方法
```

如果没有显式写：

```java
super();
```

编译器通常自动添加：

```java
super();
```

因此创建子类对象时：

```text
先执行父类构造
再执行子类构造
```

---

## 抽象类与接口

### 抽象类

使用：

```java
abstract
```

修饰。

特点：

```text
不能直接实例化
可以有抽象方法
可以有普通方法
可以有实例变量
可以有构造方法
```

记忆：

```text
有抽象方法
→ 类必须是抽象类

抽象类
→ 不一定有抽象方法
```

---

### 接口

接口主要用于定义：

```text
行为规范
```

一个类：

```text
可以实现多个接口
```

接口成员随 Java 版本扩展：

```text
Java 7 及以前
→ 常量、抽象方法

Java 8
→ default 方法、static 方法

Java 9
→ private 方法
```

---

### 接口成员默认修饰符

接口字段默认：

```text
public static final
```

例如：

```java
interface A {
    int NUM = 10;
}
```

实际相当于：

```java
public static final int NUM = 10;
```

接口抽象方法默认：

```text
public abstract
```

---

### 抽象类与接口区别

| 对比 | 抽象类 | 接口 |
| --- | --- | --- |
| 关系 | `extends` | `implements` |
| 数量 | 类只能继承一个 | 可以实现多个 |
| 构造方法 | 有 | 无 |
| 普通实例变量 | 可以有 | 不可以 |
| 侧重点 | 描述“是什么” | 描述“能做什么” |

例如：

```text
Dog 是 Animal
→ 继承

Dog 能 Run、Swim
→ 接口
```

---

# Object 与常用类

## Object

所有 Java 类都直接或间接继承：

```text
Object
```

常见方法：

```text
equals()
hashCode()
toString()
getClass()
clone()
wait()
notify()
notifyAll()
```

`finalize()` 已废弃。

---

### == 与 equals

`==`：

```text
基本类型
→ 比较值

引用类型
→ 比较是否指向同一个对象
```

`equals()`：

```text
Object 默认实现类似 ==
很多类会重写 equals() 比较内容
```

```java
String a = new String("abc");
String b = new String("abc");

a == b;      // false
a.equals(b); // true
```

---

### equals 与 hashCode

```text
equals 相等
→ hashCode 必须相等

hashCode 相等
→ equals 不一定相等
```

因此重写 `equals()` 时：

```text
通常也要重写 hashCode()
```

否则可能影响：

```text
HashMap
HashSet
```

等哈希容器。

---

### toString

Object 默认 `toString()` 形式类似：

```text
类名@十六进制哈希值
```

通常根据需要重写。

---

### getClass

`getClass()` 返回对象：

```text
运行时真实类型
```

对应的 `Class` 对象。

```java
Animal animal = new Dog();

animal.getClass(); // Dog 对应的 Class 对象
```

---

### clone

`Object.clone()` 默认进行：

```text
浅拷贝
```

`Cloneable` 是：

```text
标记接口
```

本身没有 `clone()` 方法。

使用 `Object.clone()` 通常需要：

```text
实现 Cloneable
并重写或暴露 clone()
```

否则可能抛出：

```text
CloneNotSupportedException
```

---

### wait / notify / notifyAll

用于线程间通信，定义在：

```text
Object
```

`wait()`：

```text
让当前线程等待
并释放当前持有的对象监视器
```

`notify()`：

```text
唤醒一个等待该对象监视器的线程
```

`notifyAll()`：

```text
唤醒所有等待该对象监视器的线程
```

通常必须在：

```text
synchronized
```

同步区域内调用。

---

## String

### String、StringBuilder、StringBuffer

`String`：

```text
不可变
```

`StringBuilder`：

```text
可变
线程不安全
性能较高
适合单线程字符串拼接
```

`StringBuffer`：

```text
可变
线程安全
性能通常低于 StringBuilder
```

---

### String 常量池

字符串字面量会优先复用字符串常量池中的对象：

```java
String a = "abc";
String b = "abc";

a == b; // true
```

因为：

```text
a、b 指向常量池中的同一个 "abc"
```

而：

```java
String c = new String("abc");

a == c;      // false
a.equals(c); // true
```

`new String()`：

```text
会显式创建新的 String 对象
```

---

### intern()

```java
String poolString = c.intern();
```

`intern()` 返回：

```text
字符串常量池中对应字符串的引用
```

---

# 泛型

## 泛型基础

**泛型：**

```text
参数化类型
```

即：

```text
把类型作为参数
```

例如：

```java
List<String> list = new ArrayList<>();
```

优点：

```text
类型安全
代码复用
减少强制类型转换
编译期发现类型错误
```

泛型不能直接使用基本类型：

```java
// List<int> list;   // 错误
List<Integer> list;  // 正确
```

---

### 泛型类、接口、方法

泛型类：

```java
class Box<T> {
    T value;
}
```

泛型接口：

```java
interface Container<T> {
}
```

泛型方法：

```java
public <T> T get(T value) {
    return value;
}
```

泛型方法自己的：

```text
<T>
```

写在：

```text
返回值类型之前
```

---

## 泛型原理

### 泛型不变性

虽然：

```text
Integer extends Number
```

但是：

```text
List<Integer>
```

并不是：

```text
List<Number>
```

的子类型。

```java
List<Integer> list = new ArrayList<>();

// 编译错误
// List<Number> nums = list;
```

---

### 类型擦除

Java 泛型主要通过：

```text
Type Erasure
类型擦除
```

实现。

泛型信息主要用于：

```text
编译期类型检查
```

运行期大部分泛型参数会被擦除。

例如：

```text
<T>
→ 通常擦除为 Object

<T extends Number>
→ 擦除为 Number
```

因此：

```text
List<String>
List<Integer>
```

运行时主要表现为：

```text
List
```

编译器会在需要的位置：

```text
自动插入类型转换
```

---

## 通配符

```text
<?>           任意类型

<? extends T> T 或 T 的子类

<? super T>   T 或 T 的父类
```

---

### extends

```text
适合读
生产数据
```

读取出来：

```text
至少可以安全当作 T
```

但通常不能安全写入具体对象：

```java
List<? extends Number> list =
        new ArrayList<Integer>();

// list.add(1); // 编译错误
```

除了：

```text
null
```

外，一般不能写入。

---

### super

```text
适合写
消费数据
```

```java
List<? super Integer> list =
        new ArrayList<Number>();

list.add(1); // 可以
```

读取时：

```text
通常只能安全当作 Object
```

---

### PECS

```text
Producer Extends
Consumer Super
```

即：

```text
生产者 → extends
消费者 → super
```

---

# 反射与注解

## 反射

**反射：**

> 程序在运行时动态获取类的信息，并动态创建对象、访问字段或调用方法的能力。

核心对象：

```text
Class
├── Field
├── Method
└── Constructor
```

---

### Class 对象

每个被 JVM 加载的类都有对应的：

```text
Class 对象
```

可以理解为：

```text
该类在 JVM 中的运行时说明书
```

```java
Dog dog = new Dog();

Class<?> clazz = dog.getClass();
```

其中：

```text
dog
→ Dog 实例对象

clazz
→ 描述 Dog 类本身的 Class 对象
```

---

### 获取 Class 对象

常见三种方式：

```java
Dog.class;

dog.getClass();

Class.forName("com.example.Dog");
```

同名类通过：

```text
全限定类名
```

区分。

例如：

```text
java.util.Date
java.sql.Date
```

更严格地说，一个类在 JVM 中的身份由：

```text
全限定类名
+
类加载器
```

共同决定。

---

### 反射操作

获取字段：

```java
clazz.getDeclaredFields();
```

获取方法：

```java
clazz.getDeclaredMethods();
```

获取构造方法：

```java
clazz.getDeclaredConstructors();
```

反射还可以动态：

```text
创建对象
读取 / 修改字段
调用方法
获取父类和接口信息
读取运行时注解
```

---

### getXxx 与 getDeclaredXxx

可以简单记：

```text
getXxx()
→ 更偏向获取 public 成员
→ 会考虑继承

getDeclaredXxx()
→ 获取当前类声明的成员
→ 包括 private、protected 等
→ 不包含继承而来的成员
```

---

### 反射优缺点

优点：

```text
动态
灵活
解耦
```

缺点：

```text
性能低于直接调用
可读性较差
可能破坏封装
```

---

### 反射应用

经典示例：

```java
Class.forName("com.mysql.cj.jdbc.Driver");
```

早期 JDBC 常通过：

```text
Class.forName()
```

加载数据库驱动。

现代 JDBC 4.0+ 支持：

```text
SPI 自动发现和加载驱动
```

因此通常不再需要手动 `Class.forName()`。

它仍然是理解：

```text
反射
动态加载
```

的经典例子。

反射常用于：

```text
Spring IoC
MyBatis
JDBC
注解解析
动态代理
```

---

## 注解

### 注解基础

**注解 Annotation：**

> 给类、方法、字段等代码元素添加元数据信息。

注解本身通常：

```text
不会直接执行逻辑
```

而由：

```text
编译器
JVM
框架
反射程序
```

解析。

自定义注解：

```java
public @interface MyAnnotation {
}
```

注解本质上是一种特殊接口。

运行时获得的注解实例：

```text
通常由运行时机制生成代理对象
```

---

### 元注解

**@Retention：**

决定注解保留到什么时候：

```text
SOURCE
→ 只存在源码

CLASS
→ 保留在 .class
→ 运行时不能通过普通反射读取

RUNTIME
→ 运行时仍存在
→ 可以通过反射读取
```

**@Target：**

决定注解可以标在哪里。

常见值：

```text
TYPE
→ 类、接口

METHOD
→ 方法

FIELD
→ 字段

PARAMETER
→ 参数

CONSTRUCTOR
→ 构造方法
```

**@Documented：**

```text
表示该注解是否包含在 Javadoc 中
```

**@Inherited：**

```text
表示类上的注解是否可以被子类继承
```

主要对：

```text
类级注解
```

有效。

---

# 异常

## 异常体系

```text
Throwable
├── Error
│   └── Unchecked
│
└── Exception
    ├── RuntimeException
    │   └── Unchecked
    │
    └── 其他 Exception
        └── Checked
```

注意：

```text
Checked / Unchecked
```

是：

```text
异常分类概念
```

不是 Java 中真正的类层级名称。

---

### Error

JVM 或系统级严重问题：

```text
OutOfMemoryError
StackOverflowError
```

通常：

```text
不主动捕获处理
```

---

### Checked Exception

编译器强制要求：

```text
捕获
或
声明抛出
```

例如：

```text
IOException
SQLException
```

---

### Unchecked Exception

主要包括：

```text
RuntimeException 及其子类
Error 及其子类
```

编译器：

```text
不强制捕获或声明
```

常见：

```text
NullPointerException
ArrayIndexOutOfBoundsException
ClassCastException
IllegalArgumentException
```

---

## 异常处理

常见结构：

```java
try {
    // 可能发生异常的代码
} catch (Exception e) {
    // 异常处理
} finally {
    // 通常用于资源清理
}
```

`finally`：

```text
通常都会执行
```

但例如：

```text
JVM 直接退出
进程被强制终止
```

等特殊情况下可能不会执行。

---

### throw 与 throws

`throw`：

```text
主动抛出一个异常对象
```

```java
throw new RuntimeException("错误");
```

`throws`：

```text
声明方法可能抛出的异常
```

```java
void test() throws IOException {
}
```

---

### 自定义异常

根据业务需要继承：

```text
Exception
或
RuntimeException
```

```java
class BusinessException extends RuntimeException {

    public BusinessException(String message) {
        super(message);
    }
}
```

---

### try-with-resources

Java 7 引入。

用于自动关闭实现：

```text
AutoCloseable
```

的资源。

```java
try (BufferedReader br =
             new BufferedReader(
                     new FileReader("a.txt"))) {

    System.out.println(br.readLine());
}
```

代码块结束后：

```text
资源会自动关闭
```

相比手动：

```text
finally + close()
```

通常更简洁、更安全。

常见资源：

```text
InputStream
OutputStream
Reader
Writer
JDBC Connection
Statement
ResultSet
```

---

# 内部类

## 内部类分类

```text
                 Java 嵌套类
                      │
          ┌───────────┴───────────┐
          │                       │
      定义在类中               定义在方法中
          │                       │
     ┌────┴─────┐            ┌────┴────┐
     │          │            │         │
成员内部类   static嵌套类   局部内部类   匿名内部类
```

---

### 成员内部类

依赖：

```text
外部类对象
```

可以直接访问外部类：

```text
实例成员
private 成员
```

创建：

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

成员内部类会隐式持有：

```text
外部类对象引用
```

---

### static 嵌套类

不依赖：

```text
外部类对象
```

创建：

```java
Outer.Inner inner = new Outer.Inner();
```

可以直接访问：

```text
外部类 static 成员
```

不能直接访问：

```text
外部类实例成员
```

---

### 局部内部类

定义在：

```text
方法
或
代码块
```

只能在：

```text
当前作用域
```

使用。

---

### 匿名内部类

没有显式类名。

本质：

```text
定义一个匿名子类或实现类
+
立即创建对象
```

```java
Animal animal = new Animal() {

    @Override
    void sound() {
        System.out.println("汪汪");
    }
};
```

---

## 内部类关键点

### effectively final

局部内部类和匿名内部类访问局部变量时：

```text
该变量必须是 final
或 effectively final
```

例如：

```java
int num = 10;

class Inner {

    void test() {
        System.out.println(num);
    }
}
```

只要 `num` 后续没有被重新赋值：

```text
num 就属于 effectively final
```

---

### 匿名内部类与 Lambda

```java
Runnable r = new Runnable() {

    @Override
    public void run() {
        System.out.println("hello");
    }
};
```

函数式接口场景下可以简化为：

```java
Runnable r = () -> System.out.println("hello");
```

但：

```text
Lambda 不等价于所有匿名内部类
```

只有：

```text
函数式接口
```

场景可以直接使用 Lambda。
