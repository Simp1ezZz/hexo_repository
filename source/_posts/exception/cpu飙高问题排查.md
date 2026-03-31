# CPU飙高问题排查：定位热点与性能分析

## 前言

CPU使用率飙高是线上服务常见的性能问题之一。高CPU不仅影响系统响应时间，还可能导致服务雪崩。本文将详细介绍如何排查Java应用CPU飙高的原因，包括如何使用诊断工具定位热点代码，以及如何进行性能优化。

## CPU飙高常见原因

```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU飙高常见原因                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 死循环/无限递归                                              │
│     - 代码逻辑错误                                               │
│     - 未正确终止的循环                                           │
│                                                                 │
│  2. 频繁GC                                                      │
│     - 内存分配过快                                              │
│     - 内存泄漏导致频繁Full GC                                    │
│                                                                 │
│  3. 频繁上下文切换                                              │
│     - 线程数过多                                                │
│     - 锁竞争激烈                                                │
│                                                                 │
│  4. 正则表达式回溯                                              │
│     - 复杂正则导致回溯                                           │
│     - 恶意输入触发最坏情况                                       │
│                                                                 │
│  5. 序列化/反序列化                                             │
│     - 大对象序列化                                              │
│     - 低效序列化框架                                             │
│                                                                 │
│  6. 加密/解密                                                   │
│     - 频繁加密操作                                              │
│     - 低效加密算法                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1. 排查流程

### 快速定位步骤

```bash
# 第一步：确定CPU飙高的进程
# 查看所有进程CPU使用率
top
# 或
ps aux | grep java

# 第二步：查看该进程的线程CPU占用
top -Hp <pid>

# 第三步：将线程ID转为16进制
printf "%x\n" <thread_id>

# 第四步：获取线程栈
jstack <pid> > /tmp/threaddump.txt
grep -A 10 <hex_thread_id> /tmp/threaddump.txt
```

### 完整排查示例

```bash
# 1. 查找Java进程
$ ps aux | grep java
USER       PID   %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root     12345   98.5  5.0 4194304 204800 -      S    10:00  456:07 java -jar app.jar
# 发现PID 12345的Java进程CPU占用98.5%

# 2. 查看该进程的线程
$ top -Hp 12345
PID  USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+
12346 root      20   0 4194304 204800 -      R 45.2  5.0  234:05 java
12347 root      20   0 4194304 204800 -      R 32.1  5.0  156:32 java
12348 root      20   0 4194304 204800 -      R 20.1  5.0   89:12 java

# 3. 转换为16进制
printf "%x\n" 12346
# 输出: 303a

# 4. 查看线程栈
$ jstack 12345 | grep -A 20 "nid=0x303a"
"pool-1-thread-10" #10 prio=5 os_prio=0 tid=0x00007f8a4c01a800 nid=0x303a runnable

java.lang.Thread.State: RUNNABLE
    at com.example.service.Calculator.compute(Calculator.java:45)
    at com.example.service.Calculator.calculate(Calculator.java:30)
    at com.example.controller.CalculatorController.calculate(CalculatorController.java:25)
```

## 2. jstack分析

### 线程状态解读

```bash
# jstack输出
"pool-1-thread-10" #10 prio=5 os_prio=0 tid=0x00007f8a4c01a800 nid=0x303a runnable
   ↑
   线程名称

java.lang.Thread.State: RUNNABLE
   ↑
   线程状态

# 线程状态：
# - RUNNABLE: 正在运行
# - BLOCKED: 阻塞等待锁
# - WAITING: 等待另一个线程执行特定动作
# - TIMED_WAITING: 带超时的等待
```

### 常见问题模式

```bash
# 1. 死循环（RUNNABLE，高CPU）
"pool-1-thread-1" #1 prio=5 os_prio=0 tid=0x00007f8a4c01a800 nid=0x303a runnable
java.lang.Thread.State: RUNNABLE
    at com.example.util.HashUtils.compute(HashUtils.java:45)
    at com.example.util.HashUtils.calculate(HashUtils.java:30)
    # 可能存在死循环

# 2. 锁等待（BLOCKED）
"pool-1-thread-2" #2 prio=5 os_prio=0 tid=0x00007f8a4c01b800 nid=0x303b waiting for monitor entry
java.lang.Thread.State: BLOCKED
    at com.example.service.OrderService.createOrder(OrderService.java:25)
    - waiting to lock <0x00000000a0001234> (a java.lang.Object)
    # 等待锁

