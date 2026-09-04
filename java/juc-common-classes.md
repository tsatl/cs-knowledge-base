# 原子类

Java 原子类位于 `java.util.concurrent.atomic` 包中，主要通过 CAS 等机制提供线程安全的原子操作。

## AtomicInteger

`AtomicInteger` 用于提供线程安全的整数原子操作。

常见方法：

```java
AtomicInteger count = new AtomicInteger(0);

count.incrementAndGet();
count.decrementAndGet();
count.getAndIncrement();
count.compareAndSet(0, 1);
```

**核心原理：**

```text
可见性
+
CAS 原子更新
```

可以粗略理解：

```text
读取当前值
   ↓
计算新值
   ↓
CAS 尝试更新
   │
   ├── 成功 → 返回
   │
   └── 失败 → 重试
```

因此无需使用 `synchronized`，也可以完成很多整数的线程安全更新。

适合：

- 并发计数器
- 状态值更新
- 简单的无锁原子操作

**源码补充：**

JDK 8 中很多原子操作底层会借助 `Unsafe` 提供的 CAS 等底层能力。

`Unsafe` 可以进行：

```text
CAS
内存操作
线程调度
```

等底层操作，但它属于非常底层的内部能力，不适合作为普通业务 API 使用。

---

## AtomicStampedReference

CAS 存在 ABA 问题：

```text
A → B → A
```

普通 CAS 只发现最终仍然是 A，无法知道中间是否发生过变化。

`AtomicStampedReference` 通过：

```text
对象引用
+
版本号 stamp
```

共同判断状态。

例如：

```text
A, version = 1
      ↓
B, version = 2
      ↓
A, version = 3
```

虽然值重新变成 A，但版本号已经发生变化，因此可以识别 ABA。

---

# JUC锁

JUC 锁体系可以按照下面的关系理解：

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

`LockSupport` 提供了构建锁和同步器时最基础的线程阻塞 / 唤醒能力。

核心方法：

```java
LockSupport.park();
LockSupport.unpark(thread);
```

**park()：**

阻塞当前线程，常见返回条件包括：

- 其他线程调用 `unpark(thread)`
- 当前线程被中断
- 超时时间到达
- 可能发生无理由返回，因此实际使用通常应结合条件循环

**unpark(thread)：**

给指定线程一个“许可 permit”。

每个线程可以理解为最多拥有一个许可：

```text
permit = 0 / 1
```

因此：

```text
先 unpark(thread)
再 park()
```

也是允许的。

```text
unpark
  ↓
permit = 1

park
  ↓
消费 permit
  ↓
直接返回
```

如果线程在 `park()` 时被中断：

```text
park() 返回
中断标记保留
不抛 InterruptedException
```

---

## AQS

AQS：

```text
AbstractQueuedSynchronizer
```

是 JUC 中用于构建锁和同步器的基础框架。

很多同步器都建立在 AQS 思想之上。

**核心结构：**

```text
AQS
│
├── volatile state
│
├── 同步等待队列
│
└── 获取 / 释放资源的模板方法
```

**state：**

表示同步状态，具体含义由不同同步器决定。

以 `ReentrantLock` 为例，可以粗略理解：

```text
state = 0
→ 锁空闲

state > 0
→ 锁被某线程持有
→ 数值还可以表示重入次数
```

**同步队列：**

获取同步状态失败的线程会进入一个 FIFO 风格的双向等待队列。

```text
head
 ↓
Node ←→ Node ←→ Node
                 ↑
                tail
```

Node 中通常保存：

- 等待线程
- 前驱节点
- 后继节点
- 等待状态

**获取锁流程：**

以 ReentrantLock 独占模式为例：

```text
线程调用 lock()
     ↓
尝试 tryAcquire()
     │
     ├── 成功
     │    ↓
     │   获得锁
     │
     └── 失败
          ↓
       加入同步队列
          ↓
       尝试再次获取
          ↓
       条件不满足
          ↓
       LockSupport.park()
          ↓
         等待
          ↓
       被 unpark 唤醒
          ↓
       重新竞争锁
```

