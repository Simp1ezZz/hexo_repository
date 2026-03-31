---
title: HashMap深度解析：哈希冲突、扩容与红黑树
date: 2026-04-11 10:00:00
tags:
  - Java进阶
  - 集合框架
categories: 学习
keywords: HashMap,哈希冲突,扩容,红黑树,底层原理
description: 深入理解HashMap的底层实现，哈希算法、冲突解决、扩容机制与红黑树优化
cover:
---

## 前言

HashMap 是 Java 开发中最常用的数据结构之一，但你是否真正理解它的底层实现？JDK8 之后 HashMap 发生了什么变化？为什么 HashMap 不是线程安全的？本文深入解析 HashMap 的每一个细节。

## HashMap的基本使用

### 常用API

```java
Map<String, Integer> map = new HashMap<>();

// 插入
map.put("Alice", 25);
map.put("Bob", 30);

// 读取
Integer age = map.get("Alice");  // 25
Integer unknown = map.get("Unknown");  // null

// 删除
map.remove("Bob");

// 遍历
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}

// 其他常用方法
map.containsKey("Alice");  // true
map.containsValue(25);   // true
map.size();                // 1
map.isEmpty();             // false
map.keySet();              // 返回所有key的Set
map.values();              // 返回所有value的Collection
```

### 重要特性

| 特性 | 说明 |
|------|------|
| 线程不安全 | 不保证并发安全，多线程环境用ConcurrentHashMap |
| 允许null | key和value都可以为null |
| 无序 | 不保证顺序，取决于hash后的分布 |
| 扩容 | 达到负载因子阈值时自动扩容 |
| JDK8+红黑树 | 链表过长时转化为红黑树 |

---

## 底层数据结构演进

### JDK7：数组 + 链表

```
┌─────────────────────────────────────────────────────┐
│                    HashMap (JDK7)                    │
│                                                      │
│  Node<K,V>[] table                                   │
│  ┌──────┬──────┬──────┬──────┬─────────────┐     │
│  │Node  │ null │ Node │Tree  │    ...     │     │
│  │(链表)│      │(链表)│(红黑树)│            │     │
│  └──────┴──────┴──────┴──────┴─────────────┘     │
│      │          │                    │               │
│      ▼          ▼                    ▼               │
│   [key,val]  [key,val]          [key,val]           │
│   [key,val]                                   │
│                                                      │
│  负载因子loadFactor = 0.75                          │
│  扩容阈值threshold = capacity * loadFactor          │
└─────────────────────────────────────────────────────┘
```

### JDK8：数组 + 链表 + 红黑树

当链表长度超过 8 且数组长度 >= 64 时，链表转化为红黑树。

```
┌─────────────────────────────────────────────────────┐
│                    HashMap (JDK8+)                   │
│                                                      │
│  Node<K,V>[] table                                   │
│  ┌──────┬──────┬──────┬──────┬─────────────┐     │
│  │Node  │ null │ Node │Tree  │    ...     │     │
│  │(链表)│      │(链表)│(红黑树)│            │     │
│  └──────┴──────┴──────┴──────┴─────────────┘     │
│                                                      │
│  链表长度 > 8 && 数组长度 >= 64 → 红黑树            │
│  红黑树节点数 < 6 → 链表                            │
└─────────────────────────────────────────────────────┘
```

---

## 哈希算法

### hashCode的作用

HashMap使用key的hashCode()来决定存储位置。

```java
// HashMap的put方法
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 计算hash值
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

### 扰动函数设计

```java
// hash = key.hashCode() ^ (key.hashCode() >>> 16)
// 让高位和低位混合，增加随机性

// 示例
"hello".hashCode() = 0x6E65706F  // 二进制: 0110 1110 0110 0001...
        h >>> 16 = 0x00006E65  // 右移16位，高位变低位
        hash = h ^ (h >>> 16)  // 混合，得到更分散的hash
