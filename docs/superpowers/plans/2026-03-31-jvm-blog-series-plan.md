# JVM系列博客实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完成JVM系列5篇深度技术博客，每篇面向进阶Java开发者

**Architecture:** 每个系列独立成篇，每篇遵循"场景引入 → 原理剖析 → 代码验证 → 面试总结"结构

**Tech Stack:** Hexo博客框架、Markdown格式

---

## 文件结构

每篇文章创建在 `source/_posts/jvm/` 目录下：
- `source/_posts/jvm/java-memory区域详解.md`
- `source/_posts/jvm/jvm-gc垃圾回收.md`
- `source/_posts/jvm/java类加载机制.md`
- `source/_posts/jvm/jvm性能调优实战.md`
- `source/_posts/jvm/java对象创建与布局.md`

---

## 文章1：Java内存区域详解

**Files:**
- Create: `source/_posts/jvm/java-memory区域详解.md`

- [ ] **Step 1: 撰写开头部分**

```markdown
---
title: Java内存区域详解：堆、栈、方法区深度剖析
date: 2026-03-31 10:00:00
tags:
  - Java进阶
  - JVM
categories: 学习
keywords: JVM,内存区域,堆,栈,方法区
description: 深入理解JVM内存区域划分与各区域作用
cover:
---

## 前言

为什么一个看似简单的 `new Object()` 会占用多大内存？**程序计数器、虚拟机栈、本地方法栈、堆、方法区** 各存储什么数据？本文带你彻底理解JVM内存划分。

## 运行时数据区详解

### 程序计数器（Program Counter Register）

**作用：** 当前线程执行的字节码行号指示器

**特点：**
- 线程私有，每个线程独立拥有
- 唯一一个不会出现 OutOfMemoryError 的区域
- 分支、循环、跳转、异常处理都依赖它

**代码示例：**
```java
public class PCRegisterDemo {
    public static void main(String[] args) {
        int a = 1;  // 行号0
        int b = 2;  // 行号1
        int c = a + b;  // 行号2
        System.out.println(c);  // 行号3
    }
}
```

### Java虚拟机栈（JVM Stack）

**作用：** 存储方法调用栈帧，每个方法执行时创建一个栈帧

**栈帧结构：**
- 局部变量表：方法参数和局部变量
- 操作数栈：表达式计算中间结果
- 动态链接：指向运行时常量池的引用
- 返回地址：方法返回地址

**线程请求深度受-Xss参数控制，默认1MB**
```

- [ ] **Step 2: 撰写堆与方法区部分**

```markdown
### 堆（Heap）

**作用：** 所有对象实例和数组分配内存的区域

**特点：**
- 线程共享，整个 JVM 只有一个堆
- GC主要工作区域（新生代、老年代、永久代/元空间）
- 数组和对象永远存在堆中，局部变量表只存储引用

**JVM参数示例：**
```bash
-Xms512m -Xmx512m  # 堆初始和最大值为512MB
-Xmn256m           # 新生代大小256MB
-XX:NewRatio=2     # 老年代/新生代比例=2
```

### 方法区（Method Area）

**作用：** 存储类信息（类的元数据）、常量、静态变量、JIT编译后的代码

**演进历史：**
- JDK 7及之前：永久代（PermGen），使用JVM堆内存
- JDK 8+：元空间（Metaspace），使用本地内存

**代码验证方法区存储内容：**
```java
public class MethodAreaDemo {
    // 静态变量 - 方法区
    public static int staticVar = 100;
    // 常量 - 运行时常量池
    public static final int CONSTANT = 200;

    public void test() {
        // 局部变量 - 虚拟机栈
        int localVar = 300;
    }
}
```

**验证元空间大小：**
```bash
-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m
```

### 运行时常量池（Runtime Constant Pool）

**作用：** 方法区的一部分，存储编译期生成的各种字面量和符号引用

```java
public class ConstantPoolDemo {
    public static void main(String[] args) {
        // 字符串常量池验证
        String s1 = "hello";
        String s2 = "hello";
        String s3 = new String("hello");

        System.out.println(s1 == s2);  // true，字符串常量池
        System.out.println(s1 == s3);  // false，new在堆中创建
        System.out.println(s1 == s3.intern());  // true，intern()返回池中对象
    }
}
```
```

- [ ] **Step 3: 撰写直接内存与总结部分**

