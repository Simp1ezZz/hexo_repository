---
title: Spring循环依赖：三级缓存解决方案详解
date: 2024-04-09 11:37:57
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring,循环依赖,三级缓存,singletonFactories,earlySingletonObjects
description: 深入理解Spring如何通过三级缓存解决循环依赖问题
cover:
---

## 前言

循环依赖是 Spring 中一个经典问题。当 A 依赖 B，B 又依赖 A 时，Spring 如何解决？本文深入解析 Spring 的三级缓存机制。

## 什么是循环依赖

### 代码示例

```java
@Service
public class A {
    @Autowired
    private B b;
}

@Service
public class B {
    @Autowired
    private A a;
}
```

### 三种循环依赖

| 类型 | 示例 | Spring能否解决 |
|------|------|---------------|
| 构造器循环依赖 | A构造器依赖B，B构造器依赖A | 否 |
| setter循环依赖（单例） | A setter依赖B，B setter依赖A | 能 |
| prototype循环依赖 | 都是prototype作用域 | 否 |

---

## Spring三级缓存

### 三级缓存结构

```java
public class DefaultSingletonBeanRegistry {

    // 一级缓存：完整的单例Bean实例
    // key: beanName, value: 完整的Bean实例（已创建完成）
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 二级缓存：早期Bean引用
    // key: beanName, value: 早期Bean引用（属性未填充）
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    // 三级缓存：Bean工厂
    // key: beanName, value: ObjectFactory（创建早期引用的工厂）
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
}
```

### 缓存作用

```
┌─────────────────────────────────────────────────────────────────┐
│                         三级缓存流程                               │
│                                                                   │
│  1. getSingleton(beanName) 先查一级缓存                          │
│     └→ 有 → 返回完整Bean                                        │
│     └→ 无 → 继续                                                │
│                                                                   │
│  2. 查询二级缓存（earlySingletonObjects）                        │
│     └→ 有 → 返回早期Bean                                        │
│     └→ 无 → 继续                                                │
│                                                                   │
│  3. 查询三级缓存（singletonFactories）                          │
│     └→ 有 → 创建早期Bean，放入二级缓存，删除三级缓存              │
│     └→ 无 → 返回null                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## 解决循环依赖流程

### 代码流程

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 1. 一级缓存：已完成的Bean
    Object singletonObject = singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 2. 二级缓存：早期Bean（未填充属性）
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null) {
                // 3. 三级缓存：Bean工厂
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    // 创建早期引用
                    singletonObject = singletonFactory.getObject();
                    // 放入二级缓存
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    // 移除三级缓存
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

### 循环依赖解决示例

```
场景：A 依赖 B，B 依赖 A

时间线：
─────────────────────────────────────────────────────────────────────────

T1: 创建 A
    A 调用 getSingleton("A")
    ├─ 一级缓存: 无
    ├─ 二级缓存: 无
    └─ 三级缓存: 无

    创建 A 的早期引用，放入三级缓存
    singletonFactories.put("A", () → createA早期引用)

    开始填充 A 的属性
    发现依赖 B，调用 getSingleton("B")

T2: 创建 B
    B 调用 getSingleton("B")
    ├─ 一级缓存: 无
    ├─ 二级缓存: 无
    └─ 三级缓存: 无

    创建 B 的早期引用，放入三级缓存
    singletonFactories.put("B", () → createB早期引用)

    开始填充 B 的属性
    发现依赖 A，调用 getSingleton("A")

T3: 再次获取 A
    A 调用 getSingleton("A")
    ├─ 一级缓存: 无（A 还未创建完成）
    ├─ 二级缓存: 无
    └─ 三级缓存: 有！

    从三级缓存获取 A 的早期引用
    放入二级缓存，删除三级缓存
    返回 A 的早期引用给 B

T4: B 创建完成
    B 完成属性填充和初始化
    B 放入一级缓存，删除三级缓存
    返回 B 给 A

T5: A 创建完成
    A 完成属性填充和初始化
    A 放入一级缓存，删除三级缓存
```

---

## 三级缓存详解

### 为什么要三级缓存？

**只用一级缓存的问题：**
```java
// 一级缓存只有完整Bean
// 如果创建过程中返回"不完整"的Bean给其他Bean
// 其他Bean可能基于不完整的状态做操作
```

**只用二级缓存的问题：**
```java
// 二级缓存可以存储早期引用
// 但无法保证同一个Bean只创建一个早期引用
// 每次获取都会创建新的早期引用
```

**三级缓存的作用：**
```java
// 同一个Bean只创建一个ObjectFactory
// 多次获取只调用一次ObjectFactory.getObject()
// 保证早期引用只创建一次

