# 操作系统
## 目录
1. 操作系统基础
2. 进程与线程
3. CPU 调度
4. 同步、互斥与死锁
5. 内存管理
6. 文件与磁盘
7. I/O 系统

> 学习主线：**操作系统基础 → 进程与线程 → CPU 调度 → 同步、互斥与死锁 → 内存管理 → 文件与磁盘 → I/O 系统**

### 知识地图
```text
OS
├── 基础：用户态 / 内核态、系统调用、中断、内核结构
├── CPU：进程 / 线程、调度、同步、死锁
├── Memory：地址转换、分页 / 分段、虚拟内存、COW
├── Storage：文件系统、inode / FD、磁盘调度
└── I/O：设备控制、I/O 模型、epoll、Zero-Copy
```

# 操作系统基础
## 操作系统定义
操作系统（Operating System）

作用：管理硬件资源、管理软件资源、组织任务执行、完成资源分配、向应用程序提供统一接口

一句话：

> **操作系统 = 硬件资源管理者 + 应用程序运行环境。**

## 操作系统四个基本特征
并发、共享、虚拟、异步

### 并发
Concurrency

多个事件：在同一时间段内推进

单核 CPU：宏观并发、微观交替

多核 CPU：还可以真正并行

注意：并发、≠、并行

### 共享
多个进程：共同使用系统资源

分为：互斥共享、同时共享

### 虚拟
把：一个物理资源

抽象成：多个逻辑资源

例如：Virtual Memory、Virtual CPU

### 异步
进程执行会走走停停、速度不可预知，因此需要同步和调度机制保证正确运行。
## 操作系统功能与接口
操作系统既是**资源管理者**，也是用户 / 应用与硬件之间的接口：

- **资源管理**：处理机、内存、文件、设备。
- **命令接口**：用户直接与系统交互。
- **程序接口**：应用通过系统调用请求内核服务。
- **GUI**：图形界面最终仍通过系统提供的能力完成操作。

### 常见系统类型
- **批处理系统**：按批处理作业，交互性弱；多道程序可提高 CPU 利用率。
- **分时系统**：以时间片轮转等方式让多个用户 / 任务交互使用系统，强调响应时间。
- **实时系统**：要求任务在规定时间内完成，强调及时性和可靠性。

## 用户态与内核态
CPU 通常区分 **User Mode（用户态）** 和 **Kernel Mode（内核态）**。

### 用户态
应用程序运行在：较低权限

不能直接执行：特权指令

也不能直接操作：关键硬件资源

### 内核态
内核态权限较高，可执行特权指令并直接管理硬件和系统资源。

可以：访问硬件、管理页表、处理中断、进行进程调度、执行设备 I/O

### 为什么区分两种状态
主要：保护系统资源、隔离应用程序、防止普通程序直接破坏系统

## 特权指令
只能在：Kernel Mode

执行。

例如：修改页表、关闭 / 开启中断、访问特权寄存器、直接控制设备

普通应用程序：不能直接执行

## 系统调用
System Call：应用程序请求内核服务的受控接口。

例如：

```text
open
read
write
fork
mmap
socket
```

### 为什么需要系统调用
应用不能直接：操作磁盘、管理页表、创建进程、控制设备

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

这样：统一管理资源、保证安全、保证隔离

### 系统调用常见分类
Process Control、File Management、Device Management、Memory Management、Communication

## 中断与异常
不要把所有进入内核的情况都混成一个概念。

可以这样理解：

```text
进入内核
├── System Call
├── Exception
└── Hardware Interrupt
```

### Hardware Interrupt
外部异步事件。

例如：Keyboard、Network Card、Disk、Timer

特点：与当前正在执行的指令、不一定有直接关系

### Exception
由：当前指令执行

引起。

常见：Trap、Fault、Abort

### Trap
有意触发、通常执行后返回下一条指令

典型：调试 Trap、某些系统调用入口机制

### Fault
当前指令尚未正常完成

处理成功后：可以重新执行当前指令

典型：Page Fault

### Abort
严重错误、通常无法恢复

程序可能：直接终止

## 中断处理基本流程
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

### 中断作用
如果没有中断：CPU、只能不断轮询设备

有中断：设备有事、→ 主动通知 CPU

提高：CPU 利用率、响应能力、并发能力

## 操作系统内核与结构
Kernel：操作系统最核心部分

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

### 大内核与微内核
- **大内核（Monolithic Kernel）**：大量系统服务运行在内核态，调用链短、性能高，但内核内部耦合更强。
- **微内核（Microkernel）**：只把核心机制放在内核，其余服务尽量放到用户态，隔离性和可扩展性更好，但可能增加通信和上下文切换开销。

