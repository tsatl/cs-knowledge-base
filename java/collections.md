# Java 集合框架
## 集合体系
Java 集合主要分为 `Collection` 和 `Map` 两大体系，其中 `Map` 不属于 `Collection`。

```text
Java 集合框架
│
├── Collection
│   ├── List
│   │   ├── ArrayList
│   │   ├── LinkedList
│   │   └── Vector
│   │       └── Stack
│   ├── Set
│   │   ├── HashSet
│   │   │   └── LinkedHashSet
│   │   └── TreeSet
│   └── Queue
│       ├── Deque
│       │   ├── ArrayDeque
│       │   └── LinkedList
│       └── PriorityQueue
│
└── Map
    ├── HashMap
    │   └── LinkedHashMap
    ├── TreeMap
    └── Hashtable
```

注意：

```text
LinkedList
→ 同时实现 List 和 Deque

Stack
→ extends Vector

LinkedHashMap
→ extends HashMap

LinkedHashSet
→ extends HashSet
```

## Collection 与 Map
### Collection
用于存储单个元素，主要分为：List → 有序、可重复；Set → 不允许重复；Queue → 按队列规则组织元素

### Map
用于存储：`key-value`

特点：key 不能重复、value 可以重复

## 常见集合选型
```text
需要 List？
│
├─ 随机访问多
│  → ArrayList
│
└─ 大量头尾操作
   → LinkedList / ArrayDeque
```

```text
需要 Map？
│
├─ 普通快速查找
│  → HashMap
│
├─ 保持插入顺序 / 访问顺序
│  → LinkedHashMap
│
└─ key 排序
   → TreeMap
```

```text
需要 Set？
│
├─ 普通去重
│  → HashSet
│
├─ 去重 + 插入顺序
│  → LinkedHashSet
│
└─ 去重 + 排序
   → TreeSet
```

```text
需要 Queue？
│
├─ 普通队列 / 双端队列 / 栈
│  → ArrayDeque
│
└─ 优先级
   → PriorityQueue
```

并发集合如 `ConcurrentHashMap`、`CopyOnWriteArrayList`、`BlockingQueue` 等统一放到 JUC 中学习。

# List
List 的核心特点：允许重复、保持元素顺序、可以通过索引访问

| 实现 | 底层结构 | 有序 | 线程安全 | 典型特点 |
| --- | --- | --- | --- | --- |
| ArrayList | 动态数组 | 是 | 否 | 随机访问快 |
| LinkedList | 双向链表 | 是 | 否 | 头尾增删方便 |
| Vector | 动态数组 | 是 | 是 | 老旧同步集合 |
| Stack | Vector | LIFO | 是 | 老旧栈实现 |

## ArrayList
### 底层结构
ArrayList 基于：`Object[] elementData`

动态数组实现，因此：随机访问快、中间插入 / 删除需要移动元素

### 容量与扩容
JDK 8 中，无参构造时底层先使用空数组，第一次添加元素时通常扩为默认容量：`10`

空间不足时通常扩为原容量约：1.5 倍

核心计算：newCapacity = oldCapacity + (oldCapacity >> 1);

### 核心操作
```text
按索引查询
→ O(1)

尾部追加
→ 均摊 O(1)

指定位置插入 / 删除
→ O(n)
```

元素移动主要通过：System.arraycopy(src, srcPos, dest, destPos, length);

删除元素后会将最后一个失效位置置为 `null`，切断引用，帮助 GC。

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null` | 允许 |
| 重复元素 | 允许 |
| 顺序 | 保持插入顺序 |
| 线程安全 | 否 |

### 源码要点
ArrayList 使用 `modCount` 记录结构性修改次数，其迭代器支持 fail-fast，具体机制见“集合通用机制”。

`elementData` 使用 `transient`，避免默认序列化整个底层数组。ArrayList 会通过自定义序列化逻辑只序列化 `size` 个有效元素，不保存多余容量。

ArrayList 允许存储 `null`，因为它通过 `size` 区分有效区间与未使用空间：[0, size) → 有效元素；[size, elementData.length) → 未使用空间

## LinkedList
### 底层结构
LinkedList 基于双向链表，每个节点保存：item、next、prev

同时维护：first、last

```text
null ← Node1 ⇄ Node2 ⇄ Node3 → null
       ↑                    ↑
     first                 last
