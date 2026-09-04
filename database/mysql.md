# MySQL 基础知识结构速记

> MySQL 学习主线：

```text
SQL 怎么写
   ↓
MySQL 怎么执行
   ↓
事务如何保证正确
   ↓
MVCC 如何提高并发
   ↓
索引如何提高查询效率
   ↓
锁如何解决并发冲突
   ↓
日志如何保证恢复与复制
   ↓
单机不够时如何扩展
```

---

# SQL 基础

## 基本概念

### DBMS

DBMS：

```text
Database Management System
数据库管理系统
```

作用：

```text
创建数据库
管理数据
查询数据
控制权限
保证数据一致性
```

MySQL 就是一种 DBMS。

---

### 元组

关系数据库中：

```text
一行数据
→ 一个元组
```

例如：

| id | name | age |
| --- | --- | --- |
| 1 | 张三 | 20 |

这一行就是一个元组。

---

### 码

码：

```text
能够唯一标识一个元组的属性集合
```

---

### 候选码

候选码：

```text
能够唯一标识元组
+
不存在多余属性
```

一张表可以有多个候选码。

---

### 主码

主码：

```text
从候选码中
选一个作为主要标识
```

也就是：

```text
Primary Key
```

---

### 外码

外码：

```text
一个表中的字段
引用另一个表的主键 / 唯一键
```

用于建立表之间的关联关系。

---

### 主属性与非主属性

主属性：

```text
出现在某个候选码中的属性
```

非主属性：

```text
不属于任何候选码的属性
```

---

### ER 图

ER：

```text
Entity Relationship
实体关系图
```

主要描述：

```text
实体
属性
实体之间的关系
```

常见关系：

```text
1 : 1
1 : N
N : M
```

---

## 数据库三大范式

### 1NF

第一范式：

```text
字段必须保持原子性
不能再继续拆分
```

例如不推荐：

```text
address = 浙江省杭州市余杭区
```

如果业务需要分别查询省、市、区，可以拆成：

```text
province
city
district
```

---

### 2NF

第二范式：

```text
在 1NF 基础上
消除非主属性
对候选码的部分函数依赖
```

也就是：

```text
非主属性
必须完全依赖候选码
```

主要针对：

```text
联合候选码
```

---

### 3NF

第三范式：

```text
在 2NF 基础上
消除非主属性
对候选码的传递函数依赖
```

例如：

```text
student_id
   ↓
class_id
   ↓
class_name
```

如果：

```text
student_id → class_id
class_id → class_name
```

那么：

```text
student_id → class_name
```

属于传递依赖。

---

## SQL 分类

### DDL

DDL：

```text
Data Definition Language
数据定义语言
```

用于定义数据库对象：

```text
数据库
表
字段
索引
```

常见：

```sql
CREATE
ALTER
DROP
TRUNCATE
```

---

### DML

DML：

```text
Data Manipulation Language
数据操作语言
```

用于：

```text
INSERT
UPDATE
DELETE
```

---

### DQL

DQL：

```text
Data Query Language
数据查询语言
```

主要：

```sql
SELECT
```

---

### DCL

DCL：

```text
Data Control Language
数据控制语言
```

用于：

```text
用户
权限
授权
回收权限
```

常见：

```sql
GRANT
REVOKE
```

---

## 约束

常见约束：

```text
PRIMARY KEY
FOREIGN KEY
UNIQUE
DEFAULT
CHECK
NOT NULL
```

### 主键约束

```text
唯一
+
非空
```

一张表通常只有一个主键定义。

---

### 外键约束

用于保证：

```text
引用完整性
```

常见删除 / 更新行为：

```text
RESTRICT
NO ACTION
CASCADE
SET NULL
```

其中：

```text
CASCADE
→ 父表变化时同步影响子表

SET NULL
→ 父表记录删除后
  子表外键设为 NULL
```

---

## DQL

### 单表查询

常见：

```text
基础查询
条件查询
聚合查询
分组查询
排序查询
分页查询
```

---

### 多表查询

常见：

```text
内连接
外连接
自连接
联合查询
子查询
```

---

### SQL 执行逻辑顺序

可以记：

