# Redis 底层实现速记

> 本篇只关注 Redis 的内部实现：

```text
底层数据结构
   ↓
RedisObject 对象封装
   ↓
String / List / Set / Hash / ZSet
   ↓
不同数据类型选择不同编码
   ↓
Redis 版本演进
```

核心关系：

```text
SDS
→ String

IntSet
→ 小型整数 Set

Dict
→ Hash Table

ListPack
→ 紧凑连续存储

QuickList
→ List

SkipList + Dict
→ 大型 ZSet
```

---

# Redis 底层数据结构

## SDS

SDS：

```text
Simple Dynamic String
简单动态字符串
```

Redis 没有直接使用 C 字符串作为主要字符串结构，而是封装成 SDS。

---

### 内部结构

SDS 核心可以理解为：

```text
SDS
├── len
│   → 当前已使用字节数
├── alloc
│   → 已分配容量
├── flags
│   → SDS Header 类型
└── buf[]
    → 实际字节数组
```

不同长度字符串会使用不同 Header：

```text
sdshdr8
sdshdr16
sdshdr32
sdshdr64
```

作用：

```text
字符串越短
→ Header 使用越小的字段
→ 节省内存
```

---

### buf

实际数据存储在：

```text
buf[]
```

尾部仍然保留：

```text
'\0'
```

因此 SDS：

```text
兼容部分 C 字符串函数
```

但 SDS 自己通过：

```text
len
```

记录真实长度，不需要每次遍历到 `\0`。

---

### SDS vs C String

C String 获取长度：

```text
遍历字符串
直到 '\0'
→ O(n)
```

SDS：

```text
直接读取 len
→ O(1)
```

SDS 还支持：

```text
二进制安全
动态扩容
减少频繁内存申请
```

---

### 扩容机制

假设：

```text
required = len + addlen
```

扩容采用内存预分配思想。

如果扩容后的数据量较小：

```text
required < 1MB
→ 预分配容量约为 required × 2
```

如果数据已经较大：

```text
required >= 1MB
→ 预分配约 required + 1MB
```

另外还需要：

```text
额外空间保存 '\0'
```

不要简单记成：

```text
alloc = 2 × len + 1
```

更准确是：

```text
alloc
→ 记录可用数据容量

真实申请空间
→ Header + alloc + '\0'
```

---

### SDS 特点

```text
长度获取 O(1)
二进制安全
减少缓冲区溢出风险
支持动态扩容
空间预分配减少 realloc
```

一句话：

> **SDS = 带长度和容量信息的动态字节数组。**

---

## IntSet

IntSet：

```text
Integer Set
整数集合
```

适合保存：

```text
少量整数
```

---

### 内部结构

```text
IntSet
├── encoding
│   → 当前整数编码方式
├── length
│   → 元素数量
└── contents[]
    → 整数数组
```

---

### encoding

根据整数范围选择：

```text
int16
int32
int64
```

目标：

```text
在能够表示数据的前提下
尽量使用更小整数类型
```

从而：

```text
节省内存
```

---

### 元素存储

IntSet 中：

```text
元素唯一
+
按升序保存
```

因此查询可以使用：

```text
二分查找
```

注意：

```text
升序保存
→ 是存储方式

二分查找
→ 是查询 / 定位方式
```

---

### 编码升级

如果新加入整数：

```text
超出当前 encoding 范围
```

则：

```text
升级 encoding
```

流程：

```text
加入更大整数
   ↓
判断当前 encoding 不够
   ↓
升级到更大的整数类型
   ↓
扩大数组空间
   ↓
从后向前搬迁旧元素
   ↓
插入新元素
   ↓
更新 encoding / length
```

为什么从后向前复制：

```text
避免前面的数据
被扩容后的写入覆盖
```

---

### 特点

```text
整数唯一
升序保存
支持二分查找
支持类型升级
节省小整数集合内存
```

通常：

```text
只升级
不主动降级
```

一句话：