```

### 核心操作
头尾添加 / 删除 → O(1)；按索引访问 → O(n)

查找元素时：索引在前半段 → 从 first 开始；索引在后半段 → 从 last 开始

已知节点时插入 / 删除本身是 `O(1)`，但如果需要先通过索引找节点，通常仍为 `O(n)`。

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null` | 允许 |
| 重复元素 | 允许 |
| 顺序 | 保持插入顺序 |
| 线程安全 | 否 |

### 源码要点
LinkedList → 同时实现 List 和 Deque

因此既可以作为普通 List，也可以作为队列和双端队列使用。

## Vector 与 Stack
### Vector
Vector 与 ArrayList 类似，底层都是动态数组。

特点：默认初始容量 10、大量核心方法使用 synchronized、线程安全、并发性能通常较低、属于老旧集合

如果设置：`capacityIncrement > 0`

则扩容为：`oldCapacity + capacityIncrement`

否则通常：oldCapacity × 2

### Stack
`Stack extends Vector`

主要操作：push() → 入栈；pop()  → 出栈；peek() → 查看栈顶

现代 Java 实现栈通常推荐：

```java
Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);
stack.pop();
stack.peek();
```

记忆：栈 → 优先 ArrayDeque，不推荐 Stack

# Map
Map 存储 `key-value` 键值对：key 不能重复、value 可以重复

| 实现 | 底层结构 | 顺序 | null | 线程安全 | 典型特点 |
| --- | --- | --- | --- | --- | --- |
| HashMap | 数组 + 链表 + 红黑树 | 不保证 | key/value 允许 | 否 | 通用哈希 Map |
| LinkedHashMap | HashMap + 双向链表 | 插入 / 访问顺序 | 允许 | 否 | 有序、LRU |
| TreeMap | 红黑树 | key 排序 | 取决于比较规则 | 否 | 有序 Map |
| Hashtable | 数组 + 链表 | 不保证 | key/value 都不允许 | 是 | 老旧同步 Map |

## HashMap
### 底层结构
JDK 7 → 数组 + 单向链表；JDK 8 → 数组 + 单向链表 + 红黑树

JDK 8 中数组中的每个位置称为一个桶，哈希冲突后形成链表，冲突严重时可能树化为红黑树。

### 容量与扩容
默认逻辑容量：16、默认负载因子：0.75、默认阈值：16 × 0.75 = 12、容量保持为 2 的幂

注意：`new HashMap<>()`

无参构造时通常不会立即分配长度为 16 的 table，而是在第一次 `put` 时进行实际初始化。

size 超过阈值后通常扩为原容量的：2 倍

### hash 扰动
JDK 8：(h = key.hashCode()) ^ (h >>> 16)

作用：让 hash 高位信息也参与桶下标计算

桶下标：index = (n - 1) & hash;

### 为什么容量是 2 的幂
当：n = 2^k

时，`n - 1` 的低位全部为 `1`，因此：`(n - 1) & hash`

可以高效利用 hash 的低位定位桶。

好处：位运算快、桶分布方便、扩容迁移简单

### 树化条件
重要阈值：TREEIFY_THRESHOLD = 8、UNTREEIFY_THRESHOLD = 6、MIN_TREEIFY_CAPACITY = 64

速记：`8 / 6 / 64`

含义：8 → 树化相关阈值；6 → 扩容拆分等场景中的链表化阈值；64 → table 至少达到该容量后才优先考虑树化

如果 table 容量小于 64，冲突严重时通常优先扩容而不是立即树化。

注意：节点数量达到 6、≠ 一定立即退化为链表

### put 流程
```text
put(key, value)
      ↓
计算 hash
      ↓
定位桶
      ↓
桶为空？
 ├─ 是 → 直接添加
 └─ 否
     ↓
   判断 key
     ↓
链表 / 红黑树查找
     ↓
相同 key → 覆盖 value
不同 key → 添加节点
```

JDK 8 链表采用：尾插

### get 流程
```text
get(key)
   ↓
计算 hash
   ↓
定位桶
   ↓
比较 hash
   ↓
equals()
   ↓
找到目标 key
```

平均复杂度：`O(1)`

严重冲突并树化后：`O(log n)`

### 扩容迁移
容量扩为 2 倍后，每个节点的新位置只有两种：原索引、或、原索引 + oldCap