```markdown
### 直接内存（Direct Memory）

**作用：** NIO使用，DirectByteBuffer通过堆外内存提升IO效率

**特点：**
- 不属于JVM运行时数据区
- 通过 `ByteBuffer.allocateDirect()` 分配
- 受 `-XX:MaxDirectMemorySize` 限制

**示例：**
```java
// 分配1GB直接内存
ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024 * 1024);
```

---

## 内存区域对比总结

| 区域 | 线程私有/共享 | 作用 | 异常类型 |
|------|-------------|------|---------|
| 程序计数器 | 私有 | 字节码行号 | 无 |
| 虚拟机栈 | 私有 | 方法栈帧 | StackOverflowError / OOM |
| 本地方法栈 | 私有 | Native方法栈帧 | StackOverflowError / OOM |
| 堆 | 共享 | 对象/数组 | OOM |
| 方法区 | 共享 | 类信息/常量/静态变量 | OOM（JDK7 PermGen / JDK8 Metaspace） |

---

## 面试高频问题

**Q1：对象一定在堆中吗？**
> 答：不一定。TLAB（Thread Local Allocation Buffer）允许在Eden区为每个线程预留私有空间，small object可以在TLAB中分配而非堆的公共区域。

**Q2：方法区存放什么？**
> 答：类的元数据（Class对象）、运行时常量池、静态变量、JIT编译后的代码。JDK8后使用元空间实现。

**Q3：String.intern()有什么作用？**
> 答：将字符串放入字符串常量池，如果池中已存在则返回池中引用，否则添加后返回引用。可以用来节省内存或用于字符串比较优化。
```

- [ ] **Step 4: 创建目录并保存文件**

```bash
mkdir -p source/_posts/jvm
# 保存以上内容到 source/_posts/jvm/java-memory区域详解.md
```

- [ ] **Step 5: Commit**

```bash
git add source/_posts/jvm/java-memory区域详解.md
git commit -m "feat(jvm): add Java内存区域详解 article"
```

---

## 文章2：深入理解JVM垃圾回收

**Files:**
- Create: `source/_posts/jvm/jvm-gc垃圾回收.md`

- [ ] **Step 1: 撰写GC算法基础部分**

```markdown
---
title: 深入理解JVM垃圾回收：GC算法与收集器对比
date: 2026-04-01 10:00:00
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
```

- [ ] **Step 2: 撰写四种GC算法详解**

```markdown
## 四种GC算法详解

### 1. 标记-清除算法（Mark-Sweep）

**流程：** 标记所有存活对象 → 清除所有未标记对象

**缺点：**
- 效率不稳定，标记和清除时间随对象数量增长
- 产生内存碎片，大对象分配可能失败

### 2. 复制算法（Copying）

**原理：** 将内存分为两块，每次只用一块，GC时将存活对象复制到另一块，清理原区域

**优点：** 无内存碎片，实现简单，运行高效
**缺点：** 可用内存减半

**现代JVM应用：** 新生代Eden区（80%）+ Survivor区（10%×2）

```bash
# 新生代内存布局演示
# |---Eden---|---Survivor0---|---Survivor1---|
#     80%         10%              10%
# 对象优先在Eden分配，Survivor用于复制算法的对象提升
```

### 3. 标记-整理算法（Mark-Compact）

**流程：** 标记存活对象 → 整理到一端 → 清理边界外内存

**优点：** 无内存碎片，内存利用率高
**缺点：** 移动存活对象需要更新引用，stop-the-world时间较长

**适用场景：** 老年代

### 4. 分代收集算法（Generational Collection）

**核心思想：** 对象生命周期不同，采用不同策略

| 分代 | 算法 | 收集器 |
|------|------|--------|
| 新生代（少量对象死亡） | 复制算法 | Serial、ParNew、Parallel Scavenge |
| 老年代（对象存活率高） | 标记-整理/标记-清除 | Serial Old、Parallel Old、CMS |

---

## 七种经典垃圾收集器

### 新生代收集器

#### Serial（串行收集器）
**特点：** 单线程进行GC，stop-the-world，GC时暂停所有用户线程

```bash
-XX:+UseSerialGC  # 搭配 Serial Old
```

**适用：** 单核CPU、客户端模式、堆内存较小（<100MB）

#### ParNew（并行收集器）
**特点：** Serial的多线程版本，多CPU环境效率高，同样stop-the-world

```bash
-XX:+UseParNewGC  # 搭配 Serial Old
```

**适用：** 多核服务端，JDK7/8默认的新生代收集器（配合CMS）

#### Parallel Scavenge（吞吐量优先）
**特点：** 关注吞吐量（运行用户代码时间/总时间），自适应调节

```bash
-XX:+UseParallelGC  # 搭配 Parallel Old
-XX:MaxGCPauseMillis=100  # 最大GC停顿时间（目标，不保证）
-XX:GCTimeRatio=99       # 吞吐量目标（1/(1+99)=1%时间用于GC）
```
```