> **IntSet = 有序整数数组 + 自动编码升级。**

---

## Dict

Dict：

```text
Redis 内部哈希表
```

常用于：

```text
数据库 Key Space
Hash
Set
ZSet 辅助索引
```

---

### 整体结构

可以理解为三层：

```text
dict
 ↓
dictht
 ↓
dictEntry
```

---

### dict

核心负责：

```text
类型信息
私有数据
两张 Hash Table
Rehash 状态
```

可以抽象成：

```text
dict
├── type
├── privdata
├── ht[0]
├── ht[1]
├── rehashidx
└── pause / rehash 状态
```

其中：

```text
ht[0]
→ 正常使用的哈希表

ht[1]
→ Rehash 时的新哈希表
```

---

### dictht

哈希表本体：

```text
dictht
├── table
│   → bucket 数组
├── size
├── sizemask
└── used
```

其中：

```text
sizemask
≈ size - 1
```

常用于：

```text
hash & sizemask
```

快速定位 Bucket。

---

### dictEntry

每个 Entry 保存：

```text
dictEntry
├── key
├── value
└── next
```

如果多个 Key Hash 到同一个 Bucket：

```text
bucket
  ↓
entry → entry → entry
```

也就是：

```text
链地址法
```

解决 Hash 冲突。

---

### Load Factor

负载因子：

```text
LoadFactor = used / size
```

负载因子越大：

```text
Hash 冲突概率越高
```

当哈希表过满：

```text
需要扩容
```

具体扩容阈值：

```text
与 Redis 版本 / 运行状态有关
```

不要把某一组：

```text
1
5
```

当作永久固定规则。

---

### 扩容

扩容通常会：

```text
申请新的 ht[1]
```

容量：

```text
通常取合适的 2 的幂
```

然后：

```text
把 ht[0]
逐步迁移到 ht[1]
```

不是一次性全部复制。

---

### 渐进式 Rehash

Redis 使用：

```text
Progressive Rehash
渐进式 Rehash
```

流程：

```text
ht[0]
 ↓
创建更大 ht[1]
 ↓
rehashidx = 0
 ↓
每次迁移少量 Bucket
 ↓
rehashidx++
 ↓
全部完成
 ↓
ht[1] 替换 ht[0]
 ↓
rehashidx = -1
```

---

### Rehash 期间读写

查询：

```text
先查 ht[0]
必要时再查 ht[1]
```

新增：

```text
通常优先写入 ht[1]
```

这样避免：

```text
新数据继续进入旧表
增加迁移工作量
```

---

### 为什么渐进式 Rehash

如果一次迁移全部数据：

```text
数据量大
→ 主线程长时间阻塞
```

渐进式：

```text
一次迁移一点
分摊到多次操作中
```

避免明显卡顿。

一句话：

> **Dict = 哈希表 + 链表冲突解决 + 双表渐进式 Rehash。**

---

## ZipList

ZipList：

```text
压缩列表
```

属于：

```text
历史紧凑结构
```

主要用于理解：

```text
Redis 早期内存优化
ListPack 为什么出现
连锁更新问题
```

---

### 整体结构

```text
ZipList
├── zlbytes
│   → 整个 ZipList 字节数
├── zltail
│   → 尾节点偏移量
├── zllen
│   → Entry 数量
├── entry...
└── zlend
    → 0xFF
```

它是一段：

```text
连续内存
```

---

### ZipListEntry

Entry：

```text
ZipListEntry
├── previous_entry_length
├── encoding
└── content
```

---

### previous_entry_length

记录：

```text
前一个 Entry 的总长度
```

如果前一个 Entry 较短：

```text
1 Byte
```

如果前一个 Entry 较长：

```text
5 Byte
```

大致：

```text
长度 < 254
→ 1 Byte

长度 >= 254
→ 5 Byte
```

---

### encoding

用于描述：

```text
当前 Entry
存的是字符串还是整数
+
数据长度
```

字符串：

