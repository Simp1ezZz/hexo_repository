---
title: 深入理解JVM垃圾回收：GC算法与收集器对比
date: 2024-01-21 01:22:30
tags:
  - Java进阶
  - JVM
categories: 学习
keywords: JVM,GC,垃圾回收,GC算法,收集器
description: 图文详解GC算法原理与各收集器适用场景
cover:
---

## 前言

GC（Garbage Collection）是Java自动内存管理的核心机制。但什么对象需要回收？什么时候回收？如何回收？本文深入解析GC的各个环节。

## 如何判断对象已死？

### 引用计数法

**原理：** 对象被引用时计数器+1，引用失效时-1，计数器为0即回收

**问题：** 循环引用无法回收（Python使用此算法但通过其他机制解决）

```java
public class ReferenceCountingDemo {
    public Object instance;

    public static void main(String[] args) {
        ReferenceCountingDemo a = new ReferenceCountingDemo();
        ReferenceCountingDemo b = new ReferenceCountingDemo();
        // 循环引用 - 引用计数都不为0但已不可达
        a.instance = b;
        b.instance = a;
        // 移除引用
        a = null;
        b = null;
        // Java GC可以回收，C++如果只用引用计数则不能
    }
}
```

### 可达性分析算法（根搜索）

**原理：** 从GC Roots向下搜索，搜索路径称为"引用链"，不可达则标记为可回收

**GC Roots包括：**
- 虚拟机栈中引用的对象
- 方法区中静态属性引用的对象
- 方法区中常量引用的对象（字符串常量池）
- 本地方法栈中JNI引用的对象
- JVM内部引用（Class对象、异常对象等）
- 同步锁持有的对象
- 反映JVM内部情况的JMXBean、回调类、接口类

```java
public class GCRootsDemo {
    // 静态变量 - 方法区引用，可能是GC Root
    public static GCRootsDemo staticRef;

    public void test() {
        // 局部变量 - 虚拟机栈引用，可能是GC Root
        GCRootsDemo localRef = new GCRootsDemo();
        // localRef作为局部变量表中的引用，是当前栈帧的GC Root
    }
}
```

---

## 四种GC算法详解

### 1. 标记-清除算法（Mark-Sweep）

**流程：** 标记所有存活对象 → 清除所有未标记对象

**缺点：**
- 效率不稳定，标记和清除时间随对象数量增长
- 产生内存碎片，大对象分配可能失败

```
┌─────────────┬─────────────┬─────────────┐
│  已标记对象  │  未标记对象   │   空闲     │
│   (保留)     │   (清除)     │   内存     │
└─────────────┴─────────────┴─────────────┘
     ↓ 清除后
┌───────────────────────┬───────────────┐
│      已标记对象        │    内存碎片    │
└───────────────────────┴───────────────┘
```

### 2. 复制算法（Copying）

**原理：** 将内存分为两块，每次只用一块，GC时将存活对象复制到另一块，清理原区域

**优点：** 无内存碎片，实现简单，运行高效
**缺点：** 可用内存减半

**现代JVM应用：** 新生代Eden区（80%）+ Survivor区（10%×2）

```
         新生代内存布局
┌──────┬─────────────┬─────────────┐
│ Eden │  Survivor0  │  Survivor1  │
│ 80%  │    10%      │    10%      │
└──────┴─────────────┴─────────────┘
          ↓ Minor GC
┌──────┬─────────────┬─────────────┐
│ Eden │  存活对象    │   空闲      │
│      │  (复制过来)   │   区域     │
└──────┴─────────────┴─────────────┘
```

### 3. 标记-整理算法（Mark-Compact）

**流程：** 标记存活对象 → 整理到一端 → 清理边界外内存

**优点：** 无内存碎片，内存利用率高
**缺点：** 移动存活对象需要更新引用，stop-the-world时间较长

**适用场景：** 老年代

```
标记 → 整理前：[存活][存活][存活][未标记][未标记][存活]
整理后：[存活][存活][存活][存活][空闲][空闲]
```

### 4. 分代收集算法（Generational Collection）

**核心思想：** 对象生命周期不同，采用不同策略

