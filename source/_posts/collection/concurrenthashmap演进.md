---
title: ConcurrentHashMap：分段锁到CAS+synchronized的演进
date: 2023-01-25 12:12:04
tags:
  - Java进阶
  - 集合框架
categories: 学习
keywords: ConcurrentHashMap,分段锁,CAS,synchronized,JDK7,JDK8
description: 从JDK7到JDK8，深入理解ConcurrentHashMap的实现原理与性能优化
cover:
---

## 前言

ConcurrentHashMap 是 Java 并发编程中使用最广泛的线程安全哈希表。从 JDK7 的 Segment 分段锁到 JDK8 的 CAS + synchronized，架构发生了巨大变化。本文深入解析这个演进过程。

## JDK7：Segment分段锁架构

### 核心结构

```
┌──────────────────────────────────────────────────────────────┐
│                 ConcurrentHashMap (JDK7)                      │
│                                                               │
│  Segment[]数组                                                │
│  ┌────────┬────────┬────────┬────────┬──────────────────┐  │
│  │Seg[0]  │Seg[1]  │Seg[2]  │Seg[3]  │      ...        │  │
│  │HashE[] │HashE[] │HashE[] │HashE[] │                 │  │
│  │(加锁)  │(加锁)  │(加锁)  │(加锁)  │                 │  │
│  └────────┴────────┴────────┴────────┴──────────────────┘  │
│                                                               │
│  每个Segment独立加锁，并发度 = Segment数量                      │
└──────────────────────────────────────────────────────────────┘
```

### Segment继承ReentrantLock

```java
static final class Segment<K,V> extends ReentrantLock {

    transient volatile int count;           // 元素数量
    transient int modCount;                 // 修改计数
    transient int threshold;               // 扩容阈值
    transient Node<K,V>[] table;           // 链表数组

    V put(K key, int hash, V value, boolean onlyIfAbsent) {
        // 加锁，保证线程安全
        lock();
        try {
            // 遍历链表，查找或插入
            // ...
        } finally {
            unlock();
        }
    }
}
```

### get操作无需加锁

```java
public V get(Object key) {
    int hash = hash(key);

    // 定位Segment
    Segment<K,V> s = segmentFor(hash);

    // 无锁读取
    return s.get(key, hash);
}

// Segment内部
V get(Object key, int hash) {
    if (count != 0) {  // 可见性保证
        Node<K,V> e = getFirst(hash);
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                return e.value;
            }
            e = e.next;
        }
    }
    return null;
}
```

### put操作加锁

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();  // 加锁
    try {
        int c = count;

        // 扩容检查
        if (c++ > threshold) {
            rehash();
        }

        // 定位链表
        Node<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        Node<K,V> first = tab[index];

        // 遍历链表
        Node<K,V> e = first;
        while (e != null) {
            if (e.hash == hash && key.equals(e.key)) {
                V oldValue = e.value;
                if (!onlyIfAbsent) {
                    e.value = value;
                }
                return oldValue;
            }
            e = e.next;
        }

        // 头插法
        modCount++;
        tab[index] = new Node<>(hash, key, value, first);
        count = c;
        return null;
    } finally {
        unlock();  // 解锁
    }
}
```

---

## JDK8：CAS + synchronized优化

### 核心结构变化

废弃了 Segment，使用数组 + 链表 + 红黑树：

```
┌─────────────────────────────────────────────────────┐
│           ConcurrentHashMap (JDK8+)                   │
│                                                      │
│  Node<K,V>[] table                                   │
│  ┌──────┬──────┬──────┬──────┬─────────────┐     │
│  │ Node │ null │ Node │Tree  │    ...     │     │
│  │(链表)│      │(链表)│(红黑树)│            │     │
│  └──────┴──────┴──────┴──────┴─────────────┘     │
│                                                      │
│  并发度 = table数组长度                               │
│  锁粒度 = 单个Node节点（synchronized）               │
│  读操作 = 大部分无锁（CAS）                         │
└─────────────────────────────────────────────────────┘
```

### 核心属性

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V> {

    // 数组
    transient volatile Node<K, V>[] table;

    // 下一个数组（用于扩容）
    private transient volatile Node<K, V>[] nextTable;

    // 计数（baseCount + CounterCell）
    private transient volatile long baseCount;

    // 控制变量：负数=扩容中，0=未初始化，正数=table大小
    private transient volatile int sizeCtl;

    // 扩容戳
    private transient volatile int transferIndex;
}
```

