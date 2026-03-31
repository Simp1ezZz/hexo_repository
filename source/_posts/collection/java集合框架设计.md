---
title: Java集合框架整体设计：接口体系与设计模式
date: 2023-12-27 16:44:48
tags:
  - Java进阶
  - 集合框架
categories: 学习
keywords: Java集合框架,Collection,Map,接口体系,设计模式
description: 从整体视角理解Java集合框架的设计哲学，核心接口与设计模式
cover:
---

## 前言

Java 集合框架（Java Collections Framework）是 Java SE 中最核心的类库之一。理解它的整体设计，有助于更好地使用集合类，也能从中学习优秀的设计模式。本文从架构层面解析 Java 集合框架。

## 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                       Iterable                                   │
│              iterator() / forEach() / spliterator()              │
└─────────────────────────────────────────────────────────────────┘
                              △
                              │
┌─────────────────────────────────────────────────────────────────┐
│                       Collection                                 │
│              ┌──────────┴──────────┐                           │
│         ┌────┴────┐           ┌────┴────┐                      │
│         │   Set   │           │   List  │           ┌────┐   │
│         │ 无序不重复│           │  有序可重复│           │Queue│   │
│         └─────────┘           └─────────┘           └──┬─┘   │
│         ┌────┴────┐           ┌────┴────┐              │     │
│    ┌────┴───┐ ┌───┴────┐ ┌───┴───┐ ┌──┴────┐        │     │
│    │ HashSet│ │TreeSet│ │ArrayList││LinkedList│      │     │
│    │LinkedSet│ │       │ │ Vector  │ │         │      │     │
│    └─────────┘ └───────┘ └─────────┘ └─────────┘      │     │
└─────────────────────────────────────────────────────────────┘   │
                                                                    │
┌─────────────────────────────────────────────────────────────────┐
│                          Map                                     │
│              ┌──────────┴──────────┐                           │
│         ┌────┴────┐           ┌────┴────┐                      │
│         │ HashMap │           │ TreeMap │                      │
│         │LinkedMap│           │ WeakMap │                      │
│         │IdentityHashMap│      │ Hashtable│                     │
│         └─────────┘           └─────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心接口详解

### Iterable 接口

```java
public interface Iterable<T> {
    // 返回迭代器
    Iterator<T> iterator();

    // JDK8+: forEach遍历
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    // JDK8+: 可分割迭代器（并行遍历）
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}
```

### Collection 接口

```java
public interface Collection<E> extends Iterable<E> {
    // 基本操作
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    boolean add(E e);
    boolean remove(Object o);

    // 批量操作
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);  // 交集
    void clear();

    // 转换为数组
    Object[] toArray();
    <T> T[] toArray(T[] a);

    // 流操作（JDK8+）
    default Stream<E> stream() { }
    default Stream<E> parallelStream() { }
}
```

### List 接口

```java
public interface List<E> extends Collection<E> {
    // 位置访问
    E get(int index);
    E set(int index, E element);
    void add(int index, E element);
    E remove(int index);

    // 搜索
    int indexOf(Object o);
    int lastIndexOf(Object o);

    // 迭代器
    ListIterator<E> listIterator();
    ListIterator<E> listIterator(int index);

    // 子列表
    List<E> subList(int fromIndex, int toIndex);
}
```

### Set 接口

```java
public interface Set<E> extends Collection<E> {
    // 无重复元素
    // 不维护顺序（部分实现维护）

    // Set 继承 Collection 的所有方法
    // 额外约束：add 不允许重复元素
}
```

### Map 接口

```java
public interface Map<K, V> {
    // 基本操作
    int size();
    boolean isEmpty();
    boolean containsKey(Object key);
    boolean containsValue(Object value);
    V get(Object key);
    V put(K key, V value);
    V remove(Object key);

    // 批量操作
    void putAll(Map<? extends K, ? extends V> m);
    void clear();

    // 视图
    Set<K> keySet();
    Collection<V> values();
    Set<Map.Entry<K, V>> entrySet();

    // Entry 接口
    interface Entry<K, V> {
        K getKey();
        V getValue();
        V setValue(V value);
    }
}
```

