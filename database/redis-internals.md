# Redis 底层实现速记

> 主线：**底层数据结构 → RedisObject → 五大类型编码 → 版本演进**

# 1. 底层数据结构

核心关系：

| 结构 | 主要用途 |
| --- | --- |
| SDS | String 的字符串存储 |
| IntSet | 小型整数 Set |
| Dict | Key Space、Hash、Set、ZSet 辅助索引 |
| ZipList | 历史紧凑结构 |
| ListPack | 新版紧凑连续结构 |
| QuickList | List 主线结构 |
| SkipList | ZSet 的有序索引 |

## SDS

SDS（Simple Dynamic String）可以理解为**带长度和容量信息的动态字节数组**：

```text
SDS
├── len    → 已使用字节数
├── alloc  → 已分配容量
├── flags  → Header 类型
└── buf[]  → 实际数据，末尾保留 '\0'
```

根据长度可使用不同 Header，如 `sdshdr8 / 16 / 32 / 64`，短字符串使用更小的头部以节省内存。

| 对比 | C String | SDS |
| --- | --- | --- |
| 获取长度 | 遍历到 `\0`，O(n) | 直接读取 `len`，O(1) |
| 二进制安全 | 受 `\0` 限制 | 是 |
| 扩容 | 手动处理 | 支持动态扩容和空间预分配 |

经典实现的预分配思想：

```text
数据较小 → 扩容后预留较多额外空间
数据较大 → 追加固定量级的额外空间
```

具体阈值属于版本实现细节，不建议死记固定数字。

## IntSet

IntSet = **有序整数数组 + 自动编码升级**。

```text
IntSet
├── encoding → int16 / int32 / int64
├── length   → 元素数量
└── contents[]
```

特点：

- 元素唯一、升序保存，可以二分查找。
- 新整数超出当前编码范围时升级到更大整数类型。
- 升级时扩容并重新排列数据，通常只升级、不主动降级。

```text
int16 → int32 → int64
```

## Dict

Dict 是 Redis 内部哈希表，核心可以理解成：

```text
dict
├── ht[0]      → 当前哈希表
├── ht[1]      → Rehash 时的新表
└── rehashidx  → Rehash 进度

dictht
→ bucket[]

bucket
→ dictEntry → dictEntry → ...
```

Hash 冲突通过**链地址法**解决。

负载因子：

```text
LoadFactor = used / size
```

数据增多时 Redis 会建立 `ht[1]`，再进行渐进式 Rehash：

```text
ht[0]
→ 创建 ht[1]
→ 每次迁移少量 Bucket
→ 全部完成
→ ht[1] 成为新的 ht[0]
```

Rehash 期间：

- 查询需要同时考虑两张表。
- 新增数据通常进入 `ht[1]`。
- 渐进迁移把一次大操作拆到多次请求中，避免主线程长时间阻塞。

> **Dict = 哈希表 + 链地址法 + 双表渐进式 Rehash。**

## ZipList / ListPack

ZipList 是历史紧凑结构，ListPack 是其重要替代方案。

| 对比 | ZipList | ListPack |
| --- | --- | --- |
| 存储 | 连续内存 | 连续内存 |
| Entry | `prevlen + encoding + data` | `encoding + data + backlen` |
| 长度关系 | 后一个 Entry 依赖前一个 Entry 长度 | Entry 记录自身长度相关信息 |
| 主要问题 | 可能出现 Cascade Update | 避免 ZipList 典型连锁更新 |

ZipList：

```text
前一个 Entry 变大
→ 后一个 prevlen 可能从短编码变成长编码
→ 后一个 Entry 自身变大
→ 可能继续影响后续 Entry
→ Cascade Update
```

ListPack：

```text
[encoding][data][backlen]
```

`backlen` 用于支持反向遍历，并避免后一个 Entry 依赖前一个 Entry 的长度。

> **ListPack 保留紧凑连续内存优势，同时去掉 ZipList 的典型连锁更新问题。**

## QuickList

QuickList = **双向链表 + 多个紧凑列表节点**。

```text
QuickList
→ Node ⇄ Node ⇄ Node
     │      │      │
  ListPack ListPack ListPack
```

它在两种极端之间折中：

```text
普通链表
→ 扩展灵活，但每个节点指针开销大

单个超大连续结构
→ 省指针，但 realloc / 内存搬移成本高

QuickList
→ 链表负责扩展
→ ListPack 负责节省内存
```

中间节点还可以压缩，进一步降低内存占用。

## SkipList

SkipList 是多层有序链表，Redis 中典型用于 ZSet：

```text
Level 3: A -------- D -------- F
Level 2: A ---- C ---- E ---- F
Level 1: A → B → C → D → E → F
```

