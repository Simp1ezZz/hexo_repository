---
title: volatile关键字：内存可见性与指令重排序
date: 2023-12-30 15:38:00
tags:
  - Java进阶
  - 并发编程
categories: 学习
keywords: volatile,内存屏障,可见性,有序性,JMM
description: 深入理解volatile的底层实现原理，内存屏障与JMM的关系
cover:
---

## 前言

volatile 是 Java 并发编程中最轻量的同步机制。很多同学知道它能"保证可见性"，但底层是如何实现的？它和 synchronized 有什么区别？本文深入解析 volatile 的工作原理。

## volatile的两个核心特性

### 1. 内存可见性

**问题引出：**

```java
public class VisibilityDemo {
    private static boolean flag = false;

    public static void main(String[] args) throws InterruptedException {
        // 线程1：修改flag
        new Thread(() -> {
            while (!flag) {
                // 循环等待
            }
            System.out.println("线程1检测到flag变化，退出循环");
        }).start();

        // 线程2：修改flag
        Thread.sleep(100);
        flag = true;
        System.out.println("线程2设置了flag=true");
    }
}
```

> 上述代码中，线程1可能永远无法退出循环，因为 flag 的更新对线程1不可见。这就是**缓存导致的可见性问题**。

**解决：给 flag 添加 volatile 修饰**

```java
private volatile static boolean flag = false;
// 现在线程2的修改对线程1立即可见
```

### 2. 禁止指令重排序

**问题引出：**

```java
public class ReorderDemo {
    private int a = 0;
    private int b = 0;
    private volatile boolean init = false;

    // 线程1：初始化
    public void init() {
        a = 1;          // 操作1
        b = 2;          // 操作2
        init = true;    // 操作3
    }

    // 线程2：使用
    public void use() {
        if (init) {     // 操作4
            System.out.println(a + "," + b);  // 操作5
        }
    }
}
```

> 如果没有 volatile 修饰，编译器和 CPU 可能对指令重排序，导致 init=true 在 a=1 和 b=2 之前执行，线程2可能读到 a=0, b=0。

---

## JMM与内存屏障

### Java内存模型（JMM）

JMM（Java Memory Model）定义了Java程序中线程如何与主内存交互：

```
┌──────────────────────────────────────────────────────────────┐
│                         主内存 (RAM)                         │
│                    所有线程共享的内存                          │
└──────────────────────────────────────────────────────────────┘
          ↑ read/write                ↑ read/write
          │                           │
    ┌─────┴─────┐               ┌─────┴─────┐
    │  工作内存  │               │  工作内存  │
    │ (CPU缓存)  │               │ (CPU缓存)  │
    │  线程1    │               │  线程2    │
    └───────────┘               └───────────┘
```

### 线程与主内存的交互

| 操作 | 说明 |
|------|------|
| read | 从主内存读取到工作内存 |
| load | 从工作内存到CPU缓存 |
| use | 从工作内存传递给CPU执行 |
| assign | CPU结果赋值给工作内存 |
| store | 从工作内存传递到主内存 |
| write | 从store结果写入主内存 |

### 为什么需要内存屏障？

CPU 和编译器为了优化性能，会对指令进行重排序。但重排序在多线程环境下可能导致问题。

---

## volatile的内存屏障实现

### Store Barrier / Load Barrier

volatile 变量的读写会插入内存屏障：

```java
public class VolatileBarrier {

    // volatile 变量的写操作后插入 Store Barrier
    // 强制将工作内存刷新到主内存
    public void volatileWrite() {
        this.value = 10;    // volatile写
        // 插入: Store Barrier
        // MFENCE 指令
        // 强制所有之前的store都刷新到主内存
    }

    // volatile 变量的读操作前插入 Load Barrier
    // 强制从主内存重新读取
    public int volatileRead() {
        // 插入: Load Barrier
        // LFENCE 指令
        // 强制所有之后的load从主内存读取
        return this.value;  // volatile读
    }
}
```

### x86架构下的实现

| 屏障类型 | 指令 | 说明 |
|---------|------|------|
| StoreStore | SFENCE | 所有之前的store都可见 |
| StoreLoad | MFENCE | 所有之前的store对之后的load可见 |
| LoadLoad | LFENCE | 所有之前的load都完成 |
| LoadStore | - | 所有之前的load都对之后的store可见 |

### volatile写操作的屏障

```
普通写                      volatile写
─────────                  ──────────────
a = 1;                     a = 1;
b = 2;                     b = 2;
                           [StoreStore屏障]
flag = true;               flag = true;
                           [StoreLoad屏障]
```

### volatile读操作的屏障

```
普通读                      volatile读
─────────                  ──────────────
                           [LoadLoad屏障]
                           [LoadStore屏障]
if (flag)                  if (flag)
    x = a;                     x = a;
    y = b;                     y = b;
```

---

## happens-before规则

JMM 定义了 happens-before 规则，volatile 的可见性通过这个规则保证。

### 什么是 happens-before？

> 如果操作A happens-before 操作B，那么A的执行结果对B可见，且A的执行顺序在B之前。

### 八大happens-before规则

| 规则 | 说明 |
|------|------|
| 程序顺序规则 | 单线程中，前面的操作 happens-before 后面的操作 |
| 监视器锁规则 | 锁的释放 happens-before 后续获取 |
| volatile规则 | volatile变量的写 happens-before 后续读取 |
| 线程启动规则 | start() happens-before 线程内的任何操作 |
| 线程终止规则 | 线程内的操作 happens-before 其他线程检测到终止 |
| 传递性 | A happens-before B，B happens-before C，则 A happens-before C |
| join()规则 | Thread.join() returns happens-before 检测到终止 |
| 中断规则 | interrupt() happens-before 被中断线程检测到中断 |