### Queue 接口

```java
public interface Queue<E> extends Collection<E> {
    // 添加元素
    boolean add(E e);      // 失败抛异常
    boolean offer(E e);    // 失败返回false

    // 移除元素
    E remove();           // 队列空抛异常
    E poll();              // 队列空返回null

    // 查看元素
    E element();          // 队列空抛异常
    E peek();              // 队列空返回null
}
```

---

## 设计模式应用

### 1. 迭代器模式（Iterator Pattern）

分离集合的遍历行为：

```java
public interface Iterator<E> {
    boolean hasNext();
    E next();
    default void remove() { throw new UnsupportedOperationException(); }
    default void forEachRemaining(Consumer<? super E> action) { }
}

// ArrayList 的迭代器
public class ArrayList<E> ... {
    public Iterator<E> iterator() {
        return new Itr();
    }

    private class Itr implements Iterator<E> {
        int cursor;
        int lastRet = -1;

        public boolean hasNext() {
            return cursor != size;
        }

        public E next() {
            int i = cursor;
            Object[] es = elementData;
            if (i >= size)
                throw new NoSuchElementException();
            cursor = i + 1;
            return (E) es[lastRet = i];
        }
    }
}
```

### 2. 适配器模式（Adapter Pattern）

Arrays.asList() 将数组适配为 List：

```java
@SafeVarargs
public static <T> List<T> asList(T... a) {
    return new ArrayList<>(a);
}

// 数组 → List 的适配
String[] arr = {"a", "b", "c"};
List<String> list = Arrays.asList(arr);
list.add("d");  // UnsupportedOperationException！
// 注意：返回的是 Arrays 的内部类，不支持 add/remove
```

### 3. 装饰器模式（Decorator Pattern）

Collections.unmodifiableXXX() 包装为只读：

```java
public static <T> List<T> unmodifiableList(List<? extends T> list) {
    return new UnmodifiableRandomAccessList<>(
        new UnmodifiableList<>(list));
}

// 使用
List<String> original = new ArrayList<>();
original.add("a");
List<String> readOnly = Collections.unmodifiableList(original);
readOnly.add("b");  // UnsupportedOperationException！
```

Collections.synchronizedXXX() 包装为线程安全：

```java
public static <T> List<T> synchronizedList(List<T> list) {
    return new SynchronizedList<>(list);
}

// 使用
List<String> safeList = Collections.synchronizedList(new ArrayList<>());
synchronized (safeList) {  // 建议手动加锁
    safeList.add("a");
}
```

### 4. 工厂模式（Factory Pattern）

Collections 提供了多个工厂方法：

```java
// 空集合工厂
Collections.emptyList();     // 返回空 List
Collections.emptySet();      // 返回空 Set
Collections.emptyMap();      // 返回空 Map

// 单元素工厂
Collections.singletonList("a");  // 返回只含一个元素的 List
Collections.singleton("a");       // 返回只含一个元素的 Set
Collections.singletonMap(k, v);   // 返回只含一个映射的 Map

// JDK9+ 静态工厂方法
List.of("a", "b", "c");     // 不可变 List
Set.of("a", "b");           // 不可变 Set
Map.of("k1", "v1", "k2", "v2");  // 不可变 Map
```

---

## 抽象类设计

Java 集合框架提供了抽象类，让开发者可以方便地实现自己的集合：

