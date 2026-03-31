# Java线上OOM排查：堆内存、Metaspace与DirectMemory

## 前言

OutOfMemoryError（OOM）是生产环境中常见且棘手的问题。一旦发生OOM，整个JVM进程会崩溃，影响线上服务。本文将深入探讨Java OOM的各种类型、原因分析、排查工具和方法，帮助读者在遇到OOM时能够快速定位和解决问题。

## OOM类型全景图

```
┌─────────────────────────────────────────────────────────────────┐
│                      Java OOM类型                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  java.lang.OutOfMemoryError: Java heap space                     │
│  ├─ 堆内存溢出，最常见                                           │
│  ├─ 对象创建速度 > GC回收速度                                   │
│  └─ 内存泄漏/内存碎片                                           │
│                                                                 │
│  java.lang.OutOfMemoryError: GC overhead limit exceeded         │
│  ├─ GC时间过长，占用超过98%                                      │
│  └─ 堆内存严重不足                                              │
│                                                                 │
│  java.lang.OutOfMemoryError: Metaspace                          │
│  ├─ 元空间溢出                                                  │
│  ├─ 类加载过多                                                  │
│  └─ JDK 8之前是PermGen Space                                    │
│                                                                 │
│  java.lang.OutOfMemoryError: Direct buffer memory               │
│  ├─ NIO直接内存溢出                                             │
│  ├─ ByteBuffer.allocateDirect()                                 │
│  └─ Netty等框架使用                                             │
│                                                                 │
│  java.lang.OutOfMemoryError: unable to create new native thread │
│  ├─ 线程数过多                                                  │
│  └─ 物理内存不足                                                │
│                                                                 │
│  java.lang.OutOfMemoryError: Requested array size exceeds limit │
│  ├─ 数组过大                                                    │
│  └─ 超出JVM限制                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 1. 堆内存溢出（Heap Space）

### 原因分析

```java
// 常见原因：
// 1. 内存泄漏 - 对象持有引用无法被GC
// 2. 集合数据无限增长 - 如HashMap持续put
// 3. 缓存未合理清理 - 软引用/弱引用未使用
// 4. 大对象分配 - 一次性加载大文件到内存

// 内存泄漏示例
public class MemoryLeak {
    // 静态集合持有对象引用，永远不会被回收
    private static List<Object> cache = new ArrayList<>();

    public void add(Object obj) {
        cache.add(obj);  // 只增不减，内存泄漏
    }
}
```

### 模拟代码

```java
public class HeapOOM {

    // JVM参数：-Xms10m -Xmx10m -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError
    // -XX:HeapDumpPath=/tmp/heapdump.hprof

    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        int i = 0;
        while (true) {
            // 每500MB分配一次，观察GC日志
            list.add(new byte[1024 * 1024 * 500]);  // 500MB对象
            System.out.println("分配了 " + (++i) + " MB");
        }
    }
}
```

### 排查工具

```bash
# 1. 生成堆转储文件
# JVM参数添加：
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/java/heapdump.hprof

# 或手动生成
jmap -dump:format=b,file=/tmp/heapdump.hprof <pid>

# 2. 分析堆转储
# 使用 MAT (Memory Analyzer Tool)
# https://www.eclipse.org/mat/

# 命令行分析
jmap -histo <pid>
# 输出：
#  num     #instances         #bytes  class name
#  1:         12345      12345678  [B
#  2:          8901       5678901  java.util.HashMap$Node
#  3:          5678       3456789  com.example.User
```

### MAT分析

```
堆转储分析重点：

1. Histogram视图
   - 查看对象数量和内存占用
   - 按包名/类名分组
   - 找出占用最大的对象

2. Dominator Tree视图
   - 找出谁持有大对象
   - GC Root到对象的路径
   - 快速定位内存泄漏

3. Top Consumers
   - 最大的50个对象
   - 可能的内存泄漏

4. Leak Suspects
   - 自动分析可能的泄漏点
   - 生成问题报告
