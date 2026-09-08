# Java 基础

## 基础概念

**Java 的特点：** 平台无关性、面向对象、自动内存管理、多线程。

**JVM、JRE、JDK：**

- JVM：执行 Java 字节码。
- JRE = JVM + Java 运行类库。
- JDK = JRE + 编译、调试等开发工具。

**Java 程序运行：**

```text
.java --javac--> .class 字节码 --> JVM 执行
```

---

## 数据类型

**8 种基本类型：** `byte(1B)`、`short(2B)`、`int(4B)`、`long(8B)`、`float(4B)`、`double(8B)`、`char(2B)`、`boolean`（JLS 未规定固定大小）。

- 整数默认 `int`，小数默认 `double`。
- **基本类型与引用类型**：基本类型变量直接保存值；引用类型变量保存对象引用。
- **类型转换**：小范围 → 大范围自动转换；大范围 → 小范围需要强转，可能溢出或损失精度。

### instanceof

`instanceof` 判断对象是否属于某个类型或其子类型，只能用于引用类型；`null instanceof 任意引用类型` 都为 `false`。

```java
Animal animal = new Dog();

animal instanceof Animal; // true
animal instanceof Dog;    // true
null instanceof String;   // false

int i = 1;
// i instanceof Object;   // 编译错误
```

### 包装类

基本类型对应包装类：

```text
byte → Byte        short → Short
int → Integer      long → Long
float → Float      double → Double
char → Character   boolean → Boolean
```

- 基本类型 → 包装类：装箱。
- 包装类 → 基本类型：拆箱。
- Java 5 后支持自动装箱 / 拆箱。

```java
Integer a = 10; // 自动装箱，本质类似 Integer.valueOf(10)
int b = a;      // 自动拆箱，本质类似 a.intValue()
```

### Integer 缓存

`Integer.valueOf()` 默认缓存 `-128 ~ 127`，该范围内自动装箱通常复用对象。

```java
Integer a = 127;
Integer b = 127;
Integer c = 128;
Integer d = 128;

a == b; // true
c == d; // false
```

包装类比较数值优先使用 `equals()`，不要依赖 `==`。

### BigDecimal

`float`、`double` 是二进制浮点数，某些十进制小数无法精确表示；金额、高精度计算通常使用 `BigDecimal`。

```java
BigDecimal a = new BigDecimal("0.1"); // 推荐
BigDecimal b = BigDecimal.valueOf(0.1);

// 不推荐
BigDecimal c = new BigDecimal(0.1);
```

- 数值比较通常使用 `compareTo()`。
- `equals()` 除了比较数值，还比较 `scale`。

```java
new BigDecimal("1.0")
        .equals(new BigDecimal("1.00")); // false
```

---

## 方法

### 值传递

**Java 只有值传递。**

- 基本类型：传递值的副本。
- 引用类型：传递“引用值”的副本。

因此 Java 不存在真正意义上的引用传递。

### 静态方法与实例方法

- **静态方法**：属于类，只能直接访问静态成员，不能直接使用实例成员、`this`、`super`，推荐 `ClassName.method()` 调用。
- **实例方法**：属于对象，可以访问实例成员和静态成员。

### 重载与重写

| 对比     | 重载 Overload | 重写 Override |
| -------- | ------------- | ------------- |
| 位置     | 同一个类      | 父子类        |
| 方法名   | 相同          | 相同          |
| 参数列表 | 必须不同      | 必须相同      |
| 时期     | 编译期        | 运行期        |

重写时：

- 子类方法访问权限不能比父类更严格。
- 返回值可相同，也可使用协变返回类型。

---

## 访问修饰符

| 修饰符      | 本类 | 同包 | 不同包子类 | 其他包 |
| ----------- | ---- | ---- | ---------- | ------ |
| `public`    | ✓    | ✓    | ✓          | ✓      |
| `protected` | ✓    | ✓    | ✓          | ×      |
| 默认        | ✓    | ✓    | ×          | ×      |
| `private`   | ✓    | ×    | ×          | ×      |

