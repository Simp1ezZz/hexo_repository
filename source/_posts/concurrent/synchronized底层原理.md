---
title: synchronized底层原理：从对象头到Monitor锁
date: 2026-04-06 10:00:00
tags:
  - Java进阶
  - 并发编程
categories: 学习
keywords: synchronized,Monitor,对象头,偏向锁,轻量级锁
description: 深入理解synchronized的底层实现原理，从对象头到Monitor锁的完整链路
cover:
---

## 前言

`synchronized` 是 Java 中最常用的同步关键字，但你对它的理解是否还停留在"加锁"层面？本文深入解析 synchronized 从对象头到 Monitor 锁的完整底层原理。

## synchronized的基本使用

### 三种加锁形式

```java
public class SyncDemo {

    // 1. 修饰实例方法 - 锁对象本身
    public synchronized void method1() {
        // 锁定当前this对象
    }

    // 2. 修饰静态方法 - 锁Class对象
    public static synchronized void method2() {
        // 锁定SyncDemo.class对象
    }

    // 3. 修饰代码块 - 锁指定对象
    public void method3() {
        synchronized (this) {
            // 锁定指定对象
        }
    }

    public void method4() {
        synchronized (SyncDemo.class) {
            // 锁定Class对象
        }
    }
}
```

### 锁的性质

| 性质 | 说明 |
|------|------|
| 可重入性 | 同一线程可多次获取同一把锁 |
| 不可中断 | 锁的获取和释放是确定的，不会被中断 |
| 阻塞 | 未获取锁的线程会进入阻塞状态 |

---

## 对象头与Mark Word

synchronized 的锁信息存储在对象的 Mark Word 中。

### Mark Word 结构（64位JVM）

| 锁状态 | Mark Word 内容 | 标志位 |
|--------|---------------|--------|
| 无锁 | 对象hashCode(31位) + GC年龄(4位) + 偏向锁标志(1位) | 01 |
| 偏向锁 | ThreadID(54位) + epoch(2位) + GC年龄(4位) + 偏向锁标志(1位) | 01 |
| 轻量级锁 | 指向栈中锁记录的指针(62位) | 00 |
| 重量级锁 | 指向Monitor的指针(62位) | 10 |
| GC标记 | 空 | 11 |

### Mark Word 变化过程

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
  ↓
hashCode存储  →  偏向锁ThreadID  →  栈中锁记录  →  Monitor
```

---

## 偏向锁原理

### 适用场景

只有一个线程访问同步块时，偏向锁可以省去CAS操作的开销。

### 获取偏向锁流程

```java
// 当对象第一次被线程T访问
// JVM会将对象头中的Mark Word设置为偏向锁状态
// 并用CAS将线程T的ID写入Mark Word

// 如果CAS成功，以后该线程进入同步块无需任何加锁解锁操作
// 如果CAS失败，说明有其他线程竞争，升级为轻量级锁
```

### 偏向锁的撤销

**以下情况会撤销偏向锁：**

1. **竞争发生**：其他线程尝试获取锁
2. **调用hashCode**：hashCode方法会使偏向锁升级
3. **调用wait/notify**：这两个方法只有重量级锁才支持

```java
public class BiasedLockRevoke {

    public static void main(String[] args) {
        Object obj = new Object();

        // 偏向锁在以下情况会撤销：
        obj.hashCode();  // 调用hashCode，偏向锁撤销

        // wait/notify需要Monitor，强制升级为重量级锁
        synchronized (obj) {
            obj.wait();  // 偏向锁升级为重量级锁
        }
    }
}
```

### JVM参数

```bash
# 启用偏向锁（默认）
-XX:+UseBiasedLocking

# 偏向锁延迟时间（JVM启动后多久才启用偏向锁）
-XX:BiasedLockingStartupDelay=4000  # 默认4秒

