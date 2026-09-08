# JVM
## JVM 基础
### JVM 的作用
JVM（Java Virtual Machine）负责运行 Java 字节码。

Java 程序执行过程：

```text
.java
  ↓ javac 编译
.class 字节码
  ↓
JVM
  ↓
解释执行 / JIT 编译
  ↓
机器指令
```

Java 的跨平台能力可以概括为：**一次编译，到处运行**。

即不同操作系统提供对应 JVM，而 Java 程序运行统一的字节码。

### JVM、JRE、JDK
```text
JVM
→ Java 虚拟机

JRE
→ JVM + Java 运行类库

JDK
→ JRE + 编译、调试等开发工具
```

### JVM 整体结构
可以简单理解为：

```text
                JVM
                 │
        ┌────────┼────────┐
        │        │        │
        ↓        ↓        ↓
   类加载系统  运行时数据区  执行引擎
                          │
                    ┌─────┴─────┐
                    ↓           ↓
                 解释执行       JIT
```

后文按 6 条主线展开：**JVM 内存结构 → Java 对象与内存 → 垃圾回收 → 类加载机制 → JVM 执行机制 → JVM 性能排查**。

# JVM 内存结构
## 运行时数据区
JVM 运行时数据区主要分为：线程私有 + 线程共享

结构：

```text
JVM 运行时数据区
│
├── 线程私有
│   ├── 程序计数器
│   ├── Java 虚拟机栈
│   └── 本地方法栈
│
└── 线程共享
    ├── Java 堆
    └── 方法区
```

## 线程私有区域
### 程序计数器
Program Counter Register。

作用：记录当前线程正在执行的字节码指令位置

线程切换后：依靠程序计数器恢复执行位置

特点：每个线程独立拥有、线程私有

JVM 规范中：唯一没有规定 OutOfMemoryError 的运行时数据区域

### Java 虚拟机栈
Java Virtual Machine Stack。

每个线程：独立拥有自己的虚拟机栈

每调用一个 Java 方法：创建一个栈帧

栈帧主要保存：局部变量表、操作数栈、动态链接、方法返回地址

- **局部变量表**：存放方法执行过程中使用的局部变量和参数，比如 `int a`、对象引用、`this` 等。
- **操作数栈**：JVM 执行字节码时的临时计算区域，很多加减乘除、方法调用参数传递都会先压入操作数栈再运算。
- **动态链接**：把字节码中的**符号引用**解析到实际要调用的方法、字段等运行时引用，支持方法调用等操作。
- **方法返回地址**：记录当前方法执行结束后，应该回到调用者的哪条指令继续执行。

结构：

```text
Java 虚拟机栈

┌─────────────┐
│ methodB 栈帧 │
├─────────────┤
│ methodA 栈帧 │
├─────────────┤
│ main 栈帧    │
└─────────────┘
```

方法调用：栈帧入栈

方法结束：栈帧出栈

递归调用过深可能导致：`StackOverflowError`

某些 JVM 实现中，虚拟机栈无法继续扩展时也可能出现：`OutOfMemoryError`

### 本地方法栈
Native Method Stack。

主要服务于 `native` 本地方法，例如由 C / C++ 实现的方法。

作用与 Java 虚拟机栈类似。

## 线程共享区域
### Java 堆
Heap。

主要存放：对象实例、数组

特点：线程共享、GC 管理的主要区域

传统分代 GC 中通常可以理解为：

```text
Heap
│
├── 新生代
│   ├── Eden
│   ├── Survivor 0
│   └── Survivor 1
│
└── 老年代
```

注意：Eden / Survivor / 老年代

属于典型分代 GC 的实现思想。

并不是所有现代垃圾收集器都采用完全相同的物理布局。

### 方法区
Method Area。

方法区是：JVM 规范定义的逻辑区域

主要存储：类信息、字段信息、方法信息、运行时常量池

结构：

```text
方法区
├── 类信息
├── 字段信息
├── 方法信息
└── 运行时常量池
```

HotSpot 中：JDK 7 及以前 → 主要由 PermGen 永久代实现；JDK 8 以后 → 主要由 Metaspace 元空间实现

Metaspace：使用本地内存 Native Memory

而不是 Java 堆。