速记：

- `public`：到处可访问。
- `protected`：本类 + 同包 + 子类。
- 默认：本类 + 同包。
- `private`：仅本类。

---

## 关键字

### final

- 修饰类：不能被继承。
- 修饰方法：不能被子类重写。
- 修饰变量：只能赋值一次。
- 修饰引用：引用不能重新指向其他对象，但对象内部状态仍可改变。

```java
final Person p = new Person();
p.name = "Tom";      // 可以
// p = new Person(); // 不可以
```

`final 类 ≠ 不可变类`。例如 String 的不可变性还依赖内部状态封装、不提供修改状态的方法等。

### static

- **静态变量**：属于类，所有对象共享一份。
- **静态方法**：属于类，推荐 `类名.方法()` 调用。
- **静态代码块**：类初始化阶段执行，通常只执行一次，早于对象构造。

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

普通静态变量：

```java
static int num = 10;
```

- **准备阶段**：分配内存并赋类型默认值，此时 `num = 0`。
- **初始化阶段**：执行显式静态赋值和 `static` 代码块，此时 `num = 10`。

对于带 `ConstantValue` 属性的编译期常量：

```java
static final int NUM = 100;
```

准备阶段可能直接赋常量值。

### this 与 super

- `this`：指向当前对象，可访问当前对象成员；`this()` 调用本类其他构造方法。
- `super`：用于访问父类成员；`super()` 调用父类构造方法。

---

## Java 8 新特性

### Lambda、函数式接口与方法引用

**函数式接口**：只有一个抽象方法的接口，可使用 `@FunctionalInterface` 标记。

常见：

- `Consumer`
- `Supplier`
- `Function`
- `Predicate`

Lambda 用于简化函数式接口的匿名内部类写法：

```java
Runnable r = () -> System.out.println("hello");
```

方法引用：

```java
list.forEach(System.out::println);
```

常见形式：

- `类名::静态方法`
- `对象::实例方法`
- `类名::实例方法`
- `类名::new`

### Stream API

Stream 用于对集合等数据进行声明式处理，通常不修改原集合。

常见中间操作：`filter`、`map`、`sorted`、`distinct`。  
常见终止操作：`collect`、`reduce`、`forEach`。

```java
list.stream()
    .filter(x -> x > 2)
    .map(x -> x * 2)
    .forEach(System.out::println);
```

### Optional

`Optional` 用于显式表达“值可能为空”，可减少部分直接操作 `null` 的风险，通常更适合作为方法返回值，不应滥用。

### 接口 default / static 方法

Java 8 接口可以定义 `default` 和 `static` 方法。

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

### 新日期时间 API

Java 8 引入 `java.time`，常见类：

`LocalDate`、`LocalTime`、`LocalDateTime`、`Instant`、`Duration`、`Period`、`DateTimeFormatter`。

相比传统 `Date`、`Calendar`，API 更清晰，也更符合不可变对象设计。

---

# 面向对象

## 封装、继承与多态

### 封装

把数据和操作数据的方法封装在类内部，通过访问修饰符控制外部访问。

作用：隐藏实现细节、降低耦合、提高可维护性。

### 继承

子类通过 `extends` 复用父类成员和行为：

- Java 类只支持单继承。
- 一个类可以实现多个接口。

### 多态

父类引用可以指向子类对象：

```java
Animal animal = new Dog();
animal.sound();
```

调用被重写方法时，运行期根据对象真实类型决定执行哪个实现。

```text
编译时类型：Animal
运行时类型：Dog
```

多态核心可以记为：

```text
父类引用指向子类对象
+
方法重写
+
动态绑定
```

### 向上转型与向下转型

- **向上转型**：子类对象 → 父类引用，自动完成。
- **向下转型**：父类引用 → 子类引用，需要强转，错误转换可能抛 `ClassCastException`。

```java
Animal animal = new Dog();

if (animal instanceof Dog) {
    Dog dog = (Dog) animal;
}
```

