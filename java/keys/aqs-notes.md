# AQS

> AQS（`AbstractQueuedSynchronizer`）是 JUC 中构建锁和同步器的基础框架。  
> 核心思想：**用 `state` 表示同步状态，用 FIFO 等待队列管理获取失败的线程，用 `park/unpark` 实现阻塞与唤醒。**

# 1. 核心组成

```text
AQS
├── state              同步状态
├── 同步队列            管理获取失败的线程
├── tryAcquire/...     子类定义“怎么获取资源”
├── tryRelease/...     子类定义“怎么释放资源”
└── park / unpark      阻塞与唤醒线程
```

## state

AQS 内部维护一个 `volatile int state`，不同同步器赋予它不同含义：

| 同步器 | `state` 含义 |
| --- | --- |
| `ReentrantLock` | 锁的重入次数 |
| `Semaphore` | 剩余许可证数量 |
| `CountDownLatch` | 剩余计数 |

AQS 提供：

```java
getState();
setState();
compareAndSetState(expect, update);
```

真正如何解释 `state`，由具体同步器决定。

# 2. AQS 本身不是锁

AQS 只是一个**同步器框架**，它不直接规定：

- 锁是否公平
- 是否可重入
- `state` 具体表示什么
- 获取和释放资源的规则

这些通常由子类实现：

```java
tryAcquire()
tryRelease()
tryAcquireShared()
tryReleaseShared()
```

例如：

```text
ReentrantLock
    ↓
   Sync
    ↓
   AQS
```

`ReentrantLock` 负责锁的具体语义，AQS 负责排队、阻塞和唤醒。

# 3. 独占模式获取资源

以 `ReentrantLock` 获取锁为例：

```text
线程调用 lock()
    ↓
tryAcquire()
    ↓
成功 ─────────────→ 获得锁
    ↓ 失败
加入 AQS 同步队列
    ↓
检查前驱节点
    ↓
前驱是 head？
 ├─ 是 → 再尝试 tryAcquire()
 │          ├─ 成功 → 自己成为新的 head
 │          └─ 失败 → park()
 │
 └─ 否 → park()
```

可以简化记成：

```text
抢锁
↓
失败
↓
排队
↓
轮到自己
↓
再抢
↓
失败就 park
↓
被唤醒后继续抢
```

## 为什么加入队列后还要再尝试一次

线程入队期间，原来的持锁线程可能刚好释放了锁。

因此节点进入队列后，如果发现自己的前驱已经是 `head`，会再尝试获取资源，避免不必要地直接睡眠。

# 4. 为什么通常只有 head 的后继去抢

例如同步队列：

```text
head → t1 → t2 → t3
        ↑
   第一个真实等待节点
```

通常 `t1` 才有资格尝试获取锁。

`t2`、`t3` 继续等待自己的前驱向前推进。

这样可以避免：

```text
t1、t2、t3 同时被唤醒
→ 全部竞争
→ 大量上下文切换
```

所以 AQS 是一种**队列化竞争**。

# 5. park 与 unpark

获取资源失败后，AQS 会通过：

```java
LockSupport.park();
```

阻塞当前线程。

释放资源时，会通过类似：

```java
LockSupport.unpark(thread);
```

唤醒合适的后继线程。

正常流程：

```text
线程 A 持有锁

线程 B
tryAcquire 失败
↓
进入 AQS 队列
↓
park()

线程 A unlock()
↓
release()
↓
tryRelease()
↓
state 释放成功
↓
unpark(B)
↓
B 从 park 返回
↓
再次 tryAcquire()
```

注意：

> `unpark()` 只是让线程从 `park()` 返回，并不是直接把锁“交给”线程。

被唤醒的线程仍然需要重新执行 `tryAcquire()`。

# 6. release 流程

独占模式释放资源：

```text
unlock()
  ↓
release()
  ↓
tryRelease()
  ↓
资源是否真正释放？
  ├─ 否 → 结束
  │
  └─ 是
      ↓
   唤醒后继节点
      ↓
   LockSupport.unpark()
```

例如 `ReentrantLock` 是可重入锁：

```text
state = 3
unlock → 2
unlock → 1
unlock → 0
```

只有 `state` 真正减到 `0`，锁才完全释放，才需要唤醒等待线程。

# 7. 公平锁与非公平锁

AQS **本身不是公平锁，也不是非公平锁**。

公平性由具体同步器的获取策略决定。

## 公平锁

公平锁获取前会检查前面是否已经有线程等待：

```java
!hasQueuedPredecessors()
```

大致逻辑：

```text
新线程 t3 到来
↓
发现 t1、t2 已经排队
↓
不能插队
↓
进入队尾
```

