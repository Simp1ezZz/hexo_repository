---
title: ArrayList与LinkedList：底层结构与性能对比
date: 2026-04-13 10:00:00
tags:
  - Java进阶
  - 集合框架
categories: 学习
keywords: ArrayList,LinkedList,动态数组,双向链表,性能对比
description: 深入理解ArrayList和LinkedList的底层实现，分析各自的性能特点与适用场景
cover:
---

## 前言

ArrayList 和 LinkedList 是 Java 中最常用的两种 List 实现。它们都是有序集合，但在底层结构和使用场景上有着本质区别。本文深入解析它们的实现原理和性能特点。

## 整体架构对比

```
┌─────────────────────────────────────────────────────────────────┐
│                         List 接口                                │
│         ┌───────────────────────────────────────┐              │
│         │        Collection 接口                   │              │
│         │         Iterable 接口                   │              │
│         └───────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                    △                              △
                    │                              │
         ┌─────────────────────────────────────┐  ┌─────────────────────┐
         │           ArrayList                  │  │       LinkedList     │
         │         实现 RandomAccess            │  │    实现 Deque        │
         │         动态数组                     │  │    双向链表          │
         └─────────────────────────────────────┘  └─────────────────────┘
```

---

## ArrayList：动态数组

### 底层结构

```java
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, Serializable {

    // 存储元素的数组
    transient Object[] elementData;

    // 实际元素数量
    private int size;

    // 默认容量
    private static final int DEFAULT_CAPACITY = 10;

    // 最大数组长度
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
}
```

### 内存布局

```
ArrayList 内部结构：
┌──────────────────────────────────────────────────────────────────┐
│ elementData (Object[])                                             │
│                                                                  │
│  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┬────────┐  │
│  │ [0]  │ [1]  │ [2]  │ [3]  │ [4]  │ [5]  │ [6]  │  ...   │  │
│  │  A    │  B    │  C    │  D    │  E    │ null │ null │        │  │
│  └──────┴──────┴──────┴──────┴──────┴──────┴──────┴────────┘  │
│   size = 5                                                        │
└──────────────────────────────────────────────────────────────────┘
```

### 构造函数

```java
// 默认容量为10的空数组
public ArrayList() {
    this.elementData = EMPTY_ELEMENTDATA;
}

// 指定初始容量
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity");
    }
}

// 从Collection构建
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    size = elementData.length;
    if (elementData.getClass() != Object[].class) {
        elementData = Arrays.copyOf(elementData, size, Object[].class);
    }
}
```

---

## LinkedList：双向链表

### 底层结构

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, Serializable {

    // 头节点
    transient Node<E> first;

    // 尾节点
    transient Node<E> last;

    // 元素数量
    private int size = 0;
}

// 内部节点类
private static class Node<E> {
    E item;           // 元素值
    Node<E> next;      // 后继节点
    Node<E> prev;      // 前驱节点

    Node(Node<E> prev, E item, Node<E> next) {
        this.item = item;
        this.next = next;
        this.prev = prev;
    }
}
```

### 内存布局

```
LinkedList 内部结构：

head                                                    tail
 │                                                        │
 ▼                                                        ▼
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  Node   │───▶│  Node   │───▶│  Node   │───▶│  Node   │
│ prev=null│    │prev=Node│    │prev=Node│    │prev=Node│
│ item=A  │    │ item=B  │    │ item=C  │    │ item=D  │
│ next= ──┼──▶│ next= ──┼──▶│ next= ──┼──▶│ next=null│
└─────────┘    └─────────┘    └─────────┘    └─────────┘

  first ──────────▶                                ◀────────── last
```

### 添加元素到尾部

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;  // 空链表
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

---

## 核心操作性能对比

### 按索引访问

| 操作 | ArrayList | LinkedList | 原因 |
|------|-----------|------------|------|
| get(0) | O(1) | O(1) | 数组首元素/链表首节点 |
| get(n/2) | O(1) | O(n) | 数组随机访问/链表需遍历 |
| get(size-1) | O(1) | O(1) | 数组直接索引/LinkedList记录last |

```java
// ArrayList get 实现 - O(1)
public E get(int index) {
    rangeCheck(index);
    return elementData[index];  // 直接索引访问
}

