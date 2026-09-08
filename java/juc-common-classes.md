# JUC 常用类

> 主线：**原子类 → 锁与同步器 → 并发集合 → 线程池 → 同步工具**

---

# 原子类

Java 原子类位于 `java.util.concurrent.atomic` 包，核心思想通常可以理解为：

```text
volatile + CAS + 自旋
```

CAS 原理见 `juc-basics.md`，这里重点看具体类。

## AtomicInteger

用于线程安全的整数原子操作。

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
count.decrementAndGet();
count.getAndIncrement();
count.compareAndSet(0, 1);
```

典型过程：

```text
读取当前值
→ 计算新值
→ CAS 尝试更新
→ 失败则重试
```

适合：并发计数器、状态更新、简单无锁原子操作。

JDK 8 中很多原子操作会借助 `Unsafe` 提供底层 CAS 等能力；`Unsafe` 属于底层内部能力，不适合作为普通业务 API。

## AtomicStampedReference

用于解决 CAS 的 ABA 问题。

```text
A, stamp=1
→ B, stamp=2
→ A, stamp=3
```

虽然值重新变成 A，但版本号已经变化，因此能够识别中间修改。

---

# 锁与同步器

JUC 锁体系可以沿着下面的关系理解：

```text
LockSupport
     ↓
    AQS
     ↓
ReentrantLock
     ↓
 Condition
```

## LockSupport

`LockSupport` 提供构建锁和同步器时最基础的线程阻塞 / 唤醒能力。

```java
LockSupport.park();
LockSupport.unpark(thread);
```

### permit 模型

每个线程可以理解为最多拥有一个许可：

```text
permit = 0 / 1
```

- `unpark(thread)`：给目标线程一个 permit。
- `park()`：有 permit 就消费并直接返回；没有 permit 就阻塞。

因此允许：

```text
先 unpark(thread)
→ permit = 1
→ 再 park()
→ 消费 permit
→ 直接返回
```

permit 不会累计成 2。

### 中断行为

线程在 `park()` 时被中断：

- `park()` 返回。
- 不抛 `InterruptedException`。
- 中断标记保留。

实际使用 `park()` 时仍应结合条件循环，因为它也可能发生无理由返回。

## AQS

AQS（`AbstractQueuedSynchronizer`）是 JUC 中用于构建锁和同步器的基础框架。

很多同步器都建立在 AQS 之上，例如：

- `ReentrantLock`
- `Semaphore`
- `CountDownLatch`

核心：

```text
AQS
├── volatile state
├── CAS
├── 同步等待队列
└── 获取 / 释放资源的模板方法
```

### state

AQS 内部维护：

```java
volatile int state;
```

不同同步器含义不同：

- `ReentrantLock`：重入次数。
- `Semaphore`：剩余许可数。
- `CountDownLatch`：剩余计数。

### 同步队列

获取同步状态失败的线程进入 FIFO 风格的双向等待队列：

```text
head
 ↓
Node ←→ Node ←→ Node
                 ↑
                tail
```

节点通常保存等待线程、前驱、后继、等待状态等信息。

第一个真正等待获取资源的节点通常是 `head.next`。

### 获取流程

以独占模式为例：

```text
尝试 tryAcquire()
      │
      ├── 成功 → 获得资源
      │
      └── 失败
           ↓
        加入同步队列
           ↓
       检查前驱节点
           ↓
     前驱是 head？
      │
      ├── 是 → 再次尝试
      │
      └── 否 / 仍失败
           ↓
          park()
           ↓
         等待唤醒
           ↓
        重新竞争
```

关键点：

> 加入同步队列后，通常前驱为 `head` 的节点最有资格再次尝试获取资源。

### 释放流程

```text
release()
→ tryRelease()
→ state 更新成功
→ 检查同步队列
→ unpark 合适的后继节点
→ 后继线程重新竞争
```

AQS 核心速记：

```text
state + CAS + 同步队列 + park / unpark
```

## ReentrantLock

`ReentrantLock` 是常用的可重入独占锁。

内部结构：

```text
ReentrantLock
    ↓
   Sync
  /    \
Fair  Nonfair
    ↓
   AQS
```

### 可重入

同一线程再次获得锁时：

```text
state++
```

释放一次：

```text
state--
```

直到 `state = 0` 才完全释放。

### 基本使用

```java
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();
}
```

`unlock()` 应放在 `finally` 中。

### lockInterruptibly()

```java
lock.lockInterruptibly();
```

等待锁期间可以响应中断：

```text
interrupt
→ 放弃等待
→ 抛 InterruptedException
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