记忆：方法区 → JVM 规范中的逻辑概念；PermGen / Metaspace → HotSpot 的具体实现

## 常量池
常见三个概念：Class 文件常量池、运行时常量池、字符串常量池

三者：不是同一个东西

### Class 文件常量池
位于：.class 文件

保存：字面量、类名、方法名、字段名、符号引用

流程：

```text
.java
 ↓ 编译
.class
 ↓
Class 文件常量池
```

### 运行时常量池
类加载之后：Class 文件常量池中的相关信息

进入 JVM 的运行时结构。

```text
Class 文件常量池
      ↓ 类加载
运行时常量池
```

JVM 规范上：运行时常量池属于方法区

### 字符串常量池
StringTable。

用于管理：被驻留的字符串

例如：

```java
String s1 = "abc";
String s2 = "abc";

System.out.println(s1 == s2); // true
```

HotSpot 从 JDK 7 开始：字符串常量池相关的 String 对象位于 Java 堆中

注意：运行时常量池 ≠ 字符串常量池

## JVM 内存结构与 JMM
这两个概念容易混淆。

### JVM 运行时数据区
描述：Java 程序运行时、数据存放在哪里

例如：堆、虚拟机栈、方法区、程序计数器、本地方法栈

### JMM
JMM：Java Memory Model、Java 内存模型

描述：多线程环境下共享变量如何读写、线程之间如何保证原子性、可见性、有序性

因此：JVM Runtime Data Area ≠ JMM

JMM 属于：Java 并发基础

建议放到：`juc-basics.md`

中继续学习。

# Java 对象与内存
## 对象创建过程
执行：Person p = new Person();

可以简化理解为：

```text
类加载检查
   ↓
分配内存
   ↓
初始化零值
   ↓
设置对象头
   ↓
执行 <init> 构造方法
   ↓
得到对象
```

**类加载检查：**
JVM 遇到：`new`

指令时，会先检查：对应类是否已经被加载、解析、初始化

如果没有：先执行类加载过程

**分配内存：**
对象需要的内存大小在类加载完成后基本可以确定。

常见分配方式：**指针碰撞**或**空闲列表**，具体取决于堆是否规整及垃圾收集器实现。

有关。

**初始化零值：**
对象分配完成后：实例字段先赋类型默认值

例如：int     → 0；boolean → false；引用    → null

因此 Java 对象即使没有显式初始化：实例字段也会有默认值

**设置对象头：**
JVM 会设置：对象所属类、GC 信息、锁相关信息、identity hashCode 等运行时信息

**执行构造方法：**
最后执行：`<init>`

即实例初始化逻辑和构造方法。

此时对象才按照程序代码完成初始化。

## 对象内存布局
普通 Java 对象通常由：

```text
Java 对象
│
├── 对象头 Header
│   ├── Mark Word
│   └── Klass Pointer
│
├── 实例数据 Instance Data
│
└── 对齐填充 Padding
```

数组对象：对象头中还会额外保存数组长度

### 对象头
**Mark Word：**
保存对象运行时状态信息。

常见内容：identity hashCode、GC 分代年龄、锁状态、GC 相关标记

在常见 64 位 HotSpot JVM 中：`Mark Word = 8B`

旧版本 HotSpot 还涉及：无锁、偏向锁、轻量级锁、重量级锁

注意：偏向锁属于旧版 HotSpot 优化机制、新版本 JDK 已逐步禁用并移除

不建议死记旧版本具体位布局。

**Klass Pointer：**
指向对象所属类的类元数据。

JVM 通过它判断：对象属于哪个类

64 位 HotSpot 中通常：开启压缩类指针 → 4B；未开启压缩 → 8B

压缩类指针：`-XX:+UseCompressedClassPointers`

普通对象引用压缩：`-XX:+UseCompressedOops`

不要混淆：UseCompressedOops → 压缩普通对象引用；UseCompressedClassPointers → 压缩 Klass Pointer

### 实例数据
Instance Data。

保存：对象真正的成员变量

例如：

```java
class Person {
    int age;
    long id;
}
```

其中：age、id

属于实例数据。

### 对齐填充
Padding。

HotSpot 通常要求对象：按照一定字节边界对齐

常见情况下按 **8 字节**对齐。

如果对象实际大小不能满足对齐要求：补充无意义字节