```text
FROM / JOIN
   ↓
ON
   ↓
WHERE
   ↓
GROUP BY
   ↓
HAVING
   ↓
SELECT
   ↓
DISTINCT
   ↓
ORDER BY
   ↓
LIMIT
```

注意：

```text
SQL 书写顺序
≠
逻辑执行顺序
```

---

# MySQL 架构与存储引擎

## MySQL 体系结构

可以分为：

```text
连接层
   ↓
服务层
   ↓
存储引擎层
   ↓
存储层
```

### 连接层

负责：

```text
连接管理
用户认证
权限校验
```

---

### 服务层

负责 MySQL 核心功能：

```text
SQL 解析
预处理
优化
执行
函数
存储过程
触发器
```

---

### 存储引擎层

负责：

```text
数据真正如何存储
如何读取
如何加锁
事务机制
索引实现
```

---

### 存储层

最终数据存放在：

```text
磁盘
文件系统
```

---

## SQL 执行流程

一个查询大致经历：

```text
客户端
   ↓
连接器
   ↓
解析器
   ↓
预处理
   ↓
优化器
   ↓
执行器
   ↓
存储引擎
   ↓
返回结果
```

---

### 连接器

负责：

```text
建立连接
管理连接
认证用户
检查权限
```

---

### 查询缓存

旧版本 MySQL 曾有 Query Cache：

```text
命中缓存
→ 直接返回
```

但：

```text
MySQL 8.0
→ 已移除 Query Cache
```

---

### 解析器

负责：

```text
词法分析
语法分析
构建语法结构
```

例如识别：

```text
表名
字段名
SQL 类型
```

---

### 预处理

负责：

```text
检查表是否存在
检查字段是否存在
展开 SELECT *
```

---

### 优化器

负责：

```text
生成多个可能执行计划
   ↓
估算成本
   ↓
选择成本更低的方案
```

包括：

```text
是否用索引
用哪个索引
表连接顺序
```

---

### 执行器

根据优化器给出的执行计划：

```text
调用存储引擎
读取数据
返回客户端
```

---

## 存储引擎

MySQL 的特点之一：

```text
Server 层
与
存储引擎
分离
```

常见：

```text
InnoDB
MyISAM
Memory
```

---

## InnoDB

MySQL 默认存储引擎。

特点：

```text
支持事务
支持 MVCC
支持行级锁
支持外键
支持崩溃恢复
使用 B+Tree 索引
```

适合：

```text
高并发
读写混合
事务要求高
```

---

## MyISAM

特点：

```text
不支持事务
不支持行级锁
不支持外键
主要使用表锁
```

优势：

```text
结构简单
某些只读场景性能较好
```

但现代业务系统通常优先使用 InnoDB。

---

## Memory

Memory 引擎：

```text
数据主要存储在内存
```

特点：

```text
速度快
服务器重启后数据丢失
不支持事务
```

索引：

```text
默认常用 HASH
也支持 BTREE
```

适合：

```text
临时数据
中间结果
对持久性要求低
```

---

# 事务与 MVCC

## 事务

事务：

```text
一组操作
作为一个整体执行
```

核心：

```text
要么全部成功
要么全部失败
```

---

## ACID

### Atomicity

原子性：

```text
事务不可再分
要么全部执行
要么全部回滚
```

主要由：

```text
Undo Log
```

支持。

---

### Consistency

一致性：

```text
事务执行前后
数据库都应该满足完整性规则
```

它是事务最终目标。

由：

```text
原子性
隔离性
持久性
约束
业务规则
```

共同保证。

不是简单由某一种日志独立实现。

---

### Isolation

隔离性：

```text
并发事务之间
尽量互不干扰
```

主要依赖：

```text
MVCC
+
Lock
```

---

### Durability

持久性：

```text
事务一旦提交
结果应该永久保存
```

主要由：

```text
Redo Log
```

保证。

---

## 并发事务问题

### 脏读

```text
事务 A
读取了事务 B
尚未提交的数据
```

如果 B 回滚：

```text
A 读到的数据就是脏数据
```

---

### 不可重复读

同一事务中：

```text
第一次读
→ value = 10

其他事务修改并提交

第二次读
→ value = 20
```