获取不到可直接返回，也可以设置超时：

```java
lock.tryLock(1, TimeUnit.SECONDS);
```

### 公平锁与非公平锁

```java
new ReentrantLock(true);   // 公平
new ReentrantLock(false);  // 非公平，默认
```

- **公平锁**：尽量按等待队列顺序获取。
- **非公平锁**：新线程也可以直接尝试抢锁，通常吞吐量更高。

### synchronized vs ReentrantLock

| 对比 | `synchronized` | `ReentrantLock` |
| --- | --- | --- |
| 加解锁 | 自动 | 手动 |
| 可重入 | 是 | 是 |
| 等锁可中断 | 不支持 | `lockInterruptibly()` |
| 超时获取 | 不支持 | `tryLock()` |
| 公平模式 | 不提供配置 | 支持 |
| 条件队列 | `wait/notify` | 多个 `Condition` |
| 实现层面 | JVM 语义支持 | JDK 类 + AQS |

## Condition

`Condition` 是 Lock 体系中的条件等待机制，类似 Monitor 的 `wait / notify`，但支持多个条件队列。

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

常用方法：

```java
condition.await();
condition.signal();
condition.signalAll();
```

### await()

```text
线程持有 Lock
→ await()
→ 进入 Condition 条件队列
→ 完全释放关联 Lock
→ park 阻塞
```

### signal()

```text
Condition 条件队列
→ signal()
→ 节点转移到 AQS 同步队列
→ 等待重新获取锁
→ 获得锁后 await() 返回
```

`signal()` **不会立即释放 ReentrantLock**，当前线程仍要执行 `unlock()`。

一个 `ReentrantLock` 可以创建多个 Condition：

```java
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();
```

这比单一 Monitor 等待集合更灵活。

### 三种等待 / 唤醒机制

| 机制 | 所属体系 | 是否释放锁 | 中断行为 | 特点 |
| --- | --- | --- | --- | --- |
| `wait/notify` | Object / Monitor | `wait()` 释放 Monitor | 抛 `InterruptedException` | 必须持有 Monitor |
| `park/unpark` | LockSupport | 不负责自动释放锁 | `park()` 返回，不抛异常 | permit 可先发放 |
| `await/signal` | Lock / Condition | `await()` 释放 Lock | 可抛 `InterruptedException` | 支持多个条件队列 |

---

# 并发集合

常见 JUC 并发集合：

| 类 | 核心特点 |
| --- | --- |
| `ConcurrentHashMap` | 高并发线程安全 Map |
| `CopyOnWriteArrayList` | 写时复制，适合读多写少 |
| `ConcurrentLinkedQueue` | CAS 实现的无界非阻塞 FIFO |
| `BlockingQueue` | 支持阻塞语义的生产者-消费者队列 |

## ConcurrentHashMap

### JDK 7

采用 `Segment` 分段锁：

```text
ConcurrentHashMap
├── Segment 1
├── Segment 2
└── ...
```

锁粒度主要是 Segment。

### JDK 8

取消 Segment，底层：

```text
数组 + 链表 + 红黑树
```

线程安全主要依赖：

```text
CAS + synchronized + volatile
```

### 初始化

采用懒初始化，创建对象时通常不会立即创建 table，首次真正插入时初始化；容量会规整为 2 的幂次方。

### put 流程

```text
put(key,value)
→ table 未初始化？初始化
→ 找目标桶
   ├── 桶空 → CAS 插入
   ├── 正在扩容 → 协助迁移
   └── 桶非空 → synchronized 桶头 → 链表 / 红黑树插入
```

### 树化

常见阈值：

```text
TREEIFY_THRESHOLD = 8
MIN_TREEIFY_CAPACITY = 64
```

桶内节点较多但数组容量过小时，通常优先扩容，而不是立即树化。

### 扩容

JDK 8 支持多个线程协助迁移。

关键结构：

- `sizeCtl`：控制初始化 / 扩容状态。
- `ForwardingNode`：表示桶已经迁移。

### get

`get()` 通常不对整个结构加互斥锁，主要依赖 volatile 可见性和并发结构设计，因此读性能较高。

## CopyOnWriteArrayList

底层：**数组 + 写时复制**。

### 读

直接读取当前数组快照，通常不加锁，遍历开销低。

### 写

```text
add / set / remove
→ 加锁
→ 复制旧数组
→ 修改新数组
→ 替换数组引用
```

### 迭代器

