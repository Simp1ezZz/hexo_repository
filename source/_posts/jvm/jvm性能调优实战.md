---
title: JVM性能调优实战：参数配置与问题排查
date: 2026-04-04 10:00:00
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

**推荐生产配置：**
```bash
-Xms4g -Xmx4g  # 初始和最大设为相同，避免运行时调整
-Xmn2g         # 新生代2g
-XX:SurvivorRatio=8  # Eden:Survivor = 8:1
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

### 三大目标

| 目标 | 说明 | 指标 |
|------|------|------|
| 低延迟 | 避免GC长时间停顿影响用户体验 | GC停顿时间 < 200ms |
| 高吞吐 | 最大化CPU利用率用于业务处理 | 吞吐量 > 95% |
| 低内存 | 合理利用资源，控制成本 | 内存占用合理，无OOM |

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

---

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

# 查看GC统计
jstat -gc <pid> 1000 5

# 输出字段说明：
# S0C S1C S0U S1U     - Survivor0/1容量和使用量(KB)
# EC EU                - Eden区容量和使用量(KB)
# OC OU                - 老年代容量和使用量(KB)
# MC MU                - 元空间容量和使用量(KB)
# YGC YGCT             - Young GC次数和耗时
# FGC FGCT             - Full GC次数和耗时
# GCT                  - 总GC耗时
```

### jmap - 内存映射

```bash
# 查看堆占用（按对象大小排序，前30个）
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

# 火焰图
profiler start
profiler stop --format html
```

---

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

    public void put(String key, Object value) {
        cache.put(key, value);
    }
}
```

**Step 4: 调整JVM参数**
```bash
# 减少CMS触发频率
-XX:CMSInitiatingOccupancyFraction=70  # 从68%调整到70%
-XX:+UseCMSInitiatingOccupancyOnly     # 只使用设定的阈值
```

---

## 常见问题与解决方案

### 问题1：OutOfMemoryError: Java heap space

**原因：** 对象分配过多，内存泄漏

**排查：**
```bash
# 导出堆dump
jmap -dump:format=b,file=heap.hprof <pid>

# 使用MAT分析
# 找出占用内存最大的对象
# 查看对象引用链
```

**解决：**
- 增加堆内存 `-Xmx`
- 修复内存泄漏（检查HashMap等容器是否有增无删除）

### 问题2：OutOfMemoryError: Metaspace

**原因：** 类太多（动态代理生成、CGLIB增强、频繁反射）

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

## GC日志分析模板

### Minor GC日志
```
2026-04-04T10:15:30.123+0800: [GC (Allocation Failure)
  Desired survivor size 5242880 bytes, new threshold 15 (max 15)
  - age 1: 123456 bytes, 123456 total
 eden space 65536K, 80% used
  from space 8192K, 50% used
  to space 8192K, 0% used
  Metaspace: 12345K used]
```

**关键指标：**
- `Allocation Failure`：Eden区满触发Minor GC
- `from/to space 50% used`：Survivor区使用率
- `- age N: X bytes`：年龄为N的对象占用字节数

### Full GC日志
```
2026-04-04T10:15:35.456+0800: [Full GC (Allocation Failure)
  -- 8000K->2048K(8192K), 0.820 secs]
  cms: 8000K->2048K(8192K), 0.820 secs]
  [CMS-remark: 0.123s]
  [CMS-concurrent-sweep: 0.456s]
```

**关键指标：**
- `8000K->2048K`：GC前后老年代使用量
- `0.820 secs`：Full GC耗时

---

## 生产环境推荐配置

### Web应用（低延迟优先）
```bash
java -server \
  -Xms4g -Xmx4g \
  -Xmn2g \
  -XX:SurvivorRatio=8 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/log/java_heap.hprof \
  -Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=10,filesize=10m
```

### 批处理应用（高吞吐优先）
```bash
java -server \
  -Xms8g -Xmx8g \
  -Xmn4g \
  -XX:SurvivorRatio=8 \
  -XX:+UseParallelGC \
  -XX:+UseParallelOldGC \
  -XX:MaxGCPauseMillis=500 \
  -XX:GCTimeRatio=19 \
  -XX:+UseAdaptiveSizePolicy
```

---

## 面试高频问题

**Q1：如何定位线上OOM？**

> 答：添加 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/path.hprof`，然后用MAT/JProfiler分析dump文件，找出占用内存最大的对象和引用链。

**Q2：如何减少GC次数？**

> 答：减少对象创建（循环内复用对象）、对象复用（StringBuilder、对象池）、增大新生代、选择合适收集器。使用软引用/弱引用处理缓存。

**Q3：G1和CMS区别？**

> 答：CMS是老年代并发收集器，标记-清除算法，会产生碎片但停顿短。G1是整堆收集器，标记-整理算法，不产生碎片，可预测停顿。JDK9+默认G1。

**Q4：线上Full GC频繁怎么处理？**

> 答：1. 分析GC日志确定原因；2. 用jmap分析堆对象找出内存占用大户；3. 检查代码是否有内存泄漏；4. 调整CMS/G1参数；5. 必要时增大堆内存。

---

## 总结

JVM调优是系统优化的关键能力：
- **理解参数**：堆大小、新生代比例、GC收集器是核心
- **工具辅助**：jstat、jmap、jstack、arthas是排查利器
- **实战经验**：通过真实案例积累排查思路
- **选择策略**：低延迟选G1/CMS，高吞吐选Parallel