同一行数据前后不一致。

---

### 幻读

同一事务中按条件查询：

```text
第一次
→ 10 行

其他事务插入满足条件的新记录

第二次
→ 11 行
```

像出现了“幻影”。

---

## 隔离级别

从低到高：

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
| --- | --- | --- | --- |
| READ UNCOMMITTED | 可能 | 可能 | 可能 |
| READ COMMITTED | 避免 | 可能 | 可能 |
| REPEATABLE READ | 避免 | 避免 | MySQL InnoDB 中可通过 MVCC + Next-Key Lock 等机制处理 |
| SERIALIZABLE | 避免 | 避免 | 避免 |

MySQL InnoDB 默认：

```text
REPEATABLE READ
```

---

## MVCC

MVCC：

```text
Multi-Version Concurrency Control
多版本并发控制
```

核心：

```text
同一条记录
保留多个历史版本
```

使不同事务：

```text
可以看到
适合自己的版本
```

从而：

```text
提高并发读性能
减少读写冲突
```

---

## Undo Log 与版本链

InnoDB 更新记录时：

```text
旧版本
写入 Undo Log
```

记录通过：

```text
roll_pointer
```

形成历史版本链。

可以理解：

```text
当前版本
   ↓
Undo 版本 1
   ↓
Undo 版本 2
   ↓
Undo 版本 3
```

---

## InnoDB 记录隐藏字段

常见重要隐藏字段：

```text
DB_TRX_ID
DB_ROLL_PTR
DB_ROW_ID
```

### DB_TRX_ID

```text
最近修改该记录的事务 ID
```

---

### DB_ROLL_PTR

```text
指向 Undo Log
中上一历史版本
```

---

### DB_ROW_ID

当表：

```text
没有合适的主键
或唯一非空索引
```

用于生成聚簇索引时，InnoDB 可能使用内部隐藏 Row ID。

所以：

```text
DB_ROW_ID
并不是所有表都必须实际使用
```

---

## ReadView

ReadView 可以理解：

```text
某个时刻
数据库中事务活跃情况的快照
```

常见字段：

| 字段 | 含义 |
| --- | --- |
| `m_ids` | 当前活跃事务 ID 集合 |
| `min_trx_id` | 最小活跃事务 ID |
| `max_trx_id` | 下一可分配事务 ID |
| `creator_trx_id` | 创建 ReadView 的事务 ID |

---

## ReadView 可见性判断

假设某版本记录的事务 ID：

```text
trx_id
```

### 自己修改

```text
trx_id == creator_trx_id
→ 可见
```

---

### 小于 min_trx_id

```text
trx_id < min_trx_id
```

说明：

```text
创建 ReadView 前
该事务已经提交
```

所以：

```text
可见
```

---

### 大于等于 max_trx_id

```text
trx_id >= max_trx_id
```

说明：

```text
创建 ReadView 后
事务才开始
```

所以：

```text
不可见
```

---

### 位于中间范围

如果：

```text
min_trx_id <= trx_id < max_trx_id
```

看是否在：

```text
m_ids
```

中。

如果在：

```text
当时事务仍活跃
→ 不可见
```

如果不在：

```text
事务已经提交
→ 可见
```

如果当前版本不可见：

```text
沿 roll_pointer
   ↓
继续找 Undo Log 中旧版本
```

直到找到可见版本。

---

## RC 与 RR 的 ReadView

### READ COMMITTED

RC：

```text
每次快照读
通常都会创建新的 ReadView
```

因此：

```text
可以看到
其他事务之后提交的数据
```

---

### REPEATABLE READ

RR：

```text
通常第一次快照读创建 ReadView
后续快照读复用
```

因此：

```text
同一事务中
多次快照读结果通常保持一致
```

---

## 快照读与当前读

### 快照读

普通：

```sql
SELECT ...
```

通常属于：

```text
Snapshot Read
快照读
```

主要依赖：

```text
MVCC
```

---

### 当前读

例如：

```sql
SELECT ... FOR UPDATE;
UPDATE ...;
DELETE ...;
```

需要读取：

```text
最新已提交 / 当前可锁版本
```

属于：

```text
Current Read
当前读
```