Iterator 保存创建时的数组快照，后续集合修改不会改变该快照，因此通常不会抛 `ConcurrentModificationException`，但也看不到后续最新修改。

### 适用场景

- 读远多于写。
- 数据规模不大。
- 可以接受快照式读取。

例如：配置列表、订阅者列表。

## ConcurrentLinkedQueue

无界、非阻塞、FIFO 的线程安全队列。

底层：单向链表；线程安全主要依赖 `CAS + volatile`。

### offer

```text
找到真正尾节点
→ CAS 把新节点挂到 next
→ 尝试推进 tail
```

`tail` 可以暂时落后于真正队尾，不影响正确性。

### poll

```text
从 head 查找有效节点
→ CAS 推进 head
→ 返回首元素
```

`head` 也允许暂时滞后。

优点：高并发吞吐较好、无需显式互斥锁。  
缺点：CAS 失败有重试开销、无界队列可能持续占用内存、不提供阻塞等待语义。

## BlockingQueue

支持阻塞语义，常用于生产者-消费者模型：

- 队列空 → 消费者可以等待。
- 队列满 → 生产者可以等待。

### 核心方法

| 操作 | 抛异常 | 特殊值 | 一直阻塞 | 超时 |
| --- | --- | --- | --- | --- |
| 插入 | `add` | `offer` | `put` | `offer(timeout)` |
| 删除 | `remove` | `poll` | `take` | `poll(timeout)` |
| 查看 | `element` | `peek` | — | — |

常见实现：

| 类 | 特点 |
| --- | --- |
| `ArrayBlockingQueue` | 数组、有界 |
| `LinkedBlockingQueue` | 链表，可指定容量 |
| `PriorityBlockingQueue` | 优先级、无界 |
| `DelayQueue` | 延迟获取 |
| `SynchronousQueue` | 不存元素，线程直接交接 |
| `LinkedTransferQueue` | 无界，支持 transfer |

### ArrayBlockingQueue

```text
数组 + 固定容量 + 一把 Lock + 两个 Condition
```

- `notEmpty`：队列非空。
- `notFull`：队列未满。

消费者发现空：

```text
notEmpty.await()
```

生产者插入后：

```text
notEmpty.signal()
```

生产者发现满：

```text
notFull.await()
```

消费者取走后：

```text
notFull.signal()
```

### LinkedBlockingQueue

底层：链表。

常见实现使用两把锁：

- `putLock`：入队。
- `takeLock`：出队。

并通过计数器维护元素数量，提高生产和消费之间的并发度。

---

# 线程池

## ThreadPoolExecutor

线程池作用：

- 复用线程，减少频繁创建 / 销毁开销。
- 提高任务响应速度。
- 统一管理线程。
- 控制并发数量。

核心实现：`ThreadPoolExecutor`。

## 核心参数

```java
new ThreadPoolExecutor(
    corePoolSize,
    maximumPoolSize,
    keepAliveTime,
    unit,
    workQueue,
    threadFactory,
    handler
);
```

| 参数 | 作用 |
| --- | --- |
| `corePoolSize` | 核心线程数 |
| `maximumPoolSize` | 最大线程数 |
| `keepAliveTime` | 非核心线程空闲存活时间 |
| `unit` | 时间单位 |
| `workQueue` | 任务队列 |
| `threadFactory` | 创建线程 |
| `handler` | 拒绝策略 |

## 执行流程

```text
execute(task)
→ 当前线程数 < corePoolSize？
   ├── 是 → 创建核心线程
   └── 否 → 尝试入队
            ├── 成功 → 等待执行
            └── 队列满
                 → 当前线程数 < maximumPoolSize？
                    ├── 是 → 创建非核心线程
                    └── 否 → 拒绝
```

一句话：

> **先核心线程 → 再入队 → 再扩容到最大线程 → 最后拒绝。**

## 常见任务队列

常见：`ArrayBlockingQueue`、`LinkedBlockingQueue`、`SynchronousQueue`、`DelayQueue`、`PriorityBlockingQueue`。

具体原理见前面的 `BlockingQueue`。

## 拒绝策略

| 策略 | 行为 |
| --- | --- |
| `AbortPolicy` | 默认；拒绝并抛 `RejectedExecutionException` |
| `DiscardPolicy` | 直接丢弃，不抛异常 |
| `DiscardOldestPolicy` | 丢弃队列最旧任务，再尝试提交当前任务 |
| `CallerRunsPolicy` | 由提交任务的线程自己执行，形成一定反压 |

