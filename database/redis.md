# Redis 基础知识结构速记

> Redis 学习主线：

```text
数据怎么不丢
→ RDB / AOF

读能力怎么扩展
→ 主从复制

Master 挂了怎么办
→ Sentinel

单机内存不够怎么办
→ Redis Cluster

内存满了怎么办
→ 过期删除 / 内存淘汰

Redis 为什么高并发
→ Event Loop + I/O 多路复用

缓存怎么保护数据库
→ 穿透 / 击穿 / 雪崩
```

---

# 持久化

Redis 数据主要存储在内存中，因此需要持久化机制降低故障后的数据丢失风险。

Redis 主要提供：

```text
RDB
AOF
```

---

## RDB

RDB：

```text
Redis Database
```

本质：

```text
某一时刻 Redis 内存数据的快照
```

可以理解：

```text
内存数据
   ↓
拍一张“照片”
   ↓
RDB 文件
```

### 触发方式

常见：

```text
SAVE
BGSAVE
配置的 save 条件
特定关闭 / 运维操作
```

具体自动触发条件由：

```text
redis.conf 中的 save 配置
```

决定，不要把某一组 `900 1` 之类的配置当成固定规则。

### SAVE

```text
SAVE
→ 主线程执行 RDB
→ 阻塞 Redis
```

执行期间无法正常处理其他请求。

### BGSAVE

```text
主进程
   ↓ fork
子进程
   ↓
生成 RDB
```

父进程继续处理客户端请求，子进程负责读取快照数据并写入 RDB 文件。

### Copy-On-Write

BGSAVE 的关键：

```text
fork
+
Copy-On-Write
```

流程：

```text
Redis 主进程
    ↓ fork
创建子进程
    ↓
父子进程暂时共享原有物理内存页
```

子进程：

```text
读取共享内存
生成 RDB
```

如果主进程只读：

```text
继续共享
无需复制
```

如果主进程修改某个内存页：

```text
写操作
  ↓
触发 COW
  ↓
复制对应内存页
  ↓
主进程修改新副本
```

> **fork 时不是立即复制整个 Redis 内存，而是先共享；发生写操作时才复制相关内存页。**

### RDB 优点

```text
文件紧凑
恢复速度快
适合全量备份
```

### RDB 缺点

```text
快照之间存在时间间隔
两次快照之间的数据可能丢失
fork / COW / 磁盘写入存在系统开销
```

---

## AOF

AOF：

```text
Append Only File
```

核心：

```text
记录 Redis 执行的写命令
```

例如：

```text
SET
DEL
INCR
HSET
LPUSH
...
```

普通读命令如：

```text
GET
```

不会作为恢复数据的写命令记录到 AOF。

可以理解：

```text
RDB
→ 保存“现在数据长什么样”

AOF
→ 保存“数据是怎么一步步变成现在这样的”
```

### AOF 写入流程

```text
客户端写命令
   ↓
Redis 执行
   ↓
追加到 AOF Buffer
   ↓
根据 appendfsync 策略
写入 / 刷盘
```

---

## appendfsync

常见三种：

```text
always
everysec
no
```

### always

```text
每次写命令都尽快 fsync
```

优点：

```text
数据安全性高
```

缺点：

```text
磁盘 IO 频繁
性能最低
```

### everysec

```text
通常每秒 fsync 一次
```

特点：

```text
性能
+
安全性
折中
```

宕机时理论上可能损失最近约 1 秒数据。

### no

```text
Redis 不主动控制 fsync
```

交给操作系统决定何时刷盘。

---

## AOF Rewrite

随着运行时间增加，AOF 会不断变大。

例如：

```text
SET count 1
INCR count
INCR count
INCR count
```

最终：

```text
count = 4
```

AOF Rewrite：

```text
根据 Redis 当前数据状态
重新生成一份更紧凑的 AOF
```

可近似理解为：

```text
SET count 4
```

> **AOF Rewrite 不是简单压缩旧文件，而是根据当前数据集重新构造一份能够恢复当前状态的、更紧凑的新 AOF。**