| 分代 | 对象特点 | 算法 | 收集器 |
|------|---------|------|--------|
| 新生代 | 大量对象死亡，少量存活 | 复制算法 | Serial、ParNew、Parallel Scavenge |
| 老年代 | 对象存活率高 | 标记-整理/标记-清除 | Serial Old、Parallel Old、CMS |

---

## 七种经典垃圾收集器

### 新生代收集器

#### Serial（串行收集器）

**特点：** 单线程进行GC，stop-the-world，GC时暂停所有用户线程

```bash
-XX:+UseSerialGC  # 搭配 Serial Old
```

**适用：** 单核CPU、客户端模式、堆内存较小（<100MB）

**缺点：** GC时必须暂停所有线程，停顿时间长

#### ParNew（并行收集器）

**特点：** Serial的多线程版本，多CPU环境效率高，同样stop-the-world

```bash
-XX:+UseParNewGC  # 搭配 Serial Old
```

**适用：** 多核服务端，JDK7/8默认的新生代收集器（配合CMS）

**注意：** 使用`-XX:ParallelGCThreads`指定线程数，通常与CPU核心数相同

#### Parallel Scavenge（吞吐量优先）

**特点：** 关注吞吐量（运行用户代码时间/总时间），自适应调节

```bash
-XX:+UseParallelGC  # 搭配 Parallel Old
-XX:MaxGCPauseMillis=100  # 最大GC停顿时间（目标，不保证）
-XX:GCTimeRatio=99       # 吞吐量目标（1/(1+99)=1%时间用于GC）
```

**自适应调节：**
- `-XX:+UseAdaptiveSizePolicy` 开启后，自动调整Eden/Survivor比例
- 适合后台批处理任务

---

### 老年代收集器

#### Serial Old（串行老年代）

**特点：** Serial的老年代版本，标记-整理算法

**备选场景：**
- 与Parallel Scavenge配合
- 作为CMS的后备收集器

#### Parallel Old（并行老年代）

**特点：** Parallel Scavenge的老年代版本，标记-整理

```bash
-XX:+UseParallelOldGC  # 吞吐量优先组合
```

**适用：** 注重吞吐量的场景，如后台批处理

#### CMS（并发标记清除）

**目标：** 最小化停顿时间，适合Web应用

**阶段：**

| 阶段 | 说明 | 停顿 |
|------|------|------|
| 初始标记(Initial Mark) | 标记GC Roots直接引用的对象 | stop-the-world |
| 并发标记(Concurrent Mark) | 遍历GC Roots引用链 | 并发进行 |
| 重新标记(Remark) | 修正并发标记期间变动 | stop-the-world |
| 并发清除(Concurrent Sweep) | 清除未标记对象 | 并发进行 |

```bash
-XX:+UseConcMarkSweepGC  # 搭配 ParNew 或 Serial
-XX:CMSInitiatingOccupancyFraction=68  # 老年代占用68%时触发CMS
```

**优点：** 并发收集，低停顿
**缺点：**
- 对CPU敏感，并发阶段占用CPU资源
- 无法处理浮动垃圾（并发清理阶段新产生的垃圾）
- 产生内存碎片

**浮动垃圾问题：**
```java
// 并发标记阶段产生的对象，在本次GC中无法回收
// 只能等下次GC清理
```

---

### G1（Garbage-First）

**设计思想：** 将堆划分为多个大小相等的Region（1MB-32MB），跟踪各Region垃圾比例，优先回收垃圾比例最高的Region

**特点：**
- 兼具并发与并行
- 分代收集，但分的是Region而非连续空间
- 可预测停顿（通过 `-XX:MaxGCPauseMillis` 设置目标）
- 不产生内存碎片

```bash
-XX:+UseG1GC  # JDK9+默认
-XX:G1HeapRegionSize=2    # Region大小2MB
-XX:MaxGCPauseMillis=200  # 目标停顿时间200ms
-XX:TargetSurvivorRatio=50  # Survivor区比例
```

**G1的Region结构：**
```
┌────────┬────────┬────────┬────────┐
│ Eden   │ Eden   │ Eden   │  S     │
│   R    │   R    │   R    │ (Survivor) │
├────────┼────────┼────────┼────────┤
│   O    │   O    │   O    │  O     │
│ (Old)  │ (Old)  │ (Old)  │ (Old)  │
├────────┼────────┼────────┼────────┤
│ Humongous (大对象，超过Region 50%的对象) │
└────────┴────────┴────────┴────────┘
```