节点核心信息：

```text
ele
score
backward
level[]
  ├── forward
  └── span
```

- 按 `score` 排序；score 相同时按 member/ele 字典序。
- 高层指针用于快速跳过大量节点。
- `span` 用于排名计算。
- 查找、插入、删除平均 O(log N)，范围遍历自然。

> **SkipList = 多层有序链表，用额外索引换取快速查找和范围操作。**

# 2. RedisObject 与五大类型

Redis Database 的 Key Space 可以理解为一个 Dict：

```text
Key（通常 SDS）
→ RedisObject
→ 实际底层数据结构
```

`redisObject` 主要保存：

| 字段 | 作用 |
| --- | --- |
| `type` | 用户看到的逻辑类型：STRING / LIST / SET / HASH / ZSET |
| `encoding` | 真正的底层编码 |
| `lru / LFU` | 访问相关信息，用于淘汰 |
| `refcount` | 引用计数 |
| `ptr` | 指向实际数据结构 |

> **type = 逻辑类型；encoding = 物理实现。**

## 五大类型编码

| 类型 | 常见编码 / 结构 | 说明 |
| --- | --- | --- |
| String | `INT / EMBSTR / RAW` | 整数、短字符串、普通 SDS |
| List | `QuickList + ListPack` | 链表负责扩展，ListPack 节省内存 |
| Set | `IntSet / ListPack / Dict` | 根据是否全整数、规模等选择 |
| Hash | `ListPack / Dict` | 小数据紧凑存储，大数据用哈希表 |
| ZSet | `ListPack / Dict + SkipList` | 小数据紧凑存储，大数据兼顾定位与排序 |

### String

```text
整数值
→ INT

短字符串
→ EMBSTR
→ RedisObject + SDS 一块连续内存

较长 / 需要修改的字符串
→ RAW
→ RedisObject 和 SDS 分开分配
```

EMBSTR 的具体长度阈值属于版本实现细节，不建议死记固定数字。

### Set

```text
全整数 + 小规模
→ IntSet

小型普通 Set（新版本）
→ ListPack

一般 / 较大 Set
→ Dict
```

Dict 中通常把 Set 元素作为 key，天然保证唯一。

### Hash

```text
小 Hash
→ ListPack
→ field / value 紧凑排列

大 Hash
→ Dict
```

大 Hash 使用 Dict 是为了更高效的查找和修改；具体转换阈值由版本和配置决定。

### ZSet

```text
小 ZSet
→ ListPack

普通 / 大 ZSet
→ Dict + SkipList
```

两个结构作用不同：

```text
Dict
→ member → score
→ 快速定位 member

SkipList
→ 按 score 排序
→ 范围查询 / Rank
```

# 3. 编码与版本演进

Redis 不会让一个逻辑类型永久绑定一种底层结构，而是根据数据规模、元素类型和大小，在**省内存**与**操作效率**之间切换。

| 演进 | 含义 |
| --- | --- |
| ZipList → ListPack | 保留紧凑连续存储，避免典型连锁更新 |
| LinkedList / ZipList → QuickList | List 在扩展灵活性和内存利用率之间折中 |
| 小 Hash：ZipList → ListPack | 新版紧凑结构替代历史 ZipList |
| 小 ZSet：ZipList → ListPack | 同上 |
| List 主线 | QuickList，节点由历史 ZipList 演进为 ListPack |
| 大 Hash / Set | Dict |
| 大 ZSet | Dict + SkipList |

总图：

```text
RedisObject
│
├── String
│   ├── INT
│   ├── EMBSTR
│   └── RAW → SDS
│
├── List
│   └── QuickList
│       └── ListPack
│
├── Set
│   ├── IntSet
│   ├── ListPack
│   └── Dict
│
├── Hash
│   ├── ListPack
│   └── Dict
│
└── ZSet
    ├── ListPack
    └── Dict + SkipList
```

# 4. 速记

```text
SDS
→ len + alloc + flags + buf[]
→ O(1) 取长度，二进制安全，动态扩容
```

```text
IntSet
→ 有序整数数组
→ int16 → int32 → int64
→ 只升级，不主动降级
```

```text
Dict
→ ht[0] + ht[1]
→ 链地址法
→ 渐进式 Rehash
```

```text
ZipList
→ 历史紧凑结构
→ prevlen
→ 可能 Cascade Update

ListPack
→ encoding + data + backlen
→ 避免典型连锁更新
```

```text
QuickList
→ 双向链表 + 多个 ListPack

SkipList
→ 多层有序链表
→ 平均 O(log N)
```

```text
RedisObject
→ type：逻辑类型
→ encoding：底层实现
→ ptr：实际数据
```
