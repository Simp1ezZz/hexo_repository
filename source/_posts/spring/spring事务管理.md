---
title: Spring事务管理：传播行为与事务失效场景分析
date: 2026-04-17 10:00:00
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring,事务,@Transactional,传播行为,事务隔离级别
description: 深入理解Spring事务管理机制，7种传播行为与常见失效场景
cover:
---

## 前言

事务是保证数据一致性的关键机制。Spring 提供了强大而灵活的事务管理功能，但使用不当也会导致事务失效。本文深入解析 Spring 事务的原理与常见问题。

## 事务的 ACID 特性

| 特性 | 说明 |
|------|------|
| Atomic（原子性） | 事务是最小执行单元，不可分割 |
| Consistency（一致性） | 事务执行前后，数据保持一致 |
| Isolation（隔离性） | 并发事务之间相互隔离 |
| Durability（持久性） | 事务提交后，结果永久保存 |

---

## Spring 事务管理接口

### 核心接口

```
┌─────────────────────────────────────────────────────────────┐
│                   PlatformTransactionManager                  │
│              （事务管理器接口）                               │
│   ┌───────────────────────────────────────────────────┐  │
│   │ Transaction getTransaction(TransactionDefinition)   │  │
│   │ void commit(TransactionStatus)                    │  │
│   │ void rollback(TransactionStatus)                  │  │
│   └───────────────────────────────────────────────────┘  │
│              △                                            │
│              │实现                                        │
│  ┌───────────┴───────────┬─────────────┬─────────────┐  │
│  │DataSourceTransaction  │JpaTransaction │Hibernate   │  │
│  │Manager                │Manager       │Transaction  │  │
│  │(JDBC)                │(JPA)         │Manager     │  │
│  └───────────────────────┴─────────────┴─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 常用实现

```java
// JDBC事务管理器
@Bean
public PlatformTransactionManager dataSourceTransactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}

// JPA事务管理器
@Bean
public PlatformTransactionManager jpaTransactionManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
}

// 编程式事务
TransactionTemplate template = new TransactionTemplate(platformTransactionManager);
template.execute(status -> {
    // 业务逻辑
    return null;
});
```

---

## @Transactional 注解

### 基本使用

```java
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    @Transactional  // 默认配置
    public void addUser(User user) {
        userRepository.save(user);
        // 异常时自动回滚
    }
}
```

### 配置参数

```java
@Transactional(
    propagation = Propagation.REQUIRED,  // 传播行为
    isolation = Isolation.DEFAULT,          // 隔离级别
    timeout = 30,                         // 超时时间（秒）
    readOnly = false,                     // 是否只读
    rollbackFor = Exception.class,       // 回滚异常类型
    noRollbackFor = RuntimeException.class // 不回滚异常类型
)
```

---

## 七种传播行为

### 传播行为图解

```
┌─────────────────────────────────────────────────────────────┐
│                    传播行为 (Propagation)                      │
│                                                             │
│  场景：ServiceA.methodA() 调用 ServiceB.methodB()          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                   ServiceA                            │  │
│  │                                                      │  │
│  │  @Transactional                                     │  │
│  │  methodA() {                                       │  │
│  │      // 已开启事务T1                               │  │
│  │      serviceB.methodB();  ─────────────────────────┼──┼──→ │
│  │  }                                                  │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 传播行为详解

| 传播行为 | 说明 | 场景A有事务 | 场景A无事务 |
|---------|------|------------|------------|
| REQUIRED | 必须有事务（默认） | 加入事务A | 创建新事务 |
| REQUIRES_NEW | 必须有新事务 | 挂起A，创建新事务B | 创建新事务 |
| SUPPORTS | 支持事务 | 加入事务A | 无事务执行 |
| NOT_SUPPORTED | 不支持事务 | 挂起A，无事务执行 | 无事务执行 |
| MANDATORY | 强制有事务 | 加入事务A | 抛异常 |
| NEVER | 禁止事务 | 抛异常 | 无事务执行 |
| NESTED | 嵌套事务 | 创建嵌套点 | 创建新事务 |

