# Redis 基础知识结构速记

> 主线：**持久化 → 主从复制 → Sentinel → Cluster → 内存管理 → 线程模型 → 缓存问题**

# 1. 持久化

Redis 数据主要在内存中，持久化用于降低故障后的数据丢失风险。核心是 **RDB、AOF，以及两者结合的混合持久化**。

## RDB / AOF

| 对比 | RDB | AOF |
| --- | --- | --- |
| 核心 | 某一时刻的内存快照 | 记录写命令 |
| 写入方式 | 周期性生成快照 | 持续追加 |
| 文件 | 相对紧凑 | 通常更大 |
| 恢复速度 | 较快 | 通常较慢 |
| 数据完整性 | 两次快照间可能丢数据 | 通常更高 |

### RDB

常见触发：`SAVE`、`BGSAVE`、配置的 `save` 条件和部分关闭/运维操作。

```text
SAVE
→ 主线程生成 RDB
→ 阻塞 Redis

BGSAVE
→ fork 子进程
→ 子进程生成 RDB
→ 父进程继续处理请求
```

`BGSAVE` 依赖 fork + Copy-On-Write：

```text
fork
→ 父子进程共享原有物理页
→ 主进程只读：继续共享
→ 主进程修改某页：触发 COW，只复制相关页
```

> fork 时不是立即复制整个 Redis 内存，而是写时才复制被修改的内存页。

RDB：文件紧凑、恢复快、适合全量备份；缺点是快照之间存在数据丢失窗口，fork/COW/磁盘写入也有开销。

### AOF

AOF 记录 Redis 执行的写命令，如 `SET`、`DEL`、`HSET`、`LPUSH`，普通读命令不会用于恢复数据。

```text
客户端写命令
→ Redis 执行
→ AOF Buffer
→ 根据 appendfsync 策略写入 / 刷盘
```

| `appendfsync` | 行为 | 特点 |
| --- | --- | --- |
| `always` | 每次写命令都尽快 fsync | 数据安全性高，性能最低 |
| `everysec` | 通常每秒 fsync 一次 | 性能与安全折中，故障时可能损失最近约 1 秒数据 |
| `no` | Redis 不主动控制 fsync | 交给操作系统决定 |

AOF 文件不断增大后可以 Rewrite：

```text
旧 AOF 中大量历史命令
→ 根据当前数据集重新构造
→ 生成更紧凑的新 AOF
```

> AOF Rewrite 不是简单压缩旧文件，而是根据当前数据状态重新生成可恢复当前状态的 AOF。

### 混合持久化

可理解为：

```text
RDB
→ 快速恢复基础数据

AOF
→ 恢复后续增量修改
```

# 2. 主从复制与高可用

## 主从复制

```text
Master
→ 负责写入

Replica
→ 复制 Master 数据
```

主要作用：扩展读能力、保存数据副本，并为高可用提供基础。

### 全量 / 增量同步

第一次同步或无法增量同步时：

```text
Replica 连接 Master
→ PSYNC
→ 无法增量
→ FULLRESYNC
→ Master 生成并发送 RDB
→ Replica 加载 RDB
→ 再补充同步期间的新写命令
```

复制的三个关键概念：

| 概念 | 作用 |
| --- | --- |
| `replid` | 标识一段复制历史 / 数据来源 |
| `offset` | 表示复制进度 |
| `repl_backlog` | 固定大小的环形复制缓冲区，保存最近一段复制数据和 offset |

短暂断线重连：

```text
Replica
→ 发送 replid + offset
→ Master 判断缺失内容是否仍在 repl_backlog
   ├─ 在：只发送缺失部分 → Partial Resynchronization
   └─ 不在：重新全量同步
```

## Sentinel

Sentinel 主要负责监控、故障判断、自动故障转移和服务发现/通知。

| 概念 | 含义 |
| --- | --- |
| SDOWN | 单个 Sentinel 主观认为实例下线 |
| ODOWN | 多个 Sentinel 的判断达到 `quorum`，Master 被客观判定下线 |
| `quorum` | 判断 ODOWN 的阈值；Leader 选举/故障转移授权还涉及多数派 |

故障转移：

```text
Master 无响应
→ SDOWN
→ 达到 quorum，ODOWN
→ 选举 Sentinel Leader
→ Leader 选择 Replica
→ 提升为新 Master
→ 其他 Replica 指向新 Master
→ 旧 Master 恢复后成为 Replica
```

# 3. Redis Cluster

Redis Cluster 通过多个 Master 分担 Key，解决单机容量和吞吐瓶颈；每个 Master 还可配置 Replica 做故障恢复。

## Hash Slot 与请求路由

Redis Cluster 有 **16384 个 Slot**：

```text
Key
→ CRC16
→ % 16384
→ Slot
→ 对应 Master
```

如果 Key 包含 `{...}`，通常使用大括号中的内容计算 Slot：

```text
user:{100}:name
user:{100}:age
→ 都使用 100 计算 Slot
→ 落到同一 Slot
```

Cluster 请求不是由节点透明转发，而是客户端根据重定向访问正确节点：

| 响应 | 含义 |
| --- | --- |
| `MOVED` | 永久重定向，Slot 已归其他节点，客户端通常更新 Slot → Node 映射 |
| `ASK` | 临时重定向，常见于 Slot 迁移过程中，不代表永久归属改变 |

Cluster-aware 客户端通常维护：

```text
Slot → Node
```

Master 故障时，对应 Replica 可能被提升为新 Master，继续负责原 Slot。

# 4. 内存管理

需要区分：

