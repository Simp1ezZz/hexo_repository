---
title: Java线程池原理：ThreadPoolExecutor核心机制详解
date: 2024-02-22 14:28:32
tags:
  - Java进阶
  - 并发编程
categories: 学习
keywords: 线程池,ThreadPoolExecutor,Executor框架,拒绝策略
description: 深入理解线程池的工作原理，核心参数与拒绝策略详解
cover:
---

## 前言

"每次请求都新建线程"会带来巨大的性能开销，线程池正是解决这个问题。合理使用线程池可以：
- 降低资源消耗（复用已有线程）
- 提高响应速度（无需等待线程创建）
- 提高线程可管理性（统一分配和监控）

## 线程池核心类关系

```
┌─────────────────────────────────────────────┐
│              Executor                       │
│   void execute(Runnable command)           │
└─────────────────────────────────────────────┘
                    △
                    │
┌─────────────────────────────────────────────┐
│           ExecutorService                   │
│   submit() / shutdown() / shutdownNow()    │
└─────────────────────────────────────────────┘
                    △
                    │
┌─────────────────────────────────────────────┐
│         AbstractExecutorService             │
│   submit() / invokeAll() / invokeAny()     │
└─────────────────────────────────────────────┘
                    △
                    │
┌─────────────────────────────────────────────┐
│            ThreadPoolExecutor               │
│   核心实现类                                │
└─────────────────────────────────────────────┘
```

---

## 七大参数详解

```java
public ThreadPoolExecutor(
    int corePoolSize,              // 核心线程数
    int maximumPoolSize,           // 最大线程数
    long keepAliveTime,            // 空闲线程存活时间
    TimeUnit unit,                 // keepAliveTime单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,   // 线程工厂
    RejectedExecutionHandler handler  // 拒绝策略
)
```

### 参数图解

```
┌──────────────────────────────────────────────────────────────┐
│                    ThreadPoolExecutor                        │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              线程池中的线程                           │   │
│   │   [核心线程 corePoolSize]  [非核心线程 max-core]    │   │
│   │                                                       │   │
│   │   线程1  线程2  线程3  ...  线程N                   │   │
│   └─────────────────────────────────────────────────────┘   │
│                          ↓                                   │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            任务队列 workQueue                         │   │
│   │   [Runnable1] [Runnable2] [Runnable3] ...           │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │         拒绝策略 handler                              │   │
│   │   队列满且线程数=max时触发                           │   │
│   └─────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 各参数详解

| 参数 | 说明 | 注意事项 |
|------|------|---------|
| corePoolSize | 核心线程数，即使空闲也不回收 | 可以通过allowCoreThreadTimeOut变为可回收 |
| maximumPoolSize | 最大线程数 = 核心 + 非核心 | 必须 ≥ corePoolSize |
| keepAliveTime | 非核心线程空闲存活时间 | 0表示不回收 |
| workQueue | 任务阻塞队列 | 如LinkedBlockingQueue、ArrayBlockingQueue |
| threadFactory | 线程工厂 | 用于自定义线程创建（设置名称、优先级等） |
| handler | 拒绝策略 | 队列满且线程数达最大值时触发 |

---

## 线程池工作流程

### 执行流程图

```
                    ┌─────────────────────────┐
                    │   提交新任务 execute()   │
                    └─────────────────────────┘
                                    │
                                    ↓
                    ┌─────────────────────────────┐
                    │  线程数 < corePoolSize ?   │
                    └─────────────────────────────┘
                         ↓ Yes              ↓ No
            ┌──────────┐         ┌──────────────────────┐
            │ 创建新   │         │  进入任务队列        │
            │ 核心线程 │         │  workQueue.offer()   │
            └──────────┘         └──────────────────────┘
                                                │
                                                ↓
                    ┌─────────────────────────────────────┐
                    │       队列插入成功？                  │
                    └─────────────────────────────────────┘
                            ↓ Yes              ↓ No
                    ┌──────────────┐     ┌─────────────────────┐
                    │   返回成功   │     │ 线程数 < maxPoolSize?│
                    └──────────────┘     └─────────────────────┘
                                               ↓ Yes    ↓ No
                                    ┌──────────┐   ┌──────────────┐
                                    │ 创建新   │   │ 执行拒绝策略  │
                                    │ 非核心线程│   └──────────────┘
                                    └──────────┘
```

### 代码流程

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();

    // 1. 如果线程数 < 核心线程数，创建核心线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    // 2. 尝试加入队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();

        // 2.1 队列插入成功，但线程池状态可能变化
        if (!isRunning(recheck) && remove(command))
            reject(command);  // 拒绝

        // 2.2 线程数为0，创建非核心线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }

    // 3. 队列满，尝试创建非核心线程
    else if (!addWorker(command, false))
        reject(command);  // 拒绝
}
```