通常会涉及：

```text
锁
```

---

# 索引

## 索引基础

索引：

```text
帮助 MySQL
高效获取数据的数据结构
```

优点：

```text
减少磁盘 IO
提高查询效率
降低部分排序成本
```

缺点：

```text
占用空间
INSERT / UPDATE / DELETE
需要维护索引
```

所以：

```text
索引不是越多越好
```

---

## 索引分类

### 按数据结构

```text
B+Tree
Hash
Full-Text
```

---

### 按物理存储

```text
聚簇索引
二级索引
```

---

### 按字段特性

```text
主键索引
唯一索引
普通索引
前缀索引
```

---

### 按字段数量

```text
单列索引
联合索引
```

---

## B+Tree

InnoDB 主要使用：

```text
B+Tree
```

特点：

```text
多叉
树高低
叶子节点有序
适合磁盘页
适合范围查询
适合排序
```

---

## B+Tree vs B-Tree

B-Tree：

```text
非叶子节点
也保存数据
```

B+Tree：

```text
非叶子节点主要保存索引键和指针
真正数据集中在叶子节点
```

因此一页能保存更多索引项：

```text
分叉更多
树高更低
磁盘 IO 更少
```

---

## Hash 索引

特点：

```text
等值查询快
```

适合：

```text
=
IN
```

不适合：

```text
范围查询
排序
前缀匹配
```

因为 Hash：

```text
本身无序
```

---

## 为什么 InnoDB 使用 B+Tree

可以记：

```text
层级低
磁盘 IO 少
范围查询友好
排序友好
叶子节点有序
```

相比 Hash：

```text
B+Tree
支持范围和排序
```

相比普通二叉树：

```text
B+Tree
分叉更多
高度更低
```

---

## 聚簇索引与二级索引

### 聚簇索引

InnoDB：

```text
聚簇索引叶子节点
保存完整行数据
```

通常：

```text
主键索引
就是聚簇索引
```

---

### 二级索引

二级索引叶子节点通常保存：

```text
索引列
+
主键值
```

查询流程：

```text
二级索引
   ↓
找到主键
   ↓
聚簇索引
   ↓
找到整行数据
```

---

## 回表

如果查询：

```text
使用二级索引
但需要的字段
不全在二级索引中
```

则：

```text
二级索引
   ↓
拿主键
   ↓
回到聚簇索引
   ↓
读取完整记录
```

这就是：

```text
回表
```

---

## 覆盖索引

如果：

```text
索引中
已经包含查询所需全部字段
```

则：

```text
无需回表
```

这就是：

```text
覆盖索引
```

---

## 索引下推 ICP

ICP：

```text
Index Condition Pushdown
```

核心：

```text
原本部分条件
要回表后再过滤
```

改为：

```text
在存储引擎扫描索引时
提前过滤
```

作用：

```text
减少不必要回表
降低 IO
```

---

## 联合索引与最左前缀

联合索引：

```text
(a, b, c)
```

索引顺序类似：

```text
先按 a
再按 b
再按 c
```

通常可以有效利用：

```text
a
a,b
a,b,c
```

如果跳过最左列：

```text
b
b,c
```

通常不能完整利用联合索引的有序性。

---

## 索引无法充分利用的常见情况

不要简单理解为：

```text
一定“索引失效”
```

很多情况是：

```text
优化器根据成本
可能不使用
或只能部分使用索引
```

常见情况：

### 1. 不满足最左前缀

联合索引：

```text
(a,b,c)
```

查询跳过 `a`：

```text
难以充分利用联合索引
```

---

### 2. 索引列上使用函数或运算

例如：

```sql
WHERE YEAR(create_time) = 2026
```

对索引列做函数：

```text
可能无法直接利用索引有序性
```

---

### 3. 隐式类型转换

例如：

```text
phone 为 VARCHAR
```

却写：

```sql
WHERE phone = 123456
```

可能导致：

```text
列发生隐式转换
→ 无法正常利用索引
```

---

### 4. 范围查询

联合索引遇到：

```text
>
<
BETWEEN
```

后：

```text
后续列通常不能继续
用于缩小索引扫描区间
```

但后续列：

```text
仍可能参与 ICP 等优化
```