通过：`hash & oldCap`

判断：hash & oldCap == 0 → 原索引；hash & oldCap != 0 → 原索引 + oldCap

因此 JDK 8 扩容时不需要重新完整计算每个节点的 hash。

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null key` | 允许 1 个 |
| `null value` | 允许多个 |
| key 重复 | 新 value 覆盖旧 value |
| value 重复 | 允许 |
| 顺序 | 不保证 |
| 线程安全 | 否 |

### JDK 7 与 JDK 8
| 对比 | JDK 7 | JDK 8 |
| --- | --- | --- |
| 底层结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 链表插入 | 头插 | 尾插 |
| 扩容迁移 | 重新定位 | `hash & oldCap` 拆分 |
| hash 扰动 | 较复杂 | `h ^ (h >>> 16)` |
| 严重冲突 | 最坏 O(n) | 树化后 O(log n) |

### 源码要点
```text
容量为 2 的幂
→ (n - 1) & hash 快速定位桶

高 16 位与低 16 位异或
→ 高位信息也参与桶定位

8 / 6
→ 形成缓冲区
→ 减少树和链表在临界值附近频繁转换
```

JDK 7 HashMap 在多线程扩容场景中曾存在链表成环风险，因此：HashMap 从来都不是线程安全集合

并发环境一般使用：`ConcurrentHashMap`

具体放到 JUC 学习。

## LinkedHashMap
### 底层结构
`LinkedHashMap extends HashMap`

在 HashMap 基础上额外使用双向链表维护所有节点的全局顺序。

哈希结构：；table[index] → Entry → Entry；顺序结构：；head ⇄ Entry1 ⇄ Entry2 ⇄ Entry3 ⇄ tail

节点除了 HashMap 原有的 `next` 外，还维护：before、after

### 顺序模式
accessOrder = false → 插入顺序，默认；accessOrder = true → 访问顺序

访问顺序模式下，最近访问的节点会被移动到链表尾部，因此可以用于实现 LRU。

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null key` | 允许 |
| `null value` | 允许 |
| key 重复 | 覆盖旧 value |
| value 重复 | 允许 |
| 顺序 | 插入顺序 / 访问顺序 |
| 线程安全 | 否 |

核心记忆：HashMap → 负责快查；双向链表 → 负责顺序；钩子方法 → 负责维护链表

## TreeMap
### 底层结构
TreeMap 基于红黑树，按照 key 的比较结果维护有序结构。

排序方式：实现 Comparable → 自然排序；传入 Comparator → 自定义排序

### 核心操作
put / get / remove → O(log n)

树结构不存在数组扩容，添加、删除节点后通过旋转和变色维持红黑树平衡。

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null key` | 默认自然排序通常不允许；自定义 Comparator 取决于实现 |
| `null value` | 允许 |
| key 重复 | 比较结果为 0 时覆盖 |
| value 重复 | 允许 |
| 顺序 | 按 key 排序 |
| 线程安全 | 否 |

TreeMap 适合：需要 key 排序、范围查找、firstKey / lastKey、floorKey / ceilingKey

## Hashtable
Hashtable 底层为：数组 + 单向链表

默认初始容量：`11`

默认负载因子：`0.75`

扩容通常：newCapacity = oldCapacity * 2 + 1;

注意：`2n + 1`

并不能保证结果一定是素数。

核心方法大量使用 `synchronized`，因此线程安全，但并发性能通常较低。

| 特性 | 结论 |
| --- | --- |
| `null key` | 不允许 |
| `null value` | 不允许 |
| key 重复 | 覆盖旧 value |
| value 重复 | 允许 |
| 顺序 | 不保证 |
| 线程安全 | 是 |

Hashtable 属于 Java 早期同步 Map，现代并发场景通常优先 `ConcurrentHashMap`。

# Set
Set 的核心特点：元素不能重复

| 实现 | 底层实现 | 顺序 | null | 线程安全 | 典型特点 |
| --- | --- | --- | --- | --- | --- |
| HashSet | HashMap | 不保证 | 允许 1 个 | 否 | 查询快、去重 |
| LinkedHashSet | LinkedHashMap | 插入顺序 | 允许 1 个 | 否 | 去重且保持顺序 |
| TreeSet | TreeMap | 元素排序 | 通常不允许 | 否 | 排序、去重 |

## HashSet
### 底层结构
HashSet 基于 HashMap 实现，Set 元素作为 HashMap 的 key，所有 value 使用同一个固定对象：private static final Object PRESENT = new Object();

本质类似：map.put(element, PRESENT);

因此 HashSet 不允许重复，本质来自：HashMap 的 key 不能重复

### 核心操作
```text
add()
→ HashMap.put()

