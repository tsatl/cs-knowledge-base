# 操作系统基础知识结构速记

> 本篇由两份操作系统笔记合并整理。
>
> 主线：

```text
操作系统基础
   ↓
进程与线程
   ↓
CPU 调度
   ↓
同步、互斥与死锁
   ↓
内存管理
   ↓
文件系统
   ↓
I/O 系统
```

核心理解：

```text
OS
│
├── CPU
│   ├── Process
│   ├── Thread
│   └── Scheduling
│
├── Concurrency
│   ├── Mutex
│   ├── Semaphore
│   └── Deadlock
│
├── Memory
│   ├── Virtual Memory
│   ├── Paging / TLB
│   ├── Page Fault
│   └── COW / OOM
│
├── Storage
│   └── File System
│
└── I/O
    ├── Interrupt
    ├── select / poll / epoll
    └── Zero-Copy
```

---

# 操作系统基础

## 操作系统定义

操作系统：

```text
Operating System
```

作用：

```text
管理硬件资源
管理软件资源
组织任务执行
完成资源分配
向应用程序提供统一接口
```

一句话：

> **操作系统 = 硬件资源管理者 + 应用程序运行环境。**

---

## 操作系统四个基本特征

```text
并发
共享
虚拟
异步
```

---

### 并发

```text
Concurrency
```

多个事件：

```text
在同一时间段内推进
```

单核 CPU：

```text
宏观并发
微观交替
```

多核 CPU：

```text
还可以真正并行
```

注意：

```text
并发
≠
并行
```

---

### 共享

多个进程：

```text
共同使用系统资源
```

分为：

```text
互斥共享
同时共享
```

---

### 虚拟

把：

```text
一个物理资源
```

抽象成：

```text
多个逻辑资源
```

例如：

```text
Virtual Memory
Virtual CPU
```

---

### 异步

进程：

```text
走走停停
执行速度不可预知
```

因此需要：

```text
同步机制
调度机制
```

保证程序正确运行。

---

# 用户态与内核态

CPU 通常至少区分：

```text
User Mode
Kernel Mode
```

---

## 用户态

应用程序运行在：

```text
较低权限
```

不能直接执行：

```text
特权指令
```

也不能直接操作：

```text
关键硬件资源
```

---

## 内核态

操作系统内核运行在：

```text
高权限模式
```

可以：

```text
访问硬件
管理页表
处理中断
进行进程调度
执行设备 I/O
```

---

## 为什么区分两种状态

主要：

```text
保护系统资源
隔离应用程序
防止普通程序直接破坏系统
```

---

# 特权指令

只能在：

```text
Kernel Mode
```

执行。

例如：

```text
修改页表
关闭 / 开启中断
访问特权寄存器
直接控制设备
```

普通应用程序：

```text
不能直接执行
```

---

# 系统调用

System Call：

```text
应用程序请求内核服务
的受控接口
```

例如：

```text
open
read
write
fork
mmap
socket
```

---

## 为什么需要系统调用

应用不能直接：

```text
操作磁盘
管理页表
创建进程
控制设备
```

而是：

```text
User Program
   ↓
System Call
   ↓
Kernel
   ↓
完成操作
```

这样：

```text
统一管理资源
保证安全
保证隔离
```

---

## 系统调用常见分类

```text
Process Control
File Management
Device Management
Memory Management
Communication
```

---

# 中断与异常

不要把所有进入内核的情况都混成一个概念。

可以这样理解：

```text
进入内核
├── System Call
├── Exception
└── Hardware Interrupt
```

---

## Hardware Interrupt

外部异步事件。

例如：

```text
Keyboard
Network Card
Disk
Timer
```

特点：

```text
与当前正在执行的指令
不一定有直接关系
```

---

## Exception

由：

```text
当前指令执行
```

引起。

常见：

```text
Trap
Fault
Abort
```

---

## Trap

```text
有意触发
通常执行后返回下一条指令
```

典型：

```text
调试 Trap
某些系统调用入口机制
```

---

## Fault

```text
当前指令尚未正常完成
```

处理成功后：

```text
可以重新执行当前指令
```

典型：

```text
Page Fault
```

---

## Abort

```text
严重错误
通常无法恢复
```

程序可能：

```text
直接终止
```

---

# 中断处理基本流程

```text
发生 Interrupt / Exception
   ↓
CPU 保存必要现场
   ↓
切换到 Kernel Mode
   ↓
根据 Interrupt Vector
找到处理程序
   ↓
执行 Handler
   ↓
恢复现场
   ↓
继续执行 / 调度其他任务
```

---

## 中断作用

如果没有中断：

```text
CPU
只能不断轮询设备
```

有中断：

```text
设备有事
→ 主动通知 CPU
```

提高：

```text
CPU 利用率
响应能力
并发能力
```

---

# 操作系统内核

Kernel：

```text
操作系统最核心部分
```

主要：

```text
Process Management
Memory Management
File System
Device / I/O
Interrupt
Scheduling
System Call
```

---

# 进程与线程

## 程序 vs 进程

Program：

```text
静态的可执行文件
```

