---
title: Redis分布式锁：实现原理与Redisson使用
date: 2023-06-26 14:16:20
tags:
  - 文章
categories: 学习
---


## 前言

在分布式系统中，由于多进程、多节点的存在，传统的单机锁已经无法满足需求。分布式锁应运而生，成为保障分布式系统资源互斥访问的重要手段。Redis凭借其高性能和丰富的数据结构，成为实现分布式锁的主流选择之一。本文将深入探讨Redis分布式锁的实现原理、常见问题、以及Redisson框架的使用。

## 为什么需要分布式锁

### 单机锁的局限

```java
// 单机环境：synchronized可以解决问题
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }
}

// 分布式环境：多台服务器，无法使用synchronized
// server1 和 server2 各有一个Counter实例
// synchronized只能保护本地实例
```

### 分布式锁的场景

```markdown
1. 库存扣减
   - 多个服务实例同时处理订单
   - 需要保证库存不超卖

2. 定时任务执行
   - 多节点部署定时任务
   - 同一时刻只能有一个节点执行

3. 幂等性控制
   - 防止重复提交
   - 保证接口幂等

4. 分布式session
   - 多服务实例共享session
   - 需要互斥访问
```

## 1. 基础实现：SETNX

### 最简单的分布式锁

```java
public class SimpleRedisLock {

    private Jedis jedis;
    private String lockKey;
    private String lockValue;

    public SimpleRedisLock(Jedis jedis, String lockKey) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID().toString();
    }

    /**
     * 获取锁
     * @param expireTime 过期时间（秒）
     * @return 是否获取成功
     */
    public boolean tryLock(long expireTime) {
        // SETNX + EXPIRE 组合，不是原子操作
        Long result = jedis.setnx(lockKey, lockValue);
        if (result == 1) {
            jedis.expire(lockKey, (int) expireTime);
            return true;
        }
        return false;
    }

    /**
     * 释放锁
     */
    public void unlock() {
        // 直接删除key
        jedis.del(lockKey);
    }
}
```

### 问题分析

```java
// 问题1：不是原子操作
// SETNX成功但EXPIRE失败，锁永不释放

// 问题2：误删其他人的锁
// 线程A执行超时，锁自动释放
// 线程B获取了锁
// 线程A执行完成，调用unlock删除锁
// 结果：线程B的锁被删除了！

// 问题3：无法重入
// 同一个线程不能多次获取锁
```

## 2. 改进实现：SET EX NX

### 原子性获取锁

```java
public class RedisLockV2 {

    private Jedis jedis;
    private String lockKey;
    private String lockValue;
    private long expireTime;

    public RedisLockV2(Jedis jedis, String lockKey, long expireTime) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID().toString() + ":" + Thread.currentThread().getId();
        this.expireTime = expireTime;
    }

    /**
     * 原子性获取锁（SET key value EX timeout NX）
     */
    public boolean tryLock() {
        String result = jedis.set(lockKey, lockValue, new SetParams().nx().ex(expireTime));
        return "OK".equals(result);
    }

    /**
     * 释放锁（只能释放自己的锁）
     */
    public void unlock() {
        // 使用Lua脚本保证原子性
        String script =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

        jedis.eval(script,
                   Collections.singletonList(lockKey),
                   Collections.singletonList(lockValue));
    }
}
```

### Lua脚本保证原子性

```lua
-- 解锁脚本
-- 只有锁的值匹配时才能删除，防止误删

if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
else
    return 0
end
```

## 3. 过期时间与锁续期

### 问题：锁自动释放过早

```java
// 场景：任务执行时间 > 锁过期时间

// 1. 线程A获取锁，设置过期时间30秒
// 2. 任务执行时间较长，25秒时还没执行完
// 3. 锁过期自动释放
// 4. 线程B获取锁，开始执行
// 5. 线程A执行完成，释放锁（释放了线程B的锁！）
// 6. 线程C获取锁
// 结果：多个线程同时持有锁，并发执行！
```

### 解决方案：Watch Dog