contains()
→ HashMap.containsKey()

remove()
→ HashMap.remove()
```

平均复杂度：`O(1)`

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null` | 允许 1 个 |
| 重复元素 | 不允许 |
| 顺序 | 不保证 |
| 线程安全 | 否 |

HashSet 是否认为两个对象重复，核心依赖：`hashCode() + equals()`

## LinkedHashSet
`LinkedHashSet extends HashSet`

内部利用 LinkedHashMap，实现：HashSet 去重 + LinkedHashMap 顺序维护

| 特性 | 结论 |
| --- | --- |
| `null` | 允许 1 个 |
| 重复元素 | 不允许 |
| 顺序 | 保持插入顺序 |
| 线程安全 | 否 |

添加、删除、查找平均约为：`O(1)`

## TreeSet
### 底层结构
TreeSet 基于 TreeMap 实现，Set 中元素作为 TreeMap 的 key，因此能够自动排序并去重。

### 核心操作
add / contains / remove → O(log n)

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null` | 默认自然排序通常不允许 |
| 重复元素 | 不允许 |
| 顺序 | 按元素排序 |
| 线程安全 | 否 |

TreeSet 判断元素是否重复，核心看：`compare(a, b) == 0`

因此 TreeSet 中的“相同”主要由 `Comparable` 或 `Comparator` 的比较结果决定，而不是单纯依赖 `equals()`。

# Queue 与 Deque
## Queue
Queue 用于队列操作，Deque 是双端队列。

常用 API：

| 操作 | 失败抛异常 | 失败返回特殊值 |
| --- | --- | --- |
| 添加 | `add()` | `offer()` |
| 删除 | `remove()` | `poll()` |
| 查看队头 | `element()` | `peek()` |

记忆：add / remove / element → 失败通常抛异常；offer / poll / peek → 失败返回特殊值

`poll()`、`peek()` 在队列为空时返回 `null`。

因此 `ArrayDeque`、`PriorityQueue` 不允许存储 `null`，可以让 `null` 明确表示“没有元素”。

但不是所有 Queue 实现都禁止 `null`，例如 LinkedList 可以存储 `null`。

## ArrayDeque
### 底层结构
ArrayDeque 基于数组实现双端队列，通过 `head` 和 `tail` 循环利用数组空间。

```text
逻辑顺序：
A → B → C → D

底层数组可能：
[C][D][ ][ ][ ][A][B]
 ↑              ↑
tail           head
```

数组尾部使用完后可以继续利用数组前部空间，因此逻辑上属于：环形数组

### 容量与扩容
空间不足时进行扩容，并重新整理首尾环绕的数据。

不同 JDK 版本在初始容量和具体扩容策略上存在差异，因此源码学习时要明确 JDK 版本。

核心记忆：小容量时增长较快、大容量后增长比例降低、扩容时保证逻辑顺序不变

### 核心操作
addFirst / offerFirst、addLast  / offerLast、pollFirst / pollLast、peekFirst / peekLast

头尾添加和删除通常：`O(1)`

也可以作为栈：

```java
Deque<Integer> stack = new ArrayDeque<>();

stack.push(1);
stack.pop();
stack.peek();
```

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null` | 不允许 |
| 重复元素 | 允许 |
| 顺序 | 按队列逻辑顺序 |
| 线程安全 | 否 |

### 源码要点
环形数组 → 重复利用已经释放的前部空间，减少不必要的数据移动

删除元素后会将对应数组位置置为 `null`，切断强引用，帮助 GC。

不允许 `null`，使：`poll() == null`

可以明确表示：队列为空

## PriorityQueue
### 底层结构
PriorityQueue 基于数组实现二叉堆，默认是小顶堆。

```text
        1
       / \
      3   2
     / \
    7   5
```

底层数组：`[1, 3, 2, 7, 5]`

### 数组与二叉树关系
对于下标 `k`：

```java
parent = (k - 1) >>> 1;
left   = 2 * k + 1;
right  = 2 * k + 2;
```