```

**为什么需要扰动？**
- table数组长度通常远小于hashCode范围
- 只取低几位作为索引会导致碰撞
- 扰动后让高位影响低位，减少碰撞

### index计算

```java
// 计算在table数组中的索引
int index = (n - 1) & hash;

// 等价于 hash % n，但位运算更快
// n 是数组长度，必须是 2 的幂次方

// 示例：n=16，(16-1) & hash = hash % 16
```

---

## 哈希冲突解决方法

### 1. 开放地址法

探查空位置：
- 线性探查：h(key) + 1, h(key) + 2, ...
- 平方探查：h(key) + 1², h(key) + 2², ...
- 双重哈希：h1(key) + i * h2(key)

**HashMap 不使用开放地址法。**

### 2. 链地址法（JDK7/JDK8使用）

将冲突的元素链接成链表或红黑树。

```
索引位置2：
┌──────────────────────────────────────┐
│                  index = 2            │
│                      │                │
│                      ▼                │
│                 ┌─────────┐           │
│                 │ Node 1  │           │
│                 │ key: A  │           │
│                 │ next: ──┼──→ [Node 2] → [Node 3] → null
│                 └─────────┘           │
└──────────────────────────────────────┘
```

### JDK7头插法

```java
// JDK7：newNode插入链表头部
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

> **问题**：多线程扩容时，头插法可能导致环形链表，形成死循环。

### JDK8尾插法

```java
// JDK8：newNode插入链表尾部
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    // JDK8改为尾插法，避免环形链表问题
}
```

---

## 扩容机制

### 扩容时机

当 `size > threshold`（capacity * loadFactor）时触发扩容。

```java
// threshold = capacity * loadFactor
// loadFactor 默认 0.75

// 例如：初始容量16，threshold=12
// 当put第13个元素时，触发扩容
```

### 扩容过程

```java
void resize(int newCapacity) {
    Node<K,V>[] oldTable = table;
    int oldCap = oldTable.length;
    int newCap = oldCap << 1;  // 容量翻倍

    Node<K,V>[] newTable = new Node[newCap];
    table = newTable;

    // 重新计算每个元素的索引位置
    transfer(newTable);
}
```

### 扩容后索引重新计算

```
原容量: 16
新容量: 32

元素的新索引 = (newCap - 1) & hash

key1: hash = 5
  旧: (16-1) & 5 = 15 & 5 = 5
  新: (32-1) & 5 = 31 & 5 = 5  // 不变

key2: hash = 17
  旧: (16-1) & 17 = 15 & 17 = 1
  新: (32-1) & 17 = 31 & 17 = 17  // 变化：17 = 1 + 16
```

**规律**：容量翻倍后，原索引位置上的元素，要么留在原位置，要么移动 oldCap 个位置。

### JDK7的扩容死循环问题

```java
// JDK7 扩容代码（简化）
void transfer(Node<K,V>[] newTable) {
    Entry<K,V>[] src = table;
    int newCapacity = newTable.length;

    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                // 重新计算索引
                int i = indexFor(e.hash, newCapacity);
                // 头插法
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}
```

**多线程死循环原因**：
1. 线程A和线程B同时扩容
2. 头插法导致链表反转
3. 可能形成环形链表
4. get操作遍历环形链表导致死循环

> **JDK8修复**：JDK8使用尾插法，不会形成环形链表，但HashMap本身仍非线程安全。

---

## 红黑树优化（JDK8+）

### 为什么需要红黑树？

链表查询时间复杂度 O(n)，红黑树 O(log n)。

```
链表长度: 8
  最坏情况: 8次比较

红黑树深度: log2(8) = 3
  最坏情况: 3次比较
```

### 转化条件

```java
// 链表转红黑树的条件
static final int TREEIFY_THRESHOLD = 8;      // 链表长度 >= 8
static final int MIN_TREEIFY_CAPACITY = 64;   // 数组容量 >= 64

// 红黑树转回链表的条件
static final int UNTREEIFY_THRESHOLD = 6;     // 红黑树节点数 <= 6
```