### 原语
**原语（Primitive）**是操作系统底层不可分割的一组操作，执行时间短、调用频繁，用于实现进程控制、同步等关键机制。P / V 操作可以理解为典型的原子同步操作。

# 进程与线程
## 程序与进程
Program：静态的可执行文件

Process：程序的一次运行实例

所以：程序、→ 静态、进程、→ 动态

## 进程
进程主要用于：资源隔离、资源分配、程序运行

可以简单理解：

> **进程是资源拥有与隔离的基本单位。**

#### 进程的基本特征
进程具有**动态性、并发性、独立性、异步性和结构性**。其中动态性是最基本特征；进程实体通常由程序段、数据段和 PCB 组成。

## 进程组成
经典：

```text
Process
├── PCB
├── Program
└── Data
```

## PCB
PCB：Process Control Block

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

## 进程状态
经典五状态：New、Ready、Running、Blocked、Terminated

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

#### Ready 与 Blocked
Ready：什么都准备好了、只差 CPU

Blocked：即使给 CPU、也暂时无法继续

例如：等待 I/O、等待锁、等待事件

### 进程创建、阻塞、唤醒与终止
- **创建**：为新进程建立 PCB、分配必要资源并进入就绪状态。
- **阻塞**：运行中的进程因等待 I/O、资源或事件而主动进入阻塞态。
- **唤醒**：等待条件满足后，由系统将阻塞进程转为就绪态。
- **终止**：正常完成、异常错误或外部干预都可能结束进程并回收资源。

> **调度是“决定谁运行”，切换是“真正保存 / 恢复上下文并换人运行”。**

## 线程
Thread：进程中的执行单元

一个进程：可以有多个线程

可以简单理解：

> **线程主要解决一个进程内部多个执行流并发的问题。**

### 线程共享什么
同一进程中的线程通常共享：Virtual Address Space、Code、Heap、Global Data、Open Files

### 每个线程独有
Thread Stack、Registers、Program Counter、Scheduling State、Thread Local Storage

所以：线程共享进程资源、但拥有独立执行现场

## 进程与线程区别
Process；→ Resource Ownership / Isolation；Thread；→ Scheduling / Execution

对比：

| 对比 | 进程 | 线程 |
| --- | --- | --- |
| 地址空间 | 通常独立 | 同进程共享 |
| 资源隔离 | 强 | 弱 |
| 调度 | 可调度 | 可调度 |
| 创建开销 | 较大 | 较小 |
| 切换开销 | 通常较大 | 同进程线程通常较小 |
| 崩溃影响 | 通常隔离较好 | 一个线程严重错误可能拖垮整个进程 |

一句话：进程、→ 隔离、线程、→ 并发

## 用户级线程与内核级线程
- **用户级线程（ULT）**：线程管理主要在用户态完成，切换开销小；若采用多对一模型，一个线程发生阻塞可能影响整个进程。
- **内核级线程（KLT）**：由内核直接管理和调度，可在多核 CPU 上并行执行；线程调度 / 切换需要内核参与。

Kernel Thread：由 OS Kernel、直接管理

内核为线程维护：TCB、Scheduling State、CPU Context

优点：线程阻塞、不会阻塞同进程其他线程、支持多核真正并行

缺点：线程调度 / 切换、需要内核参与

## 上下文切换
Context Switch：CPU 从一个执行实体、切换到另一个

需要保存 / 恢复：Registers、Program Counter、Stack Pointer、Scheduling State

### 线程切换
同一进程内：Address Space、通常不需要切换

主要切：CPU Context、Thread Stack

### 进程切换
除了 CPU Context：还涉及地址空间相关上下文

例如：Page Table / MMU Context

还可能影响：TLB、CPU Cache Locality

所以通常：Process Switch、>、Thread Switch

但不是因为：每次都复制全局变量、或文件描述符

## 进程间通信 IPC
IPC：Inter-Process Communication

因为进程地址空间隔离：Process A、不能直接读、Process B 的私有地址空间

所以需要：IPC Mechanism

### Pipe
Pipe：内核中的字节流缓冲区

特点：FIFO、字节流、容量有限、经典匿名管道常用于有亲缘关系进程

通常：单方向

双向：创建两个 Pipe

读空：Reader Block

写满：Writer Block

### Message Queue
消息队列：Kernel 中保存多个 Message

特点：消息有边界、可以按消息类型组织

缺点：数据通常需要、User ↔ Kernel、复制

### Shared Memory
共享内存：多个进程、把同一组物理页、映射到各自地址空间

建立共享内存时：仍需要系统调用

映射建立后：通信数据、不需要反复经过内核拷贝

因此：速度很高

但：多个进程同时读写、→ Race Condition

需要：Semaphore、Mutex、其他同步机制

### Semaphore
Semaphore：用于表示资源数量或进行同步协调的计数器。