### 代码示例

```java
@Service
public class AccountService {

    @Transactional
    public void transfer(Account from, Account to, BigDecimal amount) {
        // REQUIRED: 同一事务
        accountRepository.withdraw(from.getId(), amount);
        accountRepository.deposit(to.getId(), amount);
    }
}

@Service
public class LogService {

    // REQUIRES_NEW: 独立事务，不受外层事务影响
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String operation) {
        logRepository.save(new Log(operation));
    }
}

@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);
        // REQUIRES_NEW: 日志独立提交，即使订单失败日志也保存
        logService.log("创建订单: " + order.getId());
    }
}
```

### NESTED vs REQUIRES_NEW

```java
// NESTED: 嵌套事务，使用savepoint
// 外层回滚，嵌套事务也会回滚
// 内层回滚，不影响外层

// REQUIRES_NEW: 完完全全的两个事务
// 内层回滚，不影响外层
// 外层回滚，也不影响内层（内层已提交）
```

---

## 事务隔离级别

### 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ_UNCOMMITTED | 可能 | 可能 | 可能 |
| READ_COMMITTED | 不可能 | 可能 | 可能 |
| REPEATABLE_READ | 不可能 | 不可能 | 可能 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 |

### Spring 配置

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void updateAccount() {
    // 使用REPEATABLE_READ隔离级别
}
```

### 常见问题

```sql
-- 脏读：A事务读取B事务未提交的数据
-- T1: 读取 balance = 1000
-- T2: UPDATE balance = 2000 (未提交)
-- T1: 读取 balance = 2000 (脏读)
-- T2: ROLLBACK
-- T1: 基于脏数据操作

-- 不可重复读：同一事务两次读取结果不同
-- T1: 读取 balance = 1000
-- T2: UPDATE balance = 2000; COMMIT;
-- T1: 读取 balance = 2000 (不可重复读)

-- 幻读：同一事务两次查询结果集不同
-- T1: SELECT * FROM users (3 rows)
-- T2: INSERT INTO users ...; COMMIT;
-- T1: SELECT * FROM users (4 rows) (幻读)
```

---

## 事务失效场景

### 1. 非 public 方法

```java
@Service
public class UserService {

    @Transactional  // 失效！private方法不会被代理
    private void doSomething(User user) {
        userRepository.save(user);
    }
}
```

> 原因：Spring AOP 代理只能增强 public 方法。

### 2. 异常被 catch 吞掉

```java
@Service
public class UserService {

    @Transactional  // 失效！
    public void createUser(User user) {
        try {
            userRepository.save(user);
        } catch (Exception e) {
            // 异常被吞掉，不会触发回滚
        }
    }
}
```

> 解决：重新抛出异常，或配置 `rollbackFor = Exception.class`

### 3. 同类内部方法调用

```java
@Service
public class UserService {

    @Transactional
    public void methodA() {
        this.methodB();  // 调用同类方法，不走代理
    }

    @Transactional
    public void methodB() {  // 不会开启新事务
        userRepository.save(user);
    }
}
```

> 解决：注入自身代理对象调用

```java
@Service
public class UserService {

    @Autowired
    private UserService self;  // 注入自身代理

    public void methodA() {
        self.methodB();  // 通过代理调用
    }

    @Transactional
    public void methodB() {
        userRepository.save(user);
    }
}
```

### 4. 异常类型不匹配

```java
@Service
public class UserService {

    @Transactional  // 默认只回滚RuntimeException
    public void createUser(User user) {
        try {
            userRepository.save(user);
        } catch (IOException e) {
            throw new RuntimeException(e);  // 包装为RuntimeException
        }
    }