## 常见线程池类型

| 类型 | 特点 | 风险 |
| --- | --- | --- |
| `FixedThreadPool` | 固定线程数 | 无界队列可能堆积 |
| `SingleThreadExecutor` | 单线程顺序执行 | 无界队列可能堆积 |
| `CachedThreadPool` | `core=0` + `SynchronousQueue` | 线程数可能快速增长 |
| `ScheduledThreadPool` | 延迟 / 周期任务 | 适合定时调度 |

## 为什么不推荐直接使用 Executors

部分工厂方法隐藏关键参数，容易让资源不可控：

- `FixedThreadPool / SingleThreadExecutor`：任务队列可能大量堆积，存在 OOM 风险。
- `CachedThreadPool`：最大线程数很大，线程数量可能暴涨。

实际项目更推荐显式创建 `ThreadPoolExecutor`，明确配置核心线程、最大线程、有界队列、拒绝策略和线程工厂。

## 线程池关闭

### shutdown()

```text
停止接收新任务
→ 已提交任务继续执行
→ 完成后关闭
```

### shutdownNow()

```text
停止接收新任务
→ 尝试 interrupt 正在执行的任务
→ 返回尚未开始的队列任务
```

注意：`shutdownNow()` 只是尝试通过中断终止任务，不能保证任务立即停止。

---

# 同步工具

常见工具：

| 类 | 一句话 |
| --- | --- |
| `CountDownLatch` | 等别人做完 |
| `CyclicBarrier` | 等大家到齐 |
| `Semaphore` | 控制并发数量 |
| `Exchanger` | 两个线程交换数据 |
| `Phaser` | 更灵活的多阶段同步 |

## CountDownLatch

一个或多个线程等待其他线程完成任务。

```text
任务 A ─┐
任务 B ─┼→ countDown()
任务 C ─┘
        ↓
      count = 0
        ↓
等待线程继续
```

核心方法：

```java
latch.await();
latch.countDown();
```

特点：

- 内部维护计数器。
- `countDown()` 使计数减 1。
- 到 0 后等待线程继续。
- 一次性使用，不能重置。

场景：主线程等待多个子任务完成后统一汇总。

## CyclicBarrier

让一组线程全部到达屏障后再一起继续。

```text
线程 A ─┐
线程 B ─┼→ await()
线程 C ─┘
       ↓
     全部到齐
       ↓
     一起继续
```

特点：

- 可以重复使用。
- 可以指定屏障动作。
- 适合多阶段协作。

## Semaphore

通过许可 `permit` 控制同时访问资源的线程数量。

```java
semaphore.acquire();
try {
    // 使用资源
} finally {
    semaphore.release();
}
```

例如 `permit = 3`，表示最多允许 3 个线程同时进入。

典型场景：接口限流、数据库连接池、资源并发访问控制。

## Exchanger

用于两个线程之间交换数据。

```java
Object other = exchanger.exchange(data);
```

```text
线程 A：dataA ─┐
               ├→ exchange
线程 B：dataB ─┘

A 得到 dataB
B 得到 dataA
```

适合双线程数据交换、校验等场景。

## Phaser

更灵活的多阶段线程同步工具。

特点：

- 类似 `CyclicBarrier`。
- 支持多个阶段。
- 支持参与线程动态注册 / 注销。
- 适合参与者数量会变化的多阶段协作。

---

# 速记

```text
JUC 常用类
│
├── 原子类
│   ├── AtomicInteger
│   └── AtomicStampedReference
│
├── 锁与同步器
│   ├── LockSupport
│   ├── AQS
│   ├── ReentrantLock
│   └── Condition
│
├── 并发集合
│   ├── ConcurrentHashMap
│   ├── CopyOnWriteArrayList
│   ├── ConcurrentLinkedQueue
│   └── BlockingQueue
│
├── 线程池
│   └── ThreadPoolExecutor
│
└── 同步工具
    ├── CountDownLatch
    ├── CyclicBarrier
    ├── Semaphore
    ├── Exchanger
    └── Phaser
```

核心关系：

```text
CAS
 ↓
Atomic 类

LockSupport
 ↓
AQS
 ↓
ReentrantLock
 ↓
Condition

BlockingQueue
 ↓
ThreadPoolExecutor

AQS
 ├── ReentrantLock
 ├── Semaphore
 └── CountDownLatch
```

> **JUC 常用类篇重点回答：Java 并发框架中有哪些常用类，它们怎么用，以及核心实现机制是什么。**
