# 并发基础

## 并发三要素

并发问题主要来自三个方面：

```text
可见性
原子性
有序性
```

**可见性：**

```text
一个线程修改共享变量
→ 其他线程不一定立即看到最新值
```

原因可能包括：

```text
CPU Cache
寄存器
写缓冲区
编译器优化
```

**原子性：**

一个操作要么完整执行，要么完全不执行。

例如：

```java
i++;
```

并不是原子操作，可以拆成：

```text
读取 i
  ↓
i + 1
  ↓
写回 i
```

多个线程并发执行时可能发生更新丢失。

**有序性：**

编译器和 CPU 为提高性能可能进行指令重排序。

单线程中通常不影响最终结果，但多线程下可能破坏程序正确性。

---

## JMM

JMM：

```text
Java Memory Model
Java 内存模型
```

主要描述：

> **多线程环境下，共享变量如何读写，以及一个线程对变量的修改什么时候能够被另一个线程看到。**

注意：

```text
JMM
≠
JVM 堆 / 栈 / 方法区
```

JMM 是并发编程中的抽象内存模型。

可以粗略理解为：

```text
             主内存
        ┌──────────────┐
        │ 共享变量 A    │
        │ 共享变量 B    │
        └──────────────┘
           ↑          ↑
           │          │
      ┌────┘          └────┐
      │                    │
线程1本地内存          线程2本地内存
┌───────────┐        ┌───────────┐
│ A 的副本   │        │ A 的副本   │
└───────────┘        └───────────┘
```

这里的：

```text
主内存
线程本地内存
```

都是抽象概念。

线程本地内存可能对应：

```text
CPU Cache
寄存器
写缓冲区
编译器优化产生的临时数据
```

JMM 重点解决：

```text
原子性
可见性
有序性
```

---

## Happens-Before

Happens-Before 用来描述：

> **如果操作 A happens-before 操作 B，那么 A 的执行结果对 B 可见，并且 A 在逻辑顺序上先于 B。**

常见规则：

```text
程序次序规则
锁规则
volatile 规则
线程启动规则
线程终止规则
传递性
```

### 程序次序规则

同一线程中：

```text
前面的操作
happens-before
后面的操作
```

### 锁规则

```text
对一个锁 unlock
happens-before
后续对同一把锁 lock
```

因此：

```text
线程 A 在同步块中的修改
→ 对之后获得同一把锁的线程 B 可见
```

### volatile 规则

```text
写 volatile 变量
happens-before
之后对该变量的读
```

### 线程启动规则

```text
Thread.start()
happens-before
新线程中的操作
```

### 线程终止规则

```text
线程中的所有操作
happens-before
其他线程从 join() 返回
```

### 传递性

如果：

```text
A happens-before B
B happens-before C
```

则：

```text
A happens-before C
```

---

## 线程安全实现方式

常见方式：

```text
线程安全
│
├── 互斥同步
│   ├── synchronized
│   └── Lock
│
├── 非阻塞同步
│   └── CAS
│
└── 避免共享状态
    ├── 栈封闭
    ├── ThreadLocal
    └── 无状态对象
```

一句话：

```text
加锁
→ 不让多个线程同时修改

CAS
→ 不阻塞线程，通过原子更新竞争

ThreadLocal / 不共享
→ 从根源减少共享状态
```

---

# 线程基础

## 线程创建

常见方式：

```text
继承 Thread
实现 Runnable
实现 Callable
使用线程池
```

### Runnable

```java
Runnable task = () -> System.out.println("run");

Thread thread = new Thread(task);
thread.start();
```

### Callable

`Callable` 可以返回结果，也可以抛出受检异常。

常与：

```text
FutureTask
线程池
```

配合。

```java
Callable<Integer> task = () -> 1 + 2;

FutureTask<Integer> futureTask = new FutureTask<>(task);

new Thread(futureTask).start();

Integer result = futureTask.get();
```

### 继承 Thread

```java
class MyThread extends Thread {

    @Override
    public void run() {
        System.out.println("run");
    }
}
```

一般更推荐：

```text
Runnable
Callable
```

原因：

```text
任务逻辑与线程对象解耦
Java 不支持多继承
更方便交给线程池执行
```

---

## 线程状态

Java 线程有 6 种状态：