主要：同步、互斥、资源数量控制

经典操作：P / wait、V / signal

记：Semaphore、→ 协调

> **注意：** Semaphore 常被列在 IPC / 进程协作机制中，但它主要用于**同步、互斥和资源计数**，本身不负责传输业务数据。

### Signal
Signal：异步事件通知机制

例如：SIGINT、SIGTERM、SIGKILL、SIGCHLD

进程收到 Signal 后：Default Action、Catch、Ignore

部分 Signal：不能被捕获或忽略

例如：SIGKILL、SIGSTOP

记：Signal、→ 通知、Semaphore、→ 协调

### Socket
Socket：本机进程、或、不同主机进程

都可以通信。

常见：TCP Socket、UDP Socket、Unix Domain Socket

# CPU 调度
## 调度目标与常用指标
常见指标：

- **CPU 利用率**：CPU 忙碌时间占比。
- **吞吐量**：单位时间完成的作业数。
- **周转时间** = 完成时间 - 提交时间。
- **带权周转时间** = 周转时间 / 实际运行时间。
- **等待时间**：任务在就绪队列中等待 CPU 的总时间。
- **响应时间**：从请求提交到第一次得到响应的时间。

不同场景侧重点不同：批处理更关注吞吐量和周转时间，交互系统更关注响应时间。

## 调度层次
经典：高级调度、中级调度、低级调度

## 高级调度
又称：Job Scheduling

决定：哪些作业、从外存进入内存

频率：最低

## 中级调度
又称：Memory Scheduling

决定：哪些挂起进程、重新调入内存

与：Swap

相关。

## 低级调度
又称：CPU Scheduling、Process Scheduling

决定：Ready Queue 中、谁获得 CPU

频率：最高

## 调度时机
进程主动放弃 CPU：Exit、Block、Wait I/O、Wait Lock

被动失去 CPU：Time Slice Exhausted、Higher Priority Task、Interrupt / Preemption

### 调度与切换
Scheduling；→ 做决策：下一个运行谁；Context Switch；→ 做动作：保存当前上下文，恢复下一个上下文

发生调度不一定立刻意味着完成一次进程切换，但真正换运行实体时需要执行上下文切换。

## 抢占式与非抢占式
### Preemptive
OS、可以强制收回 CPU

适合：Interactive System、Real-Time System、Modern General OS

### Non-Preemptive
进程主动释放 CPU、之后才切换

实现简单。

## 调度算法
### FCFS
First Come First Served

先到先执行。

优点：简单

缺点：Convoy Effect、长任务可能拖累短任务

### SJF
Shortest Job First

最短作业优先。

优点：理论上可降低平均等待时间

缺点：需要预测运行时间、长任务可能饥饿

### HRRN
Highest Response Ratio Next

响应比：(Waiting Time + Service Time)、/、Service Time

等待越久：优先级逐渐提高

### RR
Round Robin

每个进程：获得一个 Time Quantum

时间片用完：重新排到 Ready Queue

适合：Interactive / Time-Sharing

### Priority Scheduling
优先级高、先执行

问题：低优先级任务、可能 Starvation

解决：Aging

### MLFQ
Multi-Level Feedback Queue

核心：多个优先级队列、+、动态调整优先级

典型思想：交互型短任务、→ 高优先级、持续占用 CPU 的任务、→ 逐渐下降

兼顾：响应时间、吞吐量、公平性

## 多核调度
常见：Global Run Queue、Per-CPU Run Queue

### 公共就绪队列
优点：天然容易负载均衡

缺点：全局锁竞争、CPU Cache Affinity 较差

### Per-CPU Queue
优点：Cache Affinity 好、竞争更小

问题：CPU 之间可能负载不均

因此需要：Load Balancing

# 同步、互斥与死锁
## Race Condition
多个线程 / 进程：同时访问共享可变数据

最终结果：依赖执行时序

就产生：Race Condition

## Critical Section
访问共享临界资源的代码区域：Critical Section

目标：同一时刻、只允许满足规则的线程进入

经典原则：空闲让进、忙则等待、有限等待、让权等待

## 互斥实现方法
经典互斥实现可以分为：

- **软件方法**：如 Peterson 算法，通过共享标志和让步规则协调两个执行者。
- **硬件原子指令**：如 Test-and-Set、Swap、CAS，为自旋锁和许多无锁算法提供基础。

软件 / 自旋类方案可能存在忙等；阻塞型 Mutex 则会在等待时间较长时让线程睡眠。

## Mutex
Mutex：Mutual Exclusion Lock、互斥锁

特点：同一时刻、只有一个线程持有

其他竞争者：阻塞 / 等待

适合：临界区可能执行较久

## Spinlock
Spinlock：获取不到锁、→ Busy Waiting