Process：

```text
程序的一次运行实例
```

所以：

```text
程序
→ 静态

进程
→ 动态
```

---

# 进程

进程主要用于：

```text
资源隔离
资源分配
程序运行
```

可以简单理解：

> **进程是资源拥有与隔离的基本单位。**

---

## 进程组成

经典：

```text
Process
├── PCB
├── Program
└── Data
```

---

# PCB

PCB：

```text
Process Control Block
```

操作系统描述和管理进程的数据结构。

通常包含：

```text
PID
Process State
CPU Registers
Program Counter
Scheduling Info
Memory Info
Open Files
Signal Info
Parent / Child Relation
```

一句话：

> **PCB = 操作系统眼中的进程。**

---

# 进程状态

经典五状态：

```text
New
Ready
Running
Blocked
Terminated
```

转换：

```text
New
 ↓
Ready
 ↓ dispatch
Running
 ├── time slice / preemption → Ready
 ├── wait I/O              → Blocked
 └── exit                  → Terminated

Blocked
 ↓ event completed
Ready
```

---

## Ready vs Blocked

Ready：

```text
什么都准备好了
只差 CPU
```

Blocked：

```text
即使给 CPU
也暂时无法继续
```

例如：

```text
等待 I/O
等待锁
等待事件
```

---

# 线程

Thread：

```text
进程中的执行单元
```

一个进程：

```text
可以有多个线程
```

可以简单理解：

> **线程主要解决一个进程内部多个执行流并发的问题。**

---

## 线程共享什么

同一进程中的线程通常共享：

```text
Virtual Address Space
Code
Heap
Global Data
Open Files
```

---

## 每个线程独有

```text
Thread Stack
Registers
Program Counter
Scheduling State
Thread Local Storage
```

所以：

```text
线程共享进程资源
但拥有独立执行现场
```

---

# 进程与线程区别

```text
Process
→ Resource Ownership / Isolation

Thread
→ Scheduling / Execution
```

对比：

| 对比 | 进程 | 线程 |
| --- | --- | --- |
| 地址空间 | 通常独立 | 同进程共享 |
| 资源隔离 | 强 | 弱 |
| 调度 | 可调度 | 可调度 |
| 创建开销 | 较大 | 较小 |
| 切换开销 | 通常较大 | 同进程线程通常较小 |
| 崩溃影响 | 通常隔离较好 | 一个线程严重错误可能拖垮整个进程 |

一句话：

```text
进程
→ 隔离

线程
→ 并发
```

---

# 内核级线程

Kernel Thread：

```text
由 OS Kernel
直接管理
```

内核为线程维护：

```text
TCB
Scheduling State
CPU Context
```

优点：

```text
线程阻塞
不会阻塞同进程其他线程

支持多核真正并行
```

缺点：

```text
线程调度 / 切换
需要内核参与
```

---

# 上下文切换

Context Switch：

```text
CPU 从一个执行实体
切换到另一个
```

需要保存 / 恢复：

```text
Registers
Program Counter
Stack Pointer
Scheduling State
```

---

## 线程切换

同一进程内：

```text
Address Space
通常不需要切换
```

主要切：

```text
CPU Context
Thread Stack
```

---

## 进程切换

除了 CPU Context：

```text
还涉及地址空间相关上下文
```

例如：

```text
Page Table / MMU Context
```

还可能影响：

```text
TLB
CPU Cache Locality
```

所以通常：

```text
Process Switch
>
Thread Switch
```

但不是因为：

```text
每次都复制全局变量
或文件描述符
```

---

# 进程间通信 IPC

IPC：

```text
Inter-Process Communication
```

因为进程地址空间隔离：

```text
Process A
不能直接读
Process B 的私有地址空间
```

所以需要：

```text
IPC Mechanism
```

---

## Pipe

Pipe：

```text
内核中的字节流缓冲区
```

特点：

```text
FIFO
字节流
容量有限
经典匿名管道常用于有亲缘关系进程
```

通常：

```text
单方向
```

双向：

```text
创建两个 Pipe
```

读空：

```text
Reader Block
```

写满：

```text
Writer Block
```

---

## Message Queue

消息队列：

```text
Kernel 中保存多个 Message
```

特点：

```text
消息有边界
可以按消息类型组织
```

缺点：

```text
数据通常需要
User ↔ Kernel
复制
```

---

## Shared Memory

共享内存：

```text
多个进程
把同一组物理页
映射到各自地址空间
```

建立共享内存时：

```text
仍需要系统调用
```

映射建立后：

```text
通信数据
不需要反复经过内核拷贝
```

因此：

```text
速度很高
```

但：

```text
多个进程同时读写
→ Race Condition
```

需要：

```text
Semaphore
Mutex
其他同步机制
```

---

## Semaphore

Semaphore：

```text
计数器
```

主要：

```text
同步
互斥
资源数量控制
```

经典操作：

```text
P / wait
V / signal
```

记：

```text
Semaphore
→ 协调
```

---

## Signal

Signal：

```text
异步事件通知机制
```

例如：