```text
NEW
RUNNABLE
BLOCKED
WAITING
TIMED_WAITING
TERMINATED
```

| 状态 | 含义 |
| --- | --- |
| `NEW` | 创建后尚未启动 |
| `RUNNABLE` | 正在运行或等待 CPU 调度 |
| `BLOCKED` | 等待获取 `synchronized` Monitor |
| `WAITING` | 无限期等待 |
| `TIMED_WAITING` | 有超时时间的等待 |
| `TERMINATED` | 线程执行结束 |

常见变化：

```text
NEW
 ↓ start()
RUNNABLE
 ├── 等待 Monitor → BLOCKED
 ├── wait()/join()/park() → WAITING
 ├── sleep()/限时 wait()/限时 join() → TIMED_WAITING
 └── run() 结束 → TERMINATED
```

注意：

> `BLOCKED` 特指等待进入 `synchronized` 临界区。

AQS / ReentrantLock 等显式锁通常使用：

```text
LockSupport.park()
```

线程状态常表现为：

```text
WAITING
TIMED_WAITING
```

---

## 基础线程方法

### start()

```java
thread.start();
```

作用：

```text
启动新线程
```

最终由新线程执行：

```java
run()
```

直接调用：

```java
thread.run();
```

只是普通方法调用，不会启动新线程。

### sleep()

```java
Thread.sleep(1000);
```

特点：

```text
让当前线程暂停一段时间
进入 TIMED_WAITING
不会释放已经持有的锁
可以被 interrupt
```

### yield()

```java
Thread.yield();
```

作用：

```text
提示调度器当前线程愿意让出 CPU
```

只是建议，不保证一定发生线程切换。

### join()

```java
thread.start();
thread.join();
```

当前线程等待目标线程执行结束：

```text
目标线程执行完成
      ↓
join() 返回
      ↓
当前线程继续执行
```

---

## 线程中断

Java 中断是：

```text
协作式取消机制
```

不是强制杀死线程。

调用：

```java
thread.interrupt();
```

### 普通运行线程

```text
interrupt()
   ↓
设置中断标记
```

线程通常主动检查：

```java
while (!Thread.currentThread().isInterrupted()) {
    // 工作
}
```

### sleep / wait / join

线程处于：

```text
sleep
wait
join
```

等可中断等待时，被中断通常会抛出：

```java
InterruptedException
```

并且抛出异常时：

```text
中断标记通常被清除
```

如果希望继续向上传递中断语义，常见写法：

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### synchronized 等锁

等待普通 `synchronized` Monitor 时：

```text
不能通过 interrupt() 直接取消等待
```

### ReentrantLock

```java
lock.lock();
```

等待锁时不会因为中断而放弃获取锁。

```java
lock.lockInterruptibly();
```

等待锁时可以响应中断，并抛出：

```java
InterruptedException
```

### LockSupport.park()

线程被中断后：

```text
park() 会返回
不会抛 InterruptedException
中断标记仍然保留
```

### isInterrupted() 与 interrupted()

```java
thread.isInterrupted();
```

```text
检查指定线程中断状态
不会清除中断标记
```

```java
Thread.interrupted();
```

```text
检查当前线程中断状态
并清除中断标记
```

速记：

```text
isInterrupted()
→ 查，不清除

interrupted()
→ 查当前线程，并清除
```

---

# 线程同步

## synchronized

`synchronized` 用于保证多个线程对共享资源的互斥访问，同时提供：

```text
原子性
可见性
有序性保证
```

### 锁对象

普通同步方法：

```java
public synchronized void method() {
}
```

锁的是：

```text
this
```

静态同步方法：

```java
public static synchronized void method() {
}
```

锁的是：

```text
当前类的 Class 对象
```

同步代码块：

```java
synchronized (lock) {
    // 临界区
}
```

锁的是：

```text
指定的 lock 对象
```

### 底层原理

同步代码块主要通过字节码：

```text
monitorenter
monitorexit
```

实现 Monitor 的获取和释放。

同步方法则通过方法的同步访问标志，由 JVM 完成加锁。

### 可重入

`synchronized` 是可重入锁。

```java
synchronized void a() {
    b();
}

synchronized void b() {
}
```

同一个线程：

```text
进入 a()
 ↓
已经持有 this 锁
 ↓
再次进入 b()
 ↓
不会把自己阻塞
```

### synchronized 与 JMM