---

## 四种常见线程池

### Executors 静态工厂方法

```java
// 1. 固定线程数线程池
ExecutorService fixedPool = Executors.newFixedThreadPool(10);
// 特点：core=max=10，LinkedBlockingQueue无界队列

// 2. 单线程线程池
ExecutorService singlePool = Executors.newSingleThreadExecutor();
// 特点：core=max=1，保证顺序执行

// 3. 缓存线程池
ExecutorService cachedPool = Executors.newCachedThreadPool();
// 特点：core=0，max=Integer.MAX_VALUE，60秒超时

// 4. 调度线程池
ScheduledExecutorService scheduledPool = Executors.newScheduledThreadPool(5);
// 特点：支持定时和周期任务
```

### 为什么不推荐直接使用Executors创建？

```java
// 问题1：FixedThreadPool/SingleThreadExecutor
// 使用无界队列（LinkedBlockingQueue），任务过多会OOM
ExecutorService pool = Executors.newFixedThreadPool(10);
while (true) {
    pool.execute(() -> ...);  // 队列无限增长
}

// 问题2：CachedThreadPool
// max=Integer.MAX_VALUE，可能创建过多线程导致OOM
ExecutorService pool = Executors.newCachedThreadPool();
for (int i = 0; i < 100000; i++) {
    pool.execute(() -> ...);  // 大量并发创建线程
}
```

### 推荐：手动创建 ThreadPoolExecutor

```java
// 推荐配置
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    8,                      // corePoolSize：CPU密集型可设CPU核数+1
    16,                     // maximumPoolSize：CPU密集型可设CPU核数*2
    60L,                    // keepAliveTime
    TimeUnit.SECONDS,       // 时间单位
    new LinkedBlockingQueue<>(100),  // 有界队列，控制任务堆积
    new ThreadFactoryBuilder()
        .setNameFormat("business-pool-%d")
        .build(),
    new ThreadPoolExecutor.AbortPolicy()  // 拒绝策略
);
```

---

## 四种拒绝策略

### 策略对比

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| AbortPolicy | 抛出RejectedExecutionException | 默认，不允许丢失任务 |
| CallerRunsPolicy | 由调用线程执行 | 限流，提供反馈 |
| DiscardPolicy | 静默丢弃 | 不关心任务丢失 |
| DiscardOldestPolicy | 丢弃最老的任务 | 优先处理新任务 |

### 自定义拒绝策略

```java
// 实现 RejectedExecutionHandler 接口
public class CustomRejectedHandler implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 记录日志
        log.error("Task rejected, executor: {}", executor);

        // 方案1：放入内存队列重试
        ((MyTask) r).retry();

        // 方案2：持久化到数据库，稍后处理
        saveToDatabase(r);

        // 方案3：降级处理
        doFallback(r);
    }
}
```

---

## 线程工厂（ThreadFactory）

### 默认线程工厂

```java
// Executors.DefaultThreadFactory 实现
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
            namePrefix + threadNumber.getAndIncrement(), 0);
        t.setDaemon(false);  // 非守护线程
        t.setPriority(Thread.NORM_PRIORITY);  // 正常优先级
        return t;
    }
}
```

### 自定义线程工厂

```java
// Guava ThreadFactoryBuilder
ThreadFactory factory = new ThreadFactoryBuilder()
    .setNameFormat("order-processing-%d")
    .setDaemon(false)  // 非守护线程
    .setPriority(Thread.MAX_PRIORITY)  // 最高优先级
    .setUncaughtExceptionHandler((t, e) ->
        log.error("Thread {} threw exception", t.getName(), e))
    .build();
```

---

## 常见问题与优化

### 问题1：线程池参数如何设置？

```java
// CPU密集型任务（计算为主）
// 建议：core = CPU核心数 + 1
int cpuCores = Runtime.getRuntime().availableProcessors();
executor = new ThreadPoolExecutor(
    cpuCores + 1, cpuCores + 1,
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(100)
);

// IO密集型任务（网络、磁盘为主）
// 建议：core = CPU核心数 * 2（或根据IO等待比例调整）
executor = new ThreadPoolExecutor(
    cpuCores * 2, cpuCores * 2,
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(100)
);
```

### 问题2：如何监控线程池？