### AOF 优点

```text
数据丢失窗口更小
恢复粒度更细
```

### AOF 缺点

```text
文件通常更大
磁盘写入更频繁
恢复通常比纯 RDB 慢
```

---

## RDB 与 AOF 对比

| 对比 | RDB | AOF |
| --- | --- | --- |
| 核心 | 内存快照 | 写命令日志 |
| 文件 | 相对紧凑 | 通常更大 |
| 恢复 | 快 | 相对慢 |
| 数据完整性 | 快照间可能丢较多 | 通常更高 |
| 写入方式 | 周期性快照 | 持续追加 |

一句话：

```text
RDB
→ 恢复快

AOF
→ 数据更完整
```

---

## 混合持久化

Redis 可以结合：

```text
RDB
+
AOF
```

思路：

```text
RDB
→ 快速恢复基础数据

AOF
→ 恢复后续增量修改
```

---

# 主从复制与高可用

## 主从复制

Redis 主从复制：

```text
Master
→ 负责写入

Replica
→ 复制 Master 数据
```

现代术语更推荐：

```text
Master / Replica
```

主要解决：

```text
读能力扩展
数据副本
高可用基础
```

---

## 全量同步

Replica 第一次连接 Master，大致：

```text
Replica
   ↓
建立连接
   ↓
PSYNC
   ↓
Master 判断能否增量同步
   ↓
不能
   ↓
FULLRESYNC
   ↓
生成 RDB
   ↓
发送 RDB
   ↓
Replica 加载 RDB
   ↓
补充同步期间的新写命令
```

---

## Replication ID

Replication ID：

```text
replid
```

用于标识：

```text
一段复制历史 / 数据复制来源
```

可以简单理解：

```text
Master 的“复制身份”
```

Replica 同步后会记录对应 replid。

---

## Offset

Offset：

```text
复制偏移量
```

表示：

```text
当前复制到哪里了
```

Master 不断执行写命令：

```text
offset 持续增加
```

Replica 完成同步：

```text
记录自己的 offset
```

如果：

```text
Replica offset
<
Master offset
```

说明 Replica 落后。

---

## replid + offset

Replica 重连 Master 时：

```text
告诉 Master：

我之前属于哪个复制历史
+
我同步到哪个 offset
```

Master 据此判断：

```text
能否增量同步
```

---

## repl_backlog

正确名称：

```text
repl_backlog
```

它可以理解为：

```text
固定大小
环形复制缓冲区
```

保存：

```text
最近一段时间的复制命令
+
对应 offset
```

环形结构：

```text
[0][1][2][3][4][5][6][7]
 ↑                   ↓
 └─────循环覆盖──────┘
```

写到末尾后：

```text
重新从头开始
旧数据逐渐被覆盖
```

---

## 增量同步

Replica 短暂断线后：

```text
重新连接 Master
   ↓
发送 replid + offset
   ↓
Master 检查缺失部分
是否仍在 repl_backlog
```

如果还在：

```text
只发送缺失部分
→ Partial Resynchronization
→ 增量同步
```

如果已被覆盖：

```text
无法补齐
→ 全量同步
```

意义：

```text
减少网络开销
降低同步成本
恢复更快
```

---

## Sentinel

Sentinel：

```text
哨兵模式
```

主要解决：

```text
Master 故障恢复
```

作用：

```text
监控
故障判断
自动故障转移
服务发现 / 通知
```

---

## 主观下线

SDOWN：

```text
Subjectively Down
```

如果某个 Sentinel 认为实例在规定时间内没有正常响应：

```text
标记为主观下线
```

这是：

```text
单个 Sentinel 的判断
```

---

## 客观下线

ODOWN：

```text
Objectively Down
```

多个 Sentinel 都认为 Master 主观下线，达到：

```text
quorum
```

后：

```text
Master 被判断为客观下线
```

---

## quorum

quorum 用于：

```text
判断 Master
是否达到客观下线条件
```

不能简单理解为：