```text
不同前缀
表示不同长度编码方式
```

整数：

```text
使用特定编码表示整数类型
```

---

### content

实际保存：

```text
字符串
或
整数
```

---

### 连锁更新

ZipList 的经典问题：

```text
前一个 Entry 变大
   ↓
下一个 Entry 的 prevlen
1B → 5B
   ↓
下一个 Entry 自己也变大
   ↓
继续影响再下一个 Entry
   ↓
Cascade Update
```

新增和删除：

```text
都有可能触发
```

---

### 特点

```text
连续内存
内存利用率高
无需普通链表指针
支持前后遍历
```

缺点：

```text
数据多时查找慢
修改可能 realloc
大数据修改可能内存拷贝
可能发生连锁更新
```

一句话：

> **ZipList 用连续内存换空间效率，但修改成本和连锁更新问题明显。**

---

## ListPack

ListPack：

```text
用于替代 ZipList 的紧凑结构
```

核心目标：

```text
继续保持连续内存
+
解决 ZipList 连锁更新问题
```

---

### 整体结构

```text
ListPack
├── total-bytes
├── num-elements
├── entry...
└── 0xFF
```

Header：

```text
total-bytes
→ 整体字节数

num-elements
→ 元素数量
```

如果元素数量超过可直接表示范围：

```text
需要遍历获取精确数量
```

---

### Entry 结构

可以理解为：

```text
Entry
├── encoding
├── data
└── backlen
```

---

### encoding

记录：

```text
数据类型
+
数据长度
```

支持：

```text
整数
字符串
```

并使用：

```text
变长编码
```

节省空间。

---

### data

实际数据。

---

### backlen

记录：

```text
当前 Entry 自己
前面 encoding + data
占用了多少字节
```

作用：

```text
支持反向遍历
```

---

### ZipList vs ListPack

ZipList：

```text
[prevlen][encoding][data]
     ↑
依赖前一个节点
```

ListPack：

```text
[encoding][data][backlen]
                    ↑
                描述自己
```

关键区别：

```text
ZipList
→ 后一个 Entry
  依赖前一个 Entry 长度

ListPack
→ Entry 记录自身长度信息
```

因此：

```text
修改某个 Entry
不会导致后续 Entry
不断扩大 prevlen
```

从而：

```text
避免 ZipList 典型连锁更新问题
```

一句话：

> **ListPack = 不记录前节点长度的紧凑连续结构。**

---

## QuickList

QuickList：

```text
双向链表
+
多个紧凑列表节点
```

它是：

```text
普通链表
和
超大连续内存结构
之间的折中
```

---

### 为什么需要 QuickList

普通 LinkedList：

```text
每个节点
都要保存前后指针
→ 内存开销大
```

单个超大 ZipList / ListPack：

```text
连续内存过大
→ realloc / 内存拷贝成本高
```

QuickList：

```text
双向链表
   ↓
每个节点保存一小块
ZipList / ListPack
```

于是兼顾：

```text
扩展灵活
+
内存紧凑
```

---

### 内部结构

整体：

```text
quicklist
├── head
├── tail
├── count
├── len
├── fill
├── compress
└── 其他控制字段
```

---

### QuickListNode

节点：

```text
quicklistNode
├── prev
├── next
├── entry
└── size / count / encoding 等
```

其中：

```text
entry
→ 指向紧凑列表
```

历史上：

```text
QuickList Node
→ ZipList
```

新版本主线：

```text
QuickList Node
→ ListPack
```

---

### 中间节点压缩

QuickList 可以让：

```text
靠近两端的节点
保持未压缩
```

而中间部分：

```text
进行压缩
```

进一步降低内存使用。

---

### 特点

```text
双向链表
节点内部使用紧凑结构
减少指针开销
避免单块连续内存过大
支持头尾高效操作
```

一句话：

> **QuickList = 双向链表负责扩展，ListPack 负责节省内存。**

---

## SkipList

SkipList：