不主动睡眠。

特点：避免 Sleep / Wakeup 开销、但会持续占 CPU

适合：临界区很短、等待时间很短、多核环境

不适合：长时间持锁

### CAS
Spinlock 常依赖：CAS、Compare-And-Swap

CAS：硬件提供的原子操作

可以：检查值、+、更新值

一次完成。

## Read-Write Lock
Read Lock、Write Lock

规则：多个 Reader、可以并发、Writer、通常独占

适合：Read Much、Write Less

## Condition Variable
Condition Variable：等待某个条件成立

通常配合：Mutex

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

被唤醒线程：重新获得 Mutex、后继续执行

## Semaphore
Semaphore 可以表示：可用资源数量

例如：S = 3

表示：最多 3 个执行者、同时使用资源

### Binary Semaphore
0 / 1

可用于：互斥

### Counting Semaphore
N

用于：限制并发数量

## Monitor
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

它把：共享状态、和、同步规则

封装在一起。

## 经典同步问题
经典题型主要用于理解 Semaphore / Monitor 的组合使用：

- **生产者-消费者**：既要保证缓冲区访问互斥，又要协调“非空 / 非满”。
- **读者-写者**：允许多个读者并发，但写者通常要求独占。
- **哲学家进餐**：展示多个资源获取顺序不当可能导致死锁。

重点不是死记代码，而是先判断：**哪些是互斥关系，哪些是前驱 / 同步关系。**

## Deadlock
Deadlock：多个进程 / 线程、互相等待对方持有的资源

最终：所有参与者都无法继续

## 死锁四个必要条件
Mutual Exclusion、Hold and Wait、No Preemption、Circular Wait

即：互斥、持有并等待、不可剥夺、循环等待

四个同时存在：才可能发生死锁

## 死锁预防
Prevention：主动破坏四个必要条件之一

例如：

```text
一次性申请全部资源
→ 破坏 Hold and Wait

资源有序申请
→ 破坏 Circular Wait

允许抢占资源
→ 破坏 No Preemption
```

## 死锁避免
Avoidance：不直接破坏必要条件

而是：每次分配资源之前、判断分配后是否仍处于安全状态

典型：Banker's Algorithm

## 银行家算法
核心：Available、Max、Allocation、Need

其中：Need = Max - Allocation

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

### 安全状态
存在一个 Safe Sequence

意味着：所有进程、都可以按某种顺序完成

### 不安全状态
注意：Unsafe、≠、已经 Deadlock

只是：存在未来死锁风险

## 死锁检测与解除
如果允许死锁发生：Detect、+、Recover

解除方式：Resource Preemption、Terminate Process、Rollback

### 死锁、饥饿与不安全状态
- **死锁**：多个执行者形成循环等待，参与者都无法继续。
- **饥饿**：某个执行者长期得不到资源或 CPU，但系统整体仍可能运行。
- **不安全状态**：不能保证存在安全序列，表示有死锁风险，**不等于已经死锁**。

# 内存管理
## 操作系统内存管理职责
主要：分配与回收、地址转换、内存保护、虚拟内存、共享

## 程序装入、链接与重定位
程序从源码到运行可以粗略理解为：Source、→ Compile、→ Link、→ Load、→ Execute

**链接方式：**
- 静态链接：运行前把目标模块和库连接成完整程序。
- 装入时动态链接：模块装入内存时再链接。
- 运行时动态链接：真正使用某模块时再完成链接，现代系统更常见。

**装入 / 重定位：**
- 绝对装入：编译时就确定实际地址，灵活性低。
- 可重定位装入（静态重定位）：装入时一次完成地址修正，运行后通常不再移动。
- 动态运行时装入：运行过程中通过硬件地址转换把逻辑地址映射到物理地址，更适合现代多任务系统。

## 虚拟地址空间
每个进程通常看到：独立 Virtual Address Space

进程使用：Virtual Address

CPU / MMU 最终访问：Physical Address

## 程序典型内存布局
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

### Text
保存：Machine Code

### Data
保存：已初始化、全局变量 / 静态变量

### BSS
保存：未显式初始化、或零初始化、的全局 / 静态变量

### Heap
用于：Dynamic Allocation

例如：malloc、new

### mmap Area
可能放：Shared Library、Memory-Mapped File、Anonymous Mapping、Shared Memory

### Stack
保存：Stack Frame、Local Variable、Return Address、Saved Register、Function Call Context

栈大小：受系统 / 进程配置限制

不要把：8MB

当作所有环境固定值。

## Heap 与 Stack
### Heap
动态分配区域

由：Allocator / Runtime

管理。

C：malloc/free

Java：GC 管理对象生命周期

### Stack
函数调用栈