```java
public class RedisLockWithWatchdog {

    private static final long EXPIRE_TIME = 30; // 锁过期时间
    private static final long RENEW_INTERVAL = 10; // 续期间隔

    private String lockKey;
    private String lockValue;
    private boolean locked;
    private ScheduledExecutorService scheduledExecutor;

    public RedisLockWithWatchdog(String lockKey) {
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID().toString();
        this.scheduledExecutor = Executors.newSingleThreadScheduledExecutor();
    }

    public boolean tryLock() {
        String result = jedis.set(lockKey, lockValue, new SetParams().nx().ex(EXPIRE_TIME));
        if ("OK".equals(result)) {
            locked = true;
            startWatchDog(); // 启动看门狗
            return true;
        }
        return false;
    }

    /**
     * 看门狗：自动续期
     * 每隔 1/3 过期时间 检查并续期
     */
    private void startWatchDog() {
        scheduledExecutor.scheduleAtFixedRate(() -> {
            if (locked && isLockValid()) {
                // 续期：重新设置过期时间
                jedis.expire(lockKey, EXPIRE_TIME);
                System.out.println("Lock renewed: " + lockKey);
            }
        }, RENEW_INTERVAL, RENEW_INTERVAL, TimeUnit.SECONDS);
    }

    private boolean isLockValid() {
        return lockValue.equals(jedis.get(lockKey));
    }

    public void unlock() {
        // 停止看门狗
        locked = false;
        scheduledExecutor.shutdown();

        // 删除锁
        String script =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

        jedis.eval(script,
                   Collections.singletonList(lockKey),
                   Collections.singletonList(lockValue));
    }
}
```

## 4. Redisson实现分布式锁

### Redisson简介

Redisson是Redis的Java客户端，提供了丰富的分布式数据结构和服务，其中最著名的就是分布式锁的实现。

### Maven依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.24.3</version>
</dependency>
```

### 基础配置

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        // 单节点
        config.useSingleServer()
              .setAddress("redis://127.0.0.1:6379")
              .setPassword("redis123")
              .setConnectionPoolSize(64)
              .setConnectionMinimumIdleSize(10);

        // 集群模式
        // config.useClusterServers()
        //       .addNodeAddress("redis://127.0.0.1:7181");

        return Redisson.create(config);
    }
}
```

### 简单使用

```java
@Service
public class OrderService {

    @Autowired
    private RedissonClient redissonClient;

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private StockMapper stockMapper;

    /**
     * 扣减库存
     */
    public void createOrder(Long productId, Long userId) {
        String lockKey = "stock:lock:" + productId;
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // 尝试获取锁，默认等待-1（不等待），自动解锁30秒
            // lock.lock() 会自动续期
            boolean locked = lock.tryLock(10, 30, TimeUnit.SECONDS);
            if (!locked) {
                throw new BusinessException("系统繁忙，请稍后重试");
            }

            // 查询库存
            Stock stock = stockMapper.selectByProductId(productId);
            if (stock.getCount() <= 0) {
                throw new BusinessException("库存不足");
            }

            // 扣减库存
            stockMapper.decreaseStock(productId, 1);

            // 创建订单
            Order order = new Order();
            order.setUserId(userId);
            order.setProductId(productId);
            orderMapper.insert(order);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new BusinessException("创建订单失败");
        } finally {
            // 释放锁
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

### 常用API

```java
// 获取锁
RLock lock = redissonClient.getLock("anyLock");

// 阻塞获取锁，等同于 lock.lock()
// 底层自动开启看门狗机制
lock.lock();

// 阻塞获取锁，指定超时时间
// 注意：如果leaseTime不为-1，则不会自动续期
lock.lock(10, TimeUnit.SECONDS);

// 尝试获取锁
// waitTime: 最大等待时间
// leaseTime: 锁持有时间
boolean locked = lock.tryLock(100, 30, TimeUnit.SECONDS);

// 尝试获取锁，指定leaseTime（不自动续期）
boolean locked = lock.tryLock(10, 30, TimeUnit.SECONDS);

// 释放锁
lock.unlock();

// 强制释放锁（不检查锁持有者）
lock.forceUnlock();

// 判断是否被锁定
boolean isLocked = lock.isLocked();

// 判断当前线程是否持有锁
boolean isHeldByCurrentThread = lock.isHeldByCurrentThread();

// 获取剩余等待时间
long remainingTime = lock.remainTimeToLive();
```

### 可重入锁（Reentrant Lock）

```java
public class ReentrantLockDemo {

    @Autowired
    private RedissonClient redissonClient;

    public void outer() {
        RLock lock = redissonClient.getLock("resource");
        lock.lock();
        try {
            System.out.println("outer");
            inner(); // 可重入，同一个线程可以再次获取锁
        } finally {
            lock.unlock();
        }
    }