# 禁用偏向锁
-XX:-UseBiasedLocking
```

---

## 轻量级锁原理

### 适用场景

多个线程交替访问同步块，不存在真正竞争的场景。

### 获取轻量级锁流程

```java
// 线程T获取轻量级锁的过程

// 1. 在线程T的栈帧中创建锁记录(Lock Record)
// 锁记录包含：displaced header 和 owner 指针

// 2. 将对象头的Mark Word复制到锁记录的displaced header

// 3. 使用CAS尝试将对象头的Mark Word更新为指向锁记录的指针
//    如果成功，获取轻量级锁
//    如果失败，说明存在竞争，升级为重量级锁

cas(markWord,
    objectMarkWord,           // 预期值：之前的Mark Word
    lockRecordPointer         // 更新值：指向锁记录的指针
)
```

### 轻量级锁的释放

```java
// 线程T释放轻量级锁

// 使用CAS将锁记录的displaced header复制回对象头
// 如果成功，释放成功
// 如果失败，说明有竞争，锁膨胀为重量级锁

cas(markWord,
    lockRecordPointer,        // 预期值：指向锁记录的指针
    displacedHeader          // 更新值：之前的Mark Word
)
```

### 自旋优化

轻量级锁获取失败时，线程不会直接进入阻塞，而是**自旋等待**。

```java
// 伪代码
while (尝试CAS获取轻量级锁失败) {
    // 空转，自旋重试
    // 自旋次数超过阈值，膨胀为重量级锁
}
```

**自旋参数：**
```bash
# JDK 1.6之前，自旋次数默认10次
# JDK 1.6之后，自旋次数自适应

-XX:PreBlockSpin=10  # 预定义自旋次数（JDK 1.6）

# JDK 1.6+自适应自旋：根据成功率动态调整
```

---

## 重量级锁原理

### Monitor 对象

重量级锁通过 Monitor 对象实现。Monitor 是操作系统级别的同步机制。

### Monitor 结构

```
┌─────────────────────────────────────────────┐
│                   Monitor                    │
├─────────────────────────────────────────────┤
│  _owner        - 持有锁的线程指针            │
│  _WaitSet      - 等待队列（调用wait的线程）  │
│  _entryList    - 阻塞队列（等待锁的线程）     │
│  _count        - 嵌套进入计数                │
│  _recursions   - 重入计数                    │
└─────────────────────────────────────────────┘
```

### 获取重量级锁流程

```java
// 线程进入synchronized代码块

// 1. 如果Monitor未被占用，线程成为Owner
if (monitor._owner == null) {
    monitor._owner = currentThread;
    monitor._count = 1;
}

// 2. 如果是同一线程重入，增加计数
if (monitor._owner == currentThread) {
    monitor._count++;
    monitor._recursions++;
}

// 3. 如果被其他线程占用，进入EntryList等待
else {
    加入monitor._entryList;
    线程状态变为BLOCKED;
}
```

### wait/notify 原理

```java
// 线程调用wait()
synchronized (obj) {
    while (condition) {
        obj.wait();  // 释放锁，进入_WaitSet等待
    }
}

// 线程调用notify()
synchronized (obj) {
    condition = true;
    obj.notify();  // 从_WaitSet随机唤醒一个线程
    obj.notifyAll();  // 唤醒所有等待线程
}
```

---

## 锁升级过程（锁膨胀）

synchronized 锁可以升级但不能降级，这就是著名的 **锁膨胀（Lock Escalation）**。

### 升级流程图

```
┌─────────────────────────────────────────────────────┐
│                      无锁                             │
│   Mark Word: hashCode + age + 0 01                  │
└─────────────────────────────────────────────────────┘
                          ↓
                    线程T首次访问
                    CAS设置偏向锁
                          ↓
┌─────────────────────────────────────────────────────┐
│                      偏向锁                           │
│   Mark Word: ThreadID(54位) + 01                    │
│                                                     │
│   其他线程尝试获取 → 撤销偏向锁 → 轻量级锁            │
└─────────────────────────────────────────────────────┘
                          ↓
                   CAS失败/锁竞争
                   撤销偏向锁
                          ↓