```text
SIGINT
SIGTERM
SIGKILL
SIGCHLD
```

进程收到 Signal 后：

```text
Default Action
Catch
Ignore
```

部分 Signal：

```text
不能被捕获或忽略
```

例如：

```text
SIGKILL
SIGSTOP
```

记：

```text
Signal
→ 通知

Semaphore
→ 协调
```

---

## Socket

Socket：

```text
本机进程
或
不同主机进程
```

都可以通信。

常见：

```text
TCP Socket
UDP Socket
Unix Domain Socket
```

---

# CPU 调度

## 调度层次

经典：

```text
高级调度
中级调度
低级调度
```

---

## 高级调度

又称：

```text
Job Scheduling
```

决定：

```text
哪些作业
从外存进入内存
```

频率：

```text
最低
```

---

## 中级调度

又称：

```text
Memory Scheduling
```

决定：

```text
哪些挂起进程
重新调入内存
```

与：

```text
Swap
```

相关。

---

## 低级调度

又称：

```text
CPU Scheduling
Process Scheduling
```

决定：

```text
Ready Queue 中
谁获得 CPU
```

频率：

```text
最高
```

---

# 调度时机

进程主动放弃 CPU：

```text
Exit
Block
Wait I/O
Wait Lock
```

被动失去 CPU：

```text
Time Slice Exhausted
Higher Priority Task
Interrupt / Preemption
```

---

# 抢占式与非抢占式

## Preemptive

```text
OS
可以强制收回 CPU
```

适合：

```text
Interactive System
Real-Time System
Modern General OS
```

---

## Non-Preemptive

```text
进程主动释放 CPU
之后才切换
```

实现简单。

---

# 调度算法

## FCFS

```text
First Come First Served
```

先到先执行。

优点：

```text
简单
```

缺点：

```text
Convoy Effect
长任务可能拖累短任务
```

---

## SJF

```text
Shortest Job First
```

最短作业优先。

优点：

```text
理论上可降低平均等待时间
```

缺点：

```text
需要预测运行时间
长任务可能饥饿
```

---

## HRRN

```text
Highest Response Ratio Next
```

响应比：

```text
(Waiting Time + Service Time)
/
Service Time
```

等待越久：

```text
优先级逐渐提高
```

---

## RR

```text
Round Robin
```

每个进程：

```text
获得一个 Time Quantum
```

时间片用完：

```text
重新排到 Ready Queue
```

适合：

```text
Interactive / Time-Sharing
```

---

## Priority Scheduling

```text
优先级高
先执行
```

问题：

```text
低优先级任务
可能 Starvation
```

解决：

```text
Aging
```

---

## MLFQ

```text
Multi-Level Feedback Queue
```

核心：

```text
多个优先级队列
+
动态调整优先级
```

典型思想：

```text
交互型短任务
→ 高优先级

持续占用 CPU 的任务
→ 逐渐下降
```

兼顾：

```text
响应时间
吞吐量
公平性
```

---

# 多核调度

常见：

```text
Global Run Queue
Per-CPU Run Queue
```

---

## 公共就绪队列

优点：

```text
天然容易负载均衡
```

缺点：

```text
全局锁竞争
CPU Cache Affinity 较差
```

---

## Per-CPU Queue

优点：

```text
Cache Affinity 好
竞争更小
```

问题：

```text
CPU 之间可能负载不均
```

因此需要：

```text
Load Balancing
```

---

# 同步、互斥与死锁

## Race Condition

多个线程 / 进程：

```text
同时访问共享可变数据
```

最终结果：

```text
依赖执行时序
```

就产生：

```text
Race Condition
```

---

# Critical Section

访问共享临界资源的代码区域：

```text
Critical Section
```

目标：

```text
同一时刻
只允许满足规则的线程进入
```

经典原则：

```text
空闲让进
忙则等待
有限等待
让权等待
```

---

# Mutex

Mutex：

```text
Mutual Exclusion Lock
互斥锁
```

特点：

```text
同一时刻
只有一个线程持有
```

其他竞争者：

```text
阻塞 / 等待
```

适合：

```text
临界区可能执行较久
```

---

# Spinlock

Spinlock：

```text
获取不到锁
→ Busy Waiting
```

不主动睡眠。

特点：

```text
避免 Sleep / Wakeup 开销
但会持续占 CPU
```

适合：

```text
临界区很短
等待时间很短
多核环境
```

不适合：

```text
长时间持锁
```

---

## CAS

Spinlock 常依赖：

```text
CAS
Compare-And-Swap
```

CAS：

```text
硬件提供的原子操作
```

可以：

```text
检查值
+
更新值
```

一次完成。

---

# Read-Write Lock

```text
Read Lock
Write Lock
```

规则：

```text
多个 Reader
可以并发

Writer
通常独占
```

适合：

```text
Read Much
Write Less
```

---

# Condition Variable

Condition Variable：

```text
等待某个条件成立
```

通常配合：

```text
Mutex
```

使用。

经典：

```text
Thread A
→ lock
→ condition not met
→ wait
→ 原子释放 mutex + 睡眠

Thread B
→ lock
→ 修改状态
→ signal / broadcast
→ unlock
```