一般：Function Enter、→ 创建 Stack Frame、Function Return、→ 回收 Stack Frame

## 连续内存分配
历史 / 教材模型：Single Continuous、Fixed Partition、Dynamic Partition

### 内部碎片
已经分配给进程：但进程用不到的空间

例如：固定分区、Page 内剩余空间

### 外部碎片
空闲空间很多：但不连续

无法满足：大块连续内存申请

## 动态分区算法
经典：First Fit、Best Fit、Worst Fit、Next Fit

### First Fit
按地址：从头找第一个够大的分区

### Best Fit
选择：最小但足够大的分区

问题：容易留下很多小碎片

### Worst Fit
选择：最大的空闲分区

### Next Fit
从：上次查找结束的位置、继续查找

## 分页
Paging：Virtual Memory、切成固定大小 Page、Physical Memory、切成同样大小 Frame

Page：可以放到任意空闲 Frame

因此：不要求物理连续

### Page Size
x86 / x86-64 Linux：常见基础 Page Size、→ 4KB

但：Page Size、与体系结构 / 配置有关

还存在：Huge Page

## 虚拟地址转换
虚拟地址：Virtual Page Number、+、Offset

经过 Page Table：VPN、→ Physical Frame Number

最终：Physical Address、=、Frame Base Address、+、Offset

## Page Table
Page Table：Virtual Page、→ Physical Frame

映射表。

通常还记录：Present、Read / Write、User / Kernel、Accessed、Dirty

等属性。

## MMU
MMU：Memory Management Unit

负责：Virtual Address、→ Physical Address

地址转换。

## TLB
TLB：Translation Lookaside Buffer

保存：最近使用的 Page Table Entry

作用：加速地址转换

### 地址转换流程
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

重点：TLB Miss、≠、Page Fault

## Page Fault
Page Fault：访问某 Virtual Page、但当前映射不能满足访问

常见：页面尚未装入 RAM、COW 写入、权限错误

如果是合法但尚未装入：Kernel、→ 分配 / 找到 Physical Page、→ 更新 Page Table、→ 重新执行指令

## 分段
Segmentation：按程序逻辑结构、划分多个长度可变 Segment

例如：Code、Data、Stack

地址：Segment Number、+、Offset

Segment Table 常包含：Base、Limit、Protection

不要把：程序固定分成 4 个 Segment

当作分段机制定义。

## 分页与分段
```text
Paging
→ 固定大小
→ 面向物理内存管理

Segmentation
→ 可变大小
→ 面向程序逻辑结构
```

碎片：Paging、→ 无外部碎片、→ 有内部碎片、Segmentation、→ 可能有外部碎片

## 段页式
段页式结合两种思想：用户地址空间、→ 按逻辑分段、→ 每个段再分页、→ 物理内存按页框管理

优点是兼顾分段的逻辑组织 / 保护共享能力与分页的离散内存管理；代价是地址转换层级更多、管理结构更复杂。

## 虚拟内存
虚拟内存的经典特征可以记为：**多次性、对换性、虚拟性**。它依赖局部性原理，只把当前需要的部分装入内存。

Virtual Memory：让程序看到的可用地址空间、可以大于当前实际 RAM

基于：Locality、Demand Paging、Page Replacement、Address Translation

### 局部性原理
程序访问内存通常：不是完全随机

存在：Temporal Locality、Spatial Locality

#### 时间局部性
刚访问的数据：很可能再次访问

#### 空间局部性
访问某地址后：附近地址、很可能被访问

## Demand Paging
程序启动：不必把所有页面、立即装进内存

只在真正访问时：

```text
Page Fault
   ↓
调入页面
```

## 页面置换
当：需要新 Page、但没有空闲 Frame

OS：选择 Victim Page、换出

目标：尽量减少 Page Fault

### OPT
Optimal

淘汰：未来最久不会访问的页

理论最优：无法真正实现

主要用于：理论比较

### FIFO
First In First Out

最早进入内存的 Page：先淘汰

可能出现：Belady Anomaly

即：增加 Frame、反而增加 Page Fault

### LRU
Least Recently Used

`OPT` 和 `LRU` 属于栈算法，不会出现 Belady 异常；`FIFO` 可能出现。

淘汰：最近最久没访问的 Page

理论效果好。

但严格实现：成本较高

### Clock
Clock：Reference Bit、+、Circular List

近似：LRU

流程：Reference = 0、→ 淘汰、Reference = 1、→ 置 0、→ 指针继续

## Thrashing
Thrashing：Page Fault 过于频繁

表现：Page In、Page Out、Page In、Page Out

大量时间：消耗在换页

CPU 实际执行程序的时间：反而下降

### 原因
典型：Working Set、>、分配给进程的 Frames