---

### 5. OR 中部分条件无索引

例如：

```text
A 有索引
OR
B 无索引
```

优化器可能：

```text
选择全表扫描
```

---

### 6. LIKE 左模糊

```sql
LIKE '%abc'
```

因为无法确定起始位置：

```text
通常无法使用普通 B+Tree 前缀定位
```

---

### 7. 否定条件范围过大

例如：

```text
<>
!=
NOT IN
NOT EXISTS
```

不是绝对不能使用索引。

如果匹配数据过多：

```text
优化器可能认为
全表扫描成本更低
```

---

### 8. 数据量小或选择性低

如果：

```text
表很小
或
索引区分度很低
```

优化器可能：

```text
直接全表扫描
```

---

### 9. NULL 条件

`IS NULL / IS NOT NULL`：

```text
能否使用索引
取决于数据分布和优化器成本
```

不能简单记成一定失效。

---

### 10. ORDER BY / GROUP BY 未匹配索引顺序

如果排序 / 分组顺序：

```text
不能利用索引有序性
```

可能出现：

```text
额外排序
临时表
```

---

# 锁

## 锁的作用

Lock 用于：

```text
解决并发修改
和当前读之间的冲突
```

需要区分：

```text
MVCC
→ 主要提高快照读并发

Lock
→ 解决当前读和写操作冲突
```

---

## 全局锁

锁住整个数据库实例。

典型用途：

```text
全库备份
整体数据一致性操作
```

代价：

```text
并发能力大幅下降
```

---

# 表级锁

## 表锁

### 表共享读锁

```text
Table Read Lock
```

允许：

```text
其他事务读
```

不允许：

```text
其他事务写
```

---

### 表独占写锁

```text
Table Write Lock
```

持有者：

```text
可以读写
```

其他事务：

```text
无法继续读写该表
```

---

## MDL

MDL：

```text
Metadata Lock
元数据锁
```

由系统自动管理。

作用：

```text
防止 DML
与 DDL
发生结构冲突
```

例如：

```text
CRUD
→ MDL 读锁

ALTER TABLE
→ MDL 写锁
```

如果有长事务一直访问某表：

```text
DDL
可能一直等待 MDL 写锁
```

---

## 意向锁

意向锁属于：

```text
表级锁
```

主要作用：

> 让表级锁快速知道某张表内部是否已经存在行级锁。

例如：

```text
事务准备给某行加共享锁
→ 表上先有 IS

事务准备给某行加排他锁
→ 表上先有 IX
```

意向锁之间：

```text
通常不会互相阻塞
```

主要和显式表锁发生兼容性判断。

---

## AUTO-INC 锁

用于：

```text
自增主键分配
```

不同：

```text
innodb_autoinc_lock_mode
```

下行为不同。

核心思想：

```text
保证并发插入时
自增值分配正确
```

具体锁模式属于版本相关实现细节，学习时重点记：

```text
传统表级 AUTO-INC 锁
和
更轻量的自增分配机制
```

---

# 行级锁

InnoDB 的“行锁”本质上：

```text
锁的是索引记录
```

不是直接锁物理“行”。

---

## Record Lock

```text
锁单个索引记录
```

主要防止：

```text
UPDATE
DELETE
当前读
```

之间冲突。

---

## Gap Lock

```text
锁索引记录之间的间隙
不锁具体记录本身
```

主要目的：

```text
阻止其他事务
向间隙插入新记录
```

常用于：

```text
RR
```

下处理幻读相关问题。

---

## Next-Key Lock

Next-Key Lock：

```text
Record Lock
+
Gap Lock
```

也就是：

```text
锁记录
+
锁记录前的间隙
```

是 InnoDB RR 下非常重要的锁形式。

---

## 锁与索引

因为 InnoDB：

```text
行锁建立在索引之上
```

如果查询：

```text
没有合理利用索引
```

可能扫描并锁定：

```text
更多索引记录
```

最终表现为：

```text
锁影响范围变大
```

---

# 日志

MySQL 常见日志：

```text
Redo Log
Undo Log
Binlog
Relay Log
Slow Query Log
```

---

## Redo Log

Redo Log：