// LinkedList get 实现 - O(n)
public E get(int index) {
    rangeCheck(index);
    if (index < (size >> 1)) {
        // 前半部分，从头遍历
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x.item;
    } else {
        // 后半部分，从尾遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x.item;
    }
}
```

### 按索引插入

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| add(0, e) | O(n) | O(1) |
| add(n/2, e) | O(n) | O(n) |
| add(size, e) | O(1)* | O(1) |

*ArrayList amortized O(1)，偶尔触发扩容需要O(n)

```java
// ArrayList 插入 - 需要移动元素
public void add(int index, E element) {
    rangeCheckForAdd(index);
    modCount++;
    int s = size;
    Object[] a = elementData;

    if (s == a.length)
        a = grow();  // 扩容
    System.arraycopy(a, index, a, index + 1, s - index);  // 移动元素
    a[index] = element;
    size = s + 1;
}

// LinkedList 插入 - 只需修改指针
public void add(int index, E element) {
    checkPositionIndex(index);
    if (index == size)
        linkLast(element);  // 尾部插入
    else
        linkBefore(element, node(index));  // 中间插入
}

void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

### 删除操作

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| remove(0) | O(n) | O(1) |
| remove(n/2) | O(n) | O(n) |
| remove(size-1) | O(1) | O(1) |

```java
// ArrayList 删除 - 移动元素
public E remove(int index) {
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null;
    return oldValue;
}

// LinkedList 删除 - 修改指针
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}

E unlink(Node<E> x) {
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) {
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) {
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

---

## 扩容机制

### ArrayList 扩容

```java
private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 新容量 = 旧容量 * 1.5
    int newCapacity = oldCapacity + (oldCapacity >> 1);

    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);

    elementData = Arrays.copyOf(elementData, newCapacity);
    return elementData;
}
```

### 扩容流程图

```
初始: elementData = new Object[10]
                │
                ▼
    ┌───────────────────────┐
    │  添加第11个元素时      │
    │  size=10, length=10   │
    │  size > length        │
    └───────────────────────┘
                │
                ▼
    ┌───────────────────────┐
    │      扩容             │
    │  newCapacity = 10*1.5 │
    │  = 15                 │
    └───────────────────────┘
                │
                ▼
    ┌───────────────────────┐
    │ Arrays.copyOf         │
    │ 复制到新数组          │
    │ 放入第11个元素        │
    └───────────────────────┘
```

---

## RandomAccess接口

### 标记接口

```java
public interface RandomAccess {
    // 标记接口，无方法
}
```

ArrayList 实现了 RandomAccess，LinkedList 没有实现。

```java
// JVM遍历优化
public static <T> int indexOfSubList(List<T> source, List<T> target) {
    int sourceSize = source.size();
    int targetSize = target.size();

    // 根据是否实现RandomAccess选择最优算法
    if (source instanceof RandomAccess) {
        // ArrayList：使用二分查找
        for (int i = 0; i <= sourceSize - targetSize; i++) {
            // ...
        }
    } else {
        // LinkedList：使用迭代器
        ListIterator<T> si = source.listIterator();
        // ...
    }
}
```

---

## 迭代器性能

### ArrayList 迭代器

```java
// ArrayList 使用 for 循环遍历更快
for (int i = 0; i < list.size(); i++) {
    list.get(i);  // O(1) 随机访问
}

// Iterator 遍历
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    it.next();  // 同样是 O(1) 随机访问
}
```

### LinkedList 迭代器

```java
// LinkedList 使用 for 循环遍历很慢
for (int i = 0; i < list.size(); i++) {
    list.get(i);  // O(n) 每次都从头遍历
}