- [ ] **Step 3: 撰写老年代收集器与G1**

```markdown
### 老年代收集器

#### Serial Old（串行老年代）
**特点：** Serial的老年代版本，标记-整理算法

#### Parallel Old（并行老年代）
**特点：** Parallel Scavenge的老年代版本，标记-整理

```bash
-XX:+UseParallelOldGC  # 吞吐量优先组合
```

#### CMS（并发标记清除）

**阶段：**
1. **初始标记（Initial Mark）：** 标记GC Roots直接引用的对象，stop-the-world
2. **并发标记（Concurrent Mark）：** 遍历GC Roots引用链，并发进行
3. **重新标记（Remark）：** 修正并发标记期间变动，stop-the-world
4. **并发清除（Concurrent Sweep）：** 清除未标记对象，并发进行

**优点：** 并发收集，低停顿
**缺点：**
- 对CPU敏感，并发阶段占用CPU资源
- 无法处理浮动垃圾（并发清理阶段新产生的垃圾）
- 产生内存碎片

```bash
-XX:+UseConcMarkSweepGC  # 搭配 ParNew 或 Serial
-XX:CMSInitiatingOccupancyFraction=68  # 老年代占用68%时触发CMS
```

### G1（Garbage-First）

**设计思想：** 将堆划分为多个大小相等的Region（1MB-32MB），跟踪各Region垃圾比例，优先回收垃圾比例最高的Region

**特点：**
- 兼具并发与并行
- 分代收集，但不分新生代老年代连续空间
- 可预测停顿（通过 -XX:MaxGCPauseMillis 设置目标）

```bash
-XX:+UseG1GC  # JDK9+默认
-XX:G1HeapRegionSize=2    # Region大小2MB
-XX:MaxGCPauseMillis=200  # 目标停顿时间200ms
```

**适用：** JDK9+默认，服务端大堆（>6GB）场景
```

- [ ] **Step 4: 撰写收集器对比与总结**

```markdown
---

## 收集器对比总结

| 收集器 | 线程 | 停顿 | 适用场景 | 备注 |
|--------|------|------|---------|------|
| Serial | 单线程 | stop-the-world | 客户端/单核 | 简单高效 |
| ParNew | 多线程 | stop-the-world | 多核服务端 | CMS默认搭档 |
| Parallel Scavenge | 多线程 | stop-the-world | 吞吐量优先 | 自适应调节 |
| Serial Old | 单线程 | stop-the-world | 客户端/备选 | 标记-整理 |
| Parallel Old | 多线程 | stop-the-world | 吞吐量优先 | 标记-整理 |
| CMS | 多线程 | 短暂停顿 | 低延迟需求 | 并发标记清除 |
| G1 | 多线程 | 可预测停顿 | 大堆/低延迟 | JDK9+默认 |

## 如何选择收集器？

**原则：**
- 停顿敏感 + 低延迟 → CMS 或 G1
- 吞吐量优先 → Parallel Scavenge + Parallel Old
- 简单客户端 → Serial + Serial Old
- JDK9+ → 直接使用 G1

## 面试高频问题

**Q1：对象分配流程？**
> 答：对象优先在Eden区分配。大对象直接进入老年代（-XX:PretenureSizeThreshold）。长期存活对象进入老年代（年龄达到-XX:MaxTenuringThreshold）。空间分配担保 Minor GC。

**Q2：Minor GC vs Full GC？**
> 答：Minor GC清理新生代，频率高，停顿短。Full GC清理整个堆和方法区，停顿长，应尽量避免。

**Q3：为什么老年代不用复制算法？**
> 答：老年代对象存活率高（80%-98%），复制操作代价太大。标记-整理算法只需移动少量存活对象。
```

- [ ] **Step 5: 保存文件并commit**

```bash
# 保存到 source/_posts/jvm/jvm-gc垃圾回收.md
git add source/_posts/jvm/jvm-gc垃圾回收.md
git commit -m "feat(jvm): add JVM垃圾回收 article"
```

---

## 文章3：类加载机制

**Files:**
- Create: `source/_posts/jvm/java类加载机制.md`