```
AbstractCollection      - 实现了 Collection 的基本方法
    │
    ├── AbstractList     - 实现了 List，添加了随机访问支持
    │       ├── ArrayList
    │       └── AbstractSequentialList - 支持顺序访问（链表）
    │               └── LinkedList
    │
    ├── AbstractSet      - 实现了 Set
    │       ├── HashSet
    │       ├── TreeSet
    │       └── LinkedHashSet
    │
    └── AbstractQueue    - 实现了 Queue
            └── AbstractSequentialList
                    └── LinkedList

AbstractMap             - 实现了 Map
    ├── HashMap
    ├── TreeMap
    └── LinkedHashMap
```

### 抽象类的好处

```java
// 如何正确实现一个只读的 Set
public class MySet<E> extends AbstractSet<E> {

    private final Set<E> internalSet = new HashSet<>();

    @Override
    public Iterator<E> iterator() {
        return internalSet.iterator();
    }

    @Override
    public int size() {
        return internalSet.size();
    }

    @Override
    public boolean add(E e) {
        return internalSet.add(e);
    }
}

// 只需要实现核心方法，其他方法由 AbstractSet 提供默认实现
```

---

## 视图与包装器

### 视图（View）

```java
// keySet() 返回 Map 的视图
Map<String, Integer> map = new HashMap<>();
map.put("a", 1);
map.put("b", 2);

Set<String> keys = map.keySet();  // 不是副本，是视图
keys.remove("a");  // 会从 map 中删除
System.out.println(map);  // {b=2}

// values() 返回值的集合视图
Collection<Integer> values = map.values();
values.clear();  // 清空 map
```

### 子列表（SubList）

```java
List<String> list = new ArrayList<>(Arrays.asList("a","b","c","d","e"));
List<String> sub = list.subList(1, 4);  // [b, c, d]
sub.set(0, "x");  // 修改 sub
System.out.println(list);  // [a, x, c, d, e] - list 也会变！
sub.clear();  // 清空 sub
System.out.println(list);  // [a] - list 也被清空
```

---

## 常用实现类对比

### Set 实现类

| 实现类 | 底层 | 顺序 | 特点 |
|--------|------|------|------|
| HashSet | 哈希表 | 无序 | O(1) 插入/查找/删除 |
| LinkedHashSet | 哈希表+链表 | 插入顺序 | 保持插入顺序 |
| TreeSet | 红黑树 | 自然顺序/比较器 | O(log n) 有序操作 |
| EnumSet | 位向量 | 枚举顺序 | 高效，限定枚举类型 |

### List 实现类

| 实现类 | 底层 | 特点 |
|--------|------|------|
| ArrayList | 动态数组 | O(1) 随机访问，O(n) 插入/删除 |
| LinkedList | 双向链表 | O(1) 头尾操作，O(n) 随机访问 |
| Vector | 动态数组 | 线程安全（已过时） |
| Stack | Vector | LIFO，已过时，用 Deque |

### Map 实现类

| 实现类 | 底层 | 顺序 | 特点 |
|--------|------|------|------|
| HashMap | 哈希表 | 无序 | O(1) 高效 |
| LinkedHashMap | 哈希表+链表 | 插入顺序 | 保持顺序 |
| TreeMap | 红黑树 | 自然顺序 | O(log n) 有序 |
| Hashtable | 哈希表 | 无序 | 线程安全（已过时） |
| WeakHashMap | 哈希表 | 无序 | 弱引用，GC时自动删除 |
| IdentityHashMap | 哈希表 | 无序 | 用 == 而非 equals() 比较 |

### Queue/Deque 实现类

| 实现类 | 底层 | 特点 |
|--------|------|------|
| LinkedList | 双向链表 | 头尾操作 O(1) |
| ArrayDeque | 循环数组 | 比 Stack 和 LinkedList 快 |
| PriorityQueue | 堆 | 按优先级出队 |
| ConcurrentLinkedQueue | CAS+链表 | 无界非阻塞队列 |
| ArrayBlockingQueue | 数组 | 有界阻塞队列 |
| LinkedBlockingQueue | 链表 | 可选有界/无界 |

---

## JDK8 新增特性