从 JMM 角度：

```text
unlock
happens-before
后续对同一把锁 lock
```

因此前一个线程同步块中的修改，对之后获得同一把锁的线程可见。

### 锁优化

旧版本 HotSpot 曾涉及：

```text
偏向锁
轻量级锁
重量级锁
自旋
```

这些属于 JVM 实现细节，并且不同 JDK 版本已经发生变化。

不建议把：

```text
偏向锁 → 轻量级锁 → 重量级锁
```

当成所有现代 JDK 都固定不变的模型。

---

## volatile

`volatile` 主要提供：

```text
可见性
+
一定程度的有序性保证
```

例如：

```java
volatile boolean flag = false;
```

线程 A：

```java
flag = true;
```

线程 B：

```java
while (!flag) {
}
```

B 可以及时看到 `flag` 的最新值。

### 可见性

从 Happens-Before 角度：

```text
写 volatile
happens-before
后续读 volatile
```

### 有序性

volatile 会通过：

```text
内存屏障
```

限制特定的指令重排序。

典型应用：

```text
双重检查单例 DCL
```

```java
private static volatile Singleton instance;
```

volatile 可避免：

```text
实例引用先被发布
对象却尚未完成初始化
```

这类重排序问题。

### 不保证复合操作原子性

```java
volatile int i = 0;

i++;
```

仍然包含：

```text
读
 ↓
加 1
 ↓
写
```

因此：

```text
volatile 不能保证 i++ 的原子性
```

常见场景：

```text
状态标志
一次写多次读
DCL 单例
```

---

## Lock 与 ReentrantLock

`Lock` 是 JUC 提供的显式锁接口。

常见实现：

```text
ReentrantLock
```

基本使用：

```java
lock.lock();

try {
    // 临界区
} finally {
    lock.unlock();
}
```

必须把 `unlock()` 放在：

```text
finally
```

中，避免异常导致锁无法释放。

### synchronized 与 ReentrantLock

| 对比 | synchronized | ReentrantLock |
| --- | --- | --- |
| 加解锁 | 自动 | 手动 `lock()/unlock()` |
| 可重入 | 是 | 是 |
| 等锁可中断 | 不支持 | `lockInterruptibly()` |
| 超时获取锁 | 不支持 | `tryLock()` |
| 公平模式 | 不提供配置 | 可公平 / 非公平 |
| 条件队列 | `wait/notify` | 多个 `Condition` |
| 实现层面 | JVM 语义支持 | JDK 类实现 |

### 公平锁与非公平锁

公平锁：

```text
等待更久的线程优先获得锁
```

非公平锁：

```text
新来的线程也可以直接竞争
```

ReentrantLock 默认：

```text
非公平锁
```

公平锁：

```java
ReentrantLock lock = new ReentrantLock(true);
```

一般：

```text
非公平锁
→ 吞吐量更高

公平锁
→ 获取顺序更可控
```

### lockInterruptibly()

```java
lock.lockInterruptibly();
```

等待锁期间：

```text
可以响应 interrupt()
```

### tryLock()

```java
if (lock.tryLock()) {
    try {
        // 临界区
    } finally {
        lock.unlock();
    }
}
```

获取不到锁：

```text
可以直接返回
```

也可以设置超时：

```java
lock.tryLock(1, TimeUnit.SECONDS);
```

---

## AQS

AQS：

```text
AbstractQueuedSynchronizer
抽象队列同步器
```

它是很多 JUC 同步器的基础框架。

例如：

```text
ReentrantLock
Semaphore
CountDownLatch
```

核心：

```text
AQS
│
├── state
│   → 同步状态
│
├── CAS
│   → 原子修改 state
│
└── 同步队列
    → 保存获取同步状态失败的线程
```

### state

AQS 内部维护：

```java
volatile int state;
```

不同同步器对 `state` 的含义不同：

```text
ReentrantLock
→ state 表示重入次数

Semaphore
→ state 表示可用许可数量

CountDownLatch
→ state 表示剩余计数
```

### 获取锁流程

以独占锁为例：

```text
尝试获取锁
   ↓
tryAcquire()
   ↓
成功？
├── 是 → 获得锁
│
└── 否
    ↓
加入同步队列
    ↓
检查前驱节点
    ↓
前驱是 head？
├── 是 → 再次尝试获取锁
│
└── 否 / 获取仍失败
    ↓
park()
    ↓
等待唤醒
    ↓
重新竞争
```