> 为什么用8而不是7？防止频繁转换（8→6→8→6...）。

### 红黑树节点结构

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    boolean red;
    // ...
}
```

---

## put流程全解析

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab;
    Node<K,V> p;
    int n, i;

    // 1. 初始化或扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

    // 2. 位置为空，直接插入
    if ((p = tabAt(tab, i = (n - 1) & hash)) == null)
        casTabAt(tab, i, null, new Node<>(hash, key, value, null));

    // 3. 位置已有元素
    else {
        Node<K,V> e;
        K k;

        // 3.1 key相同，更新值
        if (p.hash == hash && ((k = p.key) == key || key.equals(k)))
            e = p;

        // 3.2 红黑树节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>) p).putTreeVal(this, tab, hash, key, value);

        // 3.3 链表
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = new Node<>(hash, key, value, null);

                    // 链表过长，转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }

                // key已存在，更新
                if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                    break;

                p = e;
            }
        }

        // 更新已有节点的值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }

    // 4. 修改计数，扩容检查
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

---

## get流程全解析

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab;
    Node<K,V> first, e;
    int n; K k;

    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tabAt(tab, (n - 1) & hash)) != null) {

        // 第一个节点命中
        if (first.hash == hash && ((k = first.key) == key || key.equals(k)))
            return first;

        // 冲突解决
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>) first).find(hash, key, null);
            do {
                if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

---

## 线程安全性

### HashMap为什么不是线程安全？

| 操作 | 问题 |
|------|------|
| put | 可能丢失数据（后一个覆盖前一个） |
| resize | 可能形成环形链表，死循环 |
| get | 可能读到不一致的数据 |

### 多线程安全替代方案

```java
// 方案1：ConcurrentHashMap
ConcurrentHashMap<String, Integer> map1 = new ConcurrentHashMap<>();
// 分段锁/CAS，线程安全

// 方案2：Collections.synchronizedMap
Map<String, Integer> map2 = Collections.synchronizedMap(new HashMap<>());
// synchronized包装，线程安全但性能差

// 方案3：Hashtable（不推荐）
Hashtable<String, Integer> map3 = new Hashtable<>();
// 全表锁，不推荐

// 方案4：ConcurrentSkipListMap（需要有序）
ConcurrentSkipListMap<String, Integer> map4 = new ConcurrentSkipListMap<>();
// 有序，线程安全
```

---

## 常见面试题

### Q1：HashMap的put流程？

> 答：1. 计算hash扰动；2. 数组为空则初始化/扩容；3. 索引位置空则直接插入；4. 索引位置有元素，key相同则更新；是红黑树则树插入；是链表则遍历，尾插法，链表过长转红黑树；5. 更新size，扩容检查。

### Q2：JDK8为什么引入红黑树？

> 答：链表查找O(n)，红黑树O(log n)。当链表长度过长时（如hash攻击），查找性能急剧下降。红黑树保证最坏情况下的查找性能。

### Q3：HashMap什么时候扩容？

> 答：当size > capacity * loadFactor（默认16 * 0.75 = 12）时扩容。扩容后容量翻倍，重新计算每个元素的索引位置。

### Q4：为什么HashMap容量是2的幂次？

> 答：计算索引时使用`(n-1) & hash`而非`hash % n`，位运算更快。2的幂次减1的二进制全是1，保证散列均匀。

### Q5：JDK7扩容死循环的原因？

> 答：JDK7使用头插法扩容，多线程环境下链表反转可能导致A→B→A环形链表。get遍历时死循环。JDK8改用尾插法解决了这个问题。

---

## 总结

HashMap是Java集合框架的核心：
- **哈希算法**：扰动函数让hash更分散
- **冲突解决**：链地址法（JDK7头插/JDK8尾插）
- **扩容机制**：容量翻倍，索引要么不变要么+oldCap
- **红黑树**：链表过长时转化，O(log n)查找
- **线程安全**：非线程安全，并发用ConcurrentHashMap