```java
// 监控指标
ThreadPoolExecutor executor = ...;

// 1. 当前活跃线程数
executor.getActiveCount();

// 2. 已完成任务数
executor.getCompletedTaskCount();

// 3. 当前队列中的任务数
executor.getQueue().size();

// 4. 线程池配置
executor.getCorePoolSize();
executor.getMaximumPoolSize();

// 5. 历史最大线程数
executor.getLargestPoolSize();

// 自定义监控扩展
@Override
protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    // 记录任务执行时间、异常等
    recordMetrics(r, t);
}
```

### 问题3：如何优雅关闭线程池？

```java
ThreadPoolExecutor executor = ...;

// 方法1：不再接受新任务，等待队列任务完成
executor.shutdown();

// 方法2：不再接受新任务，立即中断正在执行的任务
executor.shutdownNow();

// 等待所有任务完成（带超时）
boolean done = executor.awaitTermination(60, TimeUnit.SECONDS);
if (!done) {
    executor.shutdownNow();
}

// 验证关闭
if (executor.isShutdown()) {
    // 线程池已关闭
}
```

### 问题4：父子线程池隔离

```java
// 场景：主线程提交任务，子线程又提交子任务
// 需要传递上下文

// 使用InheritableThreadLocal（注意线程池会复用线程）
InheritableThreadLocal<String> context = new InheritableThreadLocal<>();
context.set("value");

// 推荐：使用TransmittableThreadLocal（阿里开源）
// 需要引入: com.alibaba:transmittable-thread-local
TtlRunnable ttlRunnable = TtlRunnable.get(runnable);
executor.submit(ttlRunnable);
```

---

## 常见线程池类型对比

| 类型 | core=max | 队列 | 特点 |
|------|---------|------|------|
| FixedThreadPool | N | LinkedBQ(无界) | 固定大小，任务堆积OOM风险 |
| SingleThreadExecutor | 1 | LinkedBQ(无界) | 单线程，任务堆积OOM风险 |
| CachedThreadPool | 0, Integer.MAX | SynchronousQ | 按需创建，线程数OOM风险 |
| ScheduledThreadPool | 指定 | DelayedWorkQ | 定时任务 |
| WorkStealingPool | - | - | JDK8+，工作窃取算法 |

---

## 实战：配置企业级线程池

```java
public class ThreadPoolConfig {

    // 业务线程池
    @Bean("businessExecutor")
    public ThreadPoolExecutor businessExecutor() {
        return new ThreadPoolExecutor(
            8, 16, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(200),
            new ThreadFactoryBuilder()
                .setNameFormat("business-%d")
                .build(),
            (r, e) -> {
                log.error("Task rejected: {}", r);
                // 持久化或告警
                Metrics.counter("thread_pool.rejected").increment();
            }
        );
    }

    // IO密集型线程池
    @Bean("ioExecutor")
    public ThreadPoolExecutor ioExecutor() {
        int cores = Runtime.getRuntime().availableProcessors();
        return new ThreadPoolExecutor(
            cores * 2, cores * 4, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(500),
            new ThreadFactoryBuilder()
                .setNameFormat("io-%d")
                .build(),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

---

## 面试高频问题

**Q1：线程池的执行流程？**

> 答：1. 线程数<corePoolSize，创建核心线程；2. 否则加入队列；3. 队列满且线程数<maxPoolSize，创建非核心线程；4. 队列满且线程数=maxPoolSize，执行拒绝策略。

**Q2：为什么不推荐Executors创建线程池？**

> 答：FixedThreadPool和SingleThreadExecutor使用无界队列，可能OOM；CachedThreadPool最大线程数无限制，也可能OOM。应手动创建有界队列的线程池。

**Q3：核心线程会被回收吗？**

> 答：默认不会。设置allowCoreThreadTimeOut(true)后，核心线程也会在keepAliveTime超时后被回收。

**Q4：线程池大小如何设置？**

> 答：CPU密集型建议core=CPU核数+1；IO密集型建议core=CPU核数*2（或根据IO等待比例计算）。最佳实践是先预估，再压测调优。

**Q5：submit和execute区别？**

> 答：submit可以提交Callable（有返回值），返回Future；execute只能提交Runnable（无返回值）。submit内部也是调用execute。

---

## 总结

线程池是并发编程的核心工具：
- **七大参数**：coreSize、maxSize、keepAliveTime、queue、factory、handler
- **执行流程**：创建核心→入队→创建非核心→拒绝策略
- **拒绝策略**：Abort、CallerRuns、Discard、DiscardOldest
- **监控指标**：activeCount、completedTaskCount、queueSize
- **优雅关闭**：shutdown（等待完成）和shutdownNow（立即中断）