例如常见：64 位 HotSpot + 压缩类指针

环境中：Integer i = 10;

常见 64 位 HotSpot + 压缩类指针环境下，`Integer` 对象可粗略理解为：

```text
Mark Word      8B
Klass Pointer  4B
int value      4B
-----------------
总计          16B
```

## 对象引用
Java 中常见四种引用：强引用、软引用、弱引用、虚引用

**强引用：**
普通对象引用：Object obj = new Object();

只要强引用仍然存在：对象通常不会被 GC

**软引用：**
SoftReference。

特点：内存不足时可能被回收。

曾常用于：内存敏感缓存

**弱引用：**
WeakReference。

特点：只剩弱引用时，下一次 GC 通常就可以回收。

典型：`WeakHashMap`

**虚引用：**
PhantomReference。

特点：不能通过它直接获取对象，主要用于跟踪对象被回收后的状态。

通常需要配合：`ReferenceQueue`

使用。

## 对象访问方式
JVM 规范没有强制规定具体对象访问实现。

常见两种对象访问方式：**句柄**、**直接指针**。

**句柄：**
```text
reference
   ↓
句柄池
 ├── 对象地址
 └── 类元数据地址
```

优点：对象移动时、reference 本身通常不用变化

**直接指针：**
```text
reference
   ↓
对象
 ├── 实例数据
 └── Klass Pointer → 类元数据
```

特点：访问速度更直接

HotSpot 常见实现主要采用：直接指针

思想。

# 垃圾回收
垃圾回收可以分成三个问题：

```text
谁是垃圾？
   ↓
垃圾对象判断

垃圾怎么清？
   ↓
垃圾回收算法

谁来执行？
   ↓
垃圾回收器
```

## 垃圾对象判断
主要有：引用计数法、可达性分析

### 引用计数法
给对象维护：引用次数

没有任何引用时：认为对象可以回收

最大问题：难以处理循环引用

例如：

```text
A → B
↑   ↓
└───┘
```

即使 A、B 已经不能被程序访问：它们仍然互相引用

因此引用计数可能都不为 0。

HotSpot：不使用引用计数法作为主要垃圾判断方式

### 可达性分析
从：`GC Roots`

出发向下搜索。

能够到达：存活

无法到达：可以被回收

例如：

```text
GC Roots
   │
   ├── A → B → C
   │
   └── D

E → F
```

其中：A / B / C / D → 可从 GC Roots 到达，存活；E / F → 无法从 GC Roots 到达，可以回收

常见 GC Roots：虚拟机栈中引用的对象、类静态字段引用的对象、JNI 引用的对象、JVM 内部使用的一些对象

一句话：

> **能从 GC Roots 到达的对象存活，无法到达的对象可以被回收。**

## 垃圾回收算法
### 标记-清除
流程：

```text
标记垃圾对象
   ↓
直接清理
```

```text
GC 前：[活][垃圾][活][垃圾][活]
GC 后：[活][空][活][空][活]
```

优点：实现简单

缺点：容易产生内存碎片

### 复制算法
把存活对象：复制到另一块内存

然后：整体清理原区域

例如：

```text
From：

[活][垃圾][活][垃圾]

         ↓

To：

[活][活][          ]
```

优点：没有内存碎片、分配速度较快

缺点：需要额外空间、存活对象多时复制成本高

适合：对象死亡率较高区域

### 标记-整理
流程：

```text
标记存活对象
   ↓
将存活对象向一端移动
   ↓
清理剩余空间
```

```text
整理前：[活][垃圾][活][垃圾][活]
整理后：[活][活][活][空][空]
```

优点：不会产生大量内存碎片

缺点：需要移动对象、成本较高

适合：对象存活率较高区域

### 分代收集思想
根据对象生命周期不同：采用不同回收策略

例如：

```text
年轻对象
生命周期短
死亡率高
   ↓
新生代
   ↓
适合复制类算法
```

```text
长期存活对象
   ↓
老年代
   ↓
适合整理类算法
```

对象通常：先在年轻代分配、经历 GC 后仍存活、年龄增加、满足条件后进入老年代

`MaxTenuringThreshold`：影响对象晋升年龄

但实际晋升还会受到：Survivor 空间、动态年龄判断、对象大小、GC 策略

