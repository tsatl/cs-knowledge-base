# MySQL 基础知识结构速记

> 主线：**SQL → MySQL 执行 → 事务 / MVCC → 索引 → 锁 → 日志 → 复制与分片**

# 1. SQL 与 MySQL 架构

## SQL 基础

### 关系数据库概念

| 概念 | 含义 |
| --- | --- |
| DBMS | 数据库管理系统，MySQL 是一种 DBMS |
| 元组 | 一行数据 |
| 候选码 | 能唯一标识元组且不存在多余属性的属性集合 |
| 主码 | 从候选码中选出的主要标识，即 Primary Key |
| 外码 | 一个表中的字段引用另一个表的主键/唯一键 |
| 主属性 | 出现在某个候选码中的属性 |
| 非主属性 | 不属于任何候选码的属性 |

ER 图用于描述实体、属性及实体间的 `1:1 / 1:N / N:M` 关系。

### 三大范式

| 范式 | 核心 |
| --- | --- |
| 1NF | 属性保持原子性 |
| 2NF | 在 1NF 基础上，消除非主属性对候选码的部分依赖 |
| 3NF | 在 2NF 基础上，消除非主属性对候选码的传递依赖 |

```text
2NF
→ 主要解决联合候选码下的部分依赖

3NF
→ A → B → C 时，避免非主属性通过 B 间接依赖候选码
```

### SQL 分类与约束

| 分类 | 作用 | 常见语句 |
| --- | --- | --- |
| DDL | 定义数据库对象 | `CREATE / ALTER / DROP / TRUNCATE` |
| DML | 增删改数据 | `INSERT / UPDATE / DELETE` |
| DQL | 查询数据 | `SELECT` |
| DCL | 权限控制 | `GRANT / REVOKE` |

常见约束：`PRIMARY KEY`、`FOREIGN KEY`、`UNIQUE`、`DEFAULT`、`CHECK`、`NOT NULL`。

SQL 逻辑执行顺序：

```text
FROM / JOIN
→ ON
→ WHERE
→ GROUP BY
→ HAVING
→ SELECT
→ DISTINCT
→ ORDER BY
→ LIMIT
```

> SQL 书写顺序不等于逻辑执行顺序。

## MySQL 架构与 SQL 执行

MySQL 可以粗分为：

```text
连接层
→ 服务层
→ 存储引擎层
→ 存储层
```

| 层 | 主要职责 |
| --- | --- |
| 连接层 | 连接管理、认证、权限 |
| 服务层 | SQL 解析、预处理、优化、执行等 |
| 存储引擎层 | 数据存取、索引、锁、事务等 |
| 存储层 | 磁盘 / 文件系统中的实际数据 |

一条查询大致经历：

```text
Client
→ Connector
→ Parser
→ Preprocessor
→ Optimizer
→ Executor
→ Storage Engine
→ Result
```

- **Parser**：词法/语法分析。
- **Preprocessor**：检查表、字段等并做语义处理。
- **Optimizer**：评估执行计划，决定索引和连接顺序。
- **Executor**：根据执行计划调用存储引擎。

旧版 Query Cache 在 MySQL 8.0 已移除。

### 存储引擎

| 引擎 | 特点 |
| --- | --- |
| InnoDB | 默认主流；支持事务、MVCC、行级锁、外键、崩溃恢复、B+Tree |
| MyISAM | 不支持事务和行级锁，主要使用表锁；现代业务通常优先 InnoDB |
| Memory | 数据主要在内存；速度快、重启后数据丢失，不支持事务；支持 HASH / BTREE 索引 |

# 2. 事务与 MVCC

## 事务与 ACID

事务是一组作为整体执行的操作：要么全部成功，要么全部失败。

| 特性 | 含义 | InnoDB 主要机制 |
| --- | --- | --- |
| Atomicity | 原子性，要么全部执行，要么全部回滚 | Undo Log |
| Consistency | 一致性，事务前后满足约束和业务规则 | A + I + D + 约束 + 业务规则共同保证 |
| Isolation | 隔离性，并发事务尽量互不干扰 | MVCC + Lock |
| Durability | 持久性，提交后结果可恢复 | Redo Log |