**释放锁流程：**

```text
unlock()
   ↓
release()
   ↓
tryRelease()
   ↓
state 减少
   │
   ├── state != 0
   │      ↓
   │   仍处于重入状态
   │
   └── state == 0
          ↓
       完全释放锁
          ↓
       唤醒后继节点
```

AQS 的核心可以记成：

```text
state
+
CAS
+
同步队列
+
park / unpark
```

---

## ReentrantLock

`ReentrantLock` 是 JUC 中常用的可重入独占锁。

内部关系可以简单理解为：

```text
ReentrantLock
    ↓
   Sync
   /   \
FairSync  NonfairSync
    ↓
   AQS
```

**可重入：**

同一个线程已经持有锁时再次获取：

```text
state++
```

释放一次：

```text
state--
```

直到：

```text
state = 0
```

才算真正释放锁。

**lock()：**

```java
lock.lock();
```

如果暂时拿不到锁，会等待获取。

在线程等待过程中收到中断：

```text
不中断拿锁流程
不立即抛异常
保留中断状态
```

**lockInterruptibly()：**

```java
lock.lockInterruptibly();
```

在等待获取锁期间可以响应中断：

```text
收到 interrupt
   ↓
放弃继续等待锁
   ↓
抛 InterruptedException
```

**公平锁与非公平锁：**

```java
new ReentrantLock(true);   // 公平锁
new ReentrantLock(false);  // 非公平锁
```

公平锁：

> 尽量按照等待队列中的先后顺序获取锁。

非公平锁：

> 新来的线程也可以直接尝试抢锁。

默认：

```text
非公平锁
```

---

## Condition

`Condition` 用于实现类似 `wait / notify` 的条件等待机制，但必须配合 `Lock` 使用。

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

核心方法：

```java
condition.await();
condition.signal();
condition.signalAll();
```

**await()：**

```text
当前线程持有 Lock
       ↓
condition.await()
       ↓
进入 Condition 条件队列
       ↓
完全释放 Lock
       ↓
park 阻塞
```

**signal()：**

```text
Condition 条件队列
       ↓
取出等待节点
       ↓
转移到 AQS 同步队列
       ↓
重新竞争锁
       ↓
获得锁后 await() 返回
```

AQS 内部可以理解为存在两类队列：

```text
同步队列
→ 等待获取锁的线程
→ 双向队列

Condition 条件队列
→ 调用 await() 等待条件的线程
→ 单向等待链
```

一个 `ReentrantLock` 可以创建多个 `Condition`：

```java
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();
```

这是它相比一个 Monitor 的 `wait / notify` 更灵活的地方之一。

---

# JUC集合

常用 JUC 并发集合：

| 类 | 核心特点 |
| --- | --- |
| `ConcurrentHashMap` | 高并发线程安全 Map |
| `CopyOnWriteArrayList` | 写时复制，适合读多写少 |
| `ConcurrentLinkedQueue` | CAS 实现的无界非阻塞 FIFO 队列 |
| `BlockingQueue` | 支持阻塞等待的生产者-消费者队列 |

## ConcurrentHashMap

**JDK 7：**

采用：

```text
Segment 分段锁
```

一个 `ConcurrentHashMap` 被划分成多个 Segment，每个 Segment 类似一个小型 Hashtable。

```text
ConcurrentHashMap
├── Segment 1
├── Segment 2
├── Segment 3
└── Segment 4
```

锁粒度主要是 Segment。

**JDK 8：**

取消 Segment，底层结构变为：

```text
数组
+
链表
+
红黑树
```

线程安全主要依赖：

```text
CAS
+
synchronized
+
volatile
```

**初始化：**

采用懒初始化，创建 `ConcurrentHashMap` 时通常不会立刻创建实际 table，首次真正插入时再完成初始化。

容量会规整为 2 的幂次方。

**put 流程：**