```text
InnoDB 存储引擎层
```

属于：

```text
重做日志
```

核心作用：

```text
Crash Recovery
崩溃恢复
```

保证：

```text
Durability
持久性
```

---

### Redo Log 内容

可以理解为：

```text
对数据页修改
对应的物理重做信息
```

不是简单记录 SQL。

---

### 写入方式

事务执行过程中：

```text
不断产生 Redo
```

并先进入：

```text
Redo Log Buffer
```

之后根据策略刷盘。

---

### 循环写

Redo Log：

```text
固定空间
循环使用
```

写满后：

```text
覆盖已经不再需要的旧日志
```

---

### 刷盘策略

主要受：

```text
innodb_flush_log_at_trx_commit
```

影响。

#### 0

```text
大约每秒写 / 刷一次
```

崩溃时：

```text
可能丢失约 1 秒事务日志
```

#### 1

```text
每次事务提交
都刷到磁盘
```

安全性最高。

#### 2

```text
每次提交
写到 OS Cache

由操作系统
负责后续刷盘
```

---

## Undo Log

Undo Log：

```text
InnoDB 存储引擎层
```

属于：

```text
逻辑意义上的回滚日志
```

作用：

```text
事务回滚
MVCC
```

---

### 支持事务回滚

更新前：

```text
保存旧值 / 逆向操作信息
```

事务失败：

```text
利用 Undo
恢复原状态
```

因此支持：

```text
Atomicity
```

---

### 支持 MVCC

Undo 中保存：

```text
历史版本
```

通过：

```text
roll_pointer
```

形成版本链。

---

## Binlog

Binlog：

```text
Server 层
```

是：

```text
二进制逻辑日志
```

主要记录：

```text
数据库变更事件
```

用途：

```text
主从复制
数据恢复
审计
```

---

### Binlog 文件

Binlog：

```text
不是固定空间循环覆盖
```

通常：

```text
文件写满
或切换条件满足
→ 创建新文件
```

---

### Binlog 三种格式

#### STATEMENT

```text
记录 SQL 语句
```

---

#### ROW

```text
记录行数据变化
```

不是简单记录原 SQL。

---

#### MIXED

```text
STATEMENT
+
ROW
```

由 MySQL 根据情况选择。

---

## Redo Log vs Binlog

| 对比 | Redo Log | Binlog |
| --- | --- | --- |
| 层级 | InnoDB | Server |
| 类型 | 重做日志 | 逻辑变更日志 |
| 主要用途 | 崩溃恢复 | 复制、恢复 |
| 写法 | 循环使用 | 追加并生成新文件 |

---

## 两阶段提交

为什么需要：

```text
Redo Log
和
Binlog
都描述一次事务
```

如果两者状态不一致：

```text
崩溃恢复
与
主从复制
可能得到不同结果
```

所以采用：

```text
Two-Phase Commit
```

流程：

```text
事务提交
   ↓
Redo Log Prepare
   ↓
写 Binlog
   ↓
Redo Log Commit
   ↓
提交完成
```

目的：

> **保证 Redo Log 与 Binlog 之间的一致性。**

---

## Relay Log

Relay Log：

```text
中继日志
```

用于：

```text
主从复制
```

Replica 从 Source 获取 Binlog 后：

```text
先写入本地 Relay Log
```

再由复制线程：

```text
读取 Relay Log
应用数据变更
```

---

## 慢查询日志

Slow Query Log：

```text
记录执行时间超过阈值的 SQL
```

用途：

```text
发现慢 SQL
进行性能优化
```

通常需要配置：

```text
开启
+
慢查询阈值
```

---

# MySQL 架构与扩展

## 主从复制

核心链路：

```text
Source
  ↓
产生 Binlog
  ↓
Replica I/O Thread
  ↓
读取 Binlog
  ↓
写入 Relay Log
  ↓
Replica SQL / Applier Thread
  ↓
重放日志
  ↓
Replica 数据更新
```

可以记：

```text
Source
→ Binlog

Replica
→ Relay Log
```

---

## 主从复制作用

常见：

```text
读写分离
数据备份
高可用
灾备
```

需要注意：

```text
复制
并不等于自动强一致
```

