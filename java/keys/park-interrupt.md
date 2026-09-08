# LockSupport.park 与 Thread.interrupt

> 核心：`park()` 用于阻塞当前线程；`interrupt()` 用于给线程发送中断通知。  
> `interrupt()` 可以让处于 `park()` 中的线程返回，但它和 `unpark()` 不是同一种机制。

# park

`LockSupport.park()` 用于**阻塞当前线程**，常作为锁和同步器底层的等待机制。

```java
LockSupport.park();
```

线程调用 `park()` 后，如果当前没有可用 permit，一般会进入等待状态。

`park()` 可能因为以下原因返回：

- 其他线程调用 `LockSupport.unpark(thread)`
- 当前线程被 `interrupt()`
- `parkNanos()` / `parkUntil()` 超时
- 无理由返回（spurious return）

因此实际使用通常要结合条件循环：

```java
while (!condition) {
    LockSupport.park();
}
```

## permit

`LockSupport` 可以把每个线程理解成最多拥有一个 permit：

```text
permit = 0 / 1
```

`unpark(thread)` 发放 permit，`park()` 消费 permit。

```text
unpark(thread)
→ permit = 1

park()
→ 消费 permit
→ permit = 0
→ 直接返回
```

因此 `unpark()` 可以先于 `park()`。

注意：permit **不会累计**，连续多次 `unpark()` 也不会得到多个 permit。

# interrupt

`Thread.interrupt()` 是 Java 的**协作式中断通知机制**。

```java
thread.interrupt();
```

它的核心作用是：

```text
interrupt flag = true
```

它不会直接强制杀死线程，线程如何响应取决于当前状态和代码逻辑。

## 普通运行线程

线程正常运行时：

```java
thread.interrupt();
```

通常只是把中断标记设为 `true`，线程仍会继续执行。

线程可以主动检查：

```java
if (Thread.currentThread().isInterrupted()) {
    return;
}
```

## sleep / wait / join

线程处于 `sleep()`、`wait()`、`join()` 时被中断：

```text
interrupt()
→ 结束等待
→ 抛 InterruptedException
→ 中断标记通常被清除
```

常见处理：

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
```

重新调用 `interrupt()` 是为了恢复被清除的中断状态。

## park

线程处于 `park()` 时被中断：

```text
interrupt()
→ interrupt flag = true
→ park() 返回
→ 不抛 InterruptedException
→ 中断标记仍然保留
```

例如：

```java
Thread t = new Thread(() -> {
    System.out.println("park 前");

    LockSupport.park();

    System.out.println("park 后");
    System.out.println(Thread.currentThread().isInterrupted());
});

t.start();

Thread.sleep(1000);
t.interrupt();
```

输出类似：

```text
park 前
park 后
true
```

# park 与 interrupt 的关系

`park()` 会把“线程发生中断”视为一种**结束等待的条件**。

因此：

```text
线程
 ↓
park()
 ↓
阻塞

其他线程
 ↓
thread.interrupt()
 ↓
interrupt flag = true
 ↓
park() 返回
 ↓
线程继续向后执行
```

注意：`interrupt()` 让 `park()` 返回，并不代表线程一定要终止。

线程可以选择继续执行：

```java
LockSupport.park();

System.out.println("继续执行");
```

也可以主动检查中断状态后结束：

```java
LockSupport.park();

if (Thread.currentThread().isInterrupted()) {
    return;
}
```

所以：

> **interrupt 只是发送中断通知，不决定线程最终是继续还是退出。**

# park 与 unpark / interrupt 对比

| 操作 | 核心作用 | 能让 `park()` 返回 | 是否设置中断标记 |
| --- | --- | --- | --- |
| `park()` | 阻塞当前线程 | — | 否 |
| `unpark(thread)` | 给目标线程 permit | 是 | 否 |
| `thread.interrupt()` | 设置中断状态 | 是 | 是 |

两种让 `park()` 返回的方式：

```text
             park() 返回
                 ↑
        ┌────────┴────────┐
        │                 │
    unpark()          interrupt()
        │                 │
   发放 permit       设置中断标记
        │                 │
 interrupt 不变      interrupt = true
```

# 中断标记为什么重要

`park()` **不会清除中断标记**。

因此线程已经被中断后：

```java
Thread.currentThread().interrupt();

LockSupport.park();
LockSupport.park();
```

由于中断标记一直为 `true`，后续 `park()` 也可能直接返回，而不再真正阻塞。

如果需要清除当前线程的中断标记：

```java
Thread.interrupted();
```

它会：

```text
读取当前线程中断状态
+
清除中断标记
```

对比：

- `isInterrupted()`：查询，不清除。
- `Thread.interrupted()`：查询当前线程，并清除。

# 与 wait / sleep / join 对比

| 方法 | 被 `interrupt()` 后 | 是否抛异常 | 中断标记 |
| --- | --- | --- | --- |
| `wait()` | 结束等待 | `InterruptedException` | 通常清除 |
| `sleep()` | 结束等待 | `InterruptedException` | 通常清除 |
| `join()` | 结束等待 | `InterruptedException` | 通常清除 |
| `park()` | 直接返回 | 不抛异常 | **保留** |

# 速记

```text
park()
→ 阻塞当前线程
→ 可被 unpark 或 interrupt 弄醒
→ 不保证返回一定是因为 unpark

unpark(thread)
→ 发放 permit
→ 不设置中断标记
→ 可以先于 park

interrupt()
→ 设置 interrupt flag = true
→ 不等于强制终止线程
→ 可以让 park 返回
→ park 返回后中断标记仍保留
```

一句话：

> **`park()` 负责等待，`unpark()` 通过 permit 唤醒，`interrupt()` 通过中断状态让等待结束；线程收到中断后是继续还是退出，由代码决定。**
