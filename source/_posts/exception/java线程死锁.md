---
title: Java线程死锁：检测、定位与预防
date: 2023-01-08 16:19:06
tags:
  - 文章
categories: 学习
---


## 前言

线程死锁是Java并发编程中最隐蔽也最危险的问题之一。它不像OOM或CPU飙高那样容易通过监控发现，往往在特定条件下才会触发，而且一旦发生就会导致相关线程永久阻塞。本文将深入探讨死锁的形成原因、检测方法、定位工具以及预防策略。

## 死锁形成原理

### 必要条件

```
┌─────────────────────────────────────────────────────────────────┐
│                    死锁的必要条件                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 互斥条件                                                     │
│     资源只能被一个线程持有                                       │
│                                                                 │
│  2. 持有并等待                                                   │
│     线程持有资源的同时，等待其他资源                               │
│                                                                 │
│  3. 不可抢占                                                     │
│     资源不能被强制释放，只能等线程主动释放                         │
│                                                                 │
│  4. 循环等待                                                     │
│     线程之间形成循环等待资源的依赖关系                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 经典死锁场景

```java
// 场景：转账业务
// 线程A: 从账户X转100到账户Y
// 线程B: 从账户Y转50到账户X

// 线程A执行顺序：先锁X，再锁Y
synchronized (accountX) {          // 线程A获取X锁
    try {
        Thread.sleep(100);        // 模拟处理时间
    } catch (InterruptedException e) {}
    synchronized (accountY) {    // 等待Y锁
        accountX.decrease(100);
        accountY.increase(100);
    }
}

// 线程B执行顺序：先锁Y，再锁X
synchronized (accountY) {        // 线程B获取Y锁
    try {
        Thread.sleep(100);
    } catch (InterruptedException e) {}
    synchronized (accountX) {    // 等待X锁，但X被A持有
        accountY.decrease(50);
        accountX.increase(50);
    }
}

// 死锁形成：
// 线程A持有X，等待Y；线程B持有Y，等待X
// 形成循环等待 -> 死锁
```

## 1. 死锁检测

### jstack检测

```bash
# 最简单的方法：jstack
jstack <pid>

# 输出中会显示死锁信息：
# Found one Java-level deadlock:
# ============================
# "pool-1-thread-2":
#   waiting to lock monitor 0x00007f8a4c01a800 (java.lang.Object@0x00000000a0001234)
#   which is held by "pool-1-thread-1"
# "pool-1-thread-1":
#   waiting to lock monitor 0x00007f8a4c01a900 (java.lang.Object@0x00000000a0005678)
#   which is held by "pool-1-thread-2"

# Java stack information for the threads listed above:
# ===================================================
# "pool-1-thread-2":
#     at com.example.service.TransferService.transfer(TransferService.java:30)
#     ...
# "pool-1-thread-1":
#     at com.example.service.TransferService.transfer(TransferService.java:30)
```

### JMX检测

```java
// 使用ThreadMXBean检测死锁
public class DeadlockDetector {

    public static void detectAndPrint() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

        // 检测死锁线程ID
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();

        if (deadlockedThreads != null && deadlockedThreads.length > 0) {
            System.err.println("发现死锁！线程数量: " + deadlockedThreads.length);

            ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(
                deadlockedThreads, true, true);

            for (ThreadInfo info : threadInfos) {
                System.err.println("死锁线程: " + info.getThreadName());
                System.err.println("等待锁: " + info.getLockName());
                System.err.println("持有锁: " + info.getLockOwnerName());

                // 打印完整栈
                System.err.println("Stack trace:");
                for (StackTraceElement element : info.getStackTrace()) {
                    System.err.println("    at " + element);
                }
            }
        }
    }
}

// 定时检测
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.scheduleAtFixedRate(() -> {
    DeadlockDetector.detectAndPrint();
}, 1, 1, TimeUnit.MINUTES);
```

### ThreadMXBean完整API

```java
public class ThreadMXBeanDemo {