### Working Set
某时间窗口内：进程实际频繁访问的 Page 集合

### 解决
增加 Physical Memory、给进程更多 Frame、降低 Multiprogramming Degree、改进 Replacement Strategy、控制 Working Set

## fork
fork：创建子进程

经典理解：

```text
Parent
   ↓ fork
Child
```

子进程获得：自己的 Virtual Address Space

但并不是立即：复制父进程全部 Physical Memory

## Copy-On-Write
fork 后：父子页表、暂时指向相同 Physical Pages

并把相关页面：标记为只读 / COW

如果双方只读：继续共享

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

### COW 优点
减少 fork 初始复制成本、节省 Physical Memory、提高 fork 性能

尤其：fork 后马上 exec

时非常划算。

一句话：

> **fork 先共享，写的时候才复制。**

## malloc、brk 与 mmap
### malloc
malloc：C Library Function

不是：System Call

它是：User-Space Allocator

底层可能通过：brk / sbrk、mmap

向 OS 获取虚拟内存。

### brk
通过改变：Program Break

扩展 / 收缩 Heap。

### mmap
可以建立：File Mapping、Anonymous Mapping、Shared Mapping、Private Mapping

用于：Memory-Mapped File、Large Allocation、Shared Memory

等场景。

### malloc 的阈值
一些 glibc 实现：小块、→ 倾向 Heap / brk、大块、→ 倾向 mmap

但：具体阈值、与 glibc 版本 / 运行时策略有关

不要死记：128KB

为固定规则。

## Linux 内存不足
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

如果内存紧张：Memory Reclaim

### 后台回收
kswapd

内核后台线程：异步回收内存

### Direct Reclaim
如果：后台回收跟不上

当前申请进程可能：直接参与回收

特点：同步、会阻塞当前分配路径

## 可回收内存
常见：File-backed Pages、Anonymous Pages

### File-backed Page
例如：Page Cache

干净页：可直接丢弃、需要时重新从文件读取

脏页：先写回 Storage、再回收

### Anonymous Page
例如：Heap、Stack、Anonymous mmap

没有对应文件作为后备。

如果启用 Swap：可以换出到 Swap

之后需要：再从 Swap 换入

## OOM
如果：Reclaim、仍无法满足关键内存申请

系统可能进入：OOM、Out Of Memory

Linux 可能触发：OOM Killer

### OOM Killer
不是简单：谁占内存最多、就一定杀谁

而是根据：oom_badness / oom_score、oom_score_adj、内存使用、进程属性

综合选择 Victim。

目标：尽快释放足够资源

# 文件与磁盘
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

## 文件基础与打开流程
文件系统从用户角度解决“**按名存取**”，从系统角度负责文件组织、空间分配、访问控制和持久化。

常见操作：`create`、`open`、`read`、`write`、`seek`、`close`、`delete`、`truncate`。

传统教材常用 **FCB（File Control Block）** 表示文件控制信息；Unix/Linux 更常从 inode、目录项和打开文件表理解：文件名、→ 目录项、→ FCB / inode 等元数据、→ 打开文件表、→ 进程 FD

打开文件后，后续读写可通过 FD / 打开文件表定位文件，无需每次都从路径重新查找。

## 文件逻辑结构
从用户角度，文件可以看成：

- **无结构文件（流式文件）**：字节流，没有固定记录结构。
- **有结构文件（记录式文件）**：可进一步采用顺序、索引、索引顺序等组织方式。

逻辑结构描述“用户看到的数据组织方式”；连续 / 链接 / 索引分配描述的是“文件在外存上的物理组织方式”，两者不要混淆。

## File Descriptor
进程访问打开的文件 / Socket：通常通过 FD

例如：

```text
0
→ stdin

1
→ stdout

2
→ stderr
```

### FD 关系
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

## inode
inode：文件元数据结构

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

通常：File Name、不直接保存在 inode 中

目录负责：Name、→ inode

映射。

## 文件目录
Directory：Name、→ inode / File Metadata

经典目录结构：Single-Level      单级目录、Two-Level         两级目录、Tree              树形多级目录、Acyclic Graph     无环图目录（便于共享）

- **绝对路径**：从根目录开始。
- **相对路径**：从当前工作目录开始。

现代系统常见：Tree-like Hierarchy

## 文件分配方式
### 连续分配
文件占据：连续 Disk Blocks

优点：顺序 / 随机访问快

问题：外部碎片、文件增长困难

### 链接分配
每个 Block：指向下一个 Block

优点：无外部碎片、扩展方便

缺点：随机访问慢、指针有额外开销

### 索引分配
建立：Index Block

记录：Logical Block、→ Physical Block

思想类似：Page Table

### 多级索引
如果文件很大：Single Indirect、Double Indirect、Triple Indirect