---

## 构造方法

构造方法：

- 方法名与类名相同。
- 没有返回值类型。
- 创建对象时调用。

```java
class User {
    User() {
    }
}
```

如果没有显式定义任何构造方法，编译器会提供默认无参构造；一旦显式定义了构造方法，默认无参构造不再自动生成。

```java
class User {
    User(String name) {
    }
}

// new User(); // 编译错误
```

### 父子类构造顺序

子类构造方法会直接或间接调用父类构造方法；若没有显式写 `super()`，编译器通常自动添加。

```text
创建子类对象
→ 先执行父类构造
→ 再执行子类构造
```

---

## 抽象类与接口

### 抽象类

使用 `abstract` 修饰：

- 不能直接实例化。
- 可以有抽象方法、普通方法、实例变量和构造方法。
- 有抽象方法的类必须是抽象类；抽象类不一定有抽象方法。

### 接口

接口主要用于定义行为规范，一个类可以实现多个接口。

接口能力随 Java 版本扩展：

- Java 7 及以前：常量、抽象方法。
- Java 8：增加 `default`、`static` 方法。
- Java 9：增加 `private` 方法。

接口字段默认 `public static final`；接口抽象方法默认 `public abstract`。

```java
interface A {
    int NUM = 10; // public static final
}
```

### 抽象类与接口区别

| 对比         | 抽象类         | 接口           |
| ------------ | -------------- | -------------- |
| 关系         | `extends`      | `implements`   |
| 数量         | 类只能继承一个 | 可以实现多个   |
| 构造方法     | 有             | 无             |
| 普通实例变量 | 可以有         | 不可以         |
| 侧重点       | 描述“是什么”   | 描述“能做什么” |

记：

```text
Dog 是 Animal
→ 继承

Dog 能 Run、Swim
→ 接口
```

---

# Object 与常用类

## Object

所有 Java 类都直接或间接继承 `Object`。

常见方法：`equals()`、`hashCode()`、`toString()`、`getClass()`、`clone()`、`wait()`、`notify()`、`notifyAll()`；`finalize()` 已废弃。

### == 与 equals

- `==`：基本类型比较值；引用类型比较是否指向同一个对象。
- `equals()`：Object 默认实现类似 `==`，很多类会重写为比较内容。

```java
String a = new String("abc");
String b = new String("abc");

a == b;      // false
a.equals(b); // true
```

### equals 与 hashCode

规则：

```text
equals 相等
→ hashCode 必须相等

hashCode 相等
→ equals 不一定相等
```

因此重写 `equals()` 时通常也要重写 `hashCode()`，否则可能影响 `HashMap`、`HashSet` 等哈希容器。

### toString

Object 默认 `toString()` 形式类似：

```text
类名@十六进制哈希值
```

通常根据需要重写。

### getClass

`getClass()` 返回对象运行时真实类型对应的 `Class` 对象。

```java
Animal animal = new Dog();
animal.getClass(); // Dog 对应的 Class
```

### clone

`Object.clone()` 默认执行浅拷贝。`Cloneable` 是标记接口，本身没有 `clone()` 方法。

使用 `Object.clone()` 通常需要实现 `Cloneable` 并重写或暴露 `clone()`，否则可能抛 `CloneNotSupportedException`。

### wait / notify / notifyAll

这些方法定义在 `Object` 中，通常必须在 `synchronized` 区域内调用：

- `wait()`：当前线程等待，并释放持有的对象监视器。
- `notify()`：唤醒一个等待该监视器的线程。
- `notifyAll()`：唤醒所有等待该监视器的线程。

---

## String

### String、StringBuilder、StringBuffer

- `String`：不可变。
- `StringBuilder`：可变、线程不安全、性能较高，适合单线程字符串拼接。
- `StringBuffer`：可变、线程安全，性能通常低于 `StringBuilder`。

### String 常量池

字符串字面量优先复用字符串常量池中的对象：

```java
String a = "abc";
String b = "abc";

a == b; // true
```