- [ ] **Step 1: 撰写类加载流程与双亲委派**

```markdown
---
title: 类加载机制：双亲委派模型与自定义类加载器
date: 2026-04-02 10:00:00
tags:
  - Java进阶
  - JVM
categories: 学习
keywords: JVM,类加载,双亲委派,ClassLoader
description: 深入理解Java类加载机制与双亲委派模型原理
cover:
---

## 前言

当我们 `new Object()` 时，Object.class 是如何被加载到JVM的？类加载器如何保证类的唯一性？自定义类加载器有什么作用？本文深入解析类加载机制。

## 类加载的五个阶段

### 1. 加载（Loading）

**任务：**
- 通过类的全限定名获取类的二进制字节流
- 将字节流转化为方法区的运行时数据结构
- 在堆中生成对应的Class对象作为访问入口

**类加载源：**
- Class文件（本地磁盘）
- JAR/WAR包
- 网络获取（Applet）
- 动态代理生成（ProxyGenerator）
- 其他文件（.jsp转换）

### 2. 验证（Verification）

**目的：** 确保Class文件字节流符合JVM规范，不会危害JVM安全

**阶段：**
- 文件格式验证（魔数、版本号等）
- 元数据验证（语法语义分析）
- 字节码验证（数据流、控制流分析）
- 符号引用验证（常量池中引用是否存在）

### 3. 准备（Preparation）

**任务：** 为类变量（static）分配内存并设置初始值

```java
public class PrepareDemo {
    public static int value = 100;  // 准备阶段 value=0，初始化阶段 value=100
    public static final int CONST = 200;  // 常量在准备阶段直接赋值为200
}
```

### 4. 解析（Resolution）

**任务：** 将符号引用替换为直接引用

**符号引用：** 用字符串表示的类/方法/字段引用，不定位实际内存地址
**直接引用：** 实际的内存地址/偏移量

**解析时机：** 解析可能在初始化之后（动态绑定）

### 5. 初始化（Initialization）

**任务：** 执行类构造器 `<clinit>()`，真正赋值为初始值

**触发条件：**
- new对象、读取/设置静态字段（final除外）、调用静态方法
- 反射 `Class.forName()`
- 初始化子类时父类先初始化
- JVM启动的启动类（main所在类）
```

- [ ] **Step 2: 撰写双亲委派模型与代码实现**

```markdown
## 双亲委派模型（Parents Delegation Model）

### 核心原理

当类加载器收到加载请求时，首先委派给父类加载器处理，父类再委派其父类，直到Bootstrap ClassLoader。只有父加载器无法完成时，才自己尝试加载。

### 加载器层次

```
┌─────────────────────────────────────────────┐
│     Bootstrap ClassLoader (启动类加载器)      │
│   加载 JAVA_HOME/lib 中的核心类库（rt.jar）    │
├─────────────────────────────────────────────┤
│     Extension ClassLoader (扩展类加载器)        │
│    加载 JAVA_HOME/lib/ext 中的扩展类          │
├─────────────────────────────────────────────┤
│     Application ClassLoader (应用类加载器)     │
│ 加载 classpath 中的用户类（-classpath指定）    │
├─────────────────────────────────────────────┤
│     自定义ClassLoader                        │
│   用户自定义的类加载器（可打破双亲委派）         │
└─────────────────────────────────────────────┘
```

### 为什么需要双亲委派？

**1. 防止核心类被篡改**
```java
// 用户自定义java.lang.String
package java.lang;
public class String {
    public static void main(String[] args) {
        System.out.println("我的String");
    }
}
```
> 由于双亲委派，java.lang.String由Bootstrap加载，用户自定义的String永远不会被加载。保证了Java核心API的安全性。

**2. 保证类的唯一性**
> 父加载器加载的类对子加载器可见，子加载器加载的类对父加载器不可见。因此用户类永远看不到自己编写的java.lang.Object等类。

### 双亲委派实现

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 1. 检查类是否已加载
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            // 2. 父加载器优先加载
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
                // 3. 父为null，尝试Bootstrap加载器
                c = findBootstrapClassOrNull(name);
            }
        }
        // 4. 父加载不了，自己加载
        if (c == null) {
            c = findClass(name);
        }
        // 5. 解析（可选）
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```
```

- [ ] **Step 3: 撰写自定义类加载器与打破双亲委派**

```markdown
## 自定义类加载器

### 适用场景