等因素影响。

不能简单理解：年龄达到 15 → 一定进入老年代

## GC 类型与 STW
### Young GC / Minor GC
主要回收：年轻代

现代收集器中常称：`Young GC`

### Major GC
通常指：老年代相关回收

但不同资料和 JVM 实现中：Major GC 的含义可能不完全统一

不要单独死记名字。

### Full GC
通常会涉及：整个 Java 堆、并可能伴随类元数据等区域处理

Full GC 往往：停顿时间较长

实际排查时需要关注：为什么频繁 Full GC

### STW
Stop-The-World。

表示：JVM 暂停应用程序用户线程

以完成某些 GC 工作。

GC 优化的重要目标之一：降低 STW 停顿时间

## 垃圾回收器
| 收集器 | 核心特点 | 主要目标 | 备注 |
| --- | --- | --- | --- |
| Serial | 单线程 GC | 简单、小内存 | STW |
| Parallel | 多线程 GC | 高吞吐量 | STW |
| CMS | 并发标记清除 | 降低停顿 | 已淘汰 |
| G1 | Region 化管理 | 延迟与吞吐平衡 | JDK 9 起默认 |
| ZGC | 大量 GC 工作并发 | 超低停顿 | 现代低延迟 GC |

### CMS
CMS：`Concurrent Mark Sweep`

主要目标：降低老年代 GC 停顿时间

核心阶段：

```text
初始标记
   ↓
并发标记
   ↓
重新标记
   ↓
并发清除
```

**初始标记：**
`STW`

只标记：GC Roots 直接关联对象

速度较快。

**并发标记：**
与用户线程：并发执行

继续遍历整个对象图。

**重新标记：**
`STW`

修正并发标记期间：用户线程修改引用关系导致的遗漏

**并发清除：**
与用户线程：并发执行

清理垃圾对象。

主要问题：内存碎片、并发阶段占用 CPU、浮动垃圾

因为 CMS 主要采用：标记-清除

所以容易产生：内存碎片

CMS 已在较新的 JDK 中移除：主要作为历史知识学习

### G1
G1：`Garbage First`

最大特点：将堆划分为多个大小相近的 Region

例如：

```text
Heap

┌────┬────┬────┬────┐
│ R1 │ R2 │ R3 │ R4 │
├────┼────┼────┼────┤
│ R5 │ R6 │ R7 │ R8 │
└────┴────┴────┴────┘
```

不同 Region 可以动态承担：Eden、Survivor、Old、Humongous

等角色。

核心过程可简化为：

```text
初始标记
   ↓
并发标记
   ↓
重新标记
   ↓
筛选并回收 Region
```

**初始标记：**
`STW`

标记：GC Roots 直接关联对象

**并发标记：**
与用户线程：并发遍历对象图

**重新标记：**
`STW`

处理并发标记期间：引用关系变化

**筛选回收：**
根据 Region：垃圾数量、回收价值、停顿目标

等信息：优先选择值得回收的 Region

并将存活对象：复制到其他 Region

核心思想：

> **优先回收垃圾较多、收益较高的 Region，因此叫 Garbage First。**

主要特点：Region 化管理、可设置期望最大停顿时间、减少内存碎片、兼顾吞吐量和响应时间

### ZGC
ZGC 的主要目标：

> **在大堆场景下仍保持极低的 GC 停顿。**

大量 GC 工作：可以与用户线程并发执行

主要机制：指针元数据 / 染色指针、Load Barrier、并发标记、并发对象转移、并发引用修正

**Load Barrier：**
读屏障。

用户线程读取对象引用时：JVM 插入额外检查逻辑

例如对象被 GC 从旧地址搬到新地址：

```text
读取旧引用
    ↓
Load Barrier
    ↓
发现对象已经搬迁
    ↓
找到新地址
    ↓
继续访问
```

因此 ZGC 可以：在用户线程运行时搬迁对象

**染色指针 / 指针元数据：**
在对象引用中：携带部分 GC 状态信息

帮助 GC：高效判断引用状态

具体指针位布局：随 JDK 版本变化

不建议死记固定多少位。

重点记：引用携带 GC 元数据 + 屏障机制

整体：

```text
标记存活对象
   ↓
选择需要回收区域
   ↓
并发搬迁对象
   ↓
通过屏障修正引用
```