```text
put(key, value)
      ↓
table 是否初始化？
      │
      ├── 否 → 初始化
      │
      └── 是
           ↓
        找目标桶
           │
           ├── 桶为空
           │     ↓
           │   CAS 插入
           │
           ├── 正在扩容
           │     ↓
           │   协助迁移
           │
           └── 桶非空
                 ↓
            synchronized 桶头
                 ↓
            链表 / 红黑树插入
```

**树化：**

桶内节点达到树化阈值，并且数组容量达到最小树化容量时，链表可以转换成红黑树。

常见阈值：

```text
TREEIFY_THRESHOLD = 8
MIN_TREEIFY_CAPACITY = 64
```

如果数组容量太小：

```text
优先扩容
而不是立即树化
```

**扩容：**

JDK 8 ConcurrentHashMap 支持多个线程协助迁移。

常见关键点：

```text
sizeCtl
→ 控制初始化 / 扩容状态

ForwardingNode
→ 表示当前桶已经迁移
```

线程执行 `put` 时如果发现正在扩容，可以帮助完成迁移。

**get：**

`get()` 通常不对整个结构加互斥锁，主要依赖 volatile 可见性和并发数据结构设计，因此读性能较高。

---

## CopyOnWriteArrayList

**底层结构：**

```text
数组
```

核心思想：

```text
Copy-On-Write
写时复制
```

**读操作：**

```text
直接读取当前数组快照
通常不加锁
```

因此：

```java
list.get(index);
```

以及遍历操作的读开销较低。

**写操作：**

```text
add / set / remove
      ↓
     加锁
      ↓
复制旧数组
      ↓
修改新数组
      ↓
替换数组引用
```

例如：

```text
旧数组：
[A][B][C]

add(D)
   ↓
复制
   ↓
新数组：
[A][B][C][D]
   ↓
内部引用指向新数组
```

**迭代器：**

迭代器基于创建时的数组快照：

```text
创建 Iterator
     ↓
保存当前数组引用
     ↓
后续集合被修改
     ↓
Iterator 仍遍历旧快照
```

因此通常不会抛：

```text
ConcurrentModificationException
```

**优点：**

- 读操作开销低
- 遍历无需额外加锁
- 适合读多写少

**缺点：**

- 每次写操作都需要复制数组
- 数据量大时内存开销明显
- 迭代器看到的是快照，不保证看到后续最新修改

**适用场景：**

```text
读远多于写
+
数据规模不大
+
允许快照式读取
```

例如：

```text
配置列表
订阅者列表
```

---

## ConcurrentLinkedQueue

`ConcurrentLinkedQueue` 是：

```text
无界
+
非阻塞
+
FIFO
```

的线程安全队列。

**底层结构：**

```text
单向链表
```

**线程安全：**

主要依赖：

```text
CAS
+
volatile
```

而不是 `synchronized` 或显式锁。

**初始化：**

逻辑上可以理解为：

```text
head
 ↓
空节点
 ↑
tail
```

**offer 入队：**

```text
找到真正尾节点
     ↓
next == null
     ↓
CAS 把新节点挂到 next
     ↓
尝试推进 tail
```

如果发现其他线程已经推进队列：

```text
帮助推进 tail
```

`tail` 允许暂时落后于真正队尾，不影响正确性。

**poll 出队：**

```text
从 head 开始找有效节点
      ↓
队列为空？
      │
      ├── 是 → 返回 null
      │
      └── 否
           ↓
        CAS 推进 head
           ↓
        返回首元素
```

`head` 也允许暂时滞后。

**优点：**

- 无锁阻塞竞争
- 高并发下吞吐量较好
- 适合短操作队列

**缺点：**

- CAS 失败可能产生重试开销
- 无界队列可能持续占用内存
- 不提供“队列空了就阻塞等待”的语义

---

## BlockingQueue

`BlockingQueue` 是一个支持阻塞语义的队列接口，常用于生产者-消费者模型。

核心能力：

```text
队列空
→ 消费者可以等待

队列满
→ 生产者可以等待
```