因此不需要额外保存左右孩子指针。

### 容量与扩容
默认初始容量：`11`

JDK 8 中：oldCapacity < 64 → newCapacity = oldCapacity * 2 + 2；oldCapacity >= 64 → 新容量约增加 50%

### 添加元素
```text
offer / add
    ↓
放到数组末尾
    ↓
siftUp
```

复杂度：`O(log n)`

### 删除堆顶
```text
poll()
  ↓
删除 queue[0]
  ↓
最后一个元素放到根位置
  ↓
siftDown
```

复杂度：`O(log n)`

### 查看堆顶
```java
peek();
```

直接读取 `queue[0]`：`O(1)`

### 查找指定元素
contains(Object)；remove(Object) → O(n)

因为堆只保证父子节点的优先级关系，不保证数组整体有序。

### 基本特性
| 特性 | 结论 |
| --- | --- |
| `null` | 不允许 |
| 重复元素 | 允许 |
| 顺序 | 出队有序，遍历无序 |
| 线程安全 | 否 |

记忆：poll() → 每次获得当前优先级最高的元素；iterator() → 不保证按照优先级顺序遍历

因此：PriorityQueue 是堆有序、不是数组整体有序

### Floyd 建堆
已有一批数据时，从：`(size >>> 1) - 1`

即最后一个非叶子节点开始，向前依次执行 `siftDown`。

建堆整体复杂度：`O(n)`

而不是逐个插入的：`O(n log n)`

# 集合通用机制
## equals 与 hashCode
HashMap 和 HashSet 判断元素位置与相等性时：

```text
hashCode()
   ↓
计算 hash
   ↓
定位桶
   ↓
hash 相同
   ↓
equals()
   ↓
判断是否为同一个 key / 元素
```

必须遵守：equals 相等 → hashCode 必须相等；hashCode 相等 → equals 不一定相等

如果重写 `equals()`，通常也必须重写 `hashCode()`。

### HashMap key 不宜随意修改
例如：

```java
Map<User, String> map = new HashMap<>();

User user = new User(1, "Tom");
map.put(user, "A");
```

如果 `id` 参与 `equals()` 和 `hashCode()` 计算，而之后修改：user.setId(2);

则对象的 hash 结果可能变化，此时：`map.get(user)`

可能无法正常找到之前的 entry。

因此：

> **作为 HashMap key 的对象，放入后最好不要修改会影响 equals/hashCode 的字段。**

常见安全 key：String、Integer、Long、Enum、不可变对象

## Comparable 与 Comparator
### Comparable
由类本身定义自然排序，核心方法：`compareTo()`

例如：

```java
class User implements Comparable<User> {

    private int age;

    @Override
    public int compareTo(User other) {
        return Integer.compare(this.age, other.age);
    }
}
```

### Comparator
排序规则定义在类外部，核心方法：`compare()`

例如：

```java
Comparator<User> comparator =
        Comparator.comparingInt(User::getAge);
```

区别：Comparable → 类自己定义默认排序规则，compareTo()；Comparator → 外部提供排序规则，compare()

常见涉及：TreeMap、TreeSet、PriorityQueue、Collections.sort()、List.sort()

## Iterator 与 fail-fast
很多非并发集合使用：modCount + expectedModCount

实现 fail-fast。

流程：

```text
集合创建 Iterator
      ↓
expectedModCount = modCount
      ↓
开始遍历
      ↓
检查 expectedModCount 与 modCount
      ↓
不一致
      ↓
ConcurrentModificationException
```

例如：

```java
List<Integer> list =
        new ArrayList<>(List.of(1, 2, 3));

for (Integer value : list) {
    list.remove(value);
}
```

可能抛出：`ConcurrentModificationException`

### fail-fast 不等于线程安全
fail-fast 只是：快速检测结构性修改

它不是：锁、同步机制、线程安全保证

因此 `ArrayList`、`HashMap`、`HashSet` 仍然不是线程安全集合。

### Iterator.remove()
遍历期间需要删除当前元素时，可以使用：

```java
Iterator<Integer> iterator = list.iterator();

while (iterator.hasNext()) {
    Integer value = iterator.next();

    if (value == 1) {
        iterator.remove();
    }
}
```

Iterator 会正确维护自己的修改计数。

## Collections 工具类
注意：Collection → 接口体系；Collections → 工具类

