# Java 并发基础

> 主线：**并发基础 → 线程基础 → 线程同步 → 线程协作 → CAS → ThreadLocal → 死锁**

---

# 并发基础

## 并发三要素

并发问题主要来自：**原子性、可见性、有序性**。

- **原子性**：一个操作要么完整执行，要么完全不执行；例如 `i++` 实际包含“读 → 加 1 → 写回”，不是原子操作。
- **可见性**：一个线程修改共享变量后，其他线程不一定立即看到最新值；原因可能涉及 CPU Cache、寄存器、写缓冲区和编译器优化。
- **有序性**：编译器和 CPU 可能为了性能进行指令重排序，单线程结果通常不受影响，多线程下可能产生问题。

## JMM

JMM（Java Memory Model）描述：

> **多线程环境下，共享变量如何读写，以及一个线程的修改什么时候对其他线程可见。**

注意：**JMM ≠ JVM 堆 / 栈 / 方法区**，它是并发编程中的抽象内存模型。

```text
             主内存
        ┌─────────────┐
        │   共享变量   │
        └─────────────┘
           ↑       ↑
           │       │
      线程1本地   线程2本地
```

这里的“主内存、本地内存”都是抽象概念，本地内存可能对应 CPU Cache、寄存器、写缓冲区以及编译器优化产生的临时数据。

JMM 重点解决：**原子性、可见性、有序性**。

## Happens-Before

如果操作 A happens-before 操作 B，则 **A 的执行结果对 B 可见，并且 A 在逻辑顺序上先于 B**。

常见规则：

- **程序次序**：同一线程中，前面的操作 happens-before 后面的操作。
- **锁规则**：同一把锁的 `unlock` happens-before 后续的 `lock`。
- **volatile**：写 volatile 变量 happens-before 后续对该变量的读。
- **线程启动**：`Thread.start()` happens-before 新线程中的操作。
- **线程终止**：目标线程中的操作 happens-before 其他线程从 `join()` 返回。
- **传递性**：A → B，B → C，则 A → C。

## 线程安全的基本思路

```text
线程安全
├── 互斥同步：synchronized
├── 非阻塞同步：CAS
└── 减少共享：ThreadLocal / 无状态对象
```

---

# 线程基础

## 线程创建

常见方式：继承 `Thread`、实现 `Runnable`、实现 `Callable`、使用线程池。

### Runnable

```java
Runnable task = () -> System.out.println("run");
new Thread(task).start();
```

### Callable

`Callable` 可以返回结果，也可以抛出受检异常，常与 `FutureTask` 或线程池配合。

```java
Callable<Integer> task = () -> 1 + 2;
FutureTask<Integer> future = new FutureTask<>(task);

new Thread(future).start();
Integer result = future.get();
```

一般更推荐 `Runnable / Callable`，因为任务逻辑与线程对象解耦，也更方便交给线程池执行。

## 线程状态

Java 线程有 6 种状态：

| 状态 | 含义 |
| --- | --- |
| `NEW` | 创建后尚未启动 |
| `RUNNABLE` | 正在运行或等待 CPU 调度 |
| `BLOCKED` | 等待进入 `synchronized` Monitor |
| `WAITING` | 无限期等待 |
| `TIMED_WAITING` | 有超时时间的等待 |
| `TERMINATED` | 执行结束 |

常见变化：

```text
NEW
 ↓ start()
RUNNABLE
 ├── 等待 Monitor → BLOCKED
 ├── wait()/join() → WAITING
 ├── sleep()/限时 wait()/限时 join() → TIMED_WAITING
 └── run() 结束 → TERMINATED
```

`BLOCKED` 特指等待 `synchronized` Monitor；JUC 显式锁内部若使用 `LockSupport.park()`，通常表现为 `WAITING / TIMED_WAITING`。

## 基础线程方法

- `start()`：启动新线程并最终执行 `run()`；直接调用 `run()` 只是普通方法调用。
- `sleep(ms)`：让当前线程暂停一段时间，进入 `TIMED_WAITING`，**不会释放已持有的锁**，可被中断。
- `yield()`：提示调度器当前线程愿意让出 CPU，只是建议，不保证发生线程切换。
- `join()`：当前线程等待目标线程执行结束。

## 线程中断

Java 中断是**协作式取消机制**，不是强制杀死线程。

```java
thread.interrupt();
```

### 普通运行线程

`interrupt()` 通常只设置中断标记，线程需要主动检查：

```java
while (!Thread.currentThread().isInterrupted()) {
    // 工作
}
```

### 可中断等待

`sleep / wait / join` 被中断时通常会抛 `InterruptedException`，并清除中断标记。

若希望继续向上传递中断语义：

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

### 等待锁