关键点：

> **加入同步队列后，通常只有前驱是 head 的节点，才最有资格再次尝试获取锁。**

同步队列可以粗略理解为：

```text
head
 ↓
Node1
 ↓
Node2
 ↓
Node3
```

第一个真正等待获取锁的节点通常是：

```text
head.next
```

### park / unpark

等待线程最终通常借助：

```text
LockSupport.park()
```

阻塞。

释放锁时：

```text
找到合适的后继节点
 ↓
LockSupport.unpark(thread)
 ↓
线程恢复运行
 ↓
再次竞争锁
```

### 释放锁流程

```text
释放锁
  ↓
tryRelease()
  ↓
同步状态释放成功
  ↓
检查同步队列
  ↓
唤醒后继节点
  ↓
后继线程重新竞争
```

### ReentrantLock 与 AQS

```text
ReentrantLock
    ↓
内部 Sync
    ↓
继承 AQS
    ↓
state + CAS + 同步队列
```

因此 ReentrantLock 加锁可以简化理解为：

```text
尝试获取 state
   ↓
失败
   ↓
进入 AQS 同步队列
   ↓
park()
   ↓
unpark()
   ↓
重新竞争
```

---

## 死锁

死锁产生的四个必要条件：

```text
互斥
持有并等待
不可剥夺
循环等待
```

### 互斥

资源同一时刻只能被一个线程占用。

### 持有并等待

线程持有资源的同时继续等待其他资源。

### 不可剥夺

线程持有的资源不能被其他线程强制抢走。

### 循环等待

```text
线程 A 持有锁 1 → 等锁 2
                 ↑
                 ↓
线程 B 持有锁 2 → 等锁 1
```

破坏其中任意一个必要条件，都可以避免死锁。

常见方式：

```text
统一加锁顺序
减少嵌套锁
使用 tryLock / 超时
避免持锁执行耗时操作
```

死锁排查常用：

```text
jstack
jcmd
```

---

# 线程协作

## wait / notify / notifyAll

它们都是：

```text
Object
```

的方法。

### wait()

```java
lock.wait();
```

作用：

```text
当前线程进入等待状态
并释放当前对象的 Monitor
```

必须在：

```text
当前线程已经持有对应 Monitor
```

时调用。

因此通常：

```java
synchronized (lock) {
    lock.wait();
}
```

否则会抛出：

```java
IllegalMonitorStateException
```

### notify()

```java
lock.notify();
```

作用：

```text
唤醒一个等待该 Monitor 的线程
```

注意：

> **notify() 不会立即释放锁。**

流程：

```text
线程 A 持有 Monitor
      ↓
notify()
      ↓
某个等待线程被唤醒
      ↓
线程 A 仍然继续执行
      ↓
线程 A 退出 synchronized
      ↓
被唤醒线程重新竞争 Monitor
      ↓
获得锁后 wait() 才返回
```

### notifyAll()

```java
lock.notifyAll();
```

作用：

```text
唤醒所有等待该 Monitor 的线程
```

但它们仍然需要重新竞争 Monitor。

### wait 推荐使用 while

推荐：

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }

    // 条件满足后继续
}
```

而不是：

```java
if (!condition) {
    lock.wait();
}
```

原因：

```text
被唤醒
≠
条件一定成立
```

还需要考虑：

```text
虚假唤醒
多个线程竞争后条件再次失效
```

因此：

```text
wait()
→ 通常配合 while 条件检查
```

---

## sleep 与 wait

| 对比 | `sleep()` | `wait()` |
| --- | --- | --- |
| 所属类 | Thread | Object |
| 是否释放锁 | 否 | 是，释放 Monitor |
| 是否必须持有 Monitor | 否 | 是 |
| 主要作用 | 暂停当前线程 | 等待某个条件 |
| 是否可被 interrupt | 是 | 是 |

一句话：

> `sleep()` 是“睡一会儿”，不会释放锁；`wait()` 是“等待条件”，会释放 Monitor。

---

## join

```java
target.join();
```

当前线程会等待 `target` 执行结束。

例如：

```java
Thread t = new Thread(() -> {
    System.out.println("task");
});

t.start();
t.join();