`new String()` 会显式创建新的 String 对象：

```java
String c = new String("abc");

a == c;      // false
a.equals(c); // true
```

### intern()

`intern()` 返回字符串常量池中对应字符串的引用：

```java
String poolString = c.intern();
```

---

# 泛型

## 泛型基础与原理

**泛型**：参数化类型，即把类型作为参数。

优点：类型安全、代码复用、减少强制转换、编译期发现类型错误。

```java
List<String> list = new ArrayList<>();
```

泛型不能直接使用基本类型：

```java
// List<int> list;   // 错误
List<Integer> list;  // 正确
```

### 泛型类、接口、方法

```java
class Box<T> {
    T value;
}

interface Container<T> {
}

public <T> T get(T value) {
    return value;
}
```

泛型方法自己的 `<T>` 写在返回值类型之前。

### 泛型不变性

虽然 `Integer extends Number`，但 `List<Integer>` 不是 `List<Number>` 的子类型。

```java
List<Integer> list = new ArrayList<>();
// List<Number> nums = list; // 编译错误
```

### 类型擦除

Java 泛型主要通过 **Type Erasure** 实现：

- 泛型信息主要用于编译期类型检查。
- 运行期大部分泛型参数被擦除。
- `<T>` 通常擦除为 `Object`。
- `<T extends Number>` 通常擦除为 `Number`。
- 编译器会在需要的位置自动插入类型转换。

因此 `List<String>`、`List<Integer>` 运行时主要表现为 `List`。

---

## 通配符与 PECS

```text
<?>            任意类型
<? extends T>  T 或 T 的子类
<? super T>    T 或 T 的父类
```

- `<? extends T>`：适合读，读取结果至少能安全当作 `T`；通常不能安全写入具体对象，除 `null` 外。
- `<? super T>`：适合写，可以安全写入 `T` 或其子类；读取时通常只能安全当作 `Object`。

```java
List<? extends Number> a = new ArrayList<Integer>();
// a.add(1); // 编译错误

List<? super Integer> b = new ArrayList<Number>();
b.add(1); // 可以
```

**PECS：**

- Producer Extends：生产者使用 `extends`。
- Consumer Super：消费者使用 `super`。

---

# 反射与注解

## 反射

**反射**：程序在运行时动态获取类信息，并动态创建对象、访问字段或调用方法的能力。

核心对象：

```text
Class
├── Field
├── Method
└── Constructor
```

### Class 对象

每个被 JVM 加载的类都有对应的 `Class` 对象，可以理解为“类在 JVM 中的运行时说明书”。

```java
Dog dog = new Dog();
Class<?> clazz = dog.getClass();
```

这里：

- `dog`：Dog 实例对象。
- `clazz`：描述 Dog 类本身的 Class 对象。

### 获取 Class 对象

```java
Dog.class;
dog.getClass();
Class.forName("com.example.Dog");
```

同名类通过全限定类名区分，如 `java.util.Date` 和 `java.sql.Date`。

更严格地说，一个类在 JVM 中的身份由：

```text
全限定类名 + 定义它的类加载器
```

共同决定。

### 反射操作

```java
clazz.getDeclaredFields();
clazz.getDeclaredMethods();
clazz.getDeclaredConstructors();
```

反射还可动态创建对象、读取 / 修改字段、调用方法、读取父类和接口信息、读取运行时注解。

### getXxx 与 getDeclaredXxx

- `getXxx()`：更偏向获取 `public` 成员，并会考虑继承。
- `getDeclaredXxx()`：获取当前类声明的成员，包括 `private`、`protected` 等，不包含继承成员。

### 优缺点与应用

- **优点**：动态、灵活、解耦。
- **缺点**：性能低于直接调用、可读性较差、可能破坏封装。
- **常见应用**：Spring IoC、MyBatis、JDBC、注解解析、动态代理。

早期 JDBC 常见：

```java
Class.forName("com.mysql.cj.jdbc.Driver");
```

