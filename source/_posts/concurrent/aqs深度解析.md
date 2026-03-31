---
title: AQS深度解析：ReentrantLock与AbstractQueuedSynchronizer
date: 2026-04-07 10:00:00
tags:
  - Java进阶
  - 并发编程
categories: 学习
keywords: AQS,ReentrantLock,AbstractQueuedSynchronizer,CAS,CLH队列
description: 深入理解JUC并发包核心抽象类AQS的工作原理
cover:
---

## 前言

`ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReadWriteLock` 这些常用的并发工具类，它们的底层实现都依赖于同一个核心——**AbstractQueuedSynchronizer（AQS）**。本文深入解析 AQS 的工作原理。

## AQS是什么？

### 定义

AQS（AbstractQueuedSynchronizer）是 JDK 并发包的核心抽象类，定义了实现同步器的基础框架。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    // 核心数据结构：CLH变体双向队列
    private transient volatile Node head;
    private transient volatile Node tail;

    // 同步状态
    private volatile int state;

    // ... 核心方法
}
```

### 核心组件

| 组件 | 说明 |
|------|------|
| state | 同步状态，0表示未被占用，>0表示被占用 |
| CLH队列 | 双向FIFO队列，存储等待获取锁的线程 |
| Node | 队列节点，包含线程和等待状态 |

---

## 同步状态（State）

### state 的含义

- **ReentrantLock**：表示锁的重入次数，0=未占用，>0=已占用
- **Semaphore**：表示剩余许可数量
- **CountDownLatch**：表示还需要倒计时的次数

### 线程安全操作

```java
// 使用CAS保证state原子操作
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

---

## CLH队列详解

### Node 节点结构

```java
static final class Node {
    // 模式：独占模式 / 共享模式
    static final Node EXCLUSIVE = null;  // 独占
    static final Node SHARED = new Node();  // 共享

    // 等待状态
    static final int CANCELLED = 1;      // 已取消
    static final int SIGNAL = -1;        // 后续节点需要唤醒
    static final int CONDITION = -2;     // 等待条件
    static final int PROPAGATE = -3;     //  PROPAGATE状态

    volatile int waitStatus;

    volatile Node prev;    // 前驱节点
    volatile Node next;    // 后继节点
    volatile Thread thread; // 等待线程
}
```

### 队列结构

```
                    head                tail
                     ↓                  ↓
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ Node │ ←→ │ Node │ ←→ │ Node │ ←× │ null │
  │ dummy│    │ T1   │    │ T2   │    │      │
  └──────┘    └──────┘    └──────┘    └──────┘
              thread=T1     thread=T2
              status=-1     status=-1
```

CLH队列是一个虚拟头结点的双向FIFO队列。

---

## 独占模式工作流程

### 获取锁（acquire）

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&  // 1. 尝试获取锁
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) {  // 2. 加入等待队列
        selfInterrupt();  // 3. 中断自己
    }
}
```

### tryAcquire（模板方法）

AQS 定义了模板方法，子类实现具体的获取逻辑：

```java
// ReentrantLock.FairSync 实现
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();

    // 0表示未占用
    if (c == 0) {
        // 公平锁：检查是否有前驱节点
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 非公平锁：直接CAS抢锁
    else if (current == getExclusiveOwnerThread()) {
        // 重入：state + acquires
        int nextc = c + acquires;
        setState(nextc);
        return true;
    }
    return false;
}
```

### addWaiter（加入队列）

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);

    // 快速入队：尝试一次CAS
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }

    // 自旋入队
    enq(node);
    return node;
}

private Node enq(Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 初始化：创建空的头节点
            if (compareAndSetHead(new Node())) {
                tail = head;
            }
        } else {
            // 正常入队
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return node;
            }
        }
    }
}
```

### acquireQueued（队列中等待）

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取前驱节点
            final Node p = node.predecessor();

            // 如果是head，说明是第一个等待节点，尝试获取
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null;  // help GC
                failed = false;
                return interrupted;
            }

            // 前驱节点状态为SIGNAL，则阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) {
                interrupted = true;
            }
        }
    } finally {
        if (failed) {
            cancelAcquire(node);
        }
    }
}
```

### shouldParkAfterFailedAcquire

判断是否需要阻塞：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;

    // 前驱节点会唤醒自己，可以安全阻塞
    if (ws == Node.SIGNAL) {
        return true;
    }

    // 前驱节点已取消，跳过已取消的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    }

    // 等待状态为0或PROPAGATE，尝试设置为SIGNAL
    else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

---

## 释放锁（release）

### unlock

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        // 唤醒后继节点
        if (h != null && h.waitStatus != 0) {
            unparkSuccessor(h);
        }
        return true;
    }
    return false;
}
```

### tryRelease（模板方法）