### Node节点

```java
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    volatile V value;
    volatile Node<K, V> next;

    Node(int hash, K key, V value, Node<K, V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
}

// 树节点
static final class TreeNode<K, V> extends Node<K, V> {
    TreeNode<K, V> left;
    TreeNode<K, V> right;
    TreeNode<K, V> parent;
    boolean red;
}
```

### putVal核心逻辑

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw NullPointerException();

    int hash = spread(key.hashCode());
    int binCount = 0;

    for (Node<K, V>[] tab = table; ; ) {
        Node<K, V> f;
        int n, i, fh;

        // 1. 数组未初始化，初始化数组
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();
        }
        // 2. 位置为空，CAS直接插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<>(hash, key, value, null)))
                break;
        }
        // 3. 正在扩容，帮助扩容
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
        }
        // 4. 正常插入/更新，synchronized锁当前Node
        else {
            V oldVal = null;

            synchronized (f) {  // 锁住当前槽
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {  // 链表
                        binCount = 1;
                        for (Node<K, V> e = f; ; ++binCount) {
                            if (e.hash == hash && key.equals(e.key)) {
                                oldVal = e.value;
                                if (!onlyIfAbsent) {
                                    e.value = value;
                                }
                                break;
                            }
                            Node<K, V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {  // 红黑树
                        binCount = 2;
                        TreeNode<K, V> p = ((TreeBin<K, V>) f).putTreeVal(hash, key, value);
                        if (!onlyIfAbsent && p != null) {
                            p.value = value;
                        }
                    }
                }
            }

            if (binCount != 0) {
                // 链表过长，转红黑树
                if (binCount >= TREEIFY_THRESHOLD) {
                    treeifyBin(tab, i);
                }
                if (oldVal != null) {
                    return oldVal;
                }
                break;
            }
        }
    }
    // 更新计数
    addCount(1L, binCount);
    return null;
}
```

### CAS操作

```java
// 获取数组第i个位置的值
static final <K, V> Node<K, V> tabAt(Node<K, V>[] tab, int i) {
    return U.getObjectAcquire(tab, ((long) i << ASHIFT) + ABASE);
}

// CAS设置数组第i个位置的值
static final <K, V> boolean casTabAt(Node<K, V>[] tab, int i, Node<K, V> c, Node<K, V> v) {
    return U.compareAndSetObject(tab, ((long) i << ASHIFT) + ABASE, c, v);
}
```

### initTable初始化

```java
private final Node<K, V>[] initTable() {
    Node<K, V>[] tab;
    int sc;

    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0 表示其他线程正在初始化
        if ((sc = sizeCtl) < 0) {
            Thread.yield();
        }
        // CAS设置sizeCtl为-1，表示当前线程正在初始化
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);  // threshold = 0.75 * n
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

---

## JDK8的重大改进

### 对比表

| 特性 | JDK7 Segment | JDK8 CAS+synchronized |
|------|-------------|----------------------|
| 结构 | Segment数组 | 数组+链表+红黑树 |
| 锁粒度 | Segment（多个HashEntry） | 单个Node |
| 并发度 | Segment数量（默认16） | 数组长度（可很大） |
| 读操作 | 无锁 | 无锁 |
| 写操作 | 锁Segment | synchronized锁Node |
| 扩容 | Segment独立扩容 | 整体扩容，多线程协助 |
| 链表转红黑树 | 无 | 有 |

### JDK8的优势

1. **锁粒度更细**：只锁当前节点，而非整个Segment
2. **并发度更高**：数组长度可很大，锁竞争更小
3. **读操作无锁**：CAS保证原子性，无需加锁
4. **红黑树优化**：链表过长时转化为红黑树，O(log n)查找
5. **扩容优化**：支持多线程同时协助扩容

---

## size()方法的演进

### JDK7：遍历累加

```java
public int size() {
    Segment<K, V>[] segments = this.segments;
    long sum = 0;
    for (int i = 0; i < segments.length; ++i) {
        sum += segments[i].count;  // 累加每个Segment的count
    }
    return (sum >>> 32) != 0 ? Integer.MAX_VALUE : (int) sum;
}
```