    public static void analyzeThreads() {
        ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

        // 当前线程信息
        System.out.println("当前线程数: " + threadMXBean.getThreadCount());
        System.out.println("峰值线程数: " + threadMXBean.getPeakThreadCount());
        System.out.println("守护线程数: " + threadMXBean.getDaemonThreadCount());

        // 死锁检测
        long[] deadlocked = threadMXBean.findDeadlockedThreads();
        if (deadlocked != null) {
            System.out.println("发现死锁！涉及线程数: " + deadlocked.length);
        }

        // 等待监控器的线程
        long[] monitorDeadlocked = threadMXBean.findMonitorDeadlockedThreads();
        if (monitorDeadlocked != null) {
            System.out.println("发现monitor死锁！涉及线程数: " + monitorDeadlocked.length);
        }

        // 获取所有线程信息
        ThreadInfo[] allThreads = threadMXBean.dumpAllThreads(true, true);
        for (ThreadInfo info : allThreads) {
            System.out.println("线程: " + info.getThreadName()
                + ", 状态: " + info.getThreadState()
                + ", CPU: " + info.getThreadUserTime());
        }
    }
}
```

## 2. 实战案例

### 案例1：数据库连接池死锁

```java
// 问题场景：多个服务共用连接池

// 配置：HikariCP最大连接数10

// 服务A代码
@Service
public class ServiceA {
    @Autowired
    private DataSource dataSource;

    public void methodA() {
        // 获取连接1
        try (Connection conn1 = dataSource.getConnection()) {
            // 查询表A
            PreparedStatement ps1 = conn1.prepareStatement("SELECT * FROM table_a");

            // 业务处理...

            // 再获取连接2
            try (Connection conn2 = dataSource.getConnection()) {
                // 更新表B
                PreparedStatement ps2 = conn2.prepareStatement(
                    "UPDATE table_b SET ...");
            }
        }
    }
}

// 服务B代码
@Service
public class ServiceB {
    @Autowired
    private DataSource dataSource;

    public void methodB() {
        try (Connection conn2 = dataSource.getConnection()) {
            // 先更新表B
            PreparedStatement ps2 = conn2.prepareStatement("UPDATE table_b SET ...");

            try (Connection conn1 = dataSource.getConnection()) {
                // 再查询表A
                PreparedStatement ps1 = conn1.prepareStatement("SELECT * FROM table_a");
            }
        }
    }
}

// ServiceA线程1: conn1 -> conn2
// ServiceB线程1: conn2 -> conn1
// 可能形成死锁！

// 解决方案：使用单一连接，或按固定顺序获取连接
@Service
public class ServiceAFixed {
    @Autowired
    private DataSource dataSource;

    public void methodA() {
        // 统一使用单一连接
        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);

            // 先查询A
            PreparedStatement ps1 = conn.prepareStatement("SELECT * FROM table_a");
            // 更新B
            PreparedStatement ps2 = conn.prepareStatement("UPDATE table_b SET ...");

            conn.commit();
        }
    }
}
```

### 案例2：Spring事务死锁

```java
// 问题场景：Spring + MySQL + 多数据源

// ServiceA - 主数据源
@Service
public class AccountService {
    @Autowired
    private AccountMapper accountMapper;

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountMapper.selectById(fromId);
        Account to = accountMapper.selectById(toId);

        // 按ID顺序锁定，避免死锁
        if (fromId < toId) {
            accountMapper.decreaseBalance(fromId, amount);
            accountMapper.increaseBalance(toId, amount);
        } else {
            accountMapper.increaseBalance(toId, amount);
            accountMapper.decreaseBalance(fromId, amount);
        }
    }
}

// 或者使用悲观锁，按固定顺序获取
@Service
public class AccountServiceWithPessimisticLock {
    @Autowired
    private AccountMapper accountMapper;

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // 使用 SELECT FOR UPDATE 按ID顺序锁定
        Account from = accountMapper.selectByIdForUpdate(Math.min(fromId, toId));
        Account to = accountMapper.selectByIdForUpdate(Math.max(fromId, toId));

        // 先锁定编号小的账户
        Account first = fromId < toId ? from : to;
        Account second = fromId < toId ? to : from;

        accountMapper.decreaseBalance(first.getId(), amount);
        accountMapper.increaseBalance(second.getId(), amount);
    }
}

// Mapper中使用FOR UPDATE
@Select("SELECT * FROM account WHERE id = #{id} FOR UPDATE")
Account selectByIdForUpdate(Long id);
```

### 案例3：消息队列消费死锁

```java
// 问题场景：消费者处理消息时产生新消息

@Service
public class OrderMessageConsumer {

    @Autowired
    private OrderService orderService;

    @Autowired
    private MessageProducer messageProducer;

    // RocketMQ consumer
    @RocketMQMessageListener(topic = "order-topic", consumerGroup = "order-consumer")
    public void consume(OrderMessage message) {
        try {
            // 处理订单
            orderService.processOrder(message);

            // 发送延迟消息
            messageProducer.sendDelayMessage(
                "delay-topic",
                "order.delay.check",
                message.getOrderId()
            );
        } catch (Exception e) {
            // 异常处理
        }
    }
}