```java
// ReentrantLock.Sync 实现
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;

    // 只有持有锁的线程才能释放
    if (Thread.currentThread() != getExclusiveOwnerThread()) {
        throw new IllegalMonitorStateException();
    }

    // 完全释放（state=0）
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

### unparkSuccessor

唤醒后继节点：

```java
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;

    // 尝试将waitStatus设为0
    if (ws < 0) {
        compareAndSetWaitStatus(node, ws, 0);
    }

    // 找到最需要被唤醒的后继节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从尾部向前找
        for (Node t = tail; t != null && t != node; t = t.prev) {
            if (t.waitStatus <= 0) {
                s = t;
            }
        }
    }

    // 唤醒线程
    if (s != null) {
        LockSupport.unpark(s.thread);
    }
}
```

---

## 公平锁 vs 非公平锁

### ReentrantLock 的两种实现

```java
// 公平锁
static final class FairSync extends Sync {
    protected boolean tryAcquire(int acquires) {
        // 检查是否有前驱等待节点
        if (hasQueuedPredecessors()) {
            return false;
        }
        return compareAndSetState(0, acquires);
    }
}

// 非公平锁
static final class NonfairSync extends Sync {
    protected boolean tryAcquire(int acquires) {
        // 直接抢锁，不检查等待队列
        return compareAndSetState(0, acquires);
    }
}
```

### 关键区别

```java
// 公平锁：tryAcquire中多了一个检查
if (hasQueuedPredecessors()) {
    return false;  // 队列有人，等待
}

// 非公平锁：直接抢
compareAndSetState(0, acquires);  // 直接CAS抢
```

### 选择建议

| 类型 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 公平锁 | 请求按顺序执行 | 吞吐量低 | 需要严格顺序 |
| 非公平锁 | 吞吐量高 | 可能饥饿 | 大多数并发场景 |

---

## 共享模式（CountDownLatch/Semaphore）

### acquireShared

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0) {
        doAcquireShared(arg);
    }
}

private void doAcquireShared(int arg) {
    Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) {
                interrupted = true;
            }
        }
    } finally {
        if (failed) {
            cancelAcquire(node);
        }
    }
}
```

### releaseShared

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (compareAndSetWaitStatus(h, SIGNAL, 0)) {
                    unparkSuccessor(h);
                    continue;
                }
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, PROPAGATE)) {
                continue;
            }
        }
        if (h == head) {
            break;
        }
    }
}
```

---

## Condition 条件队列

### await / signal

```java
public class ConditionObject implements Condition, Serializable {

    // 条件队列（单向FIFO）
    private transient Node firstWaiter;
    private transient Node lastWaiter;

    // await：当前线程释放锁，进入条件队列等待
    public final void await() throws InterruptedException {
        if (Thread.interrupted()) {
            throw new InterruptedException();
        }
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        while (!isOnSyncQueue(node)) {
            LockSupport.park(this);
        }
        // 重新获取锁
        acquireQueued(node, savedState);
    }

    // signal：唤醒条件队列中的一个线程
    public final void signal() {
        if (!isHeldExclusively()) {
            throw new IllegalMonitorStateException();
        }
        Node first = firstWaiter;
        if (first != null) {
            doSignal(first);
        }
    }
}
```

### 实战：实现生产者-消费者

```java
public class BoundedBuffer {
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Object[] items = new Object[100];
    private int count, putIndex, takeIndex;

    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) {
                notFull.await();  // 等待不满
            }
            items[putIndex] = x;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal();  // 通知不空
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();  // 等待不空
            }
            Object x = items[takeIndex];
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal();  // 通知不满
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 面试高频问题

**Q1：AQS的核心数据结构？**

> 答：CLH变体双向队列（head/tail）和volatile的state同步状态。队列存储等待获取锁的线程节点，state表示同步资源状态。

**Q2：ReentrantLock如何实现可重入？**

> 答：state记录重入次数。获取锁时检查是否是同一线程，是则state+1，释放时state-1，直到state=0才完全释放锁。

**Q3：公平锁和非公平锁的区别？**

> 答：公平锁tryAcquire多了hasQueuedPredecessors检查，确保队列前有人就等待。非公平锁直接CAS抢锁，可能插队，吞吐量更高。

**Q4：Condition和Object.wait的区别？**

> 答：一个Lock可以创建多个Condition，精准唤醒指定条件的线程。Object.wait只能和synchronized配合，且只有一个隐式队列。

**Q5：AQS为什么用双向队列？**

> 答：双向队列便于节点移除（cancelAcquire需要前驱节点）。CLH队列用于非阻塞同步，通过CAS保证原子性。

---

## 总结

AQS是JUC并发包的基石：
- **核心结构**：state同步状态 + CLH双向队列
- **独占模式**：tryAcquire/acquire + tryRelease/release
- **共享模式**：tryAcquireShared/acquireShared + tryReleaseShared/releaseShared
- **公平与否**：公平锁多hasQueuedPredecessors检查
- **Condition**：替代Object.wait，一个锁多个条件队列