System.out.println("main");
```

执行关系：

```text
t 执行完成
   ↓
main 从 join 返回
   ↓
main 继续执行
```

从 Happens-Before 角度：

```text
目标线程中的操作
happens-before
其他线程从 join() 返回
```

---

## LockSupport

LockSupport 提供：

```text
park()
unpark()
```

可以把每个线程理解为拥有：

```text
最多 1 个 permit
```

### park()

```java
LockSupport.park();
```

如果没有许可：

```text
阻塞当前线程
```

如果已经有许可：

```text
消费许可并立即返回
```

### unpark()

```java
LockSupport.unpark(thread);
```

给目标线程发放一个 permit。

permit 最多只有：

```text
1 个
```

所以：

```text
unpark()
unpark()
```

不会累计为：

```text
2 个 permit
```

### unpark 可以先于 park

```text
unpark(thread)
   ↓
发放许可

park()
   ↓
消费许可
   ↓
直接返回
```

因此：

```text
可以先 unpark
再 park
```

---

## Condition

Condition 是：

```text
Lock 体系中的条件等待机制
```

创建：

```java
Condition condition = lock.newCondition();
```

使用时：

```java
lock.lock();

try {
    while (!ready) {
        condition.await();
    }
} finally {
    lock.unlock();
}
```

### await()

```text
线程持有 Lock
   ↓
await()
   ↓
加入 Condition 等待队列
   ↓
完全释放关联 Lock
   ↓
线程阻塞
```

### signal()

```text
Condition 等待队列
   ↓
signal()
   ↓
节点转移到 AQS 同步队列
   ↓
等待重新获取锁
```

注意：

> **signal() 不会立即释放 ReentrantLock。**

当前线程仍需要：

```text
unlock()
```

之后等待线程才有机会重新竞争锁。

最终：

```text
重新获得锁
 ↓
await() 返回
```

---

## 三种等待 / 唤醒机制

| 机制 | 提供者 | 是否释放锁 | 中断行为 | 特点 |
| --- | --- | --- | --- | --- |
| `wait/notify` | Object / Monitor | `wait()` 释放 Monitor | 抛 `InterruptedException` | 必须持有 Monitor |
| `park/unpark` | LockSupport | 不负责自动释放锁 | `park()` 返回，不抛异常 | permit 可先发放 |
| `await/signal` | Condition | `await()` 释放 Lock | 可抛 `InterruptedException` | 支持多个条件队列 |

速记：

```text
wait / notify
→ Monitor 体系

park / unpark
→ 底层阻塞 / 唤醒机制

await / signal
→ Lock / Condition 体系
```

---

# CAS

## CAS 原理

CAS：

```text
Compare And Swap
比较并交换
```

通常包含三个值：

```text
V：当前内存值
A：期望值
B：新值
```

逻辑：

```text
如果 V == A
   ↓
原子地将 V 修改为 B

如果 V != A
   ↓
修改失败
```

例如：

```text
当前值 V = 10
期望值 A = 10
新值   B = 11

10 == 10
   ↓
CAS 成功
   ↓
V = 11
```

如果其他线程已经修改：

```text
V = 12
A = 10
```

则：

```text
12 != 10
 ↓
CAS 失败
```

CAS 的核心价值：

> **无需通过互斥锁阻塞线程，也能完成某些共享状态的原子更新。**

---

## CAS 与 volatile

很多原子类可以简化理解为：

```text
volatile
   +
CAS
```

其中：

```text
volatile
→ 保证共享状态的可见性

CAS
→ 保证一次比较并更新的原子性
```

典型过程：

```text
读取当前值
 ↓
计算新值
 ↓
CAS 尝试更新
 ↓
失败？
 ├── 是 → 重新读取并重试
 └── 否 → 完成
```

---

## CAS 的问题

### ABA

```text
线程 1 读取：A

线程 2：
A → B → A

线程 1 CAS：
发现仍然是 A
→ CAS 成功
```

虽然最终仍然是 A，但实际上中间已经发生过修改。

可以通过：

```text
值 + 版本号
```

解决。

例如：

```java
AtomicStampedReference
```

### 自旋开销

CAS 失败后：

```text
重新读取
 ↓
再次 CAS
 ↓
失败
 ↓