被唤醒线程：

```text
重新获得 Mutex
后继续执行
```

---

# Semaphore

Semaphore 可以表示：

```text
可用资源数量
```

例如：

```text
S = 3
```

表示：

```text
最多 3 个执行者
同时使用资源
```

---

## Binary Semaphore

```text
0 / 1
```

可用于：

```text
互斥
```

---

## Counting Semaphore

```text
N
```

用于：

```text
限制并发数量
```

---

# Monitor

Monitor：

```text
共享数据
+
互斥访问
+
Condition Variable
+
对共享数据的操作
```

它把：

```text
共享状态
和
同步规则
```

封装在一起。

---

# Deadlock

Deadlock：

```text
多个进程 / 线程
互相等待对方持有的资源
```

最终：

```text
所有参与者都无法继续
```

---

# 死锁四个必要条件

```text
Mutual Exclusion
Hold and Wait
No Preemption
Circular Wait
```

即：

```text
互斥
持有并等待
不可剥夺
循环等待
```

四个同时存在：

```text
才可能发生死锁
```

---

# 死锁预防

Prevention：

```text
主动破坏四个必要条件之一
```

例如：

```text
一次性申请全部资源
→ 破坏 Hold and Wait

资源有序申请
→ 破坏 Circular Wait

允许抢占资源
→ 破坏 No Preemption
```

---

# 死锁避免

Avoidance：

```text
不直接破坏必要条件
```

而是：

```text
每次分配资源之前
判断分配后是否仍处于安全状态
```

典型：

```text
Banker's Algorithm
```

---

# 银行家算法

核心：

```text
Available
Max
Allocation
Need
```

其中：

```text
Need = Max - Allocation
```

每次资源请求：

```text
先假设分配
   ↓
寻找 Safe Sequence
   ↓
如果存在
→ 真正分配

如果不存在
→ 等待
```

---

## 安全状态

```text
存在一个 Safe Sequence
```

意味着：

```text
所有进程
都可以按某种顺序完成
```

---

## 不安全状态

注意：

```text
Unsafe
≠
已经 Deadlock
```

只是：

```text
存在未来死锁风险
```

---

# 死锁检测与解除

如果允许死锁发生：

```text
Detect
+
Recover
```

解除方式：

```text
Resource Preemption
Terminate Process
Rollback
```

---

# 内存管理

## 操作系统内存管理职责

主要：

```text
分配与回收
地址转换
内存保护
虚拟内存
共享
```

---

# 虚拟地址空间

每个进程通常看到：

```text
独立 Virtual Address Space
```

进程使用：

```text
Virtual Address
```

CPU / MMU 最终访问：

```text
Physical Address
```

---

# 程序典型内存布局

典型用户空间：

```text
High Address

┌───────────────┐
│ Stack         │
│       ↓       │
├───────────────┤
│ mmap Area     │
├───────────────┤
│       ↑       │
│ Heap          │
├───────────────┤
│ BSS           │
├───────────────┤
│ Data          │
├───────────────┤
│ Text / Code   │
└───────────────┘

Low Address
```

---

## Text

保存：

```text
Machine Code
```

---

## Data

保存：

```text
已初始化
全局变量 / 静态变量
```

---

## BSS

保存：

```text
未显式初始化
或零初始化
的全局 / 静态变量
```

---

## Heap

用于：

```text
Dynamic Allocation
```

例如：

```text
malloc
new
```

---

## mmap Area

可能放：

```text
Shared Library
Memory-Mapped File
Anonymous Mapping
Shared Memory
```

---

## Stack

保存：

```text
Stack Frame
Local Variable
Return Address
Saved Register
Function Call Context
```

栈大小：

```text
受系统 / 进程配置限制
```

不要把：

```text
8MB
```

当作所有环境固定值。

---

# Heap vs Stack

## Heap

```text
动态分配区域
```

由：

```text
Allocator / Runtime
```

管理。

C：

```text
malloc/free
```

Java：

```text
GC 管理对象生命周期
```

---

## Stack

```text
函数调用栈
```

一般：

```text
Function Enter
→ 创建 Stack Frame

Function Return
→ 回收 Stack Frame
```

---

# 连续内存分配

历史 / 教材模型：

```text
Single Continuous
Fixed Partition
Dynamic Partition
```

---

## 内部碎片

已经分配给进程：

```text
但进程用不到的空间
```

例如：

```text
固定分区
Page 内剩余空间
```

---

## 外部碎片

空闲空间很多：

```text
但不连续
```

无法满足：

```text
大块连续内存申请
```

---

# 动态分区算法

经典：

```text
First Fit
Best Fit
Worst Fit
Next Fit
```

---

## First Fit

按地址：

```text
从头找第一个够大的分区
```

---

## Best Fit

选择：

```text
最小但足够大的分区
```

问题：

```text
容易留下很多小碎片
```

---

## Worst Fit

选择：

```text
最大的空闲分区
```

---

## Next Fit

从：

```text
上次查找结束的位置
继续查找
```

---

# 分页

Paging：