// 如果MessageProducer内部也使用该Consumer的线程池
// 可能导致线程池耗尽 -> 死锁

// 解决方案：使用不同的线程池
@Configuration
public class MQConfig {
    @Bean("consumerExecutor")
    public Executor consumerExecutor() {
        return new ThreadPoolExecutor(10, 20, 60,
            TimeUnit.SECONDS, new ArrayBlockingQueue<>(100));
    }

    @Bean("producerExecutor")
    public Executor producerExecutor() {
        return new ThreadPoolExecutor(5, 10, 60,
            TimeUnit.SECONDS, new ArrayBlockingQueue<>(100));
    }
}
```

## 3. 死锁预防策略

### 固定加锁顺序

```java
// 原则：所有需要获取多个锁的地方，都按固定顺序加锁

// 反例
public void badMethod(Object A, Object B) {
    synchronized (A) {
        synchronized (B) {
            // 操作
        }
    }
}

// 正例：固定顺序
public void goodMethod(Object A, Object B) {
    // 始终按hash值顺序加锁
    if (A.hashCode() < B.hashCode()) {
        synchronized (A) {
            synchronized (B) {
                // 操作
            }
        }
    } else {
        synchronized (B) {
            synchronized (A) {
                // 操作
            }
        }
    }
}

// 更简洁的方式：使用Object.identityHashCode
public void bestMethod(Object A, Object B) {
    Object first = System.identityHashCode(A) < System.identityHashCode(B) ? A : B;
    Object second = first == A ? B : A;

    synchronized (first) {
        synchronized (second) {
            // 操作
        }
    }
}
```

### 减少锁粒度

```java
// 问题：粗粒度锁
public class，粗粒度 {
    private final Object lock = new Object();
    private int counter = 0;
    private long timestamp = 0;
    private String status = "";

    public void increment() {
        synchronized (lock) {
            counter++;
        }
    }

    public void update() {
        synchronized (lock) {
            timestamp = System.currentTimeMillis();
        }
    }
}

// 改进：细粒度锁（可能引入读写分离）
public class FineGrained {
    private final AtomicInteger counter = new AtomicInteger(0);
    private final AtomicLong timestamp = new AtomicLong(0);
    private final volatile String status = "";

    public void increment() {
        counter.incrementAndGet();
    }

    public void update() {
        timestamp.set(System.currentTimeMillis());
    }
}

// 或使用读写锁
public class ReadWriteLockExample {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private int counter = 0;

    public void increment() {
        rwLock.writeLock().lock();
        try {
            counter++;
        } finally {
            rwLock.writeLock().unlock();
        }
    }

    public int get() {
        rwLock.readLock().lock();
        try {
            return counter;
        } finally {
            rwLock.readLock().unlock();
        }
    }
}
```

### 使用显式锁

```java
// 尝试获取锁，超时则放弃
public class TryLockExample {

    private final Lock lock = new ReentrantLock();

    public void doSomething() {
        // 尝试获取锁，最多等待1秒
        boolean acquired = lock.tryLock(1, TimeUnit.SECONDS);

        if (acquired) {
            try {
                // 业务逻辑
            } finally {
                lock.unlock();
            }
        } else {
            // 获取锁失败，放弃或重试
            handleLockFailure();
        }
    }
}

// 避免死锁：检测到可能死锁时回退
public class DeadlockAvoidance {

    private final Lock lockA = new ReentrantLock();
    private final Lock lockB = new ReentrantLock();

    public void transfer(Account a, Account b, BigDecimal amount) {
        // 尝试按顺序获取锁
        if (a.id() < b.id()) {
            if (lockA.tryLock()) {
                try {
                    if (lockB.tryLock()) {
                        try {
                            performTransfer(a, b, amount);
                        } finally {
                            lockB.unlock();
                        }
                    } else {
                        // 获取B锁失败，释放A锁，重试
                        retry(a, b, amount);
                    }
                } finally {
                    lockA.unlock();
                }
            }
        } else {
            // 顺序相反
            if (lockB.tryLock()) {
                try {
                    if (lockA.tryLock()) {
                        try {
                            performTransfer(a, b, amount);
                        } finally {
                            lockA.unlock();
                        }
                    } else {
                        retry(a, b, amount);
                    }
                } finally {
                    lockB.unlock();
                }
            }
        }
    }
}
```

### 数据库死锁预防

```sql
-- 1. 按固定顺序操作
-- 始终先操作ID小的表
BEGIN;
SELECT * FROM account WHERE id = 1 FOR UPDATE;  -- 始终先锁ID小的
SELECT * FROM account WHERE id = 2 FOR UPDATE;  -- 再锁ID大的
UPDATE account SET balance = balance - 100 WHERE id = 1;
UPDATE account SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- 2. 减少锁持有时间
-- 批量操作改为单条操作，降低冲突概率