# 3. GC线程
"GC task thread#0" #2 prio=5 os_prio=0 tid=0x00007f8a4c01c800 nid=0x303c runnable
java.lang.Thread.State: RUNNABLE
    at com.sun.proxy.jdk.proxy1.$Proxy100.invoke(Unknown Source)
    # 频繁GC可能导致CPU高

# 4. 正则回溯
"pool-1-thread-3" #3 prio=5 os_prio=0 tid=0x00007f8a4c01d800 nid=0x303d runnable
java.lang.Thread.State: RUNNABLE
    at java.util.regex.Pattern$Loop.match(Pattern.java:5127)
    # 正则表达式回溯
```

## 3. Async-profiler

### 安装与使用

```bash
# 下载 async-profiler
# https://github.com/async-profiler/async-profiler

# 1. 采样CPU热点（火焰图）
./profiler.sh -d 60 -f /tmp/profile.svg <pid>
# -d: 采样时间60秒
# -f: 输出文件
# 生成火焰图

# 2. 查看CPU热点方法
./profiler.sh -e cpu -d 30 <pid>
# 输出:
# ---Profile---
# Frame                                   Count
# com.example.service.SearchService.search   12345
# com.example.util.StringUtils.contains      8901
# java.util.regex.Pattern$CharProperty.match  4567

# 3. 分析锁竞争
./profiler.sh -e lock -d 30 <pid>
# 显示锁等待时间

# 4. 分析内存分配
./profiler.sh -e alloc -d 30 <pid>
# 显示内存分配热点
```

### 火焰图解读

```
火焰图分析：

        __[kernel]__
        __[user]__
            |
            |
        ___[jackson]___          <-- jackson占用
        |
    ____[search]____
    |              |
 [loop]        [parse]           <-- 热点在循环和解析

阅读方式：
- 从下往上：调用栈
- 方块越大：占用越多
- 顶部是叶节点（最终方法）
- 点击可查看源码行号
```

## 4. Arthas使用

### 常用命令

```bash
# 启动Arthas
java -jar arthas-boot.jar <pid>

# 1. dashboard - 查看系统总体情况
dashboard

# 输出：
# PID     NAME           CPU%   THREADS   MEM%   BLOCKED
# 12345   java           98.5   100       45%    5
# 线程信息、内存信息、GC信息

# 2. thread - 查看线程CPU占用
thread -n 10
# 显示CPU占用最高的10个线程

thread -n 5 -i 1000
# 采样1秒内CPU占用

# 3. trace - 方法执行时间
trace com.example.service.UserService getUser '#cost > 100'
# 追踪getUser方法，执行时间超过100ms的调用

# 4. monitor - 方法调用统计
monitor -c 10 com.example.service.UserService getUser
# 每10秒统计一次getUser调用

# 5. profiler - CPU采样
profiler start
profiler stop --format html
# 生成火焰图

# 6. jad - 反编译
jad com.example.service.UserService
# 查看运行时代码
```

### 实战案例

```bash
# 场景：用户反馈接口响应慢，CPU占用高

# 1. 查看概况
$ dashboard
# 发现CPU 98%，GC正常

# 2. 查看线程
$ thread -n 5
# 发现 pool-1-thread-1 CPU占用 45%

# 3. 查看该线程栈
$ thread -n 1 -p 12346
"pool-1-thread-1" Id=10 CPU=45%
    @com.example.service.SearchService.search(SearchService.java:45)
    @com.example.service.SearchService.search(SearchService.java:30)

# 4. trace方法调用
$ trace com.example.service.SearchService search
# 发现 hashMap.get() 占用 80% 时间

# 5. 查看代码
jad com.example.service.SearchService
# 发现 HashMap 在循环中被调用，没有缓存
```

## 5. 常见问题代码

### 问题1：死循环

```java
// 问题代码
public class BadCode {
    public void process(List<String> data) {
        int i = 0;
        while (true) {  // 死循环
            if (i >= data.size()) {
                i = 0;  // 错误逻辑，应该是break
            }
            processItem(data.get(i));
            i++;
        }
    }
}

// 修复
public class GoodCode {
    public void process(List<String> data) {
        for (String item : data) {
            processItem(item);
        }
    }
}
```

### 问题2：正则回溯

```java
// 问题代码 - 恶意正则
public class RegexProblem {
    // 看似简单，但输入 "aaaaaaaaaaaaaaaaX" 时会指数级回溯
    private static final Pattern BAD_PATTERN =
        Pattern.compile("(a+)+b");

    public boolean validate(String input) {
        return BAD_PATTERN.matcher(input).matches();
    }
}