# 类加载机制
## 类的生命周期
一个类从进入 JVM 到最终卸载，大致经历：

```text
加载
 ↓
链接
 ├── 验证
 ├── 准备
 └── 解析
 ↓
初始化
 ↓
使用
 ↓
卸载
```

**加载：**
Loading。

作用：把 .class 字节码读入 JVM、生成对应 Class 对象

**验证：**
Verification。

检查：字节码是否合法、是否安全、是否符合 JVM 规范

**准备：**
Preparation。

为类的：static 变量

分配内存并设置默认值。

例如：static int a = 10;

准备阶段：`a = 0`

注意：static final 编译期常量

可能因为 `ConstantValue`：在准备阶段直接赋常量值

**解析：**
Resolution。

将常量池中的：符号引用

转换为 JVM 可以实际使用的：直接引用

**初始化：**
Initialization。

执行：`<clinit>`

正式执行：static 变量显式赋值、static 代码块

例如：

```java
static int a = 10;

static {
    System.out.println("init");
}
```

初始化阶段：a = 10、执行 static 代码块

快速记忆：加载：读进来；验证：查安全；准备：static 给默认值；解析：符号引用 → 直接引用；初始化：真正执行 static 初始化

**使用：**
Using。

正常：创建对象、访问字段、调用方法

**卸载：**
Unloading。

当类以及加载它的：`ClassLoader`

满足回收条件时：相关类元数据可以被 JVM 卸载

## 类加载器
**JDK 8 及以前：**
```text
Bootstrap ClassLoader
        ↓
Extension ClassLoader
        ↓
Application ClassLoader
```

**JDK 9 以后：**
```text
Bootstrap ClassLoader
        ↓
Platform ClassLoader
        ↓
Application ClassLoader
```

**Bootstrap ClassLoader：**
启动类加载器。

主要负责加载：Java 核心类

例如：java.lang.Object、java.lang.String

**Extension / Platform ClassLoader：**
JDK 8 及以前：`Extension ClassLoader`

负责加载扩展类库。

JDK 9 模块系统以后：`Platform ClassLoader`

替代 Extension ClassLoader。

**Application ClassLoader：**
应用类加载器。

通常负责：classpath 中的用户类、第三方类库

## 双亲委派机制
核心规则：

> **类加载器收到类加载请求后，先委托父加载器尝试加载；父加载器无法完成时，当前加载器才自己尝试加载。**

流程：

```text
Application ClassLoader
        │
        ↓ 委托
Platform / Extension
        │
        ↓ 委托
Bootstrap ClassLoader
        │
      找不到
        ↓
Platform / Extension 尝试
        │
      找不到
        ↓
Application 尝试
```

主要作用：保护 Java 核心类、避免同一加载体系中的重复加载、复用已经加载过的类

例如自己定义：

```java
package java.lang;

public class String {
}
```

正常情况下：不能替换 JDK 自带 java.lang.String

因为核心类会优先由：`Bootstrap ClassLoader`

加载。

## 类的身份
JVM 判断两个类是否相同，不只看：全限定类名

还要看：定义它们的 ClassLoader

可以记：类的身份 = 全限定类名 + 定义它的 ClassLoader

## 打破双亲委派
双亲委派：不是不能打破

一些框架或容器会根据需要：自定义类加载逻辑

典型思想：SPI、应用服务器、模块化系统、热部署

学习时重点理解：为什么需要改变默认委派顺序

而不是只记具体实现。

# JVM 执行机制
## 字节码执行
JVM 执行字节码主要有两种方式：解释执行 + JIT 编译

**解释执行：**
解释器：逐条读取字节码、并执行对应操作

优点：启动快

缺点：重复执行热点代码时效率较低

**JIT 编译：**
JIT：Just-In-Time Compiler、即时编译器

JVM 运行过程中发现：某段代码被频繁执行

可能将其编译成：本地机器码

后续直接执行。

优点：热点代码性能高

因此 JVM 常采用：解释执行 + JIT 编译

混合模式。

## 热点代码与 JIT 优化
JVM 会根据运行时统计判断：哪些方法 / 循环执行频繁

这类代码称为：Hot Spot、热点代码

热点代码可能被：JIT 编译、优化

