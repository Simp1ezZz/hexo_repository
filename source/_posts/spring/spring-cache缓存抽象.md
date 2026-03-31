---
title: Spring Cache：缓存抽象与@Cacheable注解原理
date: 2024-04-16 13:32:45
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring Cache,@Cacheable,缓存抽象,EhCache,RedisCache
description: 深入理解Spring Cache抽象机制，@Cacheable注解原理与缓存失效处理
cover:
---

## 前言

缓存是提升系统性能的关键技术。Spring Cache 提供了统一的缓存抽象，支持多种缓存实现。本文深入解析 Spring Cache 的原理与使用。

## Spring Cache抽象

### 核心接口

```java
public interface Cache {
    // 缓存名称
    String getName();

    // 底层存储
    Object getNativeCache();

    // 获取缓存值
    <T> T get(Object key);

    // 获取缓存值（带回调）
    <T> T get(Object key, Callable<T> valueLoader);

    // 放入缓存
    void put(Object key, Object value);

    // 删除缓存
    void evict(Object key);

    // 清空所有
    void clear();

    // ValueWrapper
    ValueWrapper get(Object key);
}
```

### CacheManager

```java
public interface CacheManager {
    // 根据名称获取Cache
    Cache getCache(String name);

    // 获取所有缓存名称
    Collection<String> getCacheNames();
}
```

---

## @Cacheable注解

### 基本使用

```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id);
    }

    @Cacheable(value = "users", key = "#name")
    public User getUserByName(String name) {
        return userRepository.findByName(name);
    }
}
```

### 注解属性

```java
@Cacheable(
    value = "users",           // 缓存名称（类似分区）
    key = "#id",              // SpEL表达式，计算缓存key
    keyGenerator = "myKeyGenerator",  // 自定义KeyGenerator
    cacheManager = "cacheManager",      // 指定CacheManager
    cacheResolver = "cacheResolver",    // 指定CacheResolver
    condition = "#id > 0",   // SpEL条件，true才缓存
    unless = "#result == null",  // SpEL条件，true不缓存
    sync = false              // 是否同步，true则多线程等待
)
public User getUserById(Long id) {
    return userRepository.findById(id);
}
```

### SpEL表达式

```java
@Cacheable(value = "users", key = "#id")
    // #id - 方法参数id

@Cacheable(value = "users", key = "#user.id")
    // #user.id - 方法参数user的id属性

@Cacheable(value = "users", key = "#root.methodName + ':' + #id")
    // 拼接方法名和参数

@Cacheable(value = "users", key = "#p0")
    // #p0 - 第一个参数

@Cacheable(value = "users", key = "#result.id")
    // #result - 返回值（方法执行后）

@Cacheable(value = "users", key = "#cacheName")
    // #cacheName - 注解的cacheNames属性
```

---

## 缓存管理器实现

### ConcurrentMapCacheManager

```java
// 使用ConcurrentHashMap存储的简单实现
@Configuration
public class SimpleCacheConfig {
    @Bean
    public CacheManager cacheManager() {
        ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
        cacheManager.setCacheNames(Arrays.asList("users", "orders"));
        return cacheManager;
    }
}
```

### CaffeineCacheManager

```java
// 高性能缓存实现
@Configuration
public class CaffeineCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)           // 最大缓存数
            .expireAfterWrite(10, TimeUnit.MINUTES)  // 写入后过期
            .expireAfterAccess(5, TimeUnit.MINUTES)  // 访问后过期
            .recordStats());            // 记录统计
        return cacheManager;
    }
}
```

### RedisCacheManager

```java
@Configuration
public class RedisCacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))  // 默认过期时间
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();  // 不缓存null

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withInitialCacheConfigurations(Collections.singletonMap(
                "users",
                config.entryTtl(Duration.ofMinutes(10))  // users缓存10分钟
            ))
            .transactionAware()
            .build();
    }
}
```

---

## @CacheEvict与@CachePut

### @CacheEvict删除缓存

```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        return userRepository.findById(id);
    }

    // 删除单个缓存
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }

    // 删除所有缓存
    @CacheEvict(value = "users", allEntries = true)
    public void deleteAllUsers() {
        userRepository.deleteAll();
    }

    // 方法执行前删除缓存（beforeInvocation=true）
    @CacheEvict(value = "users", allEntries = true, beforeInvocation = true)
    public void reloadUsers() {
        userRepository.reload();
    }
}
```

### @CachePut更新缓存

```java
@Service
public class UserService {

    // 每次都执行方法，结果放入缓存
    @CachePut(value = "users", key = "#result.id")
    public User updateUser(User user) {
        return userRepository.save(user);
    }

    // 不影响原方法执行，只更新缓存
    @CachePut(value = "users", key = "#user.id")
    public User refreshCache(User user) {
        // 自定义刷新逻辑
        return refreshUser(user);
    }
}
```

### @Caching组合注解

