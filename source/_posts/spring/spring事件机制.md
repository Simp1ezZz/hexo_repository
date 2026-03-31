---
title: Spring事件机制：ApplicationEvent与观察者模式
date: 2026-04-23 10:00:00
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring,ApplicationEvent,事件监听,观察者模式,@EventListener
description: 深入理解Spring事件机制，基于观察者模式的组件解耦方案
cover:
---

## 前言

Spring 事件机制是基于观察者模式的解耦方案，可以让组件之间在不直接依赖的情况下通信。本文深入解析 Spring 事件机制的原理与使用。

## 观察者模式

### 模式结构

```
┌─────────────────────────────────────────────────────────────────┐
│                      观察者模式                                    │
│                                                                   │
│  ┌─────────────┐                                                 │
│  │  Subject    │                                                 │
│  │ (被观察者)  │                                                 │
│  ├─────────────┤                                                 │
│  │ observers[] │                                                 │
│  │ attach()   │ ────────────────────────────────────────────▶  │
│  │ detach()   │                                                 │
│  │ notify()   │                                                 │
│  └─────────────┘                                                 │
│          │                                                        │
│          │ 通知                                                  │
│          ▼                                                        │
│  ┌─────────────┐                                                 │
│  │ Observer 1  │                                                 │
│  └─────────────┘                                                 │
│  ┌─────────────┐                                                 │
│  │ Observer 2  │                                                 │
│  └─────────────┘                                                 │
│  ┌─────────────┐                                                 │
│  │ Observer N  │                                                 │
│  └─────────────┘                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Spring事件核心组件

### 核心接口

```java
// 事件源
public abstract class ApplicationEvent extends EventObject {
    private final long timestamp;
    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }
}

// 事件监听器
public interface ApplicationListener<E extends ApplicationEvent> {
    void onApplicationEvent(E event);
}

// 事件发布器
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {}
    default void publishEvent(Object event) {}  // JDK9+
}

// 事件广播器
public interface ApplicationEventMulticaster {
    void addApplicationListener(ApplicationListener<?> listener);
    void removeApplicationListener(ApplicationListener<?> listener);
    void multicastEvent(ApplicationEvent event);
}
```

---

## 事件发布与监听

### 基本使用

```java
// 1. 定义事件
public class UserCreatedEvent extends ApplicationEvent {
    private final User user;

    public UserCreatedEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() {
        return user;
    }
}

// 2. 发布事件
@Service
public class UserService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void createUser(User user) {
        userRepository.save(user);
        // 发布事件
        publisher.publishEvent(new UserCreatedEvent(this, user));
    }
}

// 3. 监听事件
@Component
public class UserEventListener implements ApplicationListener<UserCreatedEvent> {

    @Override
    public void onApplicationEvent(UserCreatedEvent event) {
        User user = event.getUser();
        System.out.println("用户创建事件: " + user.getName());
        // 发送欢迎邮件、初始化权限等
    }
}
```

### @EventListener注解

```java
@Component
public class UserEventHandler {

    // 使用注解监听
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("用户创建: " + event.getUser().getName());
    }

    // 多事件监听
    @EventListener
    public void handleUserUpdated(UserUpdatedEvent event) {
        System.out.println("用户更新: " + event.getUser().getName());
    }

    // 条件监听
    @EventListener(condition = "#event.user.age >= 18")
    public void handleAdultUser(UserCreatedEvent event) {
        System.out.println("成年用户创建: " + event.getUser().getName());
    }
}
```

---

## 异步事件处理

### @Async异步监听

```java
@SpringBootApplication
@EnableAsync  // 启用异步
public class Application {}

@Component
public class UserEventHandler {

    @Async
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        // 异步处理，不阻塞主线程
        sendWelcomeEmail(event.getUser());
    }
}
```

### 线程池配置

```java
@Configuration
public class AsyncConfig {

    @Bean("eventExecutor")
    public Executor eventExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
        executor.initialize();
        return executor;
    }
}

@Component
public class UserEventHandler {

    @Async("eventExecutor")
    @EventListener
    public void handleUserCreated(UserCreatedEvent event) {
        // 使用指定线程池
    }
}
```

---

## 事务绑定事件

### TransactionSynchronization

```java
@Service
public class UserService {

    @Autowired
    private ApplicationEventPublisher publisher;

    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
        publisher.publishEvent(new UserCreatedEvent(this, user));
        // 事件监听器将在事务提交后执行
    }
}

@Component
public class UserEventHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleUserCreated(UserCreatedEvent event) {
        // 只在事务提交后执行
        sendWelcomeEmail(event.getUser());
    }

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void beforeCommit(UserCreatedEvent event) {
        // 事务提交前执行
        // 可以在这里阻止提交
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void afterRollback(UserCreatedEvent event) {
        // 事务回滚后执行
    }
}
```

---

## 泛型事件

### GenericApplicationEvent

```java
// 泛型事件类
public class GenericEvent<T> extends ApplicationEvent {
    private final T payload;