-- 3. 合理使用索引
-- 索引可以减少锁的范围

-- 4. 设置锁超时
SET innodb_lock_wait_timeout = 5;  -- 5秒超时

-- 5. 检查死锁日志
SHOW ENGINE INNODB STATUS;
-- 查看 LATEST DETECTED DEADLOCK 部分
```

## 4. 监控与检测工具

### Arthas检测

```bash
# 1. 使用thread命令
thread -n 10
# 查看所有线程，找到阻塞的线程

# 2. 使用thread -b检测死锁
thread -b
# Arthas会自动检测并输出死锁信息

# 3. 使用monitor监控
monitor -c 5 com.example.service.OrderService createOrder
# 监控方法调用，查看是否有长时间阻塞

# 4. 使用trace追踪
trace com.example.service.OrderService createOrder
# 追踪方法调用链路
```

### JConsole检测

```bash
# 启动JConsole
jconsole

# 连接到Java进程
# 在"线程"标签页可以看到：
# - 所有线程列表
# - 线程状态
# - 死锁检测按钮
```

### VisualVM检测

```bash
# 启动VisualVM
jvisualvm

# 功能：
# - 线程Dump
# - 死锁检测
# - CPU/内存分析
```

### Spring Boot Actuator

```yaml
# 暴露线程相关端点
management:
  endpoints:
    web:
      exposure:
        include: health,info,threaddump,heapdump
  endpoint:
    threaddump:
      enabled: true

# 访问 /actuator/threaddump 获取线程dump
```

## 5. 常见面试题

**Q1：什么是死锁？如何避免死锁？**

> 参考答案：死锁是多个线程相互持有对方需要的资源，形成循环等待，导致都无法继续执行。避免死锁的方法：1）固定加锁顺序；2）减少锁粒度；3）使用tryLock尝试获取锁；4）设置锁超时；5）使用读写锁分离读写操作；6）数据库操作按固定顺序执行。

**Q2：如何检测Java程序中的死锁？**

> 参考答案：1）使用jstack <pid>，输出中如果有死锁会显示"Found one Java-level deadlock"；2）使用ThreadMXBean的findDeadlockedThreads()方法编程检测；3）使用Arthas的thread -b命令；4）使用JConsole、VisualVM等工具。死锁检测的原理是周期性检查线程状态，找出形成循环等待的线程。

**Q3：synchronized和ReentrantLock在死锁方面有什么区别？**

> 参考答案：synchronized是隐式获取和释放锁，不支持尝试获取，无法设置超时；ReentrantLock是显式锁，支持tryLock()尝试获取，支持超时设置。ReentrantLock可以通过tryLock(timeout)避免无限等待，更容易编写死锁避免的代码。两者在互斥性上相同，但ReentrantLock更灵活。

**Q4：数据库死锁和Java线程死锁有什么异同？**

> 参考答案：原理相同，都是循环等待资源。但数据库死锁由数据库管理系统自动检测和回滚（通常回滚代价最小的事务），而Java线程死锁需要程序员自己处理。数据库死锁通常是不同事务对不同表的加锁顺序不一致导致。解决思路类似：按固定顺序访问资源、减少事务时长、使用低隔离级别等。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                    死锁处理方法论                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  检测工具：                                                       │
│  ├─ jstack - 打印线程栈，显示死锁信息                            │
│  ├─ ThreadMXBean - 编程检测死锁                                 │
│  ├─ Arthas thread -b - 快速检测死锁                              │
│  └─ JConsole/VisualVM - 可视化工具                              │
│                                                                │
│  预防策略：                                                       │
│  ├─ 固定加锁顺序                                                 │
│  ├─ 减少锁粒度                                                  │
│  ├─ tryLock尝试获取                                             │
│  ├─ 设置锁超时                                                  │
│  └─ 减少事务时长                                                │
│                                                                │
│  解决方案：                                                       │
│  ├─ 中断死锁线程（不推荐）                                       │
│  ├─ 使用超时自动放弃                                             │
│  ├─ 重新启动服务（紧急处理）                                     │
│  └─ 代码修复后重新部署                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

死锁是最难调试的并发问题之一，预防胜于治疗。通过良好的编码习惯（固定加锁顺序、减少锁粒度）和完善的监控告警，可以在死锁发生前发现问题。