// 同时支持AOP：若需要代理，在ObjectFactory中创建代理对象
```

### AOP与三级缓存

```java
// 如果A需要被代理
ObjectFactory<?> singletonFactory = () -> {
    // 这里是AOP代理的入口
    return createEarlySingletonReference(beanName);
};

// 三级缓存中存储的是工厂
// 实际创建时会判断是否需要创建代理
```

---

## 为什么构造器循环依赖无法解决

### 原因分析

```java
@Service
public class A {
    public A(B b) {  // 构造器注入
        this.b = b;
    }
}

@Service
public class B {
    public B(A a) {  // 构造器注入
        this.a = a;
    }
}
```

**流程：**
```
T1: new A() 需要 B
    └→ 但 B 还未创建，无法完成构造

T2: new B() 需要 A
    └→ 但 A 还未创建，无法完成构造

结论：死锁，无法解决
```

### Spring解决不了的情况

| 场景 | 原因 | 解决方案 |
|------|------|---------|
| 构造器循环依赖 | 对象无法构造 | 改用setter注入 |
| prototype循环依赖 | 每次创建新实例 | 避免循环依赖 |
| 多实例Bean循环依赖 | 非单例模式不支持 | 避免或重构代码 |

---

## setter循环依赖解决过程

### doCreateBean源码

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd,
                             @Nullable Object[] args) {

    // 1. 实例化
    BeanWrapper instanceWrapper = createBeanInstance(beanName, mbd, args);
    Object bean = instanceWrapper.getWrappedInstance();

    // 2. 加入三级缓存（解决循环依赖的关键）
    boolean earlySingletonExposure = mbd.isSingleton()
                    && this.allowCircularReferences
                    && isSingletonCurrentlyInCreation(beanName);

    if (earlySingletonExposure) {
        // 放入三级缓存
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    // 3. 属性填充（依赖注入）
    populateBean(beanName, mbd, instanceWrapper);

    // 4. 初始化
    initializeBean(beanName, bean, mbd);

    return bean;
}
```

### addSingletonFactory

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        if (!this.singletonObjects.containsKey(beanName)) {
            // 放入三级缓存
            this.singletonFactories.put(beanName, singletonFactory);
            // 从二级缓存移除（如果有）
            this.earlySingletonObjects.remove(beanName);
        }
    }
}
```

---

## @Async循环依赖问题

### 问题场景

```java
@Service
public class A {
    @Autowired
    private B b;

    @Async
    public void asyncMethod() {}
}

@Service
public class B {
    @Autowired
    private A a;
}
```

### 原因

@Async 会生成代理对象，循环依赖时无法正确创建代理。

### 解决方案

```java
// 方案1：不用@Async，改用手动线程池
@Autowired
private AsyncExecutor executor;

public void asyncMethod() {
    CompletableFuture.runAsync(() -> {
        // 业务逻辑
    }, executor);
}

// 方案2：重构代码，避免循环依赖
// 方案3：@Lazy延迟注入
@Autowired
@Lazy
private A a;
```

---

## 常见面试题

**Q1：Spring三级缓存分别是什么？**

> 答：一级缓存singletonObjects存储完整Bean；二级缓存earlySingletonObjects存储早期Bean（未填充属性）；三级缓存singletonFactories存储ObjectFactory，用于创建早期引用。

**Q2：为什么需要三级缓存而不是两级？**

> 答：三级缓存确保同一Bean的早期引用只创建一次，同时支持AOP代理创建。只用一级/二级缓存无法在创建过程中返回"不完整"的Bean给其他Bean使用。

**Q3：构造器循环依赖为什么无法解决？**

> 答：构造器注入需要在构造时完成依赖注入，但被依赖的Bean还未创建，无法完成构造，形成死锁。

**Q4：@Async循环依赖为什么有问题？**

> 答：@Async生成代理对象，循环依赖时代理创建时机冲突，导致无法正确生成代理。

**Q5：如何检测循环依赖？**

> 答：可以通过BeanCurrentlyInCreationException异常检测。Spring在发现循环依赖时会抛出此异常。

---

## 总结

Spring通过三级缓存解决setter循环依赖：
- **一级缓存**：完整Bean实例
- **二级缓存**：早期引用（未填充属性）
- **三级缓存**：ObjectFactory（创建早期引用）
- **解决过程**：A创建时→依赖B→B创建时→依赖A→从三级缓存获取A的早期引用→B完成→A完成
- **限制**：构造器循环依赖和prototype循环依赖无法解决