┌─────────────────────────────────────────────────────┐
│                    轻量级锁                           │
│   Mark Word: 指向栈中锁记录的指针  00                │
│                                                     │
│   自旋10次失败 / 锁竞争加剧                          │
└─────────────────────────────────────────────────────┘
                          ↓
                    锁膨胀
                          ↓
┌─────────────────────────────────────────────────────┐
│                    重量级锁                           │
│   Mark Word: 指向Monitor的指针  10                   │
│   Monitor: _owner + _entryList + _WaitSet           │
│   线程状态: BLOCKED / WAITING                       │
└─────────────────────────────────────────────────────┘
```

### 升级触发条件

| 升级阶段 | 触发条件 |
|---------|---------|
| 无锁 → 偏向锁 | 第一个线程访问同步块 |
| 偏向锁 → 轻量级锁 | 撤销偏向锁，且存在线程竞争 |
| 轻量级锁 → 重量级锁 | 自旋次数超过阈值 / 自适应自旋失败 |

---

## synchronized vs Lock

| 对比 | synchronized | Lock |
|------|-------------|------|
| 层面 | JVM层面（对象头Monitor） | JDK层面（java.util.concurrent） |
| 锁获取 | 阻塞式获取，不可中断 | 可中断获取（tryLockInterruptibly） |
| 锁释放 | 自动释放 | 必须手动释放（unlock） |
| 灵活性 | 单一/unfair | 多样化/fair可选 |
| 条件变量 | 无 | 多个Condition |
| 公平性 | 非公平 | 可选择公平/非公平 |

---

## 实战：验证锁状态变化

### 使用JOL查看锁状态

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

```java
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.vm.VM;

public class LockStateDemo {

    public static void main(String[] args) throws InterruptedException {
        System.out.println(VM.current().details());

        Object obj = new Object();

        // 无锁状态
        System.out.println("无锁状态:");
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());

        // 偏向锁（需要偏向锁延迟过后）
        Thread.sleep(5000);
        synchronized (obj) {
            System.out.println("偏向锁状态:");
            System.out.println(ClassLayout.parseInstance(obj).toPrintable());
        }

        // 轻量级锁
        Thread t = new Thread(() -> {
            synchronized (obj) {
                // 轻量级锁状态
            }
        });
        t.start();
        t.join();
    }
}
```

---

## 面试高频问题

**Q1：synchronized锁的升级过程？**

> 答：无锁→偏向锁（首次访问）→轻量级锁（竞争发生）→重量级锁（自旋失败）。锁只能升级不能降级。

**Q2：偏向锁的原理？**

> 答：首次访问时，通过CAS将线程ID写入Mark Word，之后同一线程进入同步块无需任何操作。如果有其他线程尝试获取，撤销偏向锁。

**Q3：轻量级锁的原理？**

> 答：在线程栈帧创建锁记录，复制Mark Word到锁记录，然后用CAS将Mark Word替换为指向锁记录的指针。释放时再CAS换回来。

**Q4：重量级锁的原理？**

> 答：通过ObjectMonitor实现，依赖操作系统Mutex。获取失败进入entryList阻塞，调用wait()进入waitSet等待。

**Q5：为什么synchronized效率不断提升？**

> 答：从偏向锁（无竞争省CAS）→轻量级锁（自旋避免阻塞）→自适应自旋（JVM动态调整），JVM不断优化锁的性能。

---

## 总结

synchronized 的底层原理涉及对象头的 Mark Word 和 Monitor 对象：
- **偏向锁**：消除无竞争下的同步开销
- **轻量级锁**：用CAS代替阻塞，自旋等待
- **重量级锁**：依赖Monitor，线程阻塞
- **锁升级**：只能升级不能降级，针对不同竞争程度优化