    @Transactional(rollbackFor = Exception.class)  // 指定所有异常回滚
    public void createUser2(User user) throws IOException {
        userRepository.save(user);
        throw new IOException("文件错误");  // 现在会回滚
    }
}
```

### 5. 多数据源配置问题

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary  // 必须指定主数据源
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}

@Service
public class UserService {

    @Transactional  // 使用@Primary的数据源
    public void operation() {
        // 操作
    }
}
```

### 6. 事务超时配置

```java
@Transactional(timeout = 5)  // 5秒超时
public void longOperation() {
    // 如果超过5秒，会自动回滚
}
```

---

## 编程式事务

### TransactionTemplate

```java
@Service
public class UserService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    public void createUsers(List<User> users) {
        transactionTemplate.executeWithoutResult(status -> {
            for (User user : users) {
                userRepository.save(user);
            }
        });
    }

    public void transfer(Account from, Account to, BigDecimal amount) {
        transactionTemplate.execute(status -> {
            try {
                accountRepository.withdraw(from.getId(), amount);
                accountDefinition.deposit(to.getId(), amount);
            } catch (Exception e) {
                status.setRollbackOnly();  // 手动回滚
                throw e;
            }
            return null;
        });
    }
}
```

### 手动获取事务

```java
@Service
public class UserService {

    @Autowired
    private PlatformTransactionManager transactionManager;

    public void transfer(Account from, Account to, BigDecimal amount) {
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        TransactionStatus status = transactionManager.getTransaction(def);

        try {
            accountRepository.withdraw(from.getId(), amount);
            accountRepository.deposit(to.getId(), amount);
            transactionManager.commit(status);
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw e;
        }
    }
}
```

---

## Spring 事务原理

### 事务切面

```java
public class TransactionInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        TransactionAttribute attr = transactionManager.getTransaction(attr);

        try {
            Object result = invocation.proceed();  // 执行目标方法
            transactionManager.commit(status);      // 提交
            return result;
        } catch (Throwable ex) {
            // 回滚判断
            if (attr.rollbackOn(ex)) {
                transactionManager.rollback(status);
            } else {
                transactionManager.commit(status);
            }
            throw ex;
        }
    }
}
```

### 代理创建

```java
@EnableTransactionManagement  // 启用事务管理
public class TransactionManagementConfigurer
        implements TransactionManagementConfigurer {

    @Override
    public PlatformTransactionManager annotationDrivenTransactionManager() {
        return new DataSourceTransactionManager(dataSource);
    }
}
```

---

## 常见面试题

**Q1：Spring事务的传播行为有哪些？**

> 答：REQUIRED（必须有事务，默认）、REQUIRES_NEW（必须新事务）、SUPPORTS（支持事务）、NOT_SUPPORTED（不支持事务）、MANDATORY（强制有事务）、NEVER（禁止事务）、NESTED（嵌套事务）。

**Q2：@Transactional 失效的场景？**

> 答：非public方法、异常被catch、内部方法调用this.xxx()、异常类型不匹配（默认只回滚RuntimeException）、多数据源未指定@Primary。

**Q3：REQUIRES_NEW 和 NESTED 的区别？**

> 答：REQUIRES_NEW创建完全独立的新事务，外层事务挂起；NESTED在当前事务中创建嵌套点，使用savepoint，外层回滚嵌套事务也回滚，但嵌套事务回滚不影响外层。

**Q4：Spring事务和数据库事务的关系？**

> 答：Spring事务是数据库事务的抽象，底层依赖数据库的ACID特性。Spring通过Connection进行事务管理，commit/rollback对应数据库的提交/回滚。

**Q5：如何实现多数据源事务？**

> 答：使用JTA（Java Transaction API）或ChainedTransactionManager。可以配置多个DataSourceTransactionManager，按顺序提交/回滚。

---

## 总结

Spring事务管理需要深入理解：
- **传播行为**：7种，决定事务如何嵌套
- **隔离级别**：解决并发问题，但影响性能
- **失效场景**：非public方法、内部调用、异常吞掉
- **编程式事务**：TransactionTemplate或手动管理
- **原理**：AOP代理 + TransactionInterceptor