```text
跳跃表
```

用于：

```text
有序数据
```

Redis 中典型应用：

```text
ZSet
```

---

### 整体结构

```text
zskiplist
├── header
├── tail
├── length
└── level
```

---

### Node

```text
zskiplistNode
├── ele
├── score
├── backward
└── level[]
    ├── forward
    └── span
```

---

### score

排序主要根据：

```text
score
```

规则：

```text
score 小
→ 靠前

score 大
→ 靠后
```

如果 score 相同：

```text
按 member / ele
字典序排序
```

---

### 多级索引

普通链表：

```text
A → B → C → D → E → F
```

SkipList 增加高层指针：

```text
Level 3: A -------- D -------- F
Level 2: A ---- C ---- E ---- F
Level 1: A → B → C → D → E → F
```

查找：

```text
先走高层
快速跳过大量节点
再逐层下降
```

---

### span

span：

```text
当前层指针
跨过多少个底层节点
```

用于：

```text
Rank
排名计算
```

---

### 时间复杂度

平均：

```text
查找
插入
删除
→ O(logN)
```

范围遍历：

```text
非常自然
```

---

### 特点

```text
结构简单
有序
支持范围查找
支持排名
平均 O(logN)
```

一句话：

> **SkipList = 多层有序链表，用空间换查找速度。**

---

# Redis 对象系统

## RedisObject

从用户角度：

```text
Redis
├── String
├── List
├── Set
├── Hash
└── ZSet
```

但不同类型底层结构不同。

Redis 需要一个统一对象：

```text
RedisObject
```

把：

```text
逻辑数据类型
+
底层编码
```

封装起来。

---

### Key Space

Redis Database 本质上可以理解为：

```text
Dict
```

映射：

```text
Key
→ Value
```

Key：

```text
通常使用 SDS
```

Value：

```text
通过 RedisObject 封装
```

---

### RedisObject 结构

核心：

```text
redisObject
├── type
├── encoding
├── lru / LFU 信息
├── refcount
└── ptr
```

---

### type

表示：

```text
用户看到的数据类型
```

例如：

```text
STRING
LIST
SET
HASH
ZSET
```

---

### encoding

表示：

```text
真正的底层实现方式
```

例如：

```text
STRING
→ INT / EMBSTR / RAW
```

所以：

```text
type
→ 逻辑类型

encoding
→ 物理实现
```

---

### lru / LFU

用于记录：

```text
对象访问相关信息
```

支持：

```text
LRU
LFU
内存淘汰
```

---

### refcount

引用计数：

```text
记录对象被引用次数
```

用于：

```text
对象共享
生命周期管理
```

---

### ptr

指向：

```text
真正的数据结构
```

例如：

```text
SDS
Dict
QuickList
SkipList
```

一句话：

> **RedisObject = Redis 数据类型和底层数据结构之间的适配层。**

---

## String

String 的编码主要：

```text
INT
EMBSTR
RAW
```

---

### INT

如果字符串内容：

```text
可以表示为整数
```

Redis 可以采用：

```text
INT encoding
```

特点：

```text
直接保存整数值
减少额外 SDS 分配
```

例如：

```text
"100"
```

可能内部直接作为整数保存。

---

### EMBSTR

EMBSTR：

```text
Embedded String
```

用于：

```text
短字符串
```

结构：

```text
[RedisObject][SDS Header][buf]
```

一块连续内存。

优点：

```text
一次内存分配
一次释放
缓存局部性好
```

经典实现中：

```text
短字符串阈值约 44 Bytes
```

但这种数值属于：

```text
具体版本实现细节
```

---

### RAW

RAW：

```text
RedisObject
   ↓ ptr
SDS
```

RedisObject 和 SDS：

```text
分别分配
```

适合：

```text
较长字符串
或需要修改的字符串
```

---

### 编码转换

大致：

```text
整数
→ INT

短字符串
→ EMBSTR

较长字符串
→ RAW
```