```

### 代码示例 - 查找内存泄漏

```java
// 使用WeakHashMap自动回收
public class CacheExample {
    // 弱引用Map，当对象不再被其他引用持有时自动回收
    private Map<String, Object> cache = new WeakHashMap<>();

    public Object get(String key) {
        if (!cache.containsKey(key)) {
            // 模拟从数据库加载
            Object value = loadFromDB(key);
            cache.put(key, value);
        }
        return cache.get(key);
    }
}

// 使用LruCache限制大小
import com.google.common.cache.Cache;
import com.google.common.cache.CacheBuilder;

public class UserCache {
    private Cache<String, User> cache = CacheBuilder.newBuilder()
        .maximumSize(1000)           // 最多1000条
        .expireAfterWrite(10, TimeUnit.MINUTES)  // 10分钟过期
        .build();

    public User getUser(String userId) {
        return cache.getIfPresent(userId);
    }
}
```

## 2. Metaspace溢出

### 原因分析

```
Metaspace存储内容：
- 类信息（Class对象）
- 方法信息
- 字段信息
- 常量池
- 即时编译器生成的代码
- 符号表

常见原因：
1. 大量动态类生成
   - CGLIB/Spring代理
   - 热部署/类加载器问题
   - JSP编译

2. 反射使用过多
   - 大量Class对象未卸载
   - 类加载器泄漏
```

### JVM参数

```bash
# 设置Metaspace大小
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# JDK 8之前是PermGen
-XX:PermSize=128m
-XX:MaxPermSize=256m
```

### 模拟代码

```java
public class MetaspaceOOM {

    // JVM: -XX:MetaspaceSize=10m -XX:MaxMetaspaceSize=20m

    public static void main(String[] args) {
        int i = 0;
        while (true) {
            // 使用CGLIB生成大量代理类
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Object.class);
            enhancer.setUseCache(false);  // 不使用缓存
            enhancer.setCallback((MethodInterceptor) (o, method, objects, methodProxy) ->
                methodProxy.invokeSuper(o, objects));

            enhancer.create();
            System.out.println("生成了 " + (++i) + " 个类");
        }
    }
}
```

### 排查方法

```bash
# 1. 查看Metaspace使用
jstat -gc <pid>

# 输出：
#  S0C   S1C   S0U   S1U   EC     EU    OC     OU    MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
#  -     -     -     -     -     -     -     -     9728.0 9651.2 1152.0 768.0      0    0.000   0      0.000    0.000
#  MC = Metaspace Capacity
#  MU = Metaspace Used

# 2. 查看类加载信息
jmap -clstats <pid>

# 3. 查看类直方图
jcmd <pid> GC.class_histogram | head -30
```

## 3. DirectMemory溢出

### 原因分析

```
DirectMemory特点：
- 不在堆上分配，受物理内存限制
- 通过本地方法分配，不受-Xmx限制
- 用于NIO的ByteBuffer.allocateDirect()
- Netty等高性能网络框架大量使用

常见原因：
1. 大量使用NIO
2. Netty未正确释放ByteBuf
3. JNI调用分配本地内存
4. 第三方库（PDF处理、图片处理）
```

### 排查方法

```bash
# 1. JVM参数
# 指定直接内存大小
-XX:MaxDirectMemorySize=512m

# 2. 查看直接内存使用
# 通过NMT (Native Memory Tracking)
java -XX:NativeMemoryTracking=detail -jar app.jar

# 3. 监控NMT
jcmd <pid> VM.native_memory summary

# 4. 使用Async-profiler
# 检测本地内存分配
```

### 代码示例 - 正确释放DirectMemory

```java
// Netty ByteBuf正确释放
public class NettyExample {

    public void handle(ByteBuf buf) {
        try {
            // 业务处理
            byte[] data = new byte[buf.readableBytes()];
            buf.readBytes(data);
            process(data);
        } finally {
            // 必须释放，避免内存泄漏
            buf.release();
        }
    }