**方法内联：**
Method Inlining。

将短小方法调用：直接展开到调用位置

减少：方法调用开销

并为后续优化创造条件。

**逃逸分析：**
Escape Analysis。

JVM 分析对象是否：逃逸出当前方法、逃逸出当前线程

如果没有逃逸：可能进行进一步优化

**标量替换：**
如果一个对象没有真正需要整体存在：JIT 可能把对象拆成若干标量变量

减少：对象分配、内存访问

# JVM 性能排查
## 常见 JVM 参数
**堆大小：**
-Xms → 初始堆大小；-Xmx → 最大堆大小

例如：

```bash
-Xms2g -Xmx2g
```

**线程栈大小：**
`-Xss`

例如：

```bash
-Xss1m
```

栈过小可能更容易：`StackOverflowError`

栈过大则会增加：每个线程的内存占用

**GC 参数：**
不同收集器有不同参数。

例如：`-XX:+UseG1GC`

表示使用 G1。

性能调优时：不要只凭经验死改参数

应结合：GC 日志、应用负载、延迟、吞吐量、堆使用情况

分析。

## 常见内存问题
**StackOverflowError：**
常见原因：递归过深、方法调用层级过深

主要关联：Java 虚拟机栈

**Java heap space：**
典型：`OutOfMemoryError: Java heap space`

表示：堆中无法继续分配对象

常见原因：堆设置过小、对象创建过快、对象无法释放、内存泄漏

**Metaspace：**
典型：`OutOfMemoryError: Metaspace`

常见原因：加载类过多、动态生成大量类、ClassLoader 无法回收

## 常用 JVM 排查工具
**jps：**
查看：Java 进程

**jstack：**
查看：线程栈

常用于：死锁、线程阻塞、CPU 异常线程分析

**jmap：**
查看：堆信息、Heap Dump

常用于：内存泄漏分析、对象分布分析

**jstat：**
查看：GC、内存区使用、类加载、JIT 等统计信息

**jcmd：**
比较综合的 JVM 诊断工具。

可用于：线程信息、GC 信息、类统计、Heap Dump、JFR

等操作。

## JVM 排查基本思路
```text
程序慢 / CPU 高
       ↓
线程情况
       ↓
jstack / jcmd

内存持续增长
       ↓
堆情况
       ↓
jmap / Heap Dump

频繁 GC
       ↓
GC 日志 / jstat
       ↓
分析对象分配、晋升、堆大小

类加载异常
       ↓
ClassLoader / Metaspace
```

# 速记

```text
JVM
│
├── JVM 内存结构
│   ├── 程序计数器
│   ├── 虚拟机栈
│   ├── 本地方法栈
│   ├── 堆
│   └── 方法区 / 常量池
│
├── Java 对象与内存
│   ├── 对象创建
│   ├── 对象头
│   ├── 实例数据
│   ├── 对齐填充
│   └── 引用 / 对象访问
│
├── 垃圾回收
│   ├── 可达性分析
│   ├── 标记-清除 / 复制 / 标记-整理
│   ├── Young GC / Full GC / STW
│   └── CMS / G1 / ZGC
│
├── 类加载机制
│   ├── 加载 → 验证 → 准备 → 解析 → 初始化
│   ├── 类加载器
│   └── 双亲委派
│
├── JVM 执行机制
│   ├── 解释执行
│   ├── JIT
│   ├── 热点代码
│   └── 内联 / 逃逸分析 / 标量替换
│
└── JVM 性能排查
    ├── JVM 参数
    ├── StackOverflowError / OOM
    ├── jps / jstack / jmap / jstat / jcmd
    └── CPU / 内存 / GC / 类加载排查
```

**核心区分：**
- JVM 运行时数据区：描述数据在 JVM 运行时放在哪里。
- JMM：描述多线程共享变量如何正确读写，属于 JUC 范畴。

**常见错误定位：**
- `StackOverflowError` → 重点看虚拟机栈。
- `OutOfMemoryError: Java heap space` → 重点看堆。
- `OutOfMemoryError: Metaspace` → 重点看类元数据和 ClassLoader。

> **JVM 主要研究 Java 程序如何组织运行时内存和对象、如何回收垃圾、如何加载类、如何执行字节码，以及如何进行性能诊断。**