如果 EMBSTR 被修改：

```text
可能转换为 RAW
```

一般：

```text
RAW 不会因为字符串后来变短
自动转回 EMBSTR
```

---

## List

List 的底层实现经历过明显演进。

---

### 历史实现

早期：

```text
元素较少
→ ZipList

元素较多
→ LinkedList
```

问题：

```text
LinkedList
→ 指针开销大

ZipList
→ 连续内存过大时修改成本高
```

---

### QuickList

后来统一为：

```text
QuickList
```

即：

```text
双向链表
+
多个紧凑节点
```

历史节点：

```text
ZipList
```

新版本：

```text
ListPack
```

---

### 当前理解

```text
List
   ↓
QuickList
   ↓
ListPack Node
```

一句话：

> **List 用 QuickList 管理多个紧凑节点，在访问效率与内存之间折中。**

---

## Set

Set 要求：

```text
元素唯一
```

底层会根据数据情况选择不同编码。

---

### IntSet

如果：

```text
所有元素都是整数
+
元素数量较少
```

可以使用：

```text
IntSet
```

特点：

```text
连续数组
升序
节省内存
```

---

### ListPack

新版本中：

```text
小型 Set
```

也可能使用：

```text
ListPack
```

进行紧凑存储。

---

### Dict

一般 Set：

```text
Dict
```

存储方式：

```text
key
→ Set 元素

value
→ 不需要实际业务值
```

因此：

```text
Dict Key
天然保证元素唯一
```

---

### Set 编码关系

```text
全整数 + 小规模
→ IntSet

小型紧凑 Set
→ ListPack

一般 / 较大 Set
→ Dict
```

---

## Hash

Hash：

```text
field → value
```

要求：

```text
Field 唯一
+
根据 Field 快速查 Value
```

---

### 小 Hash

数据较少时：

```text
ListPack
```

紧凑保存。

逻辑上：

```text
field
value
field
value
...
```

相邻存储。

旧版本：

```text
ZipList
```

---

### 大 Hash

数据增多后：

```text
Dict
```

因为：

```text
Hash 查找效率高
修改更灵活
```

---

### 为什么不用超大 ListPack

连续内存结构：

```text
数据过多
→ 查找需要遍历
→ 修改可能产生较大内存移动
```

因此大 Hash 转 Dict。

---

### Hash 编码关系

```text
小 Hash
→ ListPack

大 Hash
→ Dict
```

具体转换阈值：

```text
由 Redis 配置决定
```

不要死记历史版本固定数字。

---

## ZSet

ZSet：

```text
Sorted Set
```

要求：

```text
member 唯一
+
有 score
+
按 score 排序
+
支持范围查询
```

---

### 小 ZSet

元素较少时：

```text
ListPack
```

旧版本：

```text
ZipList
```

逻辑存储：

```text
member
score
member
score
...
```

并保持：

```text
score 有序
```

---

### 大 ZSet

普通 ZSet 核心：

```text
Dict
+
SkipList
```

为什么两个都要：

```text
Dict
→ member → score
→ 快速查某个 member

SkipList
→ 按 score 排序
→ 范围查询 / Rank
```

所以：

```text
Dict
解决“快速定位”

SkipList
解决“排序和范围”
```

---

### ZSet 编码关系

```text
小 ZSet
→ ListPack

大 ZSet
→ Dict + SkipList
```

---

# 编码与版本演进

## 为什么 Redis 有多种编码

Redis 不是：

```text
一个数据类型
永远一种底层结构
```

而是：

```text
根据数据规模
数据类型
元素大小
```

选择不同编码。

核心思想：

```text
数据少
→ 紧凑结构
→ 节省内存

数据多
→ 高效结构
→ 提高操作性能
```

---

## ZipList → ListPack

ZipList：

```text
Entry
记录前一个 Entry 长度
```

问题：

```text
前一个 Entry 变大
→ 后一个 prevlen 可能扩容
→ 继续影响后续 Entry
→ 连锁更新
```