```text
Sentinel 总数的一半
```

需要区分：

```text
ODOWN 判断
→ quorum

Leader 选举 / 故障转移授权
→ 还涉及多数派
```

---

## Sentinel 故障转移

流程：

```text
Master 无响应
   ↓
SDOWN
   ↓
多个 Sentinel 确认
   ↓
ODOWN
   ↓
选举 Sentinel Leader
   ↓
Leader 选择一个 Replica
   ↓
提升为新 Master
   ↓
其他 Replica 指向新 Master
   ↓
旧 Master 恢复
   ↓
变成新 Master 的 Replica
```

---

# Redis Cluster

## 分片集群

Redis Cluster 主要解决：

```text
单机内存容量有限
单个 Master 吞吐有限
```

通过多个 Master：

```text
共同保存不同数据
```

实现水平扩展。

基本结构：

```text
Master A
├── Replica A1
└── Replica A2

Master B
├── Replica B1
└── Replica B2

Master C
├── Replica C1
└── Replica C2
```

每个 Master：

```text
负责部分 Key
```

---

## Hash Slot

Redis Cluster 先把 Key 映射到：

```text
Hash Slot
```

一共有：

```text
16384 个 Slot
```

编号：

```text
0 ~ 16383
```

基本公式：

```text
slot = CRC16(key) % 16384
```

流程：

```text
Key
 ↓
CRC16
 ↓
% 16384
 ↓
Slot
 ↓
对应 Master
```

---

## Slot 与节点

例如：

```text
Master A
→ Slot 0 ~ 5000

Master B
→ Slot 5001 ~ 10000

Master C
→ Slot 10001 ~ 16383
```

Key 先计算 Slot，再访问负责该 Slot 的 Master。

---

## Cluster 请求路由

不要简单理解为：

```text
请求发给任意节点
节点自动透明转发
```

更准确：

```text
Client
 ↓
访问某个节点
 ↓
节点发现 Slot 不属于自己
 ↓
返回重定向信息
 ↓
Client 再访问正确节点
```

Cluster-aware 客户端通常维护：

```text
Slot → Node
```

映射。

---

## MOVED

MOVED：

```text
永久重定向
```

表示：

```text
这个 Slot
现在归另一个节点负责
```

客户端通常会更新 Slot 映射。

---

## ASK

ASK：

```text
临时重定向
```

常见于：

```text
Slot 正在迁移
```

客户端临时访问目标节点，但通常不会立即永久修改 Slot 映射。

---

## Hash Tag

如果 Key 中包含：

```text
{...}
```

Redis Cluster 通常只使用大括号中的内容计算 Slot。

例如：

```text
user:{100}:name
user:{100}:age
```

都使用：

```text
100
```

计算 Slot。

因此：

```text
两个 Key
进入同一个 Slot
```

---

## Cluster 故障恢复

Cluster 中 Master 可以配置 Replica。

如果某 Master 故障：

```text
集群节点判断失败
   ↓
对应 Replica
可能被提升为新 Master
```

从而继续负责原 Slot。

---

# 内存管理

Redis 内存管理主要关注：

```text
Key 什么时候过期
内存满了以后淘汰谁
```

注意：

```text
过期删除
→ TTL 到期

内存淘汰
→ maxmemory 不够
```

---

## Key 过期

Redis 可以给 Key 设置：

```text
TTL
```

例如：

```text
SET key value EX 60
```

表示：

```text
60 秒后过期
```

---

## 过期删除

Redis 主要采用：

```text
惰性删除
+
定期删除
```

---

## 惰性删除

访问某 Key 时：

```text
访问 Key
   ↓
检查是否过期
   ↓
过期
→ 删除
```

优点：

```text
CPU 开销低
```

缺点：

```text
过期 Key 若长期没人访问
可能继续占内存
```

---

## 定期删除

Redis 会周期性：

```text
抽样检查
设置了 TTL 的 Key
```

发现过期：

```text
删除
```

核心：

```text
不是每次扫描所有 Key
```

而是：

```text
抽样
+
控制执行时间
```