等结构。

### 混合索引
典型：

```text
Direct Block
+
Indirect Block
+
Double Indirect
...
```

优点：Small File、→ 直接地址快、Large File、→ 间接索引扩展性好

## 空闲空间管理
经典：Free Table、Free List、Bitmap、Grouped Linking

现代文件系统常见思想：Bitmap、Extent、Tree-based Free Space

## Hard Link
Hard Link：多个 Directory Entry、指向同一个 inode

因此：inode 相同、Data 相同

删除其中一个路径：只减少 Link Count

只要仍有：其他 Hard Link、或打开的引用

数据不一定立刻消失。

## Symbolic Link
Symlink：有自己的 inode

它的数据内容通常保存：目标 Path

因此：可以跨文件系统、可以指向目录

如果目标被删：可能成为 Dangling Link

#### Hard Link 与 Symlink
```text
Hard Link
→ 多个名字
  指向同一个 inode

Symlink
→ 一个独立文件
  内容是目标路径
```

## 文件保护
常见保护手段：**访问控制、口令 / 身份认证、加密**。现代系统常通过权限位、ACL 等机制限制不同用户对文件的读、写、执行等操作。

文件保护不仅针对文件本身，目录的搜索、创建、删除权限同样会影响最终访问能力。

## VFS
VFS：Virtual File System

作用：给应用统一文件 API

上层：open、read、write、close

下层可以是：ext4、XFS、tmpfs、NFS、...

## 磁盘访问与调度
一次传统磁盘访问时间主要由：寻道时间 + 旋转延迟 + 数据传输时间

组成，其中调度算法主要试图降低磁头移动带来的寻道开销。

| 算法 | 核心思想 | 特点 |
| --- | --- | --- |
| FCFS | 按请求到达顺序处理 | 公平、简单，平均寻道距离可能较大 |
| SSTF | 优先服务离当前磁头最近的请求 | 平均寻道较小，但远端请求可能饥饿 |
| SCAN | 像电梯一样沿一个方向服务，到边界后反向 | 分布较均衡 |
| C-SCAN | 只按一个方向服务，到端点后快速返回 | 等待时间更均匀 |
| LOOK | 类似 SCAN，但只走到当前方向最远请求 | 减少无效移动 |
| C-LOOK | 类似 C-SCAN，只到最远请求后回到另一端请求 | 减少无效移动 |

# I/O 系统
## I/O 设备与控制方式
按信息交换单位常分为：
- **块设备**：以块为单位，可寻址，典型如磁盘。
- **字符设备**：以字符 / 字节流为单位，典型如终端等。

经典 I/O 控制方式：

```text
程序直接控制 / 轮询
        ↓
中断驱动
        ↓
DMA
        ↓
通道控制（经典大型机思想）
```

- **轮询**：CPU 反复检查设备状态，简单但浪费 CPU。
- **中断驱动**：设备就绪后通过中断通知 CPU，减少忙等。
- **DMA**：设备与内存之间批量传输数据，CPU 主要负责初始化和完成处理，显著减少 CPU 搬运数据的负担。
- **通道**：经典体系结构中由专用 I/O 处理部件执行更完整的 I/O 控制任务。

## I/O 软件层次
```text
用户程序
  ↓
设备无关 I/O 软件 / 系统调用层
  ↓
设备驱动程序
  ↓
中断处理
  ↓
设备控制器 / 硬件
```

**设备独立性**：应用尽量使用统一的逻辑接口，具体硬件差异由设备驱动等层次屏蔽。

## 缓冲与 SPOOLing
**缓冲（Buffering）**主要用于缓和 CPU 与 I/O 设备速度不匹配、减少频繁中断和提高 CPU / I/O 并行性。常见有单缓冲、双缓冲和缓冲池。

**SPOOLing（假脱机）**利用磁盘等辅助存储和软件队列，把独占设备在逻辑上改造成可被多个任务共享的“虚拟设备”，经典例子是打印任务排队。

## Blocking I/O
应用：发起 I/O

如果数据没准备好：线程阻塞

直到：I/O 可以继续 / 完成

## Non-blocking I/O
应用：发起 I/O

如果暂时不能完成：立即返回

通常返回：EAGAIN、EWOULDBLOCK

应用可以：稍后重试

## I/O Multiplexing
一个线程：同时等待多个 FD

典型：select、poll、epoll

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

注意：Multiplexing、→ 等待 Ready、不是：、内核替应用完成所有 I/O

## select
select：fd_set Bitmap

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

特点：每轮复制 / 扫描 FD 集合、O(n)

`FD_SETSIZE`：常见默认 1024

属于：接口 / 用户态数据结构限制

不是：“重编译内核才能改变”

这种简单结论。

## poll
poll：struct pollfd[]