ListPack：

```text
Entry
记录自身长度信息
```

于是：

```text
避免典型 Cascade Update
```

演进：

```text
ZipList
   ↓
ListPack
```

核心：

> **ListPack 保留紧凑内存优势，同时去掉对前一个 Entry 长度的依赖。**

---

## LinkedList / ZipList → QuickList

LinkedList：

```text
扩展灵活
但指针开销大
```

ZipList：

```text
节省内存
但单块连续内存过大时修改成本高
```

QuickList：

```text
LinkedList
+
多个 ZipList / ListPack
```

折中：

```text
扩展能力
+
内存利用率
```

---

## List 编码演进

可以记：

```text
早期
→ ZipList / LinkedList

Redis 3.2 以后
→ QuickList

Redis 7.x 主线
→ QuickList + ListPack
```

---

## Hash 编码演进

```text
旧版本小 Hash
→ ZipList

新版本小 Hash
→ ListPack

大 Hash
→ Dict
```

---

## Set 编码演进

```text
全整数小 Set
→ IntSet

新版本小型普通 Set
→ ListPack

一般 Set
→ Dict
```

---

## ZSet 编码演进

```text
旧版本小 ZSet
→ ZipList

新版本小 ZSet
→ ListPack

普通 / 大 ZSet
→ Dict + SkipList
```

---

## 编码关系总图

```text
RedisObject
│
├── String
│   ├── INT
│   ├── EMBSTR
│   └── RAW
│       └── SDS
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

---

# 速记

## SDS

```text
len
→ 已使用长度

alloc
→ 已分配容量

flags
→ Header 类型

buf[]
→ 实际数据
```

扩容：

```text
小于约 1MB
→ 预分配约 2 倍

较大
→ 额外预留约 1MB
```

---

## IntSet

```text
整数
+
唯一
+
升序
+
二分查找
```

编码：

```text
int16
→ int32
→ int64
```

只升级：

```text
不主动降级
```

---

## Dict

```text
dict
 ↓
ht[0] + ht[1]
 ↓
bucket[]
 ↓
dictEntry
```

Rehash：

```text
双表
+
渐进迁移
```

---

## ZipList

```text
连续内存
+
prevlen
+
encoding
+
content
```

问题：

```text
Cascade Update
连锁更新
```

---

## ListPack

```text
encoding
+
data
+
backlen
```

核心：

```text
记录自己长度
不依赖前节点长度
→ 避免 ZipList 连锁更新
```

---

## QuickList

```text
双向链表
+
多个 ListPack
```

核心：

```text
链表负责扩展
ListPack 负责省内存
```

---

## SkipList

```text
多层链表
+
score
+
forward
+
span
```

平均：

```text
查找 / 插入 / 删除
→ O(logN)
```

---

## RedisObject

```text
type
→ 用户看到的数据类型

encoding
→ 真正底层实现

ptr
→ 指向实际数据结构
```

---

## 五大类型

```text
String
→ INT / EMBSTR / RAW

List
→ QuickList + ListPack

Set
→ IntSet / ListPack / Dict

Hash
→ ListPack / Dict

ZSet
→ ListPack / Dict + SkipList
```

---

## 版本演进

```text
ZipList
→ 历史紧凑结构

ListPack
→ 新版紧凑结构

QuickList
→ List 的主线结构

Dict
→ Hash / Set / ZSet 辅助结构

SkipList
→ ZSet 排序核心
```

---

# 一句话总结

```text
SDS
→ 字符串怎么存

IntSet
→ 小整数集合怎么省内存

Dict
→ 哈希数据怎么存

ZipList / ListPack
→ 小数据怎么紧凑存储

QuickList
→ List 怎么兼顾内存与扩展

SkipList
→ ZSet 怎么排序

RedisObject
→ 怎么统一封装不同数据类型

Encoding
→ Redis 如何根据数据规模
  在“省内存”和“高性能”之间切换
```