用于平衡：

```text
CPU 开销
+
内存回收
```

SLOW / FAST 等具体频率与时间预算属于版本相关实现细节，不建议死记固定数字。

---

## 内存淘汰

当 Redis 内存达到：

```text
maxmemory
```

新的写入可能触发：

```text
Eviction
```

---

## noeviction

```text
不淘汰任何 Key
```

当内存达到上限，需要新增内存的写命令可能报错。

---

## volatile 策略

只从：

```text
设置了过期时间的 Key
```

中选择。

常见：

```text
volatile-lru
volatile-lfu
volatile-random
volatile-ttl
```

---

## allkeys 策略

从：

```text
所有 Key
```

中选择。

常见：

```text
allkeys-lru
allkeys-lfu
allkeys-random
```

---

## LRU

LRU：

```text
Least Recently Used
最近最少使用
```

优先淘汰最近很久没有访问的 Key。

Redis 使用：

```text
近似 LRU
```

而不是维护严格全局 LRU。

---

## LFU

LFU：

```text
Least Frequently Used
最不经常使用
```

优先淘汰：

```text
访问频率较低的 Key
```

---

## volatile-ttl

核心：

```text
只从有 TTL 的 Key 中
优先考虑剩余 TTL 更短的 Key
```

即：

```text
越接近过期
越优先淘汰
```

---

## 过期删除 vs 内存淘汰

```text
过期删除
→ TTL 到期

内存淘汰
→ maxmemory 不够
```

两者可以同时存在。

---

# 线程模型与网络 IO

## Redis 为什么快

常见原因：

```text
内存读写
高效数据结构
事件驱动
I/O 多路复用
核心命令串行执行减少锁竞争
```

---

## Redis 是单线程吗

不能简单说：

```text
整个 Redis
只有一个线程
```

更准确：

> **Redis 的核心命令执行长期主要由主线程串行执行，但持久化、异步释放等还会使用子进程或后台线程；Redis 6+ 还可启用 I/O Threads 辅助网络读写。**

可以记：

```text
核心命令执行
→ 主要单线程

网络 I/O
→ Redis 6+ 可使用 I/O Threads

RDB
→ fork 子进程

AOF / lazyfree 等
→ 可能使用后台线程
```

所以：

```text
“Redis 单线程”
```

通常是指：

```text
核心命令执行路径
```

---

## Event Loop

Redis 网络模型可粗略理解：

```text
多个客户端 Socket
        ↓
I/O 多路复用
        ↓
Event Loop
        ↓
识别就绪事件
        ↓
读取请求
        ↓
执行命令
        ↓
返回结果
```

---

## I/O 多路复用

I/O Multiplexing：

```text
一个线程
同时监控多个 fd
```

重点：

> **不是一个线程同时真正执行多个 I/O，而是一个线程高效监控多个 I/O 事件，哪个就绪就处理哪个。**

流程：

```text
fd1
fd2
fd3
fd4
 │
 └──────┐
        ↓
select / poll / epoll
        ↓
返回就绪 fd
        ↓
read / write
```

select、poll、epoll 都属于：

```text
I/O 就绪通知机制
```

真正的：

```text
read / write
```

仍需要程序执行。

---

## select

select：

```text
使用 fd_set 位图
保存 fd 集合
```

每次调用：

```text
用户态 fd 集合
   ↓
复制到内核
   ↓
内核遍历
   ↓
判断哪些 fd 就绪
```

问题：

```text
fd_set 大小有限
每次需要复制
每次需要遍历全部 fd
```

复杂度：

```text
O(n)
```

---

## poll

poll：

```text
使用 pollfd 数组
```

相比 select：

```text
不受固定 fd_set 大小限制
```

但：

```text
每次仍需要遍历全部 fd
```

复杂度：

```text
O(n)
```

---

## epoll

epoll：

```text
Linux 高并发场景
常用 I/O 多路复用机制
```

核心：