- 等待普通 `synchronized` Monitor 时，不能通过 `interrupt()` 直接取消等待。
- `ReentrantLock.lock()` 等待锁时不会因中断而放弃获取。
- `ReentrantLock.lockInterruptibly()` 可以响应中断并抛 `InterruptedException`。

### 中断状态方法

- `thread.isInterrupted()`：查询指定线程，不清除标记。
- `Thread.interrupted()`：查询当前线程，并清除标记。

---

# 线程同步

线程同步主要解决：**多个线程如何安全访问共享可变数据**。

## synchronized

`synchronized` 提供互斥访问，同时具备原子性、可见性和有序性语义。

### 锁对象

```java
public synchronized void method() {}
```

锁：`this`

```java
public static synchronized void method() {}
```

锁：当前类的 `Class` 对象

```java
synchronized (lock) {
    // 临界区
}
```

锁：指定的 `lock` 对象。

### 底层原理

- 同步代码块主要通过字节码 `monitorenter / monitorexit` 获取和释放 Monitor。
- 同步方法由 JVM 根据同步访问标志完成加锁。

### 可重入

`synchronized` 是可重入锁，同一线程持有某对象 Monitor 后，可以再次进入该对象的同步区域。

### synchronized 与 JMM

同一把锁：

```text
unlock
  happens-before
后续 lock
```

因此前一个线程同步块中的修改，对之后获得同一把锁的线程可见。

### 锁优化说明

旧版 HotSpot 曾包含偏向锁、轻量级锁、重量级锁、自旋等实现细节；不同 JDK 版本已经发生变化，不应把“偏向锁 → 轻量级锁 → 重量级锁”当作所有现代 JDK 固定不变的模型。

## volatile

`volatile` 主要提供：**可见性 + 一定程度的有序性保证**。

```java
volatile boolean flag = false;
```

### 可见性

写 volatile 变量 happens-before 后续对该变量的读。

### 有序性

volatile 会通过内存屏障限制特定指令重排序。

典型场景：双重检查单例 DCL。

```java
private static volatile Singleton instance;
```

它可以避免“引用先发布、对象尚未完成初始化”一类重排序问题。

### 不保证复合操作原子性

```java
volatile int i = 0;
i++;
```

`i++` 仍然是：

```text
读 → 加 1 → 写
```

因此 `volatile` 不能保证 `i++` 的原子性。

常见场景：状态标志、一次写多次读、DCL。

---

# 线程协作

线程协作关注：**线程之间如何等待、通知和协调执行顺序**。

## wait / notify / notifyAll

它们都是 `Object` 的方法，必须围绕同一个 Monitor 使用。

### wait()

`wait()` 会让当前线程进入等待状态，并**释放当前持有的 Monitor**。

```java
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
}
```

必须在持有对应 Monitor 时调用，否则抛 `IllegalMonitorStateException`。

推荐使用 `while` 而不是 `if`，因为：

- 被唤醒不代表条件一定成立。
- 可能存在虚假唤醒。
- 多线程竞争后条件可能再次失效。

### notify()

`notify()` 唤醒一个等待该 Monitor 的线程，但**不会立即释放锁**。

```text
线程 A 持有 Monitor
→ notify()
→ 某等待线程被唤醒
→ A 继续执行
→ A 退出 synchronized
→ 被唤醒线程重新竞争 Monitor
→ 获得锁后 wait() 返回
```

### notifyAll()

`notifyAll()` 唤醒所有等待该 Monitor 的线程，但它们仍需重新竞争 Monitor。

## sleep 与 wait

| 对比 | `sleep()` | `wait()` |
| --- | --- | --- |
| 所属类 | `Thread` | `Object` |
| 是否释放锁 | 否 | 是，释放 Monitor |
| 是否必须持有 Monitor | 否 | 是 |
| 主要作用 | 暂停当前线程 | 等待某个条件 |
| 是否可被中断 | 是 | 是 |

一句话：

> `sleep()` 是“睡一会儿”；`wait()` 是“等待条件并释放 Monitor”。

## join

```java
target.join();
```

当前线程等待 `target` 执行结束。

从 Happens-Before 角度：

```text
目标线程中的操作
  happens-before
其他线程从 join() 返回
```

## JUC 中的协作机制

JUC 还提供：

- `LockSupport.park / unpark`
- `Condition.await / signal`

具体原理和 API 统一放在 `juc-common-classes.md`。

---

# CAS

## CAS 原理

CAS（Compare And Swap，比较并交换）通常涉及：

- `V`：当前内存值
- `A`：期望值
- `B`：新值

逻辑：

```text
V == A → 原子地把 V 更新为 B
V != A → 更新失败
```

CAS 的价值：

