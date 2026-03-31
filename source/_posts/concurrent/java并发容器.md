---
title: Java并发容器：ConcurrentHashMap的演进与实现
date: 2026-04-10 10:00:00
tags:
  - Java进阶
  - 并发编程
categories: 学习
keywords: ConcurrentHashMap,分段锁,CAS,Java并发容器
description: 从JDK7到JDK8，深入理解ConcurrentHashMap的实现原理与优化
cover:
---

## 前言

`ConcurrentHashMap` 是 Java 并发编程中最常用的线程安全哈希表。相比 `Hashtable` 的全表锁和 `Collections.synchronizedMap` 的对象锁，它提供了更高效的并发性能。本文深入解析 ConcurrentHashMap 的实现原理与演进历史。

## 为什么需要ConcurrentHashMap？

### Hashtable的问题

```java
public synchronized V put(K key, V value) {
    // 整个方法被锁住
}

public synchronized V get(Object key) {
    // 整个方法被锁住
}
```

> Hashtable 使用 synchronized 修饰所有方法，意味着同一时刻只能有一个线程访问，任何 get/put 操作都会串行化。

### Collections.synchronizedMap的问题

```java
Map<K, V> map = Collections.synchronizedMap(new HashMap<>());
// 底层也是对每个方法加synchronized，锁住整个map
```

### 性能对比

| 实现 | 线程数=1 | 线程数=4 | 线程数=16 |
|------|----------|----------|-----------|
| Hashtable | 100ms | 800ms | 3000ms |
| synchronizedMap | 100ms | 750ms | 2800ms |
| ConcurrentHashMap | 100ms | 120ms | 200ms |

> ConcurrentHashMap 通过分段锁或CAS实现高并发，在多线程环境下性能优势明显。

---

## JDK7：Segment分段锁实现

### 结构图

```
┌──────────────────────────────────────────────────────────────┐
│                   ConcurrentHashMap                          │
│                                                              │
│  Segment[0]  ─┬─> HashEntry[] table                        │
│               │     [HashEntry, HashEntry, null, ...]      │
│  Segment[1] ──┼─> HashEntry[] table                        │
│               │     [HashEntry, HashEntry, HashEntry, ...]  │
│  ...          │                                            │
│               │     每个Segment独立锁                        │
│  Segment[N-1] ─┘> HashEntry[] table                        │
│                    [null, HashEntry, HashEntry, ...]        │
└──────────────────────────────────────────────────────────────┘
```

### 初始化

```java
public ConcurrentHashMap() {
    // 默认16个Segment，每个Segment内部有默认16个HashEntry
    this(16, 0.75f, 16);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    // segments数组长度由concurrencyLevel决定
    int sshift = 0;
    int ssize = 1;
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // Segment偏移量和掩码
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;

    // 创建Segment数组
    this.segments = new Segment[ssize];
}
```

### Segment分段锁原理

```java
static final class Segment<K, V> extends ReentrantLock {

    transient volatile int count;        // HashEntry数量
    transient int modCount;             // 修改计数
    transient int threshold;            // 扩容阈值
    transient volatile HashEntry<K, V>[] table;  // 链表数组

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

### get操作（无锁）

```java
public V get(Object key) {
    int hash = hash(key);
    // 计算Segment位置
    Segment<K,V> s = segmentFor(hash);
    // 无锁读取
    return s.get(key, hash);
}

