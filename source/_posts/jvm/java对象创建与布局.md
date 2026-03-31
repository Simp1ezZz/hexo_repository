---
title: Java对象的创建、布局与访问定位
date: 2023-03-02 23:40:16
tags:
  - Java进阶
  - JVM
categories: 学习
keywords: JVM,对象创建,对象布局,对象头,Mark Word
description: 深入理解new Object()到底发生了什么，对象在内存中如何存储
cover:
---

## 前言

当我们执行 `User user = new User();` 时，JVM内部发生了什么？对象在堆内存中如何存储？如何通过引用找到对象？本文深入讲解对象创建的每个细节。

## 对象创建流程

### Step 1：类加载检查

执行 `new` 指令时，首先检查常量池中是否能定位到该类的符号引用，并检查类是否已被加载、解析和初始化。

```java
// 伪代码：new操作对应的JVM指令处理
if (User.class 未加载) {
    执行类加载：加载 → 验证 → 准备 → 解析 → 初始化
}
```

### Step 2：分配内存

类加载检查通过后，JVM为对象分配内存。内存大小在类加载后确定。

#### 分配方式一：指针碰撞（Bump The Pointer）

**适用：** 堆内存规整（无碎片），配合Serial、ParNew等带压缩的收集器

```
堆内存：
┌───────────────────┬───────────────────┐
│      已用区域      │      空闲区域      │
└───────────────────┴───────────────────┘
                        ↑
                    指针
```

分配时，指针向空闲方向移动对象大小。

#### 分配方式二：空闲列表（Free List）

**适用：** 堆内存不规整（有碎片），配合CMS等不压缩的收集器

```bash
# JVM维护一个列表记录哪些内存块可用
# 分配时从列表中找到足够大的块，修改列表记录
```

#### 并发安全问题

对象创建在并发环境下可能存在线程安全问题（两个线程同时分配到同一块内存）。

**解决方案1：CAS + 失败重试**
```java
// 悲观策略：加锁保证原子性
while (!CAS(指针, expected, newPointer)) {
    // 重试，直到分配成功
}
```

**解决方案2：本地线程分配缓冲（TLAB）**
```java
// 每个线程在Eden区预先分配一小块区域
// 线程优先在TLAB分配，TLAB不够再cas分配到公共区域

// JVM参数
-XX:+UseTLAB          // 启用TLAB（默认启用）
-XX:TLABSize=1024k    // TLAB大小
-XX:-UseTLAB          // 禁用TLAB
```

### Step 3：初始化零值

内存分配完成后，JVM将分配到的内存空间初始化为零值（不包括对象头）。

```java
public class User {
    int age;           // 零值为0
    boolean active;     // 零值为false
    Object reference;   // 零值为null
}
// 这一步保证了实例字段可以不赋初始值就直接使用
```

### Step 4：设置对象头

将对象的运行时数据填充到对象头中。

```java
// 伪代码：设置对象头
object.header.markWord = 对象的hashCode、GC年龄、锁信息等;
object.header.klassPointer = 指向User.class的指针;
object.header.arrayLength = (如果是数组)数组长度;
```

### Step 5：执行构造器

执行 `<init>` 方法（构造函数），完成对象初始化。

```java
public class User {
    int age = 10;  // 这里会在<init>中赋值

    public User() {
        this.age = 20;  // 构造函数中再次赋值
    }
}
// 对象的age最终值为20
```

---

## 对象的内存布局

在HotSpot虚拟机中，对象内存布局分为三部分：**对象头（Header）**、**实例数据（Instance Data）**、**对齐填充（Padding）**。

```
┌─────────────────────────────────────┐
│           对象头 Object Header        │
│  ┌─────────────┬───────────────────┐ │
│  │  Mark Word  │  Klass Pointer    │ │
│  │  (8字节)    │   (4/8字节)       │ │
│  └─────────────┴───────────────────┘ │
├─────────────────────────────────────┤
│          实例数据 Instance Data       │
│  父类字段 → 子类字段                  │
│  按字段大小降序排列                   │
├─────────────────────────────────────┤
│          对齐填充 Padding             │
│  填充到8字节的整数倍                  │
└─────────────────────────────────────┘
```

### 对象头（Object Header）

#### markWord（标记字段）

存储对象的哈希码、GC年龄、锁状态等信息。

| 锁状态 | 存储内容（32位JVM） | 分代年龄位 | 标志位 |
|--------|---------------------|-----------|--------|
| 无锁 | 对象hashCode(25位) + GC年龄(4位) | 4bit | 01 |
| 偏向锁 | ThreadID(23位) + epoch(2位) + GC年龄(4位) | - | 01 |
| 轻量级锁 | 指向栈中锁记录的指针 | - | 00 |
| 重量级锁 | 指向互斥量(Monitor)的指针 | - | 10 |
| GC标记 | - | - | 11 |

#### klassPointer（类型指针）

指向方法区中Klass对象的指针，JVM通过这个指针确定对象是哪个类的实例。

```java
// klassPointer大小
// 32位JVM：4字节
// 64位JVM（未开启指针压缩）：8字节
// 64位JVM（开启指针压缩）：8字节，但会压缩为4字节
-XX:+UseCompressedOops  # 开启指针压缩（默认）
```

#### arrayLength（数组长度）

如果对象是数组，对象头中还会存储数组长度（4字节）。