- 热部署（Tomcat/OSGI）
- 字节码加密/解密
- 动态类增强（埋点、AOP）
- 隔离不同版本的同名类（数据库驱动驱动商驱动）

### 实现步骤

1. 继承 `java.lang.ClassLoader`
2. 重写 `findClass()` 方法
3. 调用 `defineClass()` 将字节数组转为Class对象

```java
public class MyClassLoader extends ClassLoader {

    private String classPath;

    public MyClassLoader(String classPath) {
        this.classPath = classPath;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = loadClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException(name);
        }
        return defineClass(name, classData, 0, classData.length);
    }

    private byte[] loadClassData(String name) {
        String fileName = classPath + File.separator
            + name.replace('.', File.separatorChar) + ".class";
        try (InputStream is = new FileInputStream(fileName);
             ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[1024];
            int len;
            while ((len = is.read(buffer)) != -1) {
                bos.write(buffer, 0, len);
            }
            return bos.toByteArray();
        } catch (IOException e) {
            return null;
        }
    }
}

// 使用
public class ClassLoaderTest {
    public static void main(String[] args) throws Exception {
        MyClassLoader loader = new MyClassLoader("/Users/test/classes");
        Class<?> clazz = loader.loadClass("com.example.Test");
        System.out.println(clazz.getClassLoader());
    }
}
```

## 打破双亲委派

### 为什么需要打破？

某些场景下需要子加载器先尝试加载：
- **Tomcat：** 多个Web应用使用不同版本的Spring，需要隔离
- **OSGI：** 模块化架构，每个模块有自己的类加载器
- **JDBC驱动：** DriverManager由Bootstrap加载，但驱动实现类由应用类加载器加载

### Tomcat打破方案

```java
// Tomcat的WebappClassLoaderBase.loadClass() 先自己尝试加载
// 失败后再调用父类加载器
// 实现了自己的类加载器层次，隔离不同Web应用
```

### SPI机制（Service Provider Interface）

```java
// JDBC驱动加载示例
// java.sql.DriverManager 由Bootstrap加载
// 但数据库驱动（如com.mysql.jdbc.Driver）由AppClassLoader加载
// DriverManager使用线程上下文类加载器（ContextClassLoader）加载驱动
// 这本身就是对双亲委派的"妥协"

public class DriverManager {
    static {
        // 使用线程上下文类加载器加载SPI实现
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        // 加载 jdbc driver 实现类
    }
}
```

---

## 面试高频问题

**Q1：类加载过程？**
> 答：加载→验证→准备→解析→初始化。验证检查格式和安全，准备分配内存并设初始值，解析将符号引用转直接引用，初始化执行\<clinit\>。

**Q2：双亲委派模型好处？**
> 答：保证类的唯一性和安全性。例如用户自定义的java.lang.String不会被加载，防止核心API被篡改。

**Q3：如何打破双亲委派？**
> 答：重写loadClass()方法，不优先调用父加载器。Tomcat、OSGI、JDBC SPI都打破了双亲委派。
```

- [ ] **Step 4: 保存文件并commit**

```bash
# 保存到 source/_posts/jvm/java类加载机制.md
git add source/_posts/jvm/java类加载机制.md
git commit -m "feat(jvm): add 类加载机制 article"
```

---

## 文章4：JVM性能调优实战

**Files:**
- Create: `source/_posts/jvm/jvm性能调优实战.md`

- [ ] **Step 1: 撰写JVM参数与调优目标**

```markdown
---
title: JVM性能调优实战：参数配置与问题排查
date: 2026-04-03 10:00:00
tags:
  - Java进阶
  - JVM
categories: 学习
keywords: JVM,性能调优,JVM参数,问题排查
description: 详解JVM调优参数配置与实际生产问题排查流程
cover:
---

## 前言

JVM调优是Java开发者进阶的必备技能。什么情况下需要调优？如何选择合适的GC收集器？频繁Full GC怎么办？本文从实际案例出发讲解JVM调优。

## 什么时候需要JVM调优？

**信号：**
- 应用响应缓慢，GC停顿时间长
- 频繁Full GC但堆内存并不紧张
- OOM频繁发生
- 吞吐量下降明显

## JVM参数分类

### 堆内存参数