可能存在：

```text
复制延迟
```

---

## 分库分表

核心分类：

```text
分库分表
├── 垂直拆分
│   ├── 垂直分库
│   └── 垂直分表
│
└── 水平拆分
    ├── 水平分库
    └── 水平分表
```

一句话：

```text
垂直
→ 按业务 / 字段切

水平
→ 按数据行切
```

---

## 垂直分库

按：

```text
业务模块
```

拆数据库。

例如：

```text
用户库
订单库
支付库
积分库
```

优点：

```text
专库专用
降低单库压力
业务隔离
```

问题：

```text
跨库事务
跨库查询
系统复杂度提升
```

---

## 垂直分表

针对宽表：

```text
按字段拆
```

例如：

```text
用户核心表
├── id
├── name
└── phone

用户详情表
├── user_id
├── description
├── avatar
└── extra_info
```

作用：

```text
减少核心表宽度
减少 IO
提高热点字段访问效率
```

---

## 水平分表

同一个数据库中：

```text
把同结构大表
按数据行拆成多个子表
```

例如：

```text
order_0
order_1
order_2
order_3
```

规则：

```text
user_id % 4
```

解决：

```text
单表数据量过大
```

但：

```text
仍在同一个数据库实例
```

所以：

```text
CPU
内存
网络
磁盘 IO
```

仍共享。

---

## 水平分库

把同一张逻辑表的数据：

```text
按规则
分散到多个数据库实例
```

例如：

```text
DB0.order
DB1.order
DB2.order
DB3.order
```

可以解决：

```text
单机容量瓶颈
单机并发瓶颈
```

但增加：

```text
路由
分布式事务
跨库查询
全局 ID
扩容迁移
```

复杂度。

---

# 速记

## SQL

```text
DDL
→ 定义结构

DML
→ 增删改

DQL
→ 查询

DCL
→ 权限
```

---

## 三大范式

```text
1NF
→ 属性原子

2NF
→ 消除非主属性
  对候选码的部分依赖

3NF
→ 消除非主属性
  对候选码的传递依赖
```

---

## MySQL 执行

```text
Client
 ↓
Connector
 ↓
Parser
 ↓
Preprocessor
 ↓
Optimizer
 ↓
Executor
 ↓
Storage Engine
```

---

## ACID

```text
A
→ Undo Log

I
→ MVCC + Lock

D
→ Redo Log

C
→ A + I + D + 约束 + 业务规则
```

---

## MVCC

```text
MVCC
=
Undo Version Chain
+
ReadView
```

RC：

```text
每次快照读
新 ReadView
```

RR：

```text
通常复用第一次快照读的 ReadView
```

---

## 索引

```text
InnoDB
→ B+Tree

聚簇索引叶子
→ 完整行

二级索引叶子
→ 索引列 + 主键
```

回表：

```text
二级索引
→ 主键
→ 聚簇索引
```

覆盖索引：

```text
索引已包含全部查询字段
→ 不回表
```

---

## 锁

```text
表级
├── Table Lock
├── MDL
├── Intention Lock
└── AUTO-INC

行级
├── Record Lock
├── Gap Lock
└── Next-Key Lock
```

记：

```text
Record
→ 记录

Gap
→ 间隙

Next-Key
→ Record + Gap
```

---

## 日志

```text
Undo
→ 回滚 + MVCC
→ Atomicity

Redo
→ 崩溃恢复
→ Durability

Binlog
→ 复制 + 数据恢复

Relay Log
→ Replica 中继日志
```

两阶段提交：

```text
Redo Prepare
   ↓
Binlog
   ↓
Redo Commit
```

---

## 分库分表

```text
垂直分库
→ 按业务拆库

垂直分表
→ 按字段拆表

水平分表
→ 按行拆表，同库

水平分库
→ 按行拆到多个数据库实例
```

---

# 一句话总结

```text
SQL
→ 怎么操作数据

Storage Engine
→ 数据怎么存

Transaction / MVCC / Lock
→ 并发时怎么保证正确

Index
→ 怎么查得更快

Log
→ 崩溃后怎么恢复、数据怎么复制

Sharding / Replication
→ 单机不够以后怎么扩展
```