**适用：** JDK9+默认，服务端大堆（>6GB）场景

---

## 收集器对比总结

| 收集器 | 线程 | 停顿 | 适用场景 | 算法 |
|--------|------|------|---------|------|
| Serial | 单线程 | stop-the-world | 客户端/单核 | 复制 |
| ParNew | 多线程 | stop-the-world | 多核服务端配合CMS | 复制 |
| Parallel Scavenge | 多线程 | stop-the-world | 吞吐量优先 | 复制 |
| Serial Old | 单线程 | stop-the-world | 客户端/备选 | 标记-整理 |
| Parallel Old | 多线程 | stop-the-world | 吞吐量优先 | 标记-整理 |
| CMS | 多线程 | 短暂停顿 | 低延迟需求 | 标记-清除 |
| G1 | 多线程 | 可预测停顿 | 大堆/低延迟 | 标记-整理 |

---

## 如何选择收集器？

**原则：**

| 场景 | 推荐收集器组合 |
|------|---------------|
| 停顿敏感 + 低延迟 | CMS 或 G1 |
| 吞吐量优先（后台批处理） | Parallel Scavenge + Parallel Old |
| 简单客户端（单核/小内存） | Serial + Serial Old |
| JDK9+ | G1（默认） |

**JVM参数配置示例：**
```bash
# Web应用，追求低延迟
java -Xms4g -Xmx4g -XX:+UseG1GC -XX:MaxGCPauseMillis=200

# 批处理，追求高吞吐
java -Xms4g -Xmx4g -XX:+UseParallelGC -XX:+UseParallelOldGC
```

---

## 实战：GC日志分析

### 开启GC日志

```bash
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/var/log/gc.log
```

### 分析Minor GC日志

```
2026-04-02T10:15:30.123+0800: [GC (Allocation Failure)
  Desired survivor size 5242880 bytes, new threshold 15 (max 15)
  - age 1: 123456 bytes, 123456 total
 eden space 65536K, 80% used
  from space 8192K, 50% used
  to space 8192K, 0% used
  Metaspace: 12345K used]
```

### 分析Full GC日志

```
2026-04-02T10:15:35.456+0800: [Full GC (Allocation Failure)
  -- 8000K->2048K(8192K), 0.820 secs]
  cms: 8000K->2048K(8192K), 0.820 secs]
  [CMS-remark: 0.123s]
  [CMS-concurrent-sweep: 0.456s]
```

**观察指标：**
- GC频率：频繁GC说明内存分配过快
- GC耗时：长时间停顿影响用户体验
- 内存使用率：老年代持续增长说明可能有内存泄漏

---

## 面试高频问题

**Q1：对象分配流程？**

> 答：对象优先在Eden区分配。大对象直接进入老年代（`-XX:PretenureSizeThreshold`）。长期存活对象进入老年代（年龄达到`-XX:MaxTenuringThreshold`）。空间分配担保Minor GC。

**Q2：Minor GC vs Full GC？**

> 答：Minor GC清理新生代，频率高，停顿短。Full GC清理整个堆和方法区，停顿长，应尽量避免。

**Q3：为什么老年代不用复制算法？**

> 答：老年代对象存活率高（80%-98%），复制操作代价太大。标记-整理算法只需移动少量存活对象。

**Q4：G1和CMS区别？**

> 答：CMS是老年代并发收集器，标记-清除算法，会产生碎片但停顿短。G1是整堆收集器，标记-整理算法，不产生碎片，可预测停顿。JDK9+默认G1。

**Q5：什么是stop-the-world？**

> 答：GC时暂停所有应用线程，所有线程停止执行。CMS和G1通过并发处理减少停顿时间，但无法完全消除。

---

## 总结

GC是JVM自动内存管理的核心：
- **判断对象存活**：可达性分析优于引用计数（解决循环引用问题）
- **四种算法**：标记-清除（CMS）、复制（新生代）、标记-整理（老年代/G1）、分代收集（组合拳）
- **七种收集器**：Serial、ParNew、Parallel Scavenge、Serial Old、Parallel Old、CMS、G1
- **选择原则**：低延迟选G1/CMS，高吞吐选Parallel，关注场景而非单一指标