    public GenericEvent(T payload) {
        super(payload);
        this.payload = payload;
    }

    public T getPayload() {
        return payload;
    }
}

// 使用
@Service
public class NotificationService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void notify(String message) {
        publisher.publishEvent(new GenericEvent<>(message));
    }
}

@Component
public class NotificationHandler {

    @EventListener
    public void handleGenericEvent(GenericEvent<String> event) {
        System.out.println("收到通知: " + event.getPayload());
    }
}
```

### PayloadApplicationEvent

```java
// 直接发布任意对象
publisher.publishEvent("Hello");  // 发布字符串
publisher.publishEvent(123);      // 发布数字
publisher.publishEvent(user);      // 发布用户对象

@EventListener
public void handlePayload(PayloadApplicationEvent<String> event) {
    System.out.println(event.getPayload());
}
```

---

## 事件监听器排序

### @Order注解

```java
@Component
public class UserEventHandler1 {
    @EventListener
    @Order(1)
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("Handler1 处理");
    }
}

@Component
public class UserEventHandler2 {
    @EventListener
    @Order(2)
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("Handler2 处理");
    }
}
```

### @Priority注解

```java
@Component
public class UserEventHandler1 {
    @EventListener
    @Priority(1)  // 数值越小优先级越高
    public void handleUserCreated(UserCreatedEvent event) {
        System.out.println("Handler1 处理");
    }
}
```

---

## 简单应用事件

### Spring Boot 2.3+ 简化

```java
// 不再需要自定义事件类
@Service
public class UserService {

    public void createUser(User user) {
        userRepository.save(user);
        // 直接发布任意对象
        publisher.publishEvent(user);
    }
}

@EventListener
public void handleUserCreated(User user) {
    // 直接监听User类型
    System.out.println("用户创建: " + user.getName());
}
```

---

## 事件发布注解

### @PublishedEvent

```java
// Spring Framework 5.2+ 新增
@Service
public class UserService {

    @PublishedEvent
    public UserCreatedEvent createUser(User user) {
        userRepository.save(user);
        return new UserCreatedEvent(this, user);
    }
}
```

---

## 源码解析

### SimpleApplicationEventMulticaster

```java
public class SimpleApplicationEventMulticaster implements ApplicationEventMulticaster {

    private Executor taskExecutor;  // 异步执行器
    private ErrorHandler errorHandler;  // 错误处理器

    @Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }

    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = eventType != null ? eventType : resolveDefaultEventType(event);

        for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            Executor executor = getTaskExecutor();
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            } else {
                invokeListener(listener, event);
            }
        }
    }

    private void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
        ErrorHandler errorHandler = getErrorHandler();
        try {
            doInvokeListener(listener, event);
        } catch (Throwable err) {
            if (errorHandler != null) {
                errorHandler.handleError(err);
            }
            throw err;
        }
    }
}
```

---

## 常见使用场景

### 1. 业务解耦

```java
// 订单创建后发送邮件、短信、更新库存
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        publisher.publishEvent(new OrderCreatedEvent(this, order));
    }
}

@Component
public class EmailListener {
    @EventListener
    public void sendEmail(OrderCreatedEvent event) {
        emailService.sendOrderEmail(event.getOrder());
    }
}

@Component
public class InventoryListener {
    @EventListener
    public void updateInventory(OrderCreatedEvent event) {
        inventoryService.updateStock(event.getOrder());
    }
}
```

### 2. 数据同步

```java
@Component
public class CacheInvalidationListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void invalidateCache(EntityUpdatedEvent event) {
        cacheManager.evict(event.getEntityType(), event.getEntityId());
    }
}
```

---

## 常见面试题

**Q1：Spring事件机制的实现原理？**

> 答：通过ApplicationEventMulticaster广播事件给所有注册的ApplicationListener。事件发布时遍历监听器列表，调用监听器的onApplicationEvent方法。

**Q2：@EventListener和ApplicationListener的区别？**

> 答：@EventListener是注解方式，更简洁；ApplicationListener是接口方式，功能更完整。两者效果相同。

**Q3：事务绑定事件有什么用？**

> 答：@TransactionalEventListener可以指定在事务提交前/后执行，确保事件处理与事务状态一致，避免事务回滚但事件已处理的问题。

**Q4：如何实现异步事件处理？**

> 答：@Async注解或配置Executor，事件监听器在异步线程执行。

**Q5：事件监听的顺序如何控制？**

> 答：@Order注解或@Priority注解控制监听器执行顺序。

---

## 总结

Spring事件机制是解耦组件的利器：
- **ApplicationEvent**：事件载体
- **ApplicationEventPublisher**：事件发布
- **@EventListener**：监听注解
- **@TransactionalEventListener**：事务绑定监听
- **@Async**：异步处理