```text
head → t1 → t2 → t3
```

## 非公平锁

非公平锁允许新来的线程直接 CAS 尝试：

```text
AQS 已有：
head → t1 → t2

此时 t3 新来
↓
直接 CAS 抢锁
├─ 成功 → t3 插队
└─ 失败 → t3 入队
```

因此：

> **AQS 队列本身基本按 FIFO 推进；公平与非公平的主要区别是，是否允许队列外的新线程插队。**

# 8. interrupt 在 AQS 中的表现

AQS 正常唤醒等待线程靠：

```text
unpark()
```

不是：

```text
interrupt()
```

但 `park()` 本身可以响应中断，因此等待线程被 `interrupt()` 时也会从 `park()` 返回。

具体怎么处理，取决于调用方式。

## lock()

```java
lock.lock();
```

属于不可中断获取。

等待期间即使被 `interrupt()`：

```text
interrupt()
↓
park 返回
↓
不放弃获取锁
↓
继续等待 / 竞争
↓
最终获得锁后保留中断语义
```

所以 `lock()` 不会因为等待期间被中断就直接抛 `InterruptedException`。

## lockInterruptibly()

```java
lock.lockInterruptibly();
```

属于可中断获取。

等待期间被中断：

```text
interrupt()
↓
park 返回
↓
检测到中断
↓
取消当前等待
↓
抛 InterruptedException
```

因此：

```text
lock()
→ 中断不能让它放弃拿锁

lockInterruptibly()
→ 中断可以让它退出等待
```

# 9. Condition 与 AQS

`Condition` 还会维护自己的**条件等待队列**。

调用：

```java
condition.await();
```

大致过程：

```text
当前线程持有锁
↓
await()
↓
进入 Condition 条件队列
↓
完全释放当前锁
↓
park()
```

其他线程：

```java
condition.signal();
```

不是直接把锁交给等待线程，而是：

```text
Condition 条件队列
        ↓ signal
AQS 同步队列
        ↓
等待重新获取锁
        ↓
获取成功
        ↓
await() 返回
```

所以：

> `signal()` 的本质是把节点从 **Condition 队列转移到 AQS 同步队列**，之后还要重新竞争锁。

# 10. 独占模式与共享模式

AQS 支持两种资源获取方式：

| 模式 | 含义 | 典型类 |
| --- | --- | --- |
| 独占模式 | 同一时刻通常只有一个线程成功 | `ReentrantLock` |
| 共享模式 | 多个线程可以同时成功 | `Semaphore`、`CountDownLatch` |

对应核心方法：

```text
独占：
tryAcquire()
tryRelease()

共享：
tryAcquireShared()
tryReleaseShared()
```

# 11. AQS 与 ReentrantLock 的关系

可以这样理解：

```text
ReentrantLock
负责：
├── 可重入
├── 公平 / 非公平
├── lock / unlock
└── Condition

AQS
负责：
├── state
├── 同步队列
├── 节点排队
├── park 阻塞
└── unpark 唤醒
```

所以：

> **ReentrantLock 决定锁的规则，AQS 提供实现这些规则所需的基础设施。**

# 12. 整体流程速记

## 获取锁

```text
lock()
↓
tryAcquire()
├─ 成功 → 执行业务
│
└─ 失败
    ↓
  入 AQS 队列
    ↓
  判断前驱
    ↓
  轮到自己时再次 tryAcquire()
    ├─ 成功 → 成为新 head
    └─ 失败 → park
                  ↓
               被 unpark
                  ↓
               再次竞争
```

## 释放锁

```text
unlock()
↓
release()
↓
tryRelease()
↓
state 真正释放
↓
unpark 后继线程
↓
后继线程重新 tryAcquire()
```

# 13. 面试速记

```text
AQS
= state + 同步队列 + CAS + park/unpark

获取失败
→ 入队
→ 前驱是 head 时尝试获取
→ 获取失败 park

释放资源
→ tryRelease
→ unpark 合适的后继
→ 后继重新竞争

公平性
→ AQS 不决定
→ 具体同步器决定是否允许插队

interrupt
→ 不是 AQS 正常唤醒手段
→ 正常唤醒靠 unpark
→ 但 interrupt 也可以让 park 返回

Condition
→ await：同步队列之外进入条件队列并释放锁
→ signal：条件队列 → 同步队列
→ 仍需重新竞争锁
```

一句话：

> **AQS 就是把“抢不到资源怎么办”这件事统一处理掉：抢不到就排队，排到前面再尝试，实在不行就 `park()`，资源释放后再 `unpark()` 唤醒。**