    public void inner() {
        RLock lock = redissonClient.getLock("resource");
        lock.lock();
        try {
            System.out.println("inner");
        } finally {
            lock.unlock();
        }
    }
}
```

### 公平锁（Fair Lock）

```java
// 普通锁不保证申请顺序
// 公平锁按照申请顺序获取锁（FIFO）

RLock fairLock = redissonClient.getFairLock("fairResource");
fairLock.lock(10, TimeUnit.SECONDS);
try {
    // 业务逻辑
} finally {
    fairLock.unlock();
}
```

### 读写锁（ReadWrite Lock）

```java
public class ReadWriteLockDemo {

    @Autowired
    private RedissonClient redissonClient;

    // 缓存数据
    private Map<String, Object> cache = new ConcurrentHashMap<>();

    /**
     * 读取数据（读锁）
     * 多个线程可以同时获取读锁
     */
    public Object get(String key) {
        RReadWriteLock rwLock = redissonClient.getReadWriteLock("dataLock");
        RLock readLock = rwLock.readLock();

        readLock.lock();
        try {
            if (cache.containsKey(key)) {
                return cache.get(key);
            }
            // 缓存未命中，从数据库读取
            Object value = loadFromDb(key);
            cache.put(key, value);
            return value;
        } finally {
            readLock.unlock();
        }
    }

    /**
     * 写入数据（写锁）
     * 写锁独占，其他线程无法获取读锁或写锁
     */
    public void set(String key, Object value) {
        RReadWriteLock rwLock = redissonClient.getReadWriteLock("dataLock");
        RLock writeLock = rwLock.writeLock();

        writeLock.lock();
        try {
            cache.put(key, value);
            // 可能还需要写回数据库
        } finally {
            writeLock.unlock();
        }
    }
}
```

### 联锁（MultiLock）

```java
// 同时锁定多个资源（必须全部获取成功）

public void processOrder() {
    RLock lock1 = redissonClient.getLock("resource1");
    RLock lock2 = redissonClient.getLock("resource2");
    RLock lock3 = redissonClient.getLock("resource3");

    // 创建联锁
    RLock multiLock = redissonClient.getMultiLock(lock1, lock2, lock3);

    multiLock.lock();
    try {
        // 同时锁定了3个资源
        // 业务逻辑
    } finally {
        multiLock.unlock();
    }
}
```

### 红锁（RedLock）

```java
// Redisson提供的红锁算法
// 在多个Redis节点上获取锁，减少单点故障

// 需要多个独立的Redis实例
RLock lock1 = redissonClient1.getLock("resource");
RLock lock2 = redissonClient2.getLock("resource");
RLock lock3 = redissonClient3.getLock("resource");

RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
redLock.lock();
try {
    // 业务逻辑
} finally {
    redLock.unlock();
}
```

## 5. 分布式锁常见问题

### 问题1：锁过期，任务未完成

```java
// 场景：任务执行时间超过锁过期时间
// 解决：设置合理的过期时间 + 看门狗续期

// 方案1：设置较长的过期时间
lock.lock(60, TimeUnit.SECONDS); // 60秒后自动释放

// 方案2：使用看门狗自动续期
lock.lock(); // 不设置leaseTime，底层自动续期
```

### 问题2：Redis主从切换丢锁

```
问题场景：
1. 客户端在Master获取锁
2. Master宕机，数据未同步到Slave
3. Slave升级为Master
4. 新Master没有锁数据
5. 另一个客户端获取锁成功
结果：两个客户端同时持有锁！
```

```
解决方案：RedLock
- 在N个独立的Redis实例上获取锁
- 超过N/2+1个成功才算获取成功
- 缺点：需要多个Redis实例，增加复杂度
```

### 问题3：锁被误释放

```java
// 场景：锁过期后被其他线程获取，原线程执行完成释放了别人的锁

// 解决：使用唯一标识作为锁值，释放时校验

String uniqueValue = UUID.randomUUID().toString() + ":" + Thread.currentThread().getId();
jedis.set(lockKey, uniqueValue, new SetParams().nx().ex(30));

// Lua脚本释放：只有值匹配才能释放
"if redis.call('get', KEYS[1]) == ARGV[1] then " +
"    return redis.call('del', KEYS[1]) " +
"else " +
"    return 0 " +
"end"
```

### 问题4：锁不可重入

```java
// 普通锁不可重入，同一线程多次获取会死锁

// 解决：使用可重入锁
// Redisson的RLock本身就是可重入锁