```text
Virtual Memory
切成固定大小 Page

Physical Memory
切成同样大小 Frame
```

Page：

```text
可以放到任意空闲 Frame
```

因此：

```text
不要求物理连续
```

---

## Page Size

x86 / x86-64 Linux：

```text
常见基础 Page Size
→ 4KB
```

但：

```text
Page Size
与体系结构 / 配置有关
```

还存在：

```text
Huge Page
```

---

# 虚拟地址转换

虚拟地址：

```text
Virtual Page Number
+
Offset
```

经过 Page Table：

```text
VPN
→ Physical Frame Number
```

最终：

```text
Physical Address
=
Frame Base Address
+
Offset
```

---

# Page Table

Page Table：

```text
Virtual Page
→ Physical Frame
```

映射表。

通常还记录：

```text
Present
Read / Write
User / Kernel
Accessed
Dirty
```

等属性。

---

# MMU

MMU：

```text
Memory Management Unit
```

负责：

```text
Virtual Address
→ Physical Address
```

地址转换。

---

# TLB

TLB：

```text
Translation Lookaside Buffer
```

保存：

```text
最近使用的 Page Table Entry
```

作用：

```text
加速地址转换
```

---

## 地址转换流程

```text
Virtual Address
      ↓
     TLB
   ↙     ↘
 hit     miss
 ↓        ↓
PFN    Page Table
          ↓
      Valid PTE?
       ↙      ↘
      yes      no
       ↓        ↓
   Fill TLB   Page Fault
```

重点：

```text
TLB Miss
≠
Page Fault
```

---

# Page Fault

Page Fault：

```text
访问某 Virtual Page
但当前映射不能满足访问
```

常见：

```text
页面尚未装入 RAM
COW 写入
权限错误
```

如果是合法但尚未装入：

```text
Kernel
→ 分配 / 找到 Physical Page
→ 更新 Page Table
→ 重新执行指令
```

---

# 分段

Segmentation：

```text
按程序逻辑结构
划分多个长度可变 Segment
```

例如：

```text
Code
Data
Stack
```

地址：

```text
Segment Number
+
Offset
```

Segment Table 常包含：

```text
Base
Limit
Protection
```

不要把：

```text
程序固定分成 4 个 Segment
```

当作分段机制定义。

---

# 分页 vs 分段

```text
Paging
→ 固定大小
→ 面向物理内存管理

Segmentation
→ 可变大小
→ 面向程序逻辑结构
```

碎片：

```text
Paging
→ 无外部碎片
→ 有内部碎片

Segmentation
→ 可能有外部碎片
```

---

# 虚拟内存

Virtual Memory：

```text
让程序看到的可用地址空间
可以大于当前实际 RAM
```

基于：

```text
Locality
Demand Paging
Page Replacement
Address Translation
```

---

## 局部性原理

程序访问内存通常：

```text
不是完全随机
```

存在：

```text
Temporal Locality
Spatial Locality
```

---

### 时间局部性

刚访问的数据：

```text
很可能再次访问
```

---

### 空间局部性

访问某地址后：

```text
附近地址
很可能被访问
```

---

# Demand Paging

程序启动：

```text
不必把所有页面
立即装进内存
```

只在真正访问时：

```text
Page Fault
   ↓
调入页面
```

---

# 页面置换

当：

```text
需要新 Page
但没有空闲 Frame
```

OS：

```text
选择 Victim Page
换出
```

目标：

```text
尽量减少 Page Fault
```

---

## OPT

```text
Optimal
```

淘汰：

```text
未来最久不会访问的页
```

理论最优：

```text
无法真正实现
```

主要用于：

```text
理论比较
```

---

## FIFO

```text
First In First Out
```

最早进入内存的 Page：

```text
先淘汰
```

可能出现：

```text
Belady Anomaly
```

即：

```text
增加 Frame
反而增加 Page Fault
```

---

## LRU

```text
Least Recently Used
```

淘汰：

```text
最近最久没访问的 Page
```

理论效果好。

但严格实现：

```text
成本较高
```

---

## Clock

Clock：

```text
Reference Bit
+
Circular List
```

近似：

```text
LRU
```

流程：

```text
Reference = 0
→ 淘汰

Reference = 1
→ 置 0
→ 指针继续
```

---

# Thrashing

Thrashing：

```text
Page Fault 过于频繁
```

表现：

```text
Page In
Page Out
Page In
Page Out
```

大量时间：

```text
消耗在换页
```

CPU 实际执行程序的时间：

```text
反而下降
```

---

## 原因

典型：

```text
Working Set
>
分配给进程的 Frames
```

---

## Working Set

某时间窗口内：

```text
进程实际频繁访问的 Page 集合
```

---

## 解决

```text
增加 Physical Memory
给进程更多 Frame
降低 Multiprogramming Degree
改进 Replacement Strategy
控制 Working Set
```

---

# fork

fork：

```text
创建子进程
```

经典理解：

```text
Parent
   ↓ fork
Child
```

子进程获得：

```text
自己的 Virtual Address Space
```

但并不是立即：