```java
int[] arr = new int[10];
// 对象头中额外存储 arrayLength = 10
```

### 实例数据（Instance Data）

存储对象的实际字段数据。

```java
public class User {
    long id;           // 8字节
    int age;           // 4字节
    boolean active;    // 1字节
    Object reference;   // 4/8字节（引用类型）
}
// JVM会按8字节对齐，所以上述字段实际占用：
// 8 + 4 + 1 + 3(padding) + 4/8 = 24/28字节
```

**字段重排序规则：**
- 相同宽度的字段放在一起
- 父类字段在前，子类字段在后
- hotSpot默认按大小降序：long/double → int/float → short/char → byte/boolean → reference

### 对齐填充（Padding）

JVM要求对象大小必须是8字节的整数倍，不足时填充。

---

## 对象访问定位

### 方式一：句柄访问

JVM在堆中划分出一块区域作为句柄池，reference存储句柄地址，句柄包含对象实例数据和类型数据的地址。

```
┌──────────────────────────────────────────────────────────┐
│                        堆                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │              句柄池                                │   │
│  │  ┌─────────────────┬─────────────────────────┐    │   │
│  │  │ 实例数据指针     │    类型数据指针         │    │   │
│  │  │ (指向对象实例)   │    (指向方法区Klass)   │    │   │
│  │  └─────────────────┴─────────────────────────┘    │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↑                              │
│                        reference                        │
└──────────────────────────────────────────────────────────┘
```

**优点：** 对象移动时只需修改句柄，reference本身不变
**缺点：** 多一次指针访问

### 方式二：直接指针访问（HotSpot采用）

reference直接存储对象地址，减少一次指针访问。HotSpot采用这种方式。

```
┌──────────────────────────────────────────────────────────┐
│                        堆                                │
│  ┌──────────────────────────────────────────────────┐   │
│  │  对象头 │ 实例数据 │                              │   │
│  │  ┌──────┴─────────┴───────────────────────┐      │   │
│  │  │ markWord    │  klassPointer           │      │   │
│  │  └──────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────┘   │
│                          ↑                              │
│                        reference                         │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│                     方法区                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Klass对象                            │   │
│  │  类信息、元数据、静态变量...                       │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**优点：** 访问速度快（减少一次指针跳转）
**缺点：** 对象移动时需要修改reference

---

## 实战验证：使用JOL查看对象布局

### Maven依赖

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

### 查看对象布局

```java
import org.openjdk.jol.info.ClassLayout;
import org.openjdk.jol.vm.VM;

public class ObjectLayoutDemo {
    public static void main(String[] args) {
        // 打印JVM信息
        System.out.println(VM.current().details());

        // 分析普通对象
        Object obj = new Object();
        System.out.println(ClassLayout.parseInstance(obj).toPrintable());

        // 分析带字段的对象
        User user = new User();
        System.out.println(ClassLayout.parseInstance(user).toPrintable());

        // 分析数组
        int[] arr = new int[10];
        System.out.println(ClassLayout.parseInstance(arr).toPrintable());
    }
}

class User {
    long id;
    int age;
    boolean active;
    Object reference;
}
```

### 输出示例

```
# VM mode: 64 bits
# Compressed references (oops): 3-bit shift
# Compressed class pointers: 3-bit shift

java.lang.Object object internals:
OFFSET  TYPE        DESCRIPTION
0       object header  [Mark Word]
8       object header  [Klass Pointer]
16      object header  [padding]
Total: 24 bytes

# 说明：Object对象占用24字节
# 8字节Mark Word + 4字节Klass Pointer + 4字节padding + 8字节对齐
```

---

## 面试高频问题

**Q1：对象创建过程？**

> 答：类加载检查→分配内存（指针碰撞/空闲列表+TLAB）→初始化零值→设置对象头→执行构造器。

**Q2：对象头包含哪些信息？**

> 答：Mark Word（哈希码、GC年龄、锁状态）、Klass Pointer（类型指针，指向方法区类信息）、arrayLength（如果是数组）。

**Q3：对象一定分配在堆上吗？**

> 答：不一定。TLAB（Thread Local Allocation Buffer）是Eden区的线程私有区域。逃逸分析发现对象不逃逸出线程，可以进行栈上分配。但对象本身在堆，引用在栈。

**Q4：对象的内存布局？**

> 答：对象头（Mark Word + Klass Pointer + arrayLength）、实例数据（字段值，按降序排列）、对齐填充（补齐到8字节倍数）。

**Q5：为什么需要对齐填充？**

> 答：JVM要求对象起始地址是8字节的倍数。对齐填充保证这一点，既为了CPU高效访问，也为了对象大小可预测。

**Q6：Mark Word存储什么？**

> 答：对象的hashCode、GC分代年龄、锁状态信息。不同锁状态下Mark Word内容不同，无锁、偏向锁、轻量级锁、重量级锁、GC标记各不相同。

---

## 总结

理解对象创建与内存布局是理解JVM的基石：
- **创建流程**：类加载检查→内存分配（TLAB+CAS）→零值初始化→设置对象头→执行构造器
- **内存布局**：对象头（Mark Word + Klass Pointer）+ 实例数据 + 对齐填充
- **对象访问**：HotSpot使用直接指针访问，速度更快
- **实战工具**：JOL可以直观查看对象布局