**核心方法：**

| 操作 | 抛异常 | 返回特殊值 | 一直阻塞 | 超时等待 |
| --- | --- | --- | --- | --- |
| 插入 | `add(e)` | `offer(e)` | `put(e)` | `offer(e,time,unit)` |
| 删除 | `remove()` | `poll()` | `take()` | `poll(time,unit)` |
| 查看 | `element()` | `peek()` | — | — |

常见实现：

| 实现类 | 特点 |
| --- | --- |
| `ArrayBlockingQueue` | 数组、有界 |
| `LinkedBlockingQueue` | 链表，可指定容量，默认容量很大 |
| `PriorityBlockingQueue` | 优先级、无界 |
| `DelayQueue` | 延迟获取 |
| `SynchronousQueue` | 不存储元素，线程间直接交接 |
| `LinkedTransferQueue` | 无界，支持 transfer 语义 |

**ArrayBlockingQueue：**

```text
数组
+
固定容量
+
一把 Lock
+
两个 Condition
```

两个条件通常可以理解为：

```text
notEmpty
→ 队列非空

notFull
→ 队列未满
```

消费者：

```text
队列为空
   ↓
notEmpty.await()
```

生产者插入后：

```text
notEmpty.signal()
```

生产者：

```text
队列已满
   ↓
notFull.await()
```

消费者取走元素后：

```text
notFull.signal()
```

**LinkedBlockingQueue：**

底层：

```text
链表
```

常见实现使用两把锁：

```text
putLock
→ 控制入队

takeLock
→ 控制出队
```

并通过计数器维护元素数量，从而提高生产和消费之间的并发度。

---

# 线程池

## ThreadPoolExecutor

线程池主要作用：

- 降低频繁创建 / 销毁线程的开销
- 复用线程
- 提高任务响应速度
- 统一管理线程
- 控制系统并发数量

Java 中最核心的线程池实现：

```text
ThreadPoolExecutor
```

---

## 核心参数

`ThreadPoolExecutor` 常见核心参数：

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
| `unit` | `keepAliveTime` 时间单位 |
| `workQueue` | 任务队列 |
| `threadFactory` | 创建线程 |
| `handler` | 拒绝策略 |

---

## 执行流程

提交任务：

```text
execute(task)
     ↓
当前线程数 < corePoolSize？
     │
     ├── 是
     │    ↓
     │ 创建核心线程执行
     │
     └── 否
          ↓
       尝试进入 workQueue
          │
          ├── 入队成功
          │     ↓
          │   等待执行
          │
          └── 队列已满
                ↓
       当前线程数 < maximumPoolSize？
                │
                ├── 是
                │    ↓
                │ 创建非核心线程
                │
                └── 否
                     ↓
                   拒绝任务
```

一句话：

> **先核心线程 → 再入队 → 再扩容到最大线程 → 最后拒绝。**

---

## 常见阻塞队列

线程池中常见任务队列：

```text
ArrayBlockingQueue
LinkedBlockingQueue
SynchronousQueue
DelayQueue
PriorityBlockingQueue
```

具体底层结构可参考前面的 `BlockingQueue`，在线程池部分不再重复展开。

---

## 四种拒绝策略

### AbortPolicy

默认策略：

```text
拒绝任务
+
抛 RejectedExecutionException
```

适合不能静默丢任务、需要及时发现系统过载的场景。

### DiscardPolicy

```text
直接丢弃任务
不抛异常
```

适合允许丢失非关键任务的场景。

### DiscardOldestPolicy

```text
丢弃队列中最旧的任务
      ↓
重新尝试提交当前任务
```

### CallerRunsPolicy

```text
线程池无法执行
      ↓
由提交任务的线程自己执行
```

它可以自然降低任务提交速度，形成一定的反压效果。

---

## 常见线程池类型

`Executors` 提供了一些工厂方法。

### FixedThreadPool

```text
固定线程数
+
无界 LinkedBlockingQueue
```