```text
复制父进程全部 Physical Memory
```

---

# Copy-On-Write

fork 后：

```text
父子页表
暂时指向相同 Physical Pages
```

并把相关页面：

```text
标记为只读 / COW
```

如果双方只读：

```text
继续共享
```

某一方写：

```text
Write
 ↓
Page Fault
 ↓
Kernel Copy Page
 ↓
更新该进程 Page Table
 ↓
重新执行写操作
```

---

## COW 优点

```text
减少 fork 初始复制成本
节省 Physical Memory
提高 fork 性能
```

尤其：

```text
fork 后马上 exec
```

时非常划算。

一句话：

> **fork 先共享，写的时候才复制。**

---

# malloc、brk 与 mmap

## malloc

malloc：

```text
C Library Function
```

不是：

```text
System Call
```

它是：

```text
User-Space Allocator
```

底层可能通过：

```text
brk / sbrk
mmap
```

向 OS 获取虚拟内存。

---

## brk

通过改变：

```text
Program Break
```

扩展 / 收缩 Heap。

---

## mmap

可以建立：

```text
File Mapping
Anonymous Mapping
Shared Mapping
Private Mapping
```

用于：

```text
Memory-Mapped File
Large Allocation
Shared Memory
```

等场景。

---

## malloc 的阈值

一些 glibc 实现：

```text
小块
→ 倾向 Heap / brk

大块
→ 倾向 mmap
```

但：

```text
具体阈值
与 glibc 版本 / 运行时策略有关
```

不要死记：

```text
128KB
```

为固定规则。

---

# Linux 内存不足

大致：

```text
Application
→ malloc / mmap
→ 获得 Virtual Memory

真正访问
→ Page Fault

Kernel
→ 尝试分配 Physical Page
```

如果内存紧张：

```text
Memory Reclaim
```

---

## 后台回收

```text
kswapd
```

内核后台线程：

```text
异步回收内存
```

---

## Direct Reclaim

如果：

```text
后台回收跟不上
```

当前申请进程可能：

```text
直接参与回收
```

特点：

```text
同步
会阻塞当前分配路径
```

---

# 可回收内存

常见：

```text
File-backed Pages
Anonymous Pages
```

---

## File-backed Page

例如：

```text
Page Cache
```

干净页：

```text
可直接丢弃
需要时重新从文件读取
```

脏页：

```text
先写回 Storage
再回收
```

---

## Anonymous Page

例如：

```text
Heap
Stack
Anonymous mmap
```

没有对应文件作为后备。

如果启用 Swap：

```text
可以换出到 Swap
```

之后需要：

```text
再从 Swap 换入
```

---

# OOM

如果：

```text
Reclaim
仍无法满足关键内存申请
```

系统可能进入：

```text
OOM
Out Of Memory
```

Linux 可能触发：

```text
OOM Killer
```

---

## OOM Killer

不是简单：

```text
谁占内存最多
就一定杀谁
```

而是根据：

```text
oom_badness / oom_score
oom_score_adj
内存使用
进程属性
```

综合选择 Victim。

目标：

```text
尽快释放足够资源
```

---

# 文件系统

## 文件系统作用

文件系统负责：

```text
文件命名
目录组织
权限
磁盘空间分配
文件访问
共享
持久化
```

---

# File Descriptor

进程访问打开的文件 / Socket：

```text
通常通过 FD
```

例如：

```text
0
→ stdin

1
→ stdout

2
→ stderr
```

---

## FD 关系

可以简化：

```text
Process
  ↓
FD Table
  ↓
Open File Description
  ↓
inode
  ↓
File Data
```

---

# inode

inode：

```text
文件元数据结构
```

典型包含：

```text
File Type
Permissions
Owner
Size
Timestamp
Data Block Pointer
Link Count
```

通常：

```text
File Name
不直接保存在 inode 中
```

目录负责：

```text
Name
→ inode
```

映射。

---

# 文件目录

Directory：

```text
Name
→ inode / File Metadata
```

经典目录结构：

```text
Single-Level
Tree
Acyclic Graph
```

现代系统常见：

```text
Tree-like Hierarchy
```

---

# 文件分配方式

## 连续分配

文件占据：

```text
连续 Disk Blocks
```

优点：

```text
顺序 / 随机访问快
```

问题：

```text
外部碎片
文件增长困难
```

---

## 链接分配

每个 Block：

```text
指向下一个 Block
```

优点：

```text
无外部碎片
扩展方便
```

缺点：

```text
随机访问慢
指针有额外开销
```

---

## 索引分配

建立：

```text
Index Block
```

记录：

```text
Logical Block
→ Physical Block
```

思想类似：

```text
Page Table
```

---

## 多级索引

如果文件很大：

```text
Single Indirect
Double Indirect
Triple Indirect
```

等结构。

---

## 混合索引

典型：

```text
Direct Block
+
Indirect Block
+
Double Indirect
...
```

优点：

```text
Small File
→ 直接地址快

Large File
→ 间接索引扩展性好
```

---

# 空闲空间管理

经典：

```text
Free Table
Free List
Bitmap
Grouped Linking
```