### volatile的happ-before保证

```java
public class HappensBeforeDemo {
    private volatile int value = 0;

    // 线程1执行 writer()
    public void writer() {
        value = 1;        // volatile写
        // happens-before: volatile写 happens-before volatile读
    }

    // 线程2执行 reader()
    public int reader() {
        int r = value;    // volatile读
        // happens-before: volatile写 happens-before volatile读
        return r;         // 一定能读到1
    }
}
```

---

## volatile vs synchronized

### 对比表

| 特性 | volatile | synchronized |
|------|----------|--------------|
| 原子性 | 仅保证64位写入的原子性 | 保证代码块原子性 |
| 可见性 | 保证可见性 | 保证可见性 |
| 有序性 | 禁止重排序 | 禁止重排序 |
| 性能 | 高（仅内存屏障） | 低（加锁解锁） |
| 用途 | 状态标志、触发器 | 保护复杂逻辑 |

### 适用场景

```java
// volatile 适用场景
private volatile boolean initialized = false;

// 1. 状态标志
if (initialized) {
    // 一定能读到最新值
}

// 2. 触发器（单volatile + 多synchronized）
private volatile boolean ready = false;

public synchronized void produce() {
    // 生产数据
    ready = true;  // volatile写，通知消费者
}

public synchronized void consume() {
    while (!ready) {
        wait();
    }
    // 消费数据
}

// 3. DCL（双重检查锁定）中的可见性
private volatile Singleton instance;

public Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton();  // volatile防止指令重排
            }
        }
    }
    return instance;
}
```

### 不适用场景

```java
// 以下场景不能用 volatile

// 1. 计数器（复合操作非原子）
private volatile int counter = 0;
counter++;  // 非原子操作！读-改-写三步，可能丢失更新

// 应该用 AtomicInteger
private AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();  // 原子操作

// 2. 需要原子读取-修改-写回的场景
// volatile不够，需要 synchronized 或 CAS
```

---

## volatile的64位写入原子性

### 问题

JMM 允许把 64 位的 long 和 double 的读写操作拆成两次 32 位操作。

```java
public class LongDemo {
    private long value;  // 非volatile

    // 线程1
    public void write() {
        value = 12345678901234L;
    }

    // 线程2
    public long read() {
        return value;  // 可能读到"撕裂值"
    }
}
```

### 解决

```java
public class LongDemoFixed {
    private volatile long value;  // volatile保证64位原子性

    public void write() {
        value = 12345678901234L;  // 原子写入
    }

    public long read() {
        return value;  // 原子读取
    }
}
```

> 注意：在现代JVM中，long/double的读写默认已经是原子的，但加上volatile是最佳实践。

---

## 实战：DCL单例与volatile

### 经典DCL问题

```java
public class Singleton {
    private static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {           // 线程2可能跳过
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  // 问题！
                }
            }
        }
        return instance;
    }
}
```

**`instance = new Singleton()` 分解为三步：**
```
1. 分配内存
2. 调用构造方法
3. 将引用赋值给instance
```

**问题：** 指令重排序可能导致 3 在 2 之前执行，线程2可能看到一个未构造完全的对象。

### 正确实现

```java
public class Singleton {
    // 1. 加volatile禁止重排序
    // 2. 解决可见性问题
    private volatile static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

---

## CPU缓存一致性协议

### MESI协议

现代CPU使用MESI协议保证缓存一致性：

| 状态 | 含义 |
|------|------|
| Modified (M) | 缓存行被修改，必须写回主内存 |
| Exclusive (E) | 缓存行独占，无其他缓存副本 |
| Shared (S) | 多个缓存持有同一行，可能都有效 |
| Invalid (I) | 缓存行无效 |

### 缓存行嗅探

CPU通过总线嗅探（Bus Snooping）监控其他CPU的缓存操作：

```
CPU1 修改缓存行 → 总线广播invalidate → CPU2缓存行失效
                                      ↓
                               CPU2 重新从主存读取
```

**volatile 写操作的流程：**
1. 嗅探到其他CPU缓存了同一行
2. 发送 invalidate 信号
3. 其他CPU将缓存行置为 Invalid
4. 等待所有 CPU 确认
5. 写入新值到缓存行

---

## 面试高频问题

**Q1：volatile如何保证可见性？**

> 答：volatile写操作后插入Store Barrier，强制将工作内存刷新到主内存。volatile读操作前插入Load Barrier，强制从主内存读取最新值。

**Q2：volatile为什么能禁止指令重排序？**

> 答：通过内存屏障实现。volatile写之后插入StoreLoad屏障，volatile读之前插入LoadLoad屏障，防止上下文的指令重排序。

**Q3：volatile和synchronized区别？**

> 答：volatile是轻量级同步，仅保证可见性和64位写入原子性；synchronized是重量级同步，保证原子性、可见性和有序性。volatile用于状态标志，synchronized用于保护代码块。

**Q4：volatile能保证原子性吗？**

> 答：仅保证64位变量读写原子性（如long/double），不保证复合操作的原子性（如counter++）。复合操作需要用AtomicInteger等原子类。

**Q5：happens-before是什么？**

> 答：JMM定义的操作间偏序关系。如果A happens-before B，则A的执行结果对B可见，且A在B之前执行。volatile变量写 happens-before 后续读取。

---

## 总结

volatile 是 Java 并发编程的基础：
- **两大特性**：内存可见性 + 禁止指令重排序
- **底层实现**：内存屏障（Store Barrier / Load Barrier）
- **happens-before**：volatile写 happens-before后续volatile读
- **适用场景**：状态标志、DCL单例、触发器
- **不适用**：计数器等复合操作，需用AtomicInteger