继续重试
```

竞争非常激烈时：

```text
大量自旋
→ 消耗 CPU
```

### 多变量一致性

CAS 天然适合：

```text
更新一个目标状态
```

如果多个变量必须整体保持一致，可以：

```text
把多个状态封装成一个对象
再 CAS 替换整个引用
```

---

## 原子类基础

常见原子类：

```text
AtomicInteger
AtomicLong
AtomicBoolean
AtomicReference
AtomicStampedReference
```

这里主要理解：

```text
volatile + CAS + 自旋
```

具体 API 和常用原子类统一放到：

```text
juc-common-classes.md
```

---

# ThreadLocal

ThreadLocal 用于：

```text
为每个线程保存一份独立数据
```

核心：

> **ThreadLocal 不是让多个线程安全地共享同一个变量，而是让每个线程使用自己的独立数据。**

---

## 基本原理

真正保存数据的是线程对象内部的：

```text
ThreadLocalMap
```

结构：

```text
Thread
  │
  └── ThreadLocalMap
       │
       ├── ThreadLocal A → value A
       └── ThreadLocal B → value B
```

不同线程：

```text
线程 A
└── ThreadLocalMap
    └── local → "A"

线程 B
└── ThreadLocalMap
    └── local → "B"
```

因此：

```text
线程之间数据隔离
```

常用方法：

```java
ThreadLocal<String> local = new ThreadLocal<>();

local.set("value");

String value = local.get();

local.remove();
```

---

## ThreadLocalMap

ThreadLocalMap 中：

```text
key
→ ThreadLocal 的弱引用

value
→ 强引用
```

可以简化为：

```text
Entry
├── key   → WeakReference<ThreadLocal>
└── value → Object
```

---

## 内存泄漏

如果外部不再强引用 ThreadLocal：

```text
ThreadLocal
   ↓ GC
key 可能变成 null
```

此时：

```text
Thread
 └── ThreadLocalMap
      └── null → value
```

如果线程长期存活，例如：

```text
线程池工作线程
```

那么 value 可能仍然被 ThreadLocalMap 强引用，无法及时回收。

---

## 为什么 key 使用弱引用

如果 key 使用强引用：

```text
Thread
 ↓
ThreadLocalMap
 ↓
ThreadLocal
```

只要线程仍然存活，ThreadLocal 本身也难以被回收。

改为弱引用后：

```text
外部不再引用 ThreadLocal
→ ThreadLocal 本身可以被 GC
```

但：

```text
value 仍然是强引用
```

所以弱引用不能彻底避免内存泄漏。

---

## remove()

线程池中的线程通常生命周期很长。

因此 ThreadLocal 使用完最好：

```java
try {
    local.set(value);

    // 使用
} finally {
    local.remove();
}
```

记忆：

> **ThreadLocal 用完要 remove，尤其在线程池环境中。**

---

# 速记

```text
并发基础
├── 原子性
├── 可见性
├── 有序性
├── JMM
└── Happens-Before
```

```text
线程基础
├── NEW
├── RUNNABLE
├── BLOCKED
├── WAITING
├── TIMED_WAITING
├── TERMINATED
└── interrupt
```

```text
线程同步
├── synchronized
│   ├── Monitor
│   ├── 可重入
│   └── JVM 语义支持
│
├── volatile
│   ├── 可见性
│   ├── 有序性
│   └── 不保证 i++ 原子性
│
├── ReentrantLock
│   ├── 公平 / 非公平
│   ├── lockInterruptibly
│   └── tryLock
│
└── AQS
    ├── state
    ├── CAS
    ├── 同步队列
    ├── park
    └── unpark
```

```text
线程协作
├── wait / notify
├── sleep / wait
├── join
├── park / unpark
└── await / signal
```

```text
CAS
├── V / A / B
├── 自旋
├── ABA
├── volatile + CAS
└── Atomic 类
```

```text
ThreadLocal
├── ThreadLocalMap
├── key 弱引用
├── value 强引用
├── 线程隔离
└── 使用后 remove
```

核心关系：

```text
JMM
 ↓
原子性 / 可见性 / 有序性
 ↓
volatile / synchronized
 ↓
CAS / Lock
 ↓
AQS
 ↓
ReentrantLock / Condition
```

一句话：

> **Java 并发的核心，就是理解共享变量如何在多线程之间正确读写，以及线程如何通过锁、CAS、等待/唤醒机制安全协作。**