### Stream API

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// 过滤
List<String> filtered = names.stream()
    .filter(n -> n.length() > 4)
    .collect(Collectors.toList());

// 映射
List<Integer> lengths = names.stream()
    .map(String::length)
    .collect(Collectors.toList());

// 归约
int sum = names.stream()
    .mapToInt(String::length)
    .sum();

// 分组
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));
```

### forEach 方法

```java
// 之前的写法
for (String name : names) {
    System.out.println(name);
}

// JDK8+ 写法
names.forEach(System.out::println);
names.forEach(name -> System.out.println(name));
```

### removeIf 方法

```java
// 删除满足条件的元素
List<Integer> numbers = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
numbers.removeIf(n -> n % 2 == 0);  // 删除偶数
System.out.println(numbers);  // [1, 3, 5]
```

---

## 最佳实践

### 选择合适的集合

```java
// 需要快速查找：HashMap / HashSet
Map<String, User> userMap = new HashMap<>();

// 需要保持顺序：LinkedHashMap / LinkedHashSet
Map<String, User> orderedMap = new LinkedHashMap<>();

// 需要有序：TreeMap / TreeSet
Map<String, User> sortedMap = new TreeMap<>();

// 需要频繁头部插入删除：LinkedList 或 ArrayDeque
Deque<String> queue = new ArrayDeque<>();

// 需要优先级队列：PriorityQueue
Queue<Integer> pq = new PriorityQueue<>(Comparator.reverseOrder());

// 线程安全场景：ConcurrentHashMap / ConcurrentLinkedQueue
ConcurrentMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
```

### 避免常见问题

```java
// 问题1：数组和集合混淆
String[] arr = new String[10];  // 固定大小
List<String> list = new ArrayList<>();  // 可动态增长

// 问题2：使用已过时的类
Hashtable<String, Integer> old = new Hashtable<>();  // 已过时
Map<String, Integer> modern = new HashMap<>();  // 推荐

// 问题3：返回裸集合
// 不好：暴露内部集合
public List<String> getNames() { return names; }

// 更好：返回不可变视图或副本
public List<String> getNames() { return Collections.unmodifiableList(names); }

// 问题4：忘记初始化
List<String> list;  // null
list.add("a");  // NullPointerException！

list = new ArrayList<>();  // 正确初始化
```

---

## 面试高频问题

**Q1：Java集合框架的整体架构？**

> 答：顶层是Iterable接口，然后是Collection接口（包含Set、List、Queue）和Map接口。Set的抽象实现是AbstractSet，List是AbstractList，Map是AbstractMap。每个接口有多个实现类。

**Q2：ArrayList和Vector的区别？**

> 答：Vector是早期实现，线程安全（所有方法synchronized），性能差。ArrayList非线程安全，性能好。JDK5之后推荐使用ArrayList，需要线程安全用CopyOnWriteArrayList。

**Q3：HashMap和Hashtable的区别？**

> 答：HashMap线程不安全，允许null key/value，性能好。Hashtable线程安全（synchronized），不允许null，性能差。推荐使用ConcurrentHashMap。

**Q4：迭代器fail-fast机制是什么？**

> 答：集合的迭代器在迭代过程中如果发现集合被修改（除了迭代器的remove），会快速失败抛出ConcurrentModificationException。是一种保护机制，不是线程安全保证。

**Q5：为什么不直接返回内部集合？**

> 答：返回内部集合的引用会让调用者可以直接修改，破坏封装。应该返回Collections.unmodifiableXXX()包装的不可变视图，或返回副本。

---

## 总结

Java集合框架的设计体现了多个设计原则：
- **接口与实现分离**：接口定义契约，实现提供细节
- **迭代器模式**：分离遍历行为与集合本身
- **装饰器模式**：通过包装器添加功能（同步、只读）
- **工厂模式**：Collections提供多种工厂方法
- **抽象类支持**：AbstractXXX简化自定义实现
- **视图与副本**：灵活控制数据暴露程度