```bash
# 初始和最大堆
-Xms512m          # 堆初始大小512MB
-Xmx512m          # 堆最大大小512MB（生产应设相同值避免heap resize）
-Xmn256m          # 新生代大小256MB
-XX:NewRatio=2    # 老年代/新生代比例=2，即老年代占2/3

# 新生代细粒度参数
-XX:SurvivorRatio=8  # Eden/Survivor=8，即Eden占新生代8/10
-XX:MaxTenuringThreshold=15  # 对象进入老年代年龄阈值

# 元空间参数（JDK8+）
-XX:MetaspaceSize=128m   # 元空间初始大小
-XX:MaxMetaspaceSize=256m # 元空间最大大小
```

### GC收集器参数

```bash
# Serial收集器
-XX:+UseSerialGC

# ParNew + CMS组合
-XX:+UseParNewGC
-XX:+UseConcMarkSweepGC

# G1（推荐JDK9+）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# Parallel Scavenge吞吐量优先
-XX:+UseParallelGC
-XX:+UseParallelOldGC
```

### 其他重要参数

```bash
# 线程栈大小
-Xss1m     # 线程栈1MB（默认1MB）

# OOM时输出堆Dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java_heap.hprof

# GC日志
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/var/log/gc.log

# 直接内存
-XX:MaxDirectMemorySize=256m
```

---

## 调优目标与策略

### 调优目标

1. **低延迟：** 避免GC长时间停顿影响用户体验
2. **高吞吐：** 最大化CPU利用率用于业务处理
3. **低内存：** 合理利用资源，控制成本

### 收集器选择策略

| 场景 | 推荐收集器 |
|------|-----------|
| 吞吐量优先（后台批处理） | Parallel Scavenge + Parallel Old |
| 低延迟优先（Web应用） | CMS 或 G1 |
| 大堆+JDK9+ | G1 |
| 简单客户端 | Serial |

### 内存分配策略

```java
// 示例：合理对象分配避免频繁GC
public class AllocationDemo {
    // 避免重复创建大对象
    private static List<byte[]> cached = new ArrayList<>();

    public static void main(String[] args) {
        // 不要这样写 - 每次循环创建1MB对象，很快占满新生代
        // for (int i = 0; i < 1000; i++) {
        //     byte[] bytes = new byte[1024 * 1024];
        // }

        // 正确做法 - 复用对象
        byte[] buffer = new byte[1024 * 1024];
        for (int i = 0; i < 1000; i++) {
            Arrays.fill(buffer, (byte) i);
            process(buffer);
        }
    }
}
```
```

- [ ] **Step 2: 撰写问题排查工具与实战案例**

```markdown
## 常用排查工具

### jps - Java进程状态

```bash
jps -l        # 显示进程ID和主类名
jps -v        # 显示JVM参数
jps -m        # 显示传递给main方法的参数
```

### jstat - JVM统计信息

```bash
# 查看类加载统计（1000ms间隔，共10次）
jstat -class <pid> 1000 10

# 查看GC统计（S0C S1C S0U S1U EC EU OC OU MC MU CCSC CCSU YGC YGCT FGC FGCT GCT）
jstat -gc <pid> 1000 5

# 查看编译统计
jstat -compiler <pid>

# 示例输出：
#  S0C    S1C    S0U    S1U     EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC   YGCT    FGC    FGCT     GCT
# 1024.0 1024.0  0.0    0.0   8192.0   1024.0   20480.0     10240.0   4864.0 4690.0 512.0  475.0     5    0.123     1    0.045    0.168
```

### jmap - 内存映射

```bash
# 查看堆占用（按对象大小排序，前20个）
jmap -histo <pid> | head -30

# 导出堆dump文件
jmap -dump:format=b,file=/var/log/heap.hprof <pid>

# 查看堆配置
jmap -heap <pid>
```

### jstack - 线程栈

```bash
# 导出线程栈
jstack <pid> > /var/log/thread.log

# 查找死锁
jstack -l <pid>
```

### arthas - 阿里诊断工具（推荐）

```bash
# 启动arthas
java -jar arthas-boot.jar

# 查看dashboard（CPU、内存、线程、GC概况）
dashboard

# 反编译类
jad com.example.MyService

# 动态修改日志级别
logger -c com.example.MyService --name ROOT --level DEBUG

# 方法监控
watch com.example.MyService methodName '{params, returnObj}' -x 3
```
```

## 实战案例：频繁Full GC排查

### 症状
```
[Full GC (Allocation Failure) -- 耗时800ms]
[CMS: remark较强长停顿 820ms]
```

### 排查步骤

**Step 1: 查看GC日志**
```
GC日志显示：老年代使用率92%触发CMS，浮动垃圾来不及清理
```

**Step 2: jmap分析堆对象**
```bash
jmap -histo <pid> | head -50