相比 select：没有固定 fd_set 位图上限

但：仍需扫描全部关注 FD

复杂度：O(n)

## epoll
epoll：Linux 高并发 I/O Multiplexing

典型 API：epoll_create、epoll_ctl、epoll_wait

### epoll_ctl
用于：ADD、MOD、DEL

关注的 FD。

### epoll_wait
用于：等待 Ready Event

只返回：已经就绪的事件

### 内部理解
可以简化：Interest Set、→ 内核维护、Ready List、→ 保存就绪事件

常见源码理解：红黑树、→ 管理关注 FD、就绪链表、→ 保存 Ready FD

重点：select / poll、→ 每次问“所有 FD 谁 Ready？”、epoll、→ 内核维护状态、直接给 Ready Event

### epoll 优势
无需每轮传完整 FD 集合、无需每轮线性扫描所有关注 FD、更适合大量连接、少量活跃

不要简单死记：epoll = O(1)

## LT
LT：Level Triggered

只要：FD 仍处于 Ready 状态

就可能：继续通知

优点：实现简单、容错高

## ET
ET：Edge Triggered

主要在：状态发生变化

时通知。

通常：配合 Non-blocking FD

收到 Ready 后：循环 read / write、直到 EAGAIN / EWOULDBLOCK

不要理解成：一次 read()、必须读完所有数据

而是：一次事件处理循环、尽量处理到暂时无数据可读

#### LT 与 ET
LT；→ 状态还 Ready；继续提醒；ET；→ 状态变化时重点提醒

不要死记：ET 一定比 LT 快

更准确：ET、→ 通知次数可能更少、→ 实现更复杂、LT、→ 更容易写正确

## Signal-Driven I/O
Signal-Driven I/O：内核在 FD Ready 时、发送 Signal

重点：通知的是 Ready

应用仍然：自己调用 read/write

## Asynchronous I/O
AIO：

```text
应用发起异步请求
   ↓
内核完成真正 I/O
   ↓
完成后通知应用
```

重点：完成通知

所以：Signal-Driven、→ Ready、AIO、→ Complete

## 零拷贝
Zero-Copy：目标是减少不必要的数据复制

尤其：User Space、↔、Kernel Space

之间的 CPU Copy。

### 传统文件发送
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

还伴随：read()、write()

多个系统调用。

### Zero-Copy 核心
目标：减少：、CPU Copy、User/Kernel Copy、System Call、Context Switch

不是说：整条硬件路径、真正 0 次搬运

## sendfile
sendfile：File FD、→ Socket FD

数据：尽量在 Kernel 内部流转

避免：先 copy 到 User Buffer、再 copy 回 Kernel

## mmap
mmap：把文件映射到进程 Virtual Address Space

避免传统：read()、把文件内容复制到独立 User Buffer

## splice
splice：在两个 FD 之间、在 Kernel 内部移动数据

常与：Pipe

配合。

## Direct I/O
Direct I/O：绕过 Page Cache

适合某些：Database、Storage Engine

场景。

它和 Zero-Copy：不是完全等价概念

只是：减少内核缓存层参与

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

## User / Kernel
User Mode；→ 普通程序；Kernel Mode；→ OS 核心

进入内核常见：System Call、Exception、Interrupt

## Process / Thread
Process；→ Resource Isolation；Thread；→ Execution / Scheduling

共享：Code、Heap、Address Space、Files

线程独有：Stack、Register、PC

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

## IPC
```text
Pipe
Message Queue
Shared Memory
Semaphore
Signal
Socket
```

记：Signal、→ 通知、Semaphore、→ 协调

## Scheduling
```text
FCFS
SJF
HRRN
RR
Priority
MLFQ
```

## Deadlock
四条件：互斥、持有等待、不可剥夺、循环等待

```text
Prevention
→ 破坏条件

Avoidance
→ 保持 Safe State

Detection
→ 允许发生后检测
```

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

重点：TLB Miss、≠、Page Fault

## Paging
Virtual Memory；→ Page；Physical Memory；→ Frame

Paging；→ 无外部碎片；→ 有内部碎片

## Virtual Memory
Locality；+；Demand Paging；+；Page Replacement

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

记：先共享、写时复制

## malloc
malloc；→ Library Function；底层可能：；brk；mmap

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

## Link
Hard Link；→ 同 inode；Symlink；→ 独立 inode；→ 保存目标路径

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

## LT / ET
LT；→ Ready 还在就继续通知；ET；→ 状态变化重点通知；→ Non-blocking + 读到 EAGAIN

## Zero-Copy
核心：、减少 User ↔ Kernel Copy、减少 CPU Copy、减少 System Call / Context Switch

典型：sendfile、mmap、splice、Direct I/O

## 一句话总结
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