### JDK8：baseCount + CounterCell

```java
// put/remove时调用
private final void addCount(long x, int binCount) {
    CounterCell[] as;
    long b, s;

    // 快速路径：CAS更新baseCount
    if ((as = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 慢速路径：使用CounterCell数组分散更新热点
        CounterCell a;
        long v;
        int m;
        boolean uncolored = U.compareAndSetInt(this, CELLSBUSY, 0, 1);
        if (uncolored) {
            try {
                if ((as = counterCells) == null) {
                    CounterCell[] rs = new CounterCell[2];
                    rs[1] = new CounterCell(2, x);
                    counterCells = rs;
                }
            } finally {
                U.compareAndSetInt(this, CELLSBUSY, 1, 0);
            }
        }
    }
}

// 计算size
public int size() {
    long n = sumCount();
    return (n < 0) ? 0 : (n >= Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int) n;
}

final long sumCount() {
    CounterCell[] as = counterCells;
    long sum = baseCount;
    if (as != null) {
        for (CounterCell a : as) {
            sum += a.value;
        }
    }
    return sum;
}
```

---

## 扩容机制

### JDK8的并发扩容

```java
private final void transfer(Node<K, V>[] tab, Node<K, V>[] nextTab) {
    int n = tab.length;
    int stride = (n >>> 3) / NCPU;  // 每线程处理的槽数
    if (stride < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;

    // 分配新的数组
    if (nextTab == null) {
        nextTab = new Node<>(n << 1);
        nextTable = nextTab;
        transferIndex = n;
    }

    int nextn = nextTab.length;

    // 遍历每个槽，协助迁移
    for (int i = 0, bound = 0; ; ) {
        while (i >= 0 && i < bound && (f = tabAt(tab, i)) != null) {
            // 迁移当前槽的节点
            // 使用CAS保证线程安全
        }
    }
}
```

### 多线程协助扩容

```java
final Node<K, V>[] helpTransfer(Node<K, V>[] tab, Node<K, V> f) {
    Node<K, V>[] nextTab;
    int sc;

    if (tab != null && f instanceof ForwardingNode &&
        (nextTab = ((ForwardingNode<K, V>) f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0) {
                break;
            }
            // CAS增加并发协助线程数
            if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);  // 协助扩容
                break;
            }
        }
    }
    return nextTab;
}
```

---

## 常用方法对比

| 方法 | JDK7 | JDK8 |
|------|------|------|
| get | Segment.get，无锁 | tabAt，无锁 |
| put | Segment.put，加锁Segment | putVal，synchronized锁Node |
| remove | Segment.remove，加锁Segment | removeNode，synchronized锁Node |
| size | 遍历Segment累加 | baseCount+CounterCell |
| 扩容 | 各Segment独立扩容 | 整体扩容，多线程协助 |

---

## 面试高频问题

**Q1：ConcurrentHashMap的并发度为什么更高？**

> 答：JDK7的Segment分段锁，每个Segment独立加锁，并发度取决于Segment数量。JDK8的CAS+synchronized，锁粒度细化到单个Node，读操作无锁，并发度等于数组长度。

**Q2：JDK8为什么用synchronized而不是ReentrantLock？**

> 答：JVM对synchronized做了大量优化（偏向锁、轻量级锁、自旋优化），而且锁粒度更细（锁Node而非Segment），性能反而更好。

**Q3：get操作需要加锁吗？**

> 答：JDK8的get操作无需加锁，因为value是volatile保证可见性，Node的next也是volatile。

**Q4：扩容时其他线程可以put吗？**

> 答：JDK8支持并发扩容，其他线程检测到ForwardingNode后会协助扩容，同时允许正常的put操作。

**Q5：为什么size()不准确？**

> 答：size()统计时可能正在扩容，而且size()是近似值，不是精确值。精确计数需要加全局锁。

---

## 总结

ConcurrentHashMap是并发哈希表的最佳选择：
- **JDK7**：Segment分段锁，并发度固定
- **JDK8**：CAS+synchronized，锁粒度更细
- **读优化**：无锁读取，性能高
- **写优化**：synchronized锁Node，粒度细
- **红黑树**：链表过长时转化
- **扩容优化**：多线程协助扩容