// Iterator 遍历较快
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    it.next();  // O(1) 迭代器维护当前位置
}
```

---

## 性能总结

### 时间复杂度对比

| 操作 | ArrayList | LinkedList |
|------|-----------|------------|
| 按索引访问 get(i) | O(1) | O(n) / O(n/2) |
| 头部插入/删除 add(0, e) / remove(0) | O(n) | O(1) |
| 尾部插入/删除 add(e) / remove() | O(1)* | O(1) |
| 中间插入/删除 add(i, e) / remove(i) | O(n) | O(n) |
| 遍历 | O(n) | O(n) |

*amortized O(1)，偶尔扩容需要O(n)

### 空间复杂度

| 结构 | 空间复杂度 | 额外开销 |
|------|-----------|---------|
| ArrayList | O(n) | 无额外节点开销，但可能有空余容量 |
| LinkedList | O(n) | 每个节点需要prev/next指针（2个引用） |

---

## 使用场景选择

### 选择 ArrayList

- **随机访问频繁**：频繁使用 get(i) 或遍历
- **尾部操作为主**：主要在末尾添加/删除
- **内存效率优先**：不希望有节点指针开销
- **需要实现RandomAccess**：需要二分查找等场景

```java
// ArrayList 最佳实践
List<String> list = new ArrayList<>(initialCapacity);  // 预估容量，避免扩容

// 遍历
for (int i = 0; i < list.size(); i++) {
    process(list.get(i));  // O(1) 访问
}
```

### 选择 LinkedList

- **头部操作频繁**：频繁在头部添加/删除（如栈、队列）
- **不需要随机访问**：主要通过迭代器遍历
- **内存充足**：不介意节点指针开销
- **需要实现Deque**：需要队列或双端队列功能

```java
// LinkedList 最佳实践
Deque<String> queue = new LinkedList<>();

// 头部操作 O(1)
queue.offerFirst("A");  // 头部插入
queue.pollFirst();      // 头部删除

// 迭代器遍历 O(n)
for (String s : queue) {
    process(s);  // O(1) 迭代器遍历
}
```

---

## 线程安全版本

### 非线程安全

ArrayList 和 LinkedList 都**不是线程安全的**：

```java
// 线程不安全！
List<Integer> list = new ArrayList<>();
list.add(1);  // 多线程下可能丢失数据
```

### 线程安全替代方案

```java
// 方案1：Collections.synchronizedList
List<Integer> safeList = Collections.synchronizedList(new ArrayList<>());

// 方案2：CopyOnWriteArrayList（读多写少）
List<Integer> cowList = new CopyOnWriteArrayList<>();

// 方案3：使用线程安全的队列
Queue<Integer> safeQueue = new ConcurrentLinkedQueue<>();
```

---

## 常见面试题

### Q1：ArrayList 和 LinkedList 的区别？

> 答：ArrayList 基于动态数组，支持 O(1) 随机访问，但插入/删除需要移动元素。LinkedList 基于双向链表，插入/删除 O(1)，但随机访问 O(n)。ArrayList 内存连续，LinkedList 节点需要额外指针空间。

### Q2：ArrayList 扩容机制？

> 答：默认容量10，扩容时新容量=旧容量*1.5，使用 Arrays.copyOf 复制到新数组。扩容代价较高，应预估容量避免频繁扩容。

### Q3：遍历时为什么 LinkedList 不要用 get(i)？

> 答：get(i) 对于 LinkedList 是 O(n)，每次调用都从头/尾遍历。应该用迭代器遍历，迭代器内部维护位置指针，遍历是 O(n)。

### Q4：如何选择 ArrayList 和 LinkedList？

> 答：需要频繁随机访问选 ArrayList，需要频繁头部插入删除选 LinkedList。以尾部操作居多且需要线程安全可选 CopyOnWriteArrayList。

### Q5：ArrayList 是如何实现快速随机访问的？

> 答：通过实现 RandomAccess 接口，底层是数组，直接通过索引计算内存地址 O(1) 访问。

---

## 总结

ArrayList 和 LinkedList 是两种截然不同的实现：
- **ArrayList**：动态数组，O(1)随机访问，O(n)插入删除
- **LinkedList**：双向链表，O(n)随机访问，O(1)头尾操作
- **选择原则**：根据访问模式选择，数据结构没有绝对优劣