# 发现：
#  num     #instances         #bytes  class name
# ----  ---------  -------  ---------------
#    1:         12345      8912345  [Ljava.lang.Object;
#    2:          8900      5678900  com.example.CacheEntry  ← 缓存对象过多
```

**Step 3: 代码定位**
```java
// 发现代码问题：缓存没有上限
public class CacheService {
    private Map<String, Object> cache = new HashMap<>();

    public void put(String key, Object value) {
        cache.put(key, value);  // 无限增长
    }
}

// 修复：使用带LRU的缓存
public class CacheService {
    private Map<String, Object> cache = new LinkedHashMap<String, Object>(100, 0.75f, true) {
        @Override
        protected boolean removeEldestEntry(Map.Entry eldest) {
            return size() > 10000;  // 超过10000条自动移除最老的
        }
    };
}
```

**Step 4: 调整JVM参数**
```bash
# 减少CMS触发频率
-XX:CMSInitiatingOccupancyFraction=70  # 从68%调整到70%
-XX:+UseCMSInitiatingOccupancyOnly
```
```

- [ ] **Step 3: 撰写常见问题解决方案**

```markdown
---

## 常见问题与解决方案

### 问题1：OutOfMemoryError: Java heap space

**原因：** 对象分配过多，内存泄漏

**排查：**
```bash
jmap -dump:format=b,file=heap.hprof <pid>
# 使用MAT/JProfiler分析堆dump
```

**解决：**
- 增加堆内存 `-Xmx`
- 修复内存泄漏（检查HashMap等容器是否有增无删除）

### 问题2：OutOfMemoryError: Metaspace

**原因：** 类太多（动态代理生成、CGLIB增强）

**排查：**
```bash
jstat -gc <pid>
# MC（Metaspace Capacity）接近MU（Metaspace Used）
```

**解决：**
```bash
-XX:MaxMetaspaceSize=512m  # 增大元空间
# 或修复：减少CGLIB使用，升级框架
```

### 问题3：GC停顿时间过长

**原因：** 大对象直接进入老年代，频繁Full GC

**解决：**
```bash
# 启用G1，设置停顿目标
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200

# 减少对象大小
# 优化代码，避免大对象创建
```

### 问题4：线程栈溢出（StackOverflowError）

**原因：** 递归调用层次过深，线程栈太小

**解决：**
```bash
# 增大栈大小
-Xss2m

# 修复递归代码
public class RecursionDemo {
    // 递归改为循环
    public int sum(int n) {
        int result = 0;
        for (int i = 1; i <= n; i++) {
            result += i;
        }
        return result;
    }
}
```

---

## 面试高频问题

**Q1：如何定位线上OOM？**
> 答：添加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path.hprof`，然后用MAT/JProfiler分析dump文件，找出占用内存最大的对象。

**Q2：如何减少GC次数？**
> 答：减少对象创建、对象复用、增大新生代、选择合适收集器。使用对象池（StringBuilder、数据库连接池）。

**Q3：G1和CMS区别？**
> 答：CMS是老年代并发收集器，标记-清除算法，会产生碎片。G1是整堆收集，标记-整理，不产生碎片，可预测停顿。JDK9+默认G1。
```

- [ ] **Step 4: 保存文件并commit**

```bash
# 保存到 source/_posts/jvm/jvm性能调优实战.md
git add source/_posts/jvm/jvm性能调优实战.md
git commit -m "feat(jvm): add JVM性能调优实战 article"
```

---

## 文章5：Java对象创建与内存布局

**Files:**
- Create: `source/_posts/jvm/java对象创建与布局.md`

- [ ] **Step 1: 撰写对象创建流程**

```markdown
---
title: Java对象的创建、布局与访问定位
date: 2026-04-04 10:00:00
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
// 伪代码
if (User.class 未加载) {
    执行类加载：加载 → 验证 → 准备 → 解析 → 初始化
}
```

### Step 2：分配内存

类加载检查通过后，JVM为对象分配内存。内存大小在类加载后确定。

#### 分配方式一：指针碰撞（Bump The Pointer）

**适用：** 堆内存规整（无碎片），配合Serial、ParNew等带压缩的收集器

```bash
# 堆内存绝对规整，已用和空闲分居两侧
# |--------已用--------|--------空闲---------|
#                        ↑
#                    指针
# 分配时，指针向空闲方向移动对象大小
```

#### 分配方式二：空闲列表（Free List）

**适用：** 堆内存不规整（有碎片），配合CMS等不压缩的收集器

```bash
# JVM维护一个列表记录哪些内存块可用
# 分配时从列表中找到足够大的块，修改列表记录
```

#### 并发安全问题

对象创建在并发环境下可能存在线程安全问题（两个线程同时分配到同一块内存）。

**解决方案：**

**CAS + 失败重试：**
```java
// 悲观策略：加锁保证原子性
while (!CAS(指针, expected, newPointer)) {
    // 重试
}
```

**本地线程分配缓冲（TLAB）：**
```bash
# 每个线程在Eden区预先分配一小块区域
# -XX:+UseTLAB  # 启用TLAB（默认启用）
# -XX:TLABSize=1024k  # TLAB大小
# 线程优先在TLAB分配，TLAB不够再cas分配到公共区域
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