    // 使用ReferenceCounted
    public void compositeExample() {
        CompositeByteBuf composite = Unpooled.compositeBuffer();
        try {
            // 添加多个ByteBuf
            composite.addComponent(buf1);
            composite.addComponent(buf2);
            // 处理
        } finally {
            composite.release();
        }
    }
}
```

## 4. 线程数过多

### 原因分析

```
每个线程默认占用：
- 约1MB栈空间（-Xss配置）
- 线程本地内存
- 线程TLAB（Thread Local Allocation Buffer）

常见原因：
1. 线程池使用不当
   - 任务阻塞导致线程堆积
   - 拒绝策略设置不当

2. 连接池配置过大
   - 数据库连接池
   - HTTP连接池

3. 恶意代码
   - 定时任务创建大量线程
   - 递归调用创建线程
```

### 排查方法

```bash
# 1. 查看线程数
ps -eLf | grep java | wc -l
jcmd <pid> Thread.print | grep "Thread-" | wc -l

# 2. 查看线程栈
jstack <pid> > /tmp/threaddump.txt

# 3. 分析线程栈
# 查找BLOCKED、WAITING状态的线程
grep -A 5 "BLOCKED" /tmp/threaddump.txt

# 4. 统计线程状态
jstack <pid> | grep "State:" | sort | uniq -c
```

### 解决方案

```java
// 1. 合理配置线程池
@Configuration
public class ThreadPoolConfig {

    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);  // 队列容量
        executor.setThreadNamePrefix("biz-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());  // 拒绝策略
        executor.initialize();
        return executor;
    }
}

// 2. 使用信号量限制并发
public class SemaphoreExample {
    private Semaphore semaphore = new Semaphore(50);  // 最多50并发

    public void process() {
        try {
            semaphore.acquire();
            doWork();
        } finally {
            semaphore.release();
        }
    }
}

// 3. 监控线程数
@Component
public class ThreadMonitor {
    private static final Logger logger = LoggerFactory.getLogger(ThreadMonitor.class);

    @Scheduled(fixedRate = 60000)
    public void monitor() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
        long count = threadMXBean.getThreadCount();
        long peak = threadMXBean.getPeakThreadCount();
        long daemon = threadMXBean.getDaemonThreadCount();

        logger.info("线程数监控 - 当前: {}, 峰值: {}, 守护: {}",
                    count, peak, daemon);
    }
}
```

## 5. 实战案例

### 案例1：HashMap内存泄漏

```java
// 问题代码
public class UserService {
    private Map<String, User> userCache = new HashMap<>();  // 静态Map

    public User getUser(String userId) {
        User user = userCache.get(userId);
        if (user == null) {
            user = userDAO.findById(userId);
            userCache.put(userId, user);  // 只增不减
        }
        return user;
    }
}

// 解决方案：使用WeakHashMap或LruCache
public class UserServiceFixed {
    private Cache<String, User> userCache = CacheBuilder.newBuilder()
        .maximumSize(10000)
        .expireAfterAccess(30, TimeUnit.MINUTES)
        .build();

    public User getUser(String userId) {
        return userCache.getIfPresent(userId);
    }
}
```

### 案例2：连接池泄漏

```java
// 问题代码
public class ConnectionPoolLeak {
    private DataSource dataSource;  // HikariCP

    public void query() {
        Connection conn = null;
        try {
            conn = dataSource.getConnection();
            // 查询...
        } catch (SQLException e) {
            // 异常时未关闭连接
        }
        // 如果发生异常，连接未归还
    }
}

// 解决方案：使用try-with-resources
public class ConnectionPoolFixed {
    private DataSource dataSource;

    public void query() {
        // 自动关闭连接
        try (Connection conn = dataSource.getConnection()) {
            // 查询...
        } catch (SQLException e) {
            // 处理异常
        }
    }
}
```

## 6. 监控与预防

### 监控配置

```yaml
# Prometheus + Grafana监控JVM
prometheus:
  jvm:
    - name: jvm_memory_used
      query: jvm_memory_used_bytes{area="heap"}
    - name: jvm_memory_max
      query: jvm_memory_max_bytes{area="heap"}
    - name: jvm_gc_pause
      query: jvm_gc_pause_seconds_sum