RLock lock = redissonClient.getLock("resource");
lock.lock();
try {
    doSomething();
    lock.lock(); // 同一线程再次获取，成功
    try {
        doOtherThing();
    } finally {
        lock.unlock();
    }
} finally {
    lock.unlock();
}
```

## 6. 最佳实践

### 代码模板

```java
public class DistributedLockTemplate {

    private RedissonClient redissonClient;

    public <T> T executeWithLock(String lockKey, long waitTime, TimeUnit unit, Supplier<T> supplier) {
        RLock lock = redissonClient.getLock(lockKey);
        boolean locked = false;

        try {
            locked = lock.tryLock(waitTime, unit);
            if (!locked) {
                throw new LockException("获取锁失败: " + lockKey);
            }
            return supplier.get();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new LockException("获取锁被中断", e);
        } finally {
            if (locked && lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    public void executeWithLock(String lockKey, Runnable runnable) {
        executeWithLock(lockKey, 10, TimeUnit.SECONDS, () -> {
            runnable.run();
            return null;
        });
    }
}
```

### 使用示例

```java
@Service
public class ProductService {

    @Autowired
    private DistributedLockTemplate lockTemplate;

    @Autowired
    private ProductMapper productMapper;

    public void decreaseStock(Long productId, Integer count) {
        String lockKey = "product:stock:lock:" + productId;

        lockTemplate.executeWithLock(lockKey, () -> {
            Product product = productMapper.selectById(productId);
            if (product.getStock() < count) {
                throw new BusinessException("库存不足");
            }
            productMapper.decreaseStock(productId, count);
        });
    }
}
```

### 锁粒度控制

```java
// 好：细粒度锁
String lockKey = "order:lock:" + orderId; // 按订单ID加锁
// 不同订单可以并发处理

// 差：粗粒度锁
String lockKey = "order:lock:all"; // 全局锁
// 所有订单串行处理，并发性能差
```

## 7. 常见面试题

**Q1：Redis实现分布式锁有哪些关键点？**

> 参考答案：1）使用SET key value NX EX timeout原子性获取锁；2）锁值使用唯一标识防止误删；3）使用Lua脚本原子性释放锁；4）考虑锁续期（看门狗机制）；5）处理锁超时场景；6）考虑Redis主从切换丢锁问题（RedLock方案）。

**Q2：如何保证Redis分布式锁的可靠性？**

> 参考答案：1）使用带过期时间的原子性SET命令；2）value使用唯一标识，释放时校验；3）使用Lua脚本保证释放的原子性；4）开启看门狗自动续期；5）使用RedLock在多个独立Redis实例获取锁；6）合理评估锁的粒度和过期时间。

**Q3：Redis分布式锁能完全替代分布式事务吗？**

> 参考答案：不能完全替代。Redis分布式锁只能保证资源的互斥访问，无法保证多个操作的原子性。如果需要跨多个资源的事务保证，建议使用分布式事务方案（如Seata的AT、TCC模式），或者将操作设计成最终一致性的模式。

**Q4：Redisson的看门狗机制是什么？**

> 参考答案：看门狗是Redisson实现的一种自动锁续期机制。当使用lock()方法获取锁且不指定leaseTime时，看门狗会启动，默认每10秒检查一次，如果锁还被持有且未过期，就自动续期30秒。这个过程持续到锁被主动释放或在持有者异常退出时过期。这解决了任务执行时间超过锁过期时间的问题。

## 总结

```
┌─────────────────────────────────────────────────────────────┐
│                    Redis分布式锁实现                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  基础版：SETNX                                               │
│  ├─ 问题：非原子性、误删锁、不可重入                          │
│  └─ 适用：简单场景                                           │
│                                                             │
│  改进版：SET key value NX EX                                 │
│  ├─ 原子性获取锁                                             │
│  ├─ Lua脚本释放锁                                           │
│  └─ 适用：一般场景                                           │
│                                                             │
│  进阶版：看门狗自动续期                                       │
│  ├─ 解决锁过期任务未完成的问题                                │
│  └─ 适用：任务执行时间不确定的场景                            │
│                                                             │
│  生产级：Redisson                                            │
│  ├─ 可重入锁、公平锁、读写锁、联锁、红锁                       │
│  ├─ 看门狗自动续期                                           │
│  └─ 适用：生产环境                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Redis分布式锁是分布式系统中的重要组件，正确实现需要考虑原子性、误删、过期时间、续期等多个方面。在生产环境中，推荐使用Redisson等成熟框架，其分布式锁实现已经过大量验证，能够应对各种边界场景。