```java
@Service
public class UserService {

    @Caching(
        evict = {
            @CacheEvict(value = "users", key = "#id"),
            @CacheEvict(value = "userNames", key = "#result.name")  // 如果返回非空
        },
        put = {
            @CachePut(value = "users", key = "#result.id")
        }
    )
    public User saveUser(User user) {
        return userRepository.save(user);
    }
}
```

---

## 缓存实现原理

### EnableCaching

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
    // 代理模式：JDK代理还是CGLIB
    ProxyMode mode() default ProxyMode.TARGET_CLASS;
}
```

### CachingConfigurationSelector

```java
public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {

    @Override
    public String[] selectImports(AdviceMode adviceMode) {
        if (adviceMode == AdviceMode.PROXY) {
            return new String[] {
                AutoProxyRegistrar.class.getName(),
                ProxyCachingConfiguration.class.getName()
            };
        } else {
            return new String[] {
                AnnotationCachingConfiguration.class.getName()
            };
        }
    }
}
```

### ProxyCachingConfiguration

```java
@Configuration
public class ProxyCachingConfiguration {

    @Bean
    public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor(
            CacheOperationSource cacheOperationSource,
            CacheInterceptor cacheInterceptor) {

        BeanFactoryCacheOperationSourceAdvisor advisor =
            new BeanFactoryCacheOperationSourceAdvisor();
        advisor.setCacheOperationSource(cacheOperationSource);
        advisor.setAdvice(cacheInterceptor);
        return advisor;
    }

    @Bean
    public CacheInterceptor cacheInterceptor(
            CacheManager cacheManager) {

        CacheInterceptor interceptor = new CacheInterceptor();
        interceptor.setCacheManager(cacheManager);
        interceptor.setCacheOperationSources(cacheOperationSource);
        return interceptor;
    }
}
```

### CacheInterceptor

```java
public class CacheInterceptor extends CacheAspectSupport {

    @Override
    protected Object execute(CacheOperationInvoker invoker, Object target,
                            Method method, Object[] args) {

        CacheOperationSources cacheOpSources = getCacheOperationSources();
        if (cacheOpSources == null) {
            return invoker.invoke();
        }

        // 获取缓存操作
        Collection<CacheOperation> operations = cacheOpSources.getCacheOperations(method, targetClass);

        // 执行缓存切面
        return new CacheAspectSupport.CachingExecutor(inoker, operations).invoke();
    }
}
```

---

## 缓存失效处理

### CacheAside模式

```java
// Read: 先查缓存，缓存没有则查数据库并放入缓存
@Cacheable(value = "users", key = "#id")
public User getUserById(Long id) {
    return userRepository.findById(id);
}

// Write: 更新数据库后删除缓存
@CacheEvict(value = "users", key = "#user.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

### 双写模式

```java
// 更新数据库和缓存
@CachePut(value = "users", key = "#result.id")
public User updateUser(User user) {
    return userRepository.save(user);
}
```

### 缓存雪崩

```java
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        Random random = new Random();

        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users",
                config.entryTtl(Duration.ofMinutes(10 + random.nextInt(5))))  // 随机过期时间
            .build();
    }
}
```

### 缓存击穿

```java
// sync=true 多线程等待
@Cacheable(value = "users", key = "#id", sync = true)
public User getUserById(Long id) {
    return userRepository.findById(id);
}

// 自定义锁实现
@Service
public class UserService {

    private final ConcurrentHashMap<Long, ReentrantLock> locks = new ConcurrentHashMap<>();

    public User getUserById(Long id) {
        User user = cache.get(id);
        if (user == null) {
            ReentrantLock lock = locks.computeIfAbsent(id, k -> new ReentrantLock());
            try {
                lock.lock();
                user = cache.get(id);  // double check
                if (user == null) {
                    user = userRepository.findById(id);
                    cache.put(id, user);
                }
            } finally {
                lock.unlock();
            }
        }
        return user;
    }
}
```

---

## 常见面试题

**Q1：@Cacheable和@CachePut的区别？**

> 答：@Cacheable先查缓存，缓存命中则直接返回，不执行方法；@CachePut每次都执行方法并将结果放入缓存。

**Q2：Spring Cache的缓存淘汰策略？**

> 答：取决于底层CacheManager实现。ConcurrentMapCacheManager基于LRU；Caffeine可配置LRU、LFU、TTL等；RedisCacheManager可配置TTL。

**Q3：如何实现缓存穿透？**

> 答：使用布隆过滤器或存储null值（设置短过期时间）。

**Q4：@CacheEvict的beforeInvocation属性作用？**

> 答：true表示方法执行前删除缓存；false（默认）表示方法执行成功后删除缓存。

**Q5：Spring Cache和Spring Boot Cache的关系？**

> 答：Spring Boot自动配置spring.cache.*配置，简化了CacheManager的创建，提供统一的缓存配置入口。

---

## 总结

Spring Cache提供了统一的缓存抽象：
- **@Cacheable**：先查缓存，命中则跳过方法执行
- **@CachePut**：每次执行方法并更新缓存
- **@CacheEvict**：删除缓存条目
- **CacheManager**：管理多种缓存实现
- **SpEL表达式**：灵活定义缓存key和条件