```

### 预警规则

```yaml
# AlertManager规则
groups:
  - name: jvm_alerts
    rules:
      - alert: JVMMemoryHigh
        expr: jvm_memory_used_bytes / jvm_memory_max_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM内存使用率超过85%"

      - alert: JVMMemoryOOM
        expr: increase(jvm_gc_pause_seconds_count[5m]) > 10
        labels:
          severity: critical
        annotations:
          summary: "GC频繁，可能即将OOM"
```

### JVM参数推荐

```bash
# 生产环境推荐JVM参数
java -server \
  -Xms4g -Xmx4g \                    # 堆大小
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  -XX:MaxDirectMemorySize=512m \
  -XX:+UseG1GC \                     # G1垃圾收集器
  -XX:MaxGCPauseMillis=200 \         # 最大GC停顿
  -XX:+HeapDumpOnOutOfMemoryError \  # OOM时生成堆转储
  -XX:HeapDumpPath=/var/log/java/ \  # 堆转储路径
  -XX:+PrintGCDetails \              # 打印GC详情
  -Xloggc:/var/log/java/gc.log \    # GC日志
  -XX:+UseGCLogFileRotation \        # GC日志滚动
  -XX:NumberOfGCLogFiles=10 \
  -XX:GCLogFileSize=50M \
  -Dfile.encoding=UTF-8 \
  -jar application.jar
```

## 常见面试题

**Q1：JVM中有哪些内存区域？哪些会OutOfMemoryError？**

> 参考答案：JVM内存区域包括堆（Heap）、虚拟机栈（VM Stack）、本地方法栈（Native Method Stack）、方法区（Metaspace）、程序计数器（PC Register）、运行时常量池。堆溢出是最常见的OOM；虚拟机栈溢出（StackOverflowError）；Metaspace溢出；直接内存溢出；线程数过多无法创建新线程。每个区域在特定条件下都可能OOM。

**Q2：如何定位Java内存泄漏？**

> 参考答案：1）添加JVM参数`-XX:+HeapDumpOnOutOfMemoryError`生成堆转储；2）使用MAT分析堆转储文件；3）查看Histogram找出占用最大的对象；4）通过Dominator Tree查看引用链；5）分析 Leak Suspects报告；6）对比正常时和OOM时的对象数量差异。常用命令包括jmap、jstack、jstat等。

**Q3：如何避免和解决OOM？**

> 参考答案：预防措施：1）合理设置堆大小和Metaspace大小；2）使用软引用/弱引用做缓存；3）及时释放资源（try-with-resources）；4）监控JVM内存使用；5）代码review检查集合操作。解决方案：1）增加堆内存；2）修复内存泄漏代码；3）使用LruCache限制缓存大小；4）优化数据结构；5）使用对象池减少创建。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                    Java OOM排查方法论                          │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  排查步骤：                                                       │
│  1. 确定OOM类型（看错误信息）                                      │
│  2. 查看GC日志，分析GC是否正常                                      │
│  3. 生成堆转储，分析对象分布                                      │
│  4. 追踪对象引用链                                               │
│  5. 定位代码问题                                                 │
│                                                                │
│  工具：                                                           │
│  ├─ jmap: 生成堆转储                                            │
│  ├─ MAT: 分析堆转储文件                                          │
│  ├─ jstat: 查看GC统计                                           │
│  ├─ jstack: 查看线程栈                                          │
│  └─ Async-profiler: 性能分析工具                                 │
│                                                                │
│  预防措施：                                                       │
│  ├─ 合理设置JVM参数                                             │
│  ├─ 开启OOM时生成堆转储                                         │
│  ├─ 监控内存使用率                                               │
│  ├─ 代码规范：及时释放资源                                       │
│  └─ 使用WeakHashMap/LruCache做缓存                              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

OOM是生产环境中需要高度重视的问题。通过合理的监控、正确的JVM参数配置和良好的编码习惯，可以有效预防OOM的发生。