```text
注册关注的 fd
   ↓
内核维护监听集合
   ↓
fd 就绪后
放入就绪集合
   ↓
epoll_wait
直接获取就绪事件
```

### epoll_create

创建 epoll 实例。

### epoll_ctl

用于：

```text
添加 fd
修改 fd
删除 fd
```

### epoll_wait

用于：

```text
等待已经就绪的事件
```

---

## epoll 内部理解

常见简化理解：

```text
红黑树
→ 管理关注的 fd

就绪链表
→ 保存已经就绪的 fd
```

重点：

```text
select / poll
→ 每次问“所有 fd 谁好了？”

epoll
→ 内核维护状态
  直接告诉你“这些 fd 好了”
```

相比 select / poll：

```text
无需每次复制完整 fd 集合
无需每次线性检查所有监听 fd
只返回活跃事件
```

不要简单死记：

```text
epoll = O(1)
```

---

## LT

LT：

```text
Level Triggered
水平触发
```

特点：

```text
只要 fd
仍然可读 / 可写
就可能继续通知
```

优点：

```text
实现简单
容错高
```

---

## ET

ET：

```text
Edge Triggered
边缘触发
```

通常在：

```text
状态发生变化
```

时通知。

例如：

```text
未就绪
→ 就绪
```

一般需要：

```text
非阻塞 I/O
+
循环读 / 写直到 EAGAIN
```

不能简单理解为：

```text
ET 一定优于 LT
```

---

## select / poll / epoll 对比

| 对比 | select | poll | epoll |
| --- | --- | --- | --- |
| fd 管理 | fd_set 位图 | pollfd 数组 | 内核维护监听集合 |
| fd 固定上限 | 常见有 fd_set 限制 | 无固定 fd_set 上限 | 受系统资源限制 |
| 每次遍历全部 fd | 是 | 是 | 不需要同样方式遍历全部监听 fd |
| 就绪集合 | 每次重新检查 | 每次重新检查 | 内核维护 |
| 适合 | 兼容性、小规模 | 中小规模 | Linux 高并发 |

一句话：

```text
select / poll
→ 扫全部

epoll
→ 看就绪
```

---

# 缓存常见问题

## 缓存穿透

缓存穿透：

```text
客户端请求的数据
Redis 中不存在
数据库中也不存在
```

于是：

```text
每次请求
都绕过缓存
直接访问数据库
```

---

## 缓存穿透解决

### 参数校验

先过滤明显非法请求：

```text
非法 ID
非法参数
```

### 缓存空对象

数据库查询不到：

```text
Key
→ 缓存一个空值
```

优点：

```text
实现简单
```

缺点：

```text
额外内存
可能短期不一致
```

通常空值 TTL 设置较短。

### Bloom Filter

布隆过滤器：

```text
Bit Array
+
多个 Hash Function
```

查询：

```text
多个位置中
有任何一个为 0
→ 一定不存在

所有位置都为 1
→ 可能存在
```

特点：

```text
有假阳性
无假阴性
```

简单记：

```text
说“不存在”
→ 基本可信

说“存在”
→ 还要继续查
```

---

## 缓存击穿

缓存击穿：

```text
一个热点 Key
被高并发访问
```

突然失效：

```text
大量请求
瞬间访问数据库
```

核心：

```text
一个热点 Key
```

### 互斥锁

```text
缓存失效
   ↓
只有一个线程获得锁
   ↓
查询数据库
   ↓
重建缓存
   ↓
其他线程等待
```

优点：

```text
一致性较好
实现直接
```

缺点：

```text
串行
性能下降
需要防止死锁
```

### 逻辑过期

缓存数据中额外保存：

```text
逻辑过期时间
```

即使逻辑上过期：

```text
旧数据仍可先返回
```

然后后台异步重建缓存。

优点：

```text
热点请求不阻塞
性能好
```

缺点：

```text
可能短时间读到旧数据
实现更复杂
```

### 热点 Key 后台刷新

对于重要热点 Key，可以：

```text
后台主动刷新
```

避免瞬时失效。

---

## 缓存雪崩

缓存雪崩：