现代文件系统常见思想：

```text
Bitmap
Extent
Tree-based Free Space
```

---

# Hard Link

Hard Link：

```text
多个 Directory Entry
指向同一个 inode
```

因此：

```text
inode 相同
Data 相同
```

删除其中一个路径：

```text
只减少 Link Count
```

只要仍有：

```text
其他 Hard Link
或打开的引用
```

数据不一定立刻消失。

---

# Symbolic Link

Symlink：

```text
有自己的 inode
```

它的数据内容通常保存：

```text
目标 Path
```

因此：

```text
可以跨文件系统
可以指向目录
```

如果目标被删：

```text
可能成为 Dangling Link
```

---

## Hard Link vs Symlink

```text
Hard Link
→ 多个名字
  指向同一个 inode

Symlink
→ 一个独立文件
  内容是目标路径
```

---

# VFS

VFS：

```text
Virtual File System
```

作用：

```text
给应用统一文件 API
```

上层：

```text
open
read
write
close
```

下层可以是：

```text
ext4
XFS
tmpfs
NFS
...
```

---

# I/O 系统

## Blocking I/O

应用：

```text
发起 I/O
```

如果数据没准备好：

```text
线程阻塞
```

直到：

```text
I/O 可以继续 / 完成
```

---

# Non-blocking I/O

应用：

```text
发起 I/O
```

如果暂时不能完成：

```text
立即返回
```

通常返回：

```text
EAGAIN
EWOULDBLOCK
```

应用可以：

```text
稍后重试
```

---

# I/O Multiplexing

一个线程：

```text
同时等待多个 FD
```

典型：

```text
select
poll
epoll
```

流程：

```text
Many FDs
   ↓
Multiplexer
   ↓
Ready FDs
   ↓
Application read/write
```

注意：

```text
Multiplexing
→ 等待 Ready

不是：
内核替应用完成所有 I/O
```

---

# select

select：

```text
fd_set Bitmap
```

每次：

```text
把关注集合交给内核
   ↓
内核扫描
   ↓
返回 Ready 状态
   ↓
用户态再扫描
```

特点：

```text
每轮复制 / 扫描 FD 集合
O(n)
```

`FD_SETSIZE`：

```text
常见默认 1024
```

属于：

```text
接口 / 用户态数据结构限制
```

不是：

```text
“重编译内核才能改变”
```

这种简单结论。

---

# poll

poll：

```text
struct pollfd[]
```

相比 select：

```text
没有固定 fd_set 位图上限
```

但：

```text
仍需扫描全部关注 FD
```

复杂度：

```text
O(n)
```

---

# epoll

epoll：

```text
Linux 高并发 I/O Multiplexing
```

典型 API：

```text
epoll_create
epoll_ctl
epoll_wait
```

---

## epoll_ctl

用于：

```text
ADD
MOD
DEL
```

关注的 FD。

---

## epoll_wait

用于：

```text
等待 Ready Event
```

只返回：

```text
已经就绪的事件
```

---

## 内部理解

可以简化：

```text
Interest Set
→ 内核维护

Ready List
→ 保存就绪事件
```

常见源码理解：

```text
红黑树
→ 管理关注 FD

就绪链表
→ 保存 Ready FD
```

重点：

```text
select / poll
→ 每次问“所有 FD 谁 Ready？”

epoll
→ 内核维护状态
  直接给 Ready Event
```

---

## epoll 优势

```text
无需每轮传完整 FD 集合
无需每轮线性扫描所有关注 FD
更适合大量连接、少量活跃
```

不要简单死记：

```text
epoll = O(1)
```

---

# LT

LT：

```text
Level Triggered
```

只要：

```text
FD 仍处于 Ready 状态
```

就可能：

```text
继续通知
```

优点：

```text
实现简单
容错高
```

---

# ET

ET：

```text
Edge Triggered
```

主要在：

```text
状态发生变化
```

时通知。

通常：

```text
配合 Non-blocking FD
```

收到 Ready 后：

```text
循环 read / write
直到 EAGAIN / EWOULDBLOCK
```

不要理解成：

```text
一次 read()
必须读完所有数据
```

而是：

```text
一次事件处理循环
尽量处理到暂时无数据可读
```

---

## LT vs ET

```text
LT
→ 状态还 Ready
  继续提醒

ET
→ 状态变化时重点提醒
```

不要死记：

```text
ET 一定比 LT 快
```

更准确：

```text
ET
→ 通知次数可能更少
→ 实现更复杂

LT
→ 更容易写正确
```

---

# Signal-Driven I/O

Signal-Driven I/O：

```text
内核在 FD Ready 时
发送 Signal
```

重点：

```text
通知的是 Ready
```

应用仍然：

```text
自己调用 read/write
```

---

# Asynchronous I/O

AIO：

```text
应用发起异步请求
   ↓
内核完成真正 I/O
   ↓
完成后通知应用
```

重点：

```text
完成通知
```

所以：

```text
Signal-Driven
→ Ready

AIO
→ Complete
```

---

# 零拷贝

Zero-Copy：