// 修复 - 使用 possessive 或 atomic grouping
public class RegexFixed {
    // 使用原子组或简化为普通正则
    private static final Pattern GOOD_PATTERN =
        Pattern.compile("a+b");

    public boolean validate(String input) {
        return GOOD_PATTERN.matcher(input).matches();
    }
}
```

### 问题3：频繁GC

```java
// 问题代码 - 大量临时对象
public class GCProblem {
    public String concat(List<String> parts) {
        String result = "";
        for (String part : parts) {
            result += part;  // 每次创建新String
        }
        return result;
    }
}

// 修复 - 使用StringBuilder
public class GCFixed {
    public String concat(List<String> parts) {
        StringBuilder sb = new StringBuilder();
        for (String part : parts) {
            sb.append(part);
        }
        return sb.toString();
    }
}
```

### 问题4：锁竞争

```java
// 问题代码 - 粗粒度锁
public class LockProblem {
    private final Object lock = new Object();
    private int counter = 0;

    public void increment() {
        synchronized (lock) {
            counter++;
            // 模拟业务处理
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}

// 修复 - 减小锁粒度
public class LockFixed {
    private AtomicInteger counter = new AtomicInteger(0);

    public void increment() {
        counter.incrementAndGet();
        // 业务处理移出锁
        try {
            Thread.sleep(10);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## 6. 监控与预防

### Prometheus监控

```yaml
# JVM监控指标
jvm:
  metrics:
    - name: process_cpu_usage
      type: gauge
    - name: jvm_threads_current
      type: gauge
    - name: jvm_gc_pause_seconds_sum
      type: counter
```

### 预警规则

```yaml
# AlertManager
groups:
  - name: cpu_alerts
    rules:
      - alert: HighCPUUsage
        expr: process_cpu_usage > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU使用率超过80%"

      - alert: ThreadCountHigh
        expr: jvm_threads_current > 1000
        labels:
          severity: warning
```

### JVM参数优化

```bash
# 减少GC压力
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1ReservePercent=10

# 减少上下文切换
-XX:+UseCondCardMark
-XX:PreBlockSpin=8
```

## 常见面试题

**Q1：如何排查Java应用CPU飙高的问题？**

> 参考答案：1）使用top/ps找到CPU高的进程；2）top -Hp <pid>查看该进程的线程；3）将高CPU线程ID转16进制；4）jstack <pid>获取线程栈，grep线程ID找到问题代码；5）使用Arthas/async-profiler采样分析热点方法；6）分析代码定位具体问题（死循环、频繁GC、锁竞争等）。

**Q2：jstack中BLOCKED和RUNNABLE状态分别代表什么？**

> 参考答案：RUNNABLE表示线程正在运行或等待CPU执行；BLOCKED表示线程在等待获取对象的监视器锁（synchronized）。BLOCKED状态的线程说明存在锁竞争，需要分析哪些线程持有锁、锁的粒度是否过大。WAITING表示线程在等待另一个线程执行特定操作（如Object.wait()）。

**Q3：正则表达式可能导致什么问题？**

> 参考答案：某些正则表达式（如`(a+)+b`）在处理特定输入时会产生指数级回溯，导致CPU 100%。这是因为正则引擎在尝试所有可能的匹配路径。解决方案包括：1）避免嵌套量词；2）使用 possessive quantifier（如`a++`）；3）使用atomic grouping；4）用简单的正则替代复杂正则。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                    CPU飙高排查方法论                             │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  排查步骤：                                                       │
│  1. top -> 找到高CPU进程                                         │
│  2. top -Hp -> 找到高CPU线程                                     │
│  3. jstack -> 获取线程栈                                         │
│  4. 分析线程栈 -> 定位问题代码                                    │
│  5. async-profiler/Arthas -> 热点分析                           │
│                                                                │
│  常见原因：                                                       │
│  ├─ 死循环/无限递归                                              │
│  ├─ 正则回溯                                                     │
│  ├─ 频繁GC                                                      │
│  ├─ 锁竞争                                                       │
│  └─ 序列化/加密等CPU密集操作                                     │
│                                                                │
│  优化手段：                                                       │
│  ├─ 代码优化：减少对象创建                                       │
│  ├─ 架构优化：减少锁竞争                                         │
│  ├─ 参数优化：合理GC参数                                         │
│  └─ 监控预警：及时发现问题                                       │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

CPU飙高排查需要结合系统工具和代码分析。通过系统性排查，可以快速定位热点代码并采取针对性优化措施。