### 并发问题与隔离级别

| 问题 | 含义 |
| --- | --- |
| 脏读 | 读取其他事务尚未提交的数据 |
| 不可重复读 | 同一事务两次读取同一行，值发生变化 |
| 幻读 | 同一条件两次查询，结果行数发生变化 |

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |
| READ UNCOMMITTED | 可能 | 可能 | 可能 |
| READ COMMITTED | 避免 | 可能 | 可能 |
| REPEATABLE READ | 避免 | 避免 | InnoDB 可结合 MVCC、Next-Key Lock 等机制处理 |
| SERIALIZABLE | 避免 | 避免 | 避免 |

InnoDB 默认隔离级别是 `REPEATABLE READ`。

## MVCC

MVCC（Multi-Version Concurrency Control）让同一条记录保留多个历史版本，使不同事务看到适合自己的版本，从而提高并发读能力、减少读写冲突。

核心：

```text
MVCC
=
Undo Version Chain
+
ReadView
```

### Undo 版本链

更新记录时，旧版本进入 Undo Log，并通过 `roll_pointer` 形成版本链：

```text
当前版本
→ Undo 版本 1
→ Undo 版本 2
→ Undo 版本 3
```

常见隐藏字段：

| 字段 | 作用 |
| --- | --- |
| `DB_TRX_ID` | 最近修改记录的事务 ID |
| `DB_ROLL_PTR` | 指向上一历史版本 |
| `DB_ROW_ID` | 没有合适主键/唯一非空索引时，InnoDB 可能使用的隐藏 Row ID |

### ReadView

ReadView 可以理解为创建快照时的活跃事务视图：

| 字段 | 含义 |
| --- | --- |
| `m_ids` | 活跃事务 ID 集合 |
| `min_trx_id` | 最小活跃事务 ID |
| `max_trx_id` | 下一可分配事务 ID |
| `creator_trx_id` | 创建 ReadView 的事务 ID |

版本可见性简化判断：

| 条件 | 结果 |
| --- | --- |
| `trx_id == creator_trx_id` | 自己修改，可见 |
| `trx_id < min_trx_id` | 创建 ReadView 前已提交，可见 |
| `trx_id >= max_trx_id` | ReadView 创建后才开始，不可见 |
| 位于中间且在 `m_ids` | 当时仍活跃，不可见 |
| 位于中间且不在 `m_ids` | 当时已提交，可见 |

如果当前版本不可见，就沿 `roll_pointer` 向旧版本查找。

RC / RR：

```text
READ COMMITTED
→ 每次快照读通常创建新的 ReadView

REPEATABLE READ
→ 通常第一次快照读创建 ReadView
→ 后续快照读复用
```

### 快照读 / 当前读

| 类型 | 常见操作 | 特点 |
| --- | --- | --- |
| 快照读 | 普通 `SELECT` | 主要依赖 MVCC |
| 当前读 | `SELECT ... FOR UPDATE`、`UPDATE`、`DELETE` | 读取并锁定当前可操作版本，涉及锁 |

# 3. 索引

索引是帮助 MySQL 高效获取数据的数据结构；可以减少磁盘 IO 和部分排序成本，但会占空间，并增加 INSERT / UPDATE / DELETE 的维护成本。

## B+Tree 与索引类型

InnoDB 主要使用 B+Tree：

```text
多叉
→ 树高低
→ 一页可容纳更多索引项
→ 磁盘 IO 少

叶子节点有序
→ 范围查询 / 排序友好
```

B+Tree 与 B-Tree 的核心区别：B+Tree 非叶子节点主要保存索引键和指针，真正数据集中在叶子节点，因此分叉更多、树高更低。

Hash 索引适合等值查询，但本身无序，不适合范围查询、排序和前缀匹配。