```text
目标是减少不必要的数据复制
```

尤其：

```text
User Space
↔
Kernel Space
```

之间的 CPU Copy。

---

## 传统文件发送

典型：

```text
Disk
 ↓ DMA
Page Cache
 ↓ CPU Copy
User Buffer
 ↓ CPU Copy
Socket Buffer
 ↓ DMA
NIC
```

还伴随：

```text
read()
write()
```

多个系统调用。

---

## Zero-Copy 核心

目标：

```text
减少：
CPU Copy
User/Kernel Copy
System Call
Context Switch
```

不是说：

```text
整条硬件路径
真正 0 次搬运
```

---

# sendfile

sendfile：

```text
File FD
→ Socket FD
```

数据：

```text
尽量在 Kernel 内部流转
```

避免：

```text
先 copy 到 User Buffer
再 copy 回 Kernel
```

---

# mmap

mmap：

```text
把文件映射到进程 Virtual Address Space
```

避免传统：

```text
read()
把文件内容复制到独立 User Buffer
```

---

# splice

splice：

```text
在两个 FD 之间
在 Kernel 内部移动数据
```

常与：

```text
Pipe
```

配合。

---

# Direct I/O

Direct I/O：

```text
绕过 Page Cache
```

适合某些：

```text
Database
Storage Engine
```

场景。

它和 Zero-Copy：

```text
不是完全等价概念
```

只是：

```text
减少内核缓存层参与
```

---

# 速记

## OS

```text
管理：
CPU
Memory
File
Device
Process
```

---

## User / Kernel

```text
User Mode
→ 普通程序

Kernel Mode
→ OS 核心
```

进入内核常见：

```text
System Call
Exception
Interrupt
```

---

## Process / Thread

```text
Process
→ Resource Isolation

Thread
→ Execution / Scheduling
```

共享：

```text
Code
Heap
Address Space
Files
```

线程独有：

```text
Stack
Register
PC
```

---

## PCB

```text
PID
State
Register
Scheduling
Memory
File
Signal
```

---

## IPC

```text
Pipe
Message Queue
Shared Memory
Semaphore
Signal
Socket
```

记：

```text
Signal
→ 通知

Semaphore
→ 协调
```

---

## Scheduling

```text
FCFS
SJF
HRRN
RR
Priority
MLFQ
```

---

## Deadlock

四条件：

```text
互斥
持有等待
不可剥夺
循环等待
```

```text
Prevention
→ 破坏条件

Avoidance
→ 保持 Safe State

Detection
→ 允许发生后检测
```

---

## Memory

```text
Virtual Address
   ↓
TLB
   ↓
Page Table
   ↓
Physical Frame
```

重点：

```text
TLB Miss
≠
Page Fault
```

---

## Paging

```text
Virtual Memory
→ Page

Physical Memory
→ Frame
```

```text
Paging
→ 无外部碎片
→ 有内部碎片
```

---

## Virtual Memory

```text
Locality
+
Demand Paging
+
Page Replacement
```

---

## Replacement

```text
OPT
→ 理论最优

FIFO
→ 先进先出
→ Belady

LRU
→ 最近最久未用

Clock
→ Reference Bit
→ 近似 LRU
```

---

## COW

```text
fork
 ↓
父子共享 Physical Page
 ↓
Write
 ↓
Page Fault
 ↓
Copy Page
```

记：

```text
先共享
写时复制
```

---

## malloc

```text
malloc
→ Library Function

底层可能：
brk
mmap
```

---

## OOM

```text
内存紧张
 ↓
Reclaim
 ↓
kswapd / Direct Reclaim
 ↓
仍失败
 ↓
OOM Killer
```

---

## File System

```text
Process
 ↓
FD
 ↓
Open File
 ↓
inode
 ↓
Data
```

---

## Link

```text
Hard Link
→ 同 inode

Symlink
→ 独立 inode
→ 保存目标路径
```

---

## I/O

```text
Blocking
→ 等

Non-blocking
→ 立即返回

Multiplexing
→ 等多个 FD Ready

AIO
→ 内核完成后通知
```

---

## select / poll / epoll

```text
select
→ fd_set
→ 扫全部

poll
→ pollfd[]
→ 扫全部

epoll
→ 内核维护 Interest Set
→ 返回 Ready Event
```

---

## LT / ET

```text
LT
→ Ready 还在就继续通知

ET
→ 状态变化重点通知
→ Non-blocking + 读到 EAGAIN
```

---

## Zero-Copy

```text
核心：
减少 User ↔ Kernel Copy
减少 CPU Copy
减少 System Call / Context Switch
```

典型：

```text
sendfile
mmap
splice
Direct I/O
```

---

# 一句话总结

```text
Process / Thread
→ 程序怎么运行

Scheduling
→ CPU 给谁

Synchronization
→ 并发怎么正确

Deadlock
→ 资源为什么互相卡死

Virtual Memory
→ 地址怎么映射、内存怎么扩展

COW / OOM
→ Linux 怎么管理真实内存压力

File System
→ 数据怎么持久化

I/O
→ 程序怎么高效和设备交互
```