### 对象头（Object Header）

#### markWord（标记字段）

32位/64位（未压缩），存储对象的哈希码、GC年龄、锁状态等信息。

| 锁状态 | 25bit | 4bit | 1bit | 2bit |
|--------|-------|------|------|------|
| 无锁 | hashCode(25bit) | 对象年龄 | 分代年龄 | 01 |
| 偏向锁 | ThreadID(23bit) | epoch | 分代年龄 | 01 |
| 轻量级锁 | 指向栈中锁记录的指针 | - | - | 00 |
| 重量级锁 | 指向互斥量(Monitor)的指针 | - | - | 10 |
| GC标记 | - | - | - | 11 |

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

### 实例数据（Instance Data）

存储对象的实际字段数据。

```java
public class User {
    long birthday;         // 8字节
    int id;                // 4字节
    boolean active;         // 1字节
    // 但JVM会按8字节对齐，所以实际占用12字节
    // 对齐后：8 + 4 + 1 + 3(padding) = 16字节
}
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

```bash
# 访问流程
reference → 句柄 → 实例数据指针（堆） 或 类型数据指针（方法区）
# 优点：对象移动时只需修改句柄，reference本身不变
# 缺点：多一次指针访问
```

### 方式二：直接指针访问（HotSpot采用）

reference直接存储对象地址，减少一次指针访问。

```bash
# 访问流程
reference → 对象头(klassPointer) → 类型数据（方法区）
# 优点：访问速度快
# 缺点：对象移动时需要修改reference
```

---

## 实战验证

### 使用JOL查看对象布局

```xml
<!-- Maven依赖 -->
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.17</version>
</dependency>
```

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
    }
}

class User {
    long id;
    int age;
    boolean active;
    Object reference;
}

// 输出示例：
// java.lang.Object object internals:
// OFFSET  TYPE        DESCRIPTION
// 0       object header  [Mark Word]
// 8       object header  [Klass Pointer]
// 16      object header  [padding]
// Total: 16 bytes
```

---

## 面试高频问题

**Q1：对象创建过程？**
> 答：类加载检查→分配内存（指针碰撞/空闲列表+TLAB）→初始化零值→设置对象头→执行构造器。

**Q2：对象头包含哪些信息？**
> 答：Mark Word（哈希码、GC年龄、锁状态）、Klass Pointer（类型指针，指向方法区类信息）、数组长度（如果是数组）。

**Q3：对象一定分配在堆上吗？**
> 答：不一定。栈上分配（逃逸分析发现对象不逃逸出线程，可栈上分配）、TLAB（线程本地分配缓冲）都可以在堆上分配。对象本身在堆，但引用在栈。
```

- [ ] **Step 2: 保存文件并commit**

```bash
# 保存到 source/_posts/jvm/java对象创建与布局.md
git add source/_posts/jvm/java对象创建与布局.md
git commit -m "feat(jvm): add Java对象创建与布局 article"
```

---

## 汇总

完成JVM系列5篇文章：

| 文章 | 文件 | 状态 |
|------|------|------|
| Java内存区域详解 | `source/_posts/jvm/java-memory区域详解.md` | ✓ |
| JVM垃圾回收 | `source/_posts/jvm/jvm-gc垃圾回收.md` | ✓ |
| 类加载机制 | `source/_posts/jvm/java类加载机制.md` | ✓ |
| JVM性能调优实战 | `source/_posts/jvm/jvm性能调优实战.md` | ✓ |
| Java对象创建与布局 | `source/_posts/jvm/java对象创建与布局.md` | ✓ |