```text
过期删除
→ Key 的 TTL 到期

内存淘汰
→ Redis 达到 maxmemory，需要释放空间
```

## 过期删除

Redis 主要结合：

```text
惰性删除 + 定期删除
```

| 机制 | 行为 | 特点 |
| --- | --- | --- |
| 惰性删除 | 访问 Key 时检查，已过期则删除 | CPU 开销低，但长期不访问的过期 Key 可能继续占内存 |
| 定期删除 | 周期性抽样检查带 TTL 的 Key | 在 CPU 开销和内存回收间折中，不会每次全量扫描 |

## 内存淘汰

| 策略 | 范围 / 行为 |
| --- | --- |
| `noeviction` | 不淘汰，内存不足时需要新增内存的写命令可能报错 |
| `volatile-lru` | 从有 TTL 的 Key 中按近似 LRU 淘汰 |
| `volatile-lfu` | 从有 TTL 的 Key 中按 LFU 淘汰 |
| `volatile-random` | 从有 TTL 的 Key 中随机淘汰 |
| `volatile-ttl` | 从有 TTL 的 Key 中优先淘汰剩余 TTL 较短的 Key |
| `allkeys-lru` | 从所有 Key 中按近似 LRU 淘汰 |
| `allkeys-lfu` | 从所有 Key 中按 LFU 淘汰 |
| `allkeys-random` | 从所有 Key 中随机淘汰 |

- **LRU**：Least Recently Used，最近最少使用；Redis 使用近似 LRU。
- **LFU**：Least Frequently Used，优先淘汰访问频率较低的 Key。

# 5. 线程模型与网络 I/O

Redis 快的常见原因：内存读写、高效数据结构、事件驱动、I/O 多路复用，以及核心命令串行执行减少锁竞争。

> “Redis 单线程”通常指核心命令执行路径主要由主线程串行执行；持久化、异步释放等还会使用子进程或后台线程，Redis 6+ 还可启用 I/O Threads 辅助网络读写。

```text
多个客户端 Socket
→ I/O 多路复用
→ Event Loop
→ 获取就绪事件
→ 读取请求
→ 执行命令
→ 返回结果
```

I/O 多路复用的重点：

> 一个线程高效监控多个 fd，哪个 fd 就绪就处理哪个；并不是一个线程同时真正执行多个 I/O。

## select / poll / epoll

| 对比 | select | poll | epoll |
| --- | --- | --- | --- |
| fd 管理 | `fd_set` 位图 | `pollfd[]` | 内核维护监听集合 |
| 固定 fd_set 上限 | 常见存在 | 无固定 fd_set 限制 | 主要受系统资源限制 |
| 每次线性检查全部监听 fd | 是 | 是 | 不按 select/poll 的方式扫描全部监听 fd |
| 就绪事件 | 每次重新检查 | 每次重新检查 | 内核维护就绪事件 |
| 常见场景 | 兼容性、小规模 | 中小规模 | Linux 高并发 |

epoll：

```text
epoll_create
→ 创建实例

epoll_ctl
→ 添加 / 修改 / 删除关注 fd

epoll_wait
→ 获取已经就绪的事件
```

简化理解：

```text
监听集合
→ 管理关注的 fd

就绪集合
→ 保存已就绪 fd
```

不要死记 `epoll = O(1)`，它的优势主要在于避免每次复制和线性扫描完整监听集合，只返回活跃事件。

### LT / ET

| 模式 | 特点 |
| --- | --- |
| LT（水平触发） | 只要 fd 仍可读/可写，就可能继续通知；实现简单 |
| ET（边缘触发） | 状态发生变化时通知；通常配合非阻塞 I/O，并循环处理到 `EAGAIN` |

ET 不一定优于 LT，应根据程序模型选择。

# 6. 缓存常见问题

| 问题 | 核心原因 | 常见解决 |
| --- | --- | --- |
| 缓存穿透 | Redis 和 DB 中都不存在该数据 | 参数校验、缓存空对象、Bloom Filter |
| 缓存击穿 | 单个热点 Key 突然失效，大量请求打到 DB | 互斥锁、逻辑过期、后台刷新 |
| 缓存雪崩 | 大量 Key 同时失效或 Redis 整体不可用 | TTL 随机化、高可用、限流/降级、多级缓存、预热 |

### Bloom Filter

```text
多个 Hash Function
→ 映射到 Bit Array

有任意一位为 0
→ 一定不存在

所有位都为 1
→ 可能存在
```

特点：有假阳性，无假阴性。

### 击穿：互斥锁 / 逻辑过期

```text
互斥锁
→ 只有一个线程重建缓存
→ 其他线程等待
→ 一致性较好，但有等待成本

逻辑过期
→ 旧数据仍可返回
→ 后台异步重建
→ 性能好，但可能短时间读旧数据
```

# 7. 速记

```text
持久化
RDB → 快照，恢复快
AOF → 写命令日志，数据更完整
COW → fork 后先共享，写时才复制
```

```text
复制
replid       → 复制历史身份
offset       → 复制进度
repl_backlog → 最近复制数据的环形缓冲区
```

```text
Sentinel
SDOWN → 单个 Sentinel 判断
ODOWN → 达到 quorum
```

```text
Cluster
Key → CRC16 → % 16384 → Slot → Master
MOVED → 永久重定向
ASK   → 临时重定向
```

```text
内存
过期删除 → TTL 到期
内存淘汰 → maxmemory 不够
```

```text
缓存问题
穿透 → 数据根本不存在
击穿 → 一个热点 Key 失效
雪崩 → 大量 Key / 整个 Redis 同时出问题
```