线程数不会无限增长，但任务可能不断堆积。

### SingleThreadExecutor

```text
单工作线程
+
任务按顺序执行
```

### CachedThreadPool

```text
corePoolSize = 0
+
最大线程数很大
+
SynchronousQueue
```

任务很多时可能快速创建大量线程。

### ScheduledThreadPool

支持：

```text
延迟任务
周期任务
```

---

## 为什么通常不推荐直接使用 Executors 创建线程池

因为部分工厂方法隐藏了关键参数，容易造成资源不可控：

```text
FixedThreadPool
SingleThreadExecutor
→ 队列可能大量堆积
→ OOM 风险

CachedThreadPool
→ 最大线程数非常大
→ 线程数量可能暴涨
```

实际项目中通常更推荐根据业务特点显式配置：

```text
ThreadPoolExecutor
```

包括：

- 核心线程数
- 最大线程数
- 有界队列容量
- 拒绝策略
- 线程工厂

---

## 线程池关闭

**shutdown()：**

```text
停止接收新任务
+
继续执行已经提交的任务
+
等待任务完成后关闭
```

**shutdownNow()：**

```text
停止接收新任务
+
尝试 interrupt 正在执行的任务
+
返回队列中尚未开始的任务
```

需要注意：

> `shutdownNow()` 只是尝试通过中断让任务结束，并不能保证任务一定立即停止。

---

# JUC同步工具

常见同步工具类：

| 类 | 一句话记忆 |
| --- | --- |
| `CountDownLatch` | 等别人做完 |
| `CyclicBarrier` | 等大家到齐 |
| `Semaphore` | 控制同时访问资源的线程数量 |
| `Exchanger` | 两个线程交换数据 |
| `Phaser` | 更灵活的多阶段同步 |

## CountDownLatch

**作用：**

一个或多个线程等待其他线程完成任务。

```text
任务 A ─┐
任务 B ─┼→ countDown()
任务 C ─┘
        ↓
      count = 0
        ↓
等待线程继续执行
```

核心方法：

```java
latch.await();
latch.countDown();
```

特点：

- 内部维护计数器
- `countDown()` 使计数减 1
- 计数到 0 后等待线程继续执行
- 一次性使用，不能重置

典型场景：

```text
主线程等待多个子任务完成
然后统一汇总结果
```

---

## CyclicBarrier

**作用：**

让一组线程全部到达某个屏障后，再一起继续。

```text
线程 A ─┐
线程 B ─┼→ await()
线程 C ─┘
       ↓
    全部到齐
       ↓
    一起继续
```

核心方法：

```java
barrier.await();
```

特点：

- 可以重复使用
- 可以指定屏障动作
- 适合多阶段协作

例如：

```text
第一阶段计算
   ↓
所有线程到齐
   ↓
第二阶段计算
```

---

## Semaphore

`Semaphore` 通过许可证 `permit` 控制同时访问资源的线程数量。

```text
permit = 3

最多允许 3 个线程同时进入
```

核心方法：

```java
semaphore.acquire();
semaphore.release();
```

执行逻辑：

```text
acquire()
   ↓
获得 permit
   ↓
访问资源
   ↓
release()
   ↓
归还 permit
```

典型场景：

- 接口限流
- 数据库连接池
- 控制同时访问某资源的线程数量

---

## Exchanger

`Exchanger` 用于两个线程之间交换数据。

核心方法：

```java
exchanger.exchange(data);
```

可以理解：

```text
线程 A：dataA ─┐
               ├→ exchange
线程 B：dataB ─┘

结果：

线程 A 得到 dataB
线程 B 得到 dataA
```

适合两个线程之间进行数据交换、校验等场景。

---

## Phaser

`Phaser` 是更灵活的多阶段线程同步工具。

特点：

- 类似 `CyclicBarrier`
- 支持多个阶段
- 支持参与线程动态注册
- 支持参与线程动态注销

适合：

```text
参与者数量可能动态变化
+
需要多阶段协作
```