索引常见分类：

| 维度 | 类型 |
| --- | --- |
| 数据结构 | B+Tree、Hash、Full-Text |
| 物理存储 | 聚簇索引、二级索引 |
| 字段特性 | 主键、唯一、普通、前缀 |
| 字段数量 | 单列、联合 |

## 聚簇索引 / 二级索引

```text
聚簇索引叶子
→ 保存完整行数据

二级索引叶子
→ 索引列 + 主键值
```

回表：

```text
二级索引
→ 找到主键
→ 聚簇索引
→ 完整行
```

覆盖索引：

```text
二级索引已经包含查询所需全部字段
→ 不需要回表
```

ICP（Index Condition Pushdown）把部分过滤条件下推到存储引擎扫描索引时执行，减少不必要回表。

## 联合索引与常见利用问题

联合索引 `(a,b,c)` 按 `a → b → c` 的顺序组织，最左前缀常见可利用：

```text
a
a,b
a,b,c
```

不要把下面情况机械记成“索引一定失效”，更准确是**优化器可能无法充分利用索引或认为全表扫描成本更低**。

| 情况 | 影响 |
| --- | --- |
| 跳过联合索引最左列 | 难以充分利用联合索引有序性 |
| 索引列上使用函数/运算 | 可能无法直接利用索引顺序 |
| 隐式类型转换发生在列上 | 可能影响索引利用 |
| 范围查询 | 后续列通常不能继续用于缩小索引扫描区间，但可能参与 ICP |
| OR 部分条件无索引 | 优化器可能选择全表扫描 |
| `LIKE '%abc'` | 无法用普通 B+Tree 前缀定位 |
| `!= / <> / NOT IN` 等 | 匹配范围大时可能选择全表扫描 |
| 表很小 / 选择性低 | 全表扫描可能更便宜 |
| `IS NULL / IS NOT NULL` | 是否用索引取决于数据分布和成本 |
| ORDER BY / GROUP BY 不匹配索引顺序 | 可能产生额外排序或临时表 |

# 4. 锁

需要区分：

```text
MVCC
→ 主要提高快照读并发

Lock
→ 解决当前读和写操作之间的冲突
```

InnoDB 的“行锁”本质上锁的是**索引记录**。

## 锁类型

| 层级 | 锁 | 作用 |
| --- | --- | --- |
| 全局 | Global Lock | 锁住整个实例，常见于全库一致性操作 |
| 表级 | Table Read / Write Lock | 表共享读锁 / 表独占写锁 |
| 表级 | MDL | 防止 DML 与 DDL 的元数据冲突 |
| 表级 | IS / IX 意向锁 | 让表级锁快速知道表内是否存在行锁 |
| 表级 | AUTO-INC | 协调并发插入中的自增值分配 |
| 行级 | Record Lock | 锁单个索引记录 |
| 行级 | Gap Lock | 锁索引记录之间的间隙，阻止插入 |
| 行级 | Next-Key Lock | Record Lock + Gap Lock |

MDL：

```text
CRUD
→ MDL 读锁

ALTER TABLE
→ MDL 写锁

长事务持有 MDL 读锁
→ DDL 可能长期等待
```

Next-Key Lock：

```text
Record Lock
+
Gap Lock
```

在 RR 下是处理当前读和幻读相关问题的重要机制。

如果查询没有合理利用索引，可能扫描并锁定更多索引记录，导致锁影响范围扩大。

# 5. 日志

## Redo / Undo / Binlog

| 日志 | 层级 | 核心作用 |
| --- | --- | --- |
| Redo Log | InnoDB | 崩溃恢复，支持 Durability |
| Undo Log | InnoDB | 事务回滚 + MVCC，支持 Atomicity |
| Binlog | Server | 数据变更日志，复制和数据恢复 |
| Relay Log | Replica | 保存从 Source 接收的复制日志 |
| Slow Query Log | Server | 记录慢 SQL，辅助性能分析 |

### Redo Log