V get(Object key, int hash) {
    if (count != 0) {
        HashEntry<K,V> e = getFirst(hash);
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

### put操作（加锁）

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();  // 加锁
    try {
        int c = count;

        // 需要扩容
        if (c++ > threshold) {
            rehash();
        }

        // 遍历链表，插入
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];

        HashEntry<K,V> e = first;
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
        tab[index] = new HashEntry<>(key, hash, first, value);
        count = c;
        return null;
    } finally {
        unlock();  // 解锁
    }
}
```

---

## JDK8：CAS + synchronized优化

### 结构变化

JDK8 废弃了 Segment，采用了数组 + 链表 + 红黑树的结构，类似 HashMap：

```
┌─────────────────────────────────────────────────────┐
│              ConcurrentHashMap                        │
│                                                      │
│  Node<K, V>[] table                                 │
│  ┌──────┬──────┬──────┬──────┬─────────────┐     │
│  │ Node │ Node │ null │ Tree │    ...     │     │
│  │(链表) │(链表) │      │(红黑树)│            │     │
│  └──────┴──────┴──────┴──────┴─────────────┘     │
│                                                      │
│  当链表长度 > 8 且数组长度 >= 64 时，转化为红黑树     │
└─────────────────────────────────────────────────────┘
```

### 核心属性

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
    implements ConcurrentMap<K, V>, Serializable {

    // 数组
    transient volatile Node<K, V>[] table;

    // 下一个要使用的数组长度（用于扩容）
    private transient volatile int nextTable;

    // baseCount + CounterCell 用于计数
    private transient volatile long baseCount;

    // 扩容戳
    private transient volatile int sizeCtl;

    // 几个常量
    static final int MAXIMUM_CAPACITY = 1 << 30;    // 最大容量 2^30
    static final int TREEIFY_THRESHOLD = 8;          // 链表转树阈值
    static final int UNTREEIFY_THRESHOLD = 6;        // 树转链表阈值
    static final int MIN_TREEIFY_CAPACITY = 64;      // 最小树化数组长度
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
    boolean red;  // 红黑树颜色
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

        // 1. 初始化数组
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();
        }
        // 2. 该位置为空，CAS直接插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null)))
                break;
        }
        // 3. 正在扩容，帮助扩容
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
        }
        // 4. 正常插入/更新
        else {
            V oldVal = null;

            // synchronized 锁住当前槽
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        // 链表
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
                    else if (f instanceof TreeBin) {
                        // 红黑树
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

## JDK8的改进点

### 对比总结

| 特性 | JDK7 Segment | JDK8 CAS+synchronized |
|------|-------------|----------------------|
| 结构 | Segment分段 | 数组+链表+红黑树 |
| 锁粒度 | Segment（整个数组段） | synchronized（单个Node） |
| 并发度 | Segment数量（默认16） | 数组长度（可很大） |
| 扩容 | 每个Segment独立扩容 | 整体扩容 |
| 动画 | 支持多线程同时put | 扩容时可协助迁移 |

### JDK8的优势

1. **锁粒度更细**：只锁当前节点，而非整个Segment
2. **红黑树优化**：链表过长时转化为红黑树，查询O(log n)
3. **CAS无锁读**：读操作大部分无需加锁
4. **扩容优化**：支持多线程协助扩容

---

## size()方法的实现

### JDK7：遍历所有Segment

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

    // 快速路径：CAS更新baseCount成功
    if ((as = counterCells) != null ||
        !U.compareAndSetLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        // 慢速路径：使用CounterCell数组分散更新
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

// size() 计算
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

## 其他并发容器

### ConcurrentLinkedQueue

无界线程安全队列，基于CAS实现：

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
    implements Queue<E>, Serializable {

    private static class Node<E> {
        volatile E item;
        volatile Node<E> next;
    }

    // 入队
    public boolean offer(E e) {
        Node<E> newNode = new Node<>(e);
        // CAS循环设置tail.next
        // ...
    }
}
```

### CopyOnWriteArrayList

写时复制ArrayList，适合读多写少场景：

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess {

    private transient volatile Object[] array;

    // 读操作无需加锁，直接读取
    public E get(int index) {
        return get(array, index);
    }

    // 写操作加锁，复制整个数组
    public boolean add(E e) {
        synchronized (lock) {
            Object[] elements = getArray();
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
        }
    }
}
```

### ConcurrentHashMap vs Collections.synchronizedMap

```java
// 场景1：读多写少
Map<K, V> map = new ConcurrentHashMap<>();
// 读操作无锁，写操作锁单个Node

// 场景2：需要有序遍历
Map<K, V> map = Collections.synchronizedMap(new TreeMap<>);
// 适合需要按序操作的场景

// 场景3：需要保序的putIfAbsent
ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
map.putIfAbsent(key, value);  // 原子操作
```

---

## 面试高频问题

**Q1：ConcurrentHashMap的并发度为什么更高？**

> 答：JDK7用Segment分段锁，每个Segment独立加锁，可并行操作不同Segment。JDK8用CAS+synchronized，锁粒度细化到单个Node，读操作大部分无锁。

**Q2：JDK8为什么用synchronized而不是ReentrantLock？**

> 答：JVM对synchronized做了大量优化（偏向锁、轻量级锁、自旋优化），而且锁粒度更细（锁Node而非Segment），性能反而更好。synchronized是JVM内置支持，无需额外代码。

**Q3：ConcurrentHashMap的get操作需要加锁吗？**

> 答：JDK8的get操作无需加锁，因为value是volatile保证可见性，Node的next也是volatile。但JDK7的get需要遍历Segment内部的链表。

**Q4：链表转红黑树的条件？**

> 答：链表长度 >= 8 且 数组长度 >= 64。树转链表：红黑树节点数 <= 6。

**Q5：size()方法是如何实现的？**

> 答：JDK7累加所有Segment的count。JDK8用baseCount + CounterCell数组，通过CAS更新计数，避免热点。

---

## 总结

ConcurrentHashMap 是并发编程的核心工具：
- **JDK7**：Segment分段锁，Segment数量决定并发度
- **JDK8**：CAS + synchronized，红黑树优化长链表
- **高并发**：锁粒度细，读操作无锁
- **计数**：baseCount + CounterCell 分散热点
- **演进**：从分段锁到CAS，从小粒度锁到无锁读