> **无需通过互斥锁阻塞线程，也能完成某些共享状态的原子更新。**

## CAS 与 volatile

很多原子类可以简化理解为：

```text
volatile + CAS + 自旋
```

- `volatile`：保证共享状态可见。
- `CAS`：保证一次比较并更新的原子性。
- CAS 失败时重新读取并重试。

## CAS 的问题

### ABA

```text
线程1读取 A
线程2执行 A → B → A
线程1再次 CAS，发现仍是 A
```

虽然最终值还是 A，但中间发生过变化。

常用解决方案：**值 + 版本号**，如 `AtomicStampedReference`。

### 自旋开销

竞争激烈时 CAS 可能频繁失败并不断重试，造成 CPU 消耗。

### 多变量一致性

CAS 天然适合更新一个目标状态；若多个变量必须整体一致，可以把它们封装成一个对象，再 CAS 替换整个引用。

具体原子类 API 统一放到 `juc-common-classes.md`。

---

# ThreadLocal

`ThreadLocal` 为每个线程保存一份独立数据。

> **它不是让多个线程安全共享同一个变量，而是让不同线程使用各自的数据。**

## 基本原理

真正保存数据的是线程对象内部的 `ThreadLocalMap`。

```text
Thread
└── ThreadLocalMap
    ├── ThreadLocal A → value A
    └── ThreadLocal B → value B
```

不同线程拥有不同的 `ThreadLocalMap`，因此数据天然隔离。

## ThreadLocalMap

Entry 可以简化为：

```text
key   → WeakReference<ThreadLocal>
value → Object（强引用）
```

## 内存泄漏

如果外部不再强引用 ThreadLocal：

```text
ThreadLocal 被 GC
→ Entry.key 可能变成 null
→ value 仍被 ThreadLocalMap 强引用
```

在线程池中工作线程通常长期存活，因此 value 可能无法及时回收。

### 为什么 key 是弱引用

若 key 是强引用，只要线程存活，ThreadLocal 本身也很难被回收。

弱引用可以让外部不再引用的 ThreadLocal 被 GC，但 **value 仍然是强引用**，因此不能彻底避免泄漏。

## remove()

使用完最好主动清理：

```java
try {
    local.set(value);
    // 使用
} finally {
    local.remove();
}
```

尤其在线程池环境中要注意。

---

# 死锁

死锁：多个线程相互等待对方持有的资源，最终都无法继续执行。

产生死锁需要同时满足四个条件：

- **互斥**：资源同一时刻只能由一个线程使用。
- **持有并等待**：持有已有资源的同时继续等待其他资源。
- **不可剥夺**：资源不能被其他线程强制抢走。
- **循环等待**：多个线程形成循环资源等待关系。

```text
线程 A：持有锁1 → 等锁2
                    ↑
                    ↓
线程 B：持有锁2 → 等锁1
```

常见避免方式：

- 统一加锁顺序。
- 减少嵌套锁。
- 使用 `tryLock()` / 超时。
- 避免持锁执行耗时操作。

常用排查工具：`jstack`、`jcmd`。

---

# 速记

```text
Java 并发
│
├── 并发基础
│   ├── 原子性
│   ├── 可见性
│   ├── 有序性
│   ├── JMM
│   └── Happens-Before
│
├── 线程基础
│   ├── 创建 / 状态
│   ├── start / sleep / yield / join
│   └── interrupt
│
├── 线程同步
│   ├── synchronized
│   └── volatile
│
├── 线程协作
│   ├── wait / notify / notifyAll
│   ├── sleep vs wait
│   └── join
│
├── CAS
│   ├── volatile + CAS
│   ├── 自旋
│   └── ABA
│
├── ThreadLocal
│   ├── ThreadLocalMap
│   ├── key 弱引用
│   ├── value 强引用
│   └── remove
│
└── 死锁
    └── 四个必要条件
```

> **并发基础篇重点回答：共享变量如何正确读写、线程如何同步和协作，以及如何避免典型并发问题。**







# 交替打0-100

```
public class AlternatePrint {

    private static final Object lock = new Object();
    private static int num = 0;

    public static void main(String[] args) {

        Thread t1 = new Thread(() -> print(0), "线程A");
        Thread t2 = new Thread(() -> print(1), "线程B");

        t1.start();
        t2.start();
    }

    private static void print(int parity) {
        while (true) {
            synchronized (lock) {

                // 不是自己该打印的数字，就等待
                while (num <= 100 && num % 2 != parity) {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }

                // 打印结束
                if (num > 100) {
                    lock.notifyAll();
                    break;
                }

                System.out.println(
                        Thread.currentThread().getName() + ": " + num
                );

                num++;

                // 唤醒另一个线程
                lock.notifyAll();
            }
        }
    }
}
```