Redo 记录数据页修改对应的重做信息，不是简单记录 SQL：

```text
事务修改
→ Redo Log Buffer
→ 根据策略写入 / 刷盘
```

`innodb_flush_log_at_trx_commit`：

| 值 | 行为 |
| --- | --- |
| 0 | 通常由后台周期写/刷，崩溃时可能丢最近一段日志 |
| 1 | 每次提交刷盘，安全性最高 |
| 2 | 每次提交写到 OS Cache，再由 OS 后续刷盘 |

Redo 使用固定空间循环复用，旧的、已不再需要的日志可以被覆盖。

### Undo Log

```text
更新前
→ 保存旧版本 / 回滚信息

事务失败
→ Undo 恢复原状态

MVCC
→ Undo 保存历史版本
→ roll_pointer 形成版本链
```

### Binlog

Binlog 是 Server 层逻辑变更日志，通常追加写并生成新文件，不像 Redo 一样固定空间循环覆盖。

| 格式 | 含义 |
| --- | --- |
| STATEMENT | 记录 SQL 语句 |
| ROW | 记录行数据变化 |
| MIXED | 根据情况使用 STATEMENT / ROW |

Redo vs Binlog：

| 对比 | Redo Log | Binlog |
| --- | --- | --- |
| 层级 | InnoDB | Server |
| 类型 | 重做日志 | 逻辑变更日志 |
| 主要用途 | 崩溃恢复 | 复制、数据恢复 |
| 写法 | 循环使用 | 追加并生成新文件 |

### 两阶段提交

为了保证 Redo Log 与 Binlog 对一次事务的描述一致：

```text
事务提交
→ Redo Prepare
→ 写 Binlog
→ Redo Commit
```

> 两阶段提交的核心目的：协调 Redo Log 与 Binlog 的一致性。

# 6. 复制与分库分表

## 主从复制

```text
Source
→ 产生 Binlog
→ Replica I/O Thread 获取日志
→ Relay Log
→ Replica SQL / Applier Thread
→ 应用变更
→ Replica 数据更新
```

作用：读写分离、数据副本、高可用、灾备。

> 复制不等于自动强一致，可能存在复制延迟。

## 分库分表

| 方式 | 怎么拆 | 主要作用 / 问题 |
| --- | --- | --- |
| 垂直分库 | 按业务模块拆库 | 业务隔离、降低单库压力；增加跨库事务/查询复杂度 |
| 垂直分表 | 按字段拆宽表 | 减少核心表宽度和 IO |
| 水平分表 | 同库中按数据行拆多个同结构表 | 降低单表数据量，但仍共享同一数据库实例资源 |
| 水平分库 | 按数据行分散到多个数据库实例 | 解决单机容量/并发瓶颈，但增加路由、分布式事务、全局 ID、扩容迁移等复杂度 |

```text
垂直
→ 按业务 / 字段切

水平
→ 按数据行切
```

# 7. 速记

```text
SQL 执行
Client
→ Connector
→ Parser
→ Preprocessor
→ Optimizer
→ Executor
→ Storage Engine
```

```text
ACID
A → Undo
I → MVCC + Lock
D → Redo
C → A + I + D + 约束 + 业务规则
```

```text
MVCC
→ Undo Version Chain + ReadView

RC
→ 每次快照读通常新 ReadView

RR
→ 通常复用第一次快照读的 ReadView
```

```text
索引
聚簇索引叶子 → 完整行
二级索引叶子 → 索引列 + 主键
回表 → 二级索引 → 主键 → 聚簇索引
覆盖索引 → 索引已包含全部查询字段
```

```text
锁
Record   → 索引记录
Gap      → 间隙
Next-Key → Record + Gap
```

```text
日志
Undo   → 回滚 + MVCC
Redo   → 崩溃恢复
Binlog → 复制 + 数据恢复
Relay  → Replica 中继日志
```

```text
两阶段提交
Redo Prepare
→ Binlog
→ Redo Commit
```