现代 JDBC 4.0+ 支持 SPI 自动发现和加载驱动，通常不再需要手动 `Class.forName()`。

---

## 注解

**Annotation**：给类、方法、字段等代码元素添加元数据信息。

注解本身通常不直接执行逻辑，而由编译器、JVM、框架或反射程序解析。

```java
public @interface MyAnnotation {
}
```

注解本质上是一种特殊接口，运行时获得的注解实例通常由运行时机制生成代理对象。

### 元注解

- `@Retention`：决定注解保留到什么时候。
  - `SOURCE`：只存在源码。
  - `CLASS`：保留在 `.class`，运行时不能通过普通反射读取。
  - `RUNTIME`：运行时保留，可通过反射读取。
- `@Target`：决定注解可标记的位置，如 `TYPE`、`METHOD`、`FIELD`、`PARAMETER`、`CONSTRUCTOR`。
- `@Documented`：是否包含在 Javadoc 中。
- `@Inherited`：类上的注解是否可被子类继承，主要对类级注解有效。

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

`Checked / Unchecked` 是异常分类概念，不是真正的 Java 类层级名称。

### Error

JVM 或系统级严重问题，如 `OutOfMemoryError`、`StackOverflowError`，通常不主动捕获处理。

### Checked Exception

编译器强制要求捕获或声明抛出，例如 `IOException`、`SQLException`。

### Unchecked Exception

主要包括 `RuntimeException` 及其子类、`Error` 及其子类，编译器不强制捕获或声明。

常见：`NullPointerException`、`ArrayIndexOutOfBoundsException`、`ClassCastException`、`IllegalArgumentException`。

---

## 异常处理

```java
try {
    // 可能发生异常
} catch (Exception e) {
    // 异常处理
} finally {
    // 通常用于资源清理
}
```

`finally` 通常都会执行，但 JVM 直接退出、进程被强制终止等特殊情况下可能不会执行。

### throw 与 throws

- `throw`：主动抛出一个异常对象。
- `throws`：声明方法可能抛出的异常。

```java
throw new RuntimeException("错误");

void test() throws IOException {
}
```

### 自定义异常

根据业务需要继承 `Exception` 或 `RuntimeException`。

```java
class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}
```

### try-with-resources

Java 7 引入，用于自动关闭实现 `AutoCloseable` 的资源。

```java
try (BufferedReader br =
         new BufferedReader(new FileReader("a.txt"))) {
    System.out.println(br.readLine());
}
```

代码块结束后资源自动关闭，比手动 `finally + close()` 更简洁、安全。

常见资源：`InputStream`、`OutputStream`、`Reader`、`Writer`、JDBC `Connection`、`Statement`、`ResultSet`。

---

# 内部类

## 内部类分类

```text
Java 嵌套类
├── 定义在类中
│   ├── 成员内部类
│   └── static 嵌套类
│
└── 定义在方法 / 代码块中
    ├── 局部内部类
    └── 匿名内部类
```

### 成员内部类

依赖外部类对象，可直接访问外部类实例成员和 `private` 成员，并会隐式持有外部类对象引用。

```java
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
```

### static 嵌套类

不依赖外部类对象，可直接访问外部类 `static` 成员，不能直接访问外部类实例成员。

```java
Outer.Inner inner = new Outer.Inner();
```

### 局部内部类

定义在方法或代码块中，只能在当前作用域使用。

### 匿名内部类

没有显式类名，本质是“定义匿名子类 / 实现类并立即创建对象”。

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

局部内部类和匿名内部类访问局部变量时，该变量必须是 `final` 或 effectively final。

```java
int num = 10;

class Inner {
    void test() {
        System.out.println(num);
    }
}
```

只要 `num` 后续没有重新赋值，它就是 effectively final。

### 匿名内部类与 Lambda

函数式接口场景：

```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("hello");
    }
};
```

可简化为：

```java
Runnable r = () -> System.out.println("hello");
```

但 **Lambda 不等价于所有匿名内部类**，只有函数式接口场景可以直接这样替换。