常见方法：

```java
Collections.sort(list);
Collections.reverse(list);
Collections.shuffle(list);
Collections.max(list);
Collections.min(list);
```

### synchronizedXxx
可以将普通集合包装成同步集合：

```java
List<Integer> list =
        Collections.synchronizedList(
                new ArrayList<>());
```

```java
Map<String, String> map =
        Collections.synchronizedMap(
                new HashMap<>());
```

高并发场景通常优先考虑 JUC 并发集合。

### unmodifiableXxx
可以创建不可修改视图：

```java
List<Integer> list =
        Collections.unmodifiableList(source);
```

注意：它通常只是禁止通过该视图修改

如果底层 `source` 被修改，视图内容仍可能变化，所以它和真正的不可变集合需要区分。

## 常见复杂度
| 集合 | 查询 | 添加 / 删除 | 顺序特点 |
| --- | --- | --- | --- |
| ArrayList | 索引 O(1) | 尾部均摊 O(1)，中间 O(n) | 插入顺序 |
| LinkedList | O(n) | 头尾 O(1) | 插入顺序 |
| HashMap | 平均 O(1) | 平均 O(1) | 无序 |
| LinkedHashMap | 平均 O(1) | 平均 O(1) | 插入 / 访问顺序 |
| TreeMap | O(log n) | O(log n) | key 有序 |
| HashSet | 平均 O(1) | 平均 O(1) | 无序 |
| TreeSet | O(log n) | O(log n) | 元素有序 |
| ArrayDeque | 头尾 O(1) | 头尾 O(1) | 队列顺序 |
| PriorityQueue | peek O(1) | offer/poll O(log n) | 堆有序 |

实际性能还会受到：CPU Cache、对象数量、内存占用、扩容、GC、数据规模

等影响。

## 集合选型总结
### List
```text
ArrayList
→ 默认首选
→ 查询多
→ 尾部添加多

LinkedList
→ 需要频繁头尾操作时考虑
→ 随机访问差
→ 节点对象额外占内存
```

### Map
```text
HashMap
→ 默认首选

LinkedHashMap
→ 需要保持插入顺序 / 访问顺序

TreeMap
→ 需要 key 排序 / 范围操作

Hashtable
→ 老旧同步 Map
→ 一般不作为现代代码首选
```

### Set
```text
HashSet
→ 普通去重

LinkedHashSet
→ 去重 + 插入顺序

TreeSet
→ 去重 + 排序
```

### Queue / Deque
```text
ArrayDeque
→ 普通队列
→ 双端队列
→ 栈

PriorityQueue
→ 优先级队列
```

# 速记
```text
Collection
├── List
│   ├── ArrayList
│   │   → 动态数组
│   │   → 随机访问 O(1)
│   │   → 扩容约 1.5 倍
│   │
│   ├── LinkedList
│   │   → 双向链表
│   │   → 头尾增删 O(1)
│   │
│   └── Vector / Stack
│       → 老旧同步集合
│       → 栈优先 ArrayDeque
│
├── Set
│   ├── HashSet
│   │   → HashMap
│   ├── LinkedHashSet
│   │   → HashSet + LinkedHashMap
│   └── TreeSet
│       → TreeMap
│       → 排序
│
└── Queue
    ├── ArrayDeque
    │   → 环形数组
    │   → 队列 / 双端队列 / 栈
    └── PriorityQueue
        → 二叉堆
        → offer/poll O(log n)
        → peek O(1)

Map
├── HashMap
│   → 数组 + 链表 + 红黑树
│   → 2 的幂
│   → (n - 1) & hash
│   → 8 / 6 / 64
│
├── LinkedHashMap
│   → HashMap + 双向链表
│   → 插入顺序 / 访问顺序
│   → LRU
│
├── TreeMap
│   → 红黑树
│   → key 有序
│   → O(log n)
│
└── Hashtable
    → synchronized
    → 不允许 null
    → 老旧同步 Map
```

最后记住：

```text
默认 List
→ ArrayList

默认 Map
→ HashMap

默认 Set
→ HashSet

默认 Queue / Stack
→ ArrayDeque

需要优先级
→ PriorityQueue

需要排序
→ TreeMap / TreeSet

需要保持插入顺序
→ LinkedHashMap / LinkedHashSet

并发集合
→ 放到 JUC 学习
```