```text
大量 Key
同一时间失效
```

或者：

```text
Redis 整体不可用
```

导致：

```text
大量请求
同时打到数据库
```

核心：

```text
大量缓存一起出问题
```

### TTL 随机化

```text
基础 TTL
+
随机值
```

打散过期时间。

### Redis 高可用

通过：

```text
主从
Sentinel
Cluster
```

降低 Redis 整体不可用风险。

### 限流 / 降级 / 熔断

缓存异常时：

```text
限制请求量
关闭非核心功能
快速失败
```

避免数据库被压垮。

### 多级缓存

例如：

```text
本地缓存
   ↓
Redis
   ↓
数据库
```

### 热点预热 / 后台刷新

提前加载热点 Key，或后台异步刷新。

---

## 穿透 / 击穿 / 雪崩对比

| 问题 | 核心原因 | 关键词 |
| --- | --- | --- |
| 缓存穿透 | 数据根本不存在 | 查不到 |
| 缓存击穿 | 单个热点 Key 失效 | 一个热点没了 |
| 缓存雪崩 | 大量 Key 同时失效 / Redis 故障 | 一大片没了 |

一句话：

```text
穿透
→ 查不到

击穿
→ 热点没了

雪崩
→ 一大片没了
```

---

# 速记

## 持久化

```text
RDB
→ 快照
→ 恢复快
→ 两次快照间可能丢数据

AOF
→ 写命令日志
→ 数据更完整
→ 文件更大
```

COW：

```text
fork 后先共享
写时才复制
```

AOF：

```text
always
→ 最安全

everysec
→ 常用折中

no
→ 交给 OS
```

---

## 主从复制

```text
第一次
→ 全量同步

断线重连
→ 尝试增量同步
```

关键：

```text
replid
→ 复制历史身份

offset
→ 同步进度

repl_backlog
→ 最近复制命令的环形缓冲区
```

---

## Sentinel

```text
SDOWN
→ 单个 Sentinel 认为下线

ODOWN
→ 达到 quorum
  多个 Sentinel 认为下线
```

故障转移：

```text
Master Down
   ↓
SDOWN
   ↓
ODOWN
   ↓
Leader
   ↓
选 Replica
   ↓
新 Master
```

---

## Cluster

```text
Key
 ↓
CRC16
 ↓
% 16384
 ↓
Slot
 ↓
Master
```

```text
MOVED
→ 永久重定向

ASK
→ 临时重定向
```

---

## 内存管理

```text
过期删除
→ TTL 到期

内存淘汰
→ maxmemory 不够
```

过期：

```text
惰性删除
+
定期删除
```

淘汰：

```text
noeviction

volatile-lru
volatile-lfu
volatile-random
volatile-ttl

allkeys-lru
allkeys-lfu
allkeys-random
```

---

## 线程模型

```text
核心命令
→ 主要主线程串行执行

Redis 6+
→ 可使用 I/O Threads 辅助网络读写
```

I/O 多路复用：

```text
一个线程
监控多个 fd
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
→ 注册 fd
→ 内核维护就绪集合
→ 返回活跃 fd
```

---

## 缓存问题

```text
穿透
→ Redis 没有
→ DB 也没有

击穿
→ 一个热点 Key 失效

雪崩
→ 大量 Key 同时失效
  或 Redis 整体故障
```

解决：

```text
穿透
→ 参数校验 / 空值 / Bloom Filter

击穿
→ 互斥锁 / 逻辑过期 / 后台刷新

雪崩
→ TTL 随机化 / 高可用 / 限流 / 多级缓存
```

---

# 一句话总结

```text
RDB / AOF
→ 数据怎么不丢

Replication
→ 数据怎么复制

Sentinel
→ Master 挂了怎么办

Cluster
→ 数据太多一台机器放不下怎么办

Expiration / Eviction
→ 内存怎么管理

I/O Multiplexing
→ Redis 为什么能处理大量连接

穿透 / 击穿 / 雪崩
→ 缓存异常时怎么保护数据库
```
