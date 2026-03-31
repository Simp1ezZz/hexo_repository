---
title: Spring IoC容器：Bean的生命周期与依赖注入原理
date: 2026-04-15 10:00:00
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring,IoC,Bean生命周期,依赖注入,BeanFactory
description: 深入理解Spring IoC容器的核心原理，Bean的创建过程与依赖注入机制
cover:
---

## 前言

Spring 最核心的概念是 IoC（控制反转）和 DI（依赖注入）。理解它们的工作原理，是掌握 Spring 框架的基础。本文深入解析 Spring IoC 容器的实现机制。

## IoC 与 DI 概念

### 控制反转（IoC）

传统方式：程序主动创建依赖对象
```java
public class OrderService {
    private UserService userService = new UserService();  // 主动创建
    private OrderDAO orderDAO = new OrderDAO();
}
```

IoC 方式：控制权反转，由容器创建和管理
```java
public class OrderService {
    private UserService userService;  // 被动接收

    public void setUserService(UserService userService) {
        this.userService = userService;  // 容器注入
    }
}
```

### 依赖注入（DI）

容器自动注入依赖对象，有三种方式：

| 方式 | 说明 |
|------|------|
| 构造器注入 | 通过构造函数注入 |
| Setter注入 | 通过setter方法注入 |
| 字段注入 | 直接注入字段（不推荐） |

---

## BeanFactory 与 ApplicationContext

### 体系结构

```
┌─────────────────────────────────────────────────────────┐
│                  BeanFactory                             │
│   - 基本的IoC容器接口                                   │
│   - 只支持Bean的getBean等基本功能                       │
└─────────────────────────────────────────────────────────┘
                        △
                        │extends
┌─────────────────────────────────────────────────────────┐
│              ApplicationContext                          │
│   - BeanFactory的扩展                                   │
│   - 支持国际化、事件机制、资源加载等                     │
│   - 常用的实现：                                        │
│     - ClassPathXmlApplicationContext                   │
│     - FileSystemXmlApplicationContext                   │
│     - AnnotationConfigApplicationContext                 │
└─────────────────────────────────────────────────────────┘
                        △
                        │extends
┌─────────────────────────────────────────────────────────┐
│              WebApplicationContext                      │
│   - Web应用上下文                                       │
│   - 配合Spring MVC使用                                  │
└─────────────────────────────────────────────────────────┘
```

### BeanFactory 实现

```java
public interface BeanFactory {
    // 基本方法
    Object getBean(String name);
    <T> T getBean(String name, Class<T> requiredType);
    Object getBean(String name, Object... args);

    // 其他方法
    boolean containsBean(String name);
    boolean isSingleton(String name);
    Class<?> getType(String name);
}
```

### ApplicationContext 实现

```java
public interface ApplicationContext extends BeanFactory, EnvironmentCapable,
        ListableBeanFactory, HierarchicalBeanFactory, MessageSource,
        ApplicationEventPublisher, ResourcePatternResolver {

    String getId();
    String getApplicationName();
    String getDisplayName();
    long getStartupDate();
    ApplicationParent getParent();
}
```

---

## Bean 的定义与注册

### XML 配置方式

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userService" class="com.example.UserService">
        <property name="userDAO" ref="userDAO"/>
        <property name="name" value="TestService"/>
    </bean>

    <bean id="userDAO" class="com.example.UserDAO"/>

</beans>
```

### 注解方式

```java
// @Component 标注为Bean
@Component
public class UserService {

    @Autowired  // 自动注入
    private UserDAO userDAO;

    @Value("${app.name}")  // 注入配置
    private String appName;
}

// @Configuration + @Bean 配置类
@Configuration
public class AppConfig {

    @Bean
    public UserService userService() {
        return new UserService();
    }
}

// 组件扫描
@ComponentScan(basePackages = "com.example")
```

### Bean 作用域

| 作用域 | 说明 |
|--------|------|
| singleton | 单例，默认作用域 |
| prototype | 每次获取创建新实例 |
| request | 每次HTTP请求创建新实例 |
| session | 同一HTTP Session共享 |
| application | ServletContext级别 |
| websocket | WebSocket级别 |

---

## Bean 生命周期

### 完整生命周期流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Bean 生命周期                            │
│                                                             │
│  1. 实例化（Instantiation）                                 │
│     └→ Bean构造器执行，创建实例                             │
│                                                             │
│  2. 属性填充（Populate）                                   │
│     └→ Spring注入依赖（@Autowired, @Value等）              │
│                                                             │
│  3. 初始化（Initialization）                                │
│     ├→ BeanNameAware.setBeanName()                         │
│     ├→ BeanFactoryAware.setBeanFactory()                   │
│     ├→ ApplicationContextAware.setApplicationContext()       │
│     ├→ BeanPostProcessor.postProcessBeforeInitialization()  │
│     ├→ @PostConstruct 标注的方法                           │
│     ├→ InitializingBean.afterPropertiesSet()                │
│     ├→ 自定义init-method                                   │
│     └→ BeanPostProcessor.postProcessAfterInitialization()   │
│                                                             │
│  4. 销毁（Destruction）                                     │
│     ├→ @PreDestroy 标注的方法                              │
│     ├→ DisposableBean.destroy()                            │
│     └→ 自定义destroy-method                                │
└─────────────────────────────────────────────────────────────┘
```

### 代码示例

```java
@Component
public class UserService implements InitializingBean, DisposableBean {

    private UserDAO userDAO;

    @PostConstruct
    public void init() {
        System.out.println("@PostConstruct 执行");
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("InitializingBean.afterPropertiesSet() 执行");
    }

    public void customInit() {
        System.out.println("custom init-method 执行");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("@PreDestroy 执行");
    }

    @Override
    public void destroy() {
        System.out.println("DisposableBean.destroy() 执行");
    }

    public void customDestroy() {
        System.out.println("custom destroy-method 执行");
    }
}
```

### BeanPostProcessor

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("Before: " + beanName);
        return bean;  // 可以返回代理对象
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        System.out.println("After: " + beanName);
        return bean;
    }
}
```

---

## 依赖注入原理

### @Autowired 注入原理

```java
// AutowiredAnnotationBeanPostProcessor 实现
public class AutowiredAnnotationBeanPostProcessor
        implements BeanPostProcessor {

    // 在bean属性填充阶段处理 @Autowired
    @Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) {
        // 1. 找到所有标注 @Autowired 的字段/方法
        // 2. 通过 getBean() 获取依赖的bean
        // 3. 通过反射设置字段或调用方法
        injectionMetadata.inject(bean, beanName, pvs);
        return pvs;
    }
}
```

### 注入方式对比

```java
@Component
public class UserService {

    // 字段注入（不推荐）
    @Autowired
    private UserDAO userDAO;

    // Setter注入
    private UserDAO userDAO;

    @Autowired
    public void setUserDAO(UserDAO userDAO) {
        this.userDAO = userDAO;
    }

    // 构造器注入（推荐）
    private UserDAO userDAO;

    @Autowired
    public UserService(UserDAO userDAO) {
        this.userDAO = userDAO;
    }
}
```

### 循环依赖问题

```java
// 循环依赖示例
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

**Spring 三级缓存解决循环依赖：**
```
一级缓存（singletonObjects）：完整Bean实例
二级缓存（earlySingletonObjects）：早期Bean引用（未完成属性填充）
三级缓存（singletonFactories）：Bean工厂（用于创建早期引用）
```

---

## 启动流程

### BeanFactory 启动

```java
// 简化的启动流程
public void refresh() {
    // 1. 创建BeanFactory
    DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

    // 2. 加载BeanDefinition
    XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(beanFactory);
    reader.loadBeanDefinitions("applicationContext.xml");

    // 3. 预实例化单例Bean
    beanFactory.preInstantiateSingletons();
}
```

### 懒加载 vs 非懒加载

```java
// 默认非懒加载，容器启动时创建
@Component
public class UserService { }

// 懒加载，首次使用时创建
@Lazy
@Component
public class OrderService { }
```

---

## 面试高频问题

**Q1：Spring IoC 的工作流程？**

> 答：1. 创建BeanFactory；2. 加载BeanDefinition；3. 调用BeanFactoryPostProcessor；4. 实例化Bean；5. 属性填充（依赖注入）；6. 初始化；7. 注册销毁回调。Bean使用后由GC或容器关闭时销毁。

**Q2：BeanFactory 和 ApplicationContext 的区别？**

> 答：BeanFactory是基础容器，只提供getBean等基本功能。ApplicationContext是扩展容器，提供国际化、事件机制、资源加载等高级功能，且会在启动时预实例化所有单例Bean。

**Q3：Bean 的生命周期？**

> 答：实例化→属性填充→初始化（Aware回调→BeanPostProcessor前置→@PostConstruct/InitializingBean→BeanPostProcessor后置）→使用→销毁（@PreDestroy/DisposableBean→destroy-method）。

**Q4：@Autowired 和 @Resource 的区别？**

> 答：@Autowired是Spring专用注解，默认byType，可配合@QualifierbyName；@Resource是JSR-250标准注解，默认byName。构造器注入推荐使用@Autowired。

**Q5：Spring 如何解决循环依赖？**

> 答：通过三级缓存：singletonObjects（一级）、earlySingletonObjects（二级）、singletonFactories（三级）。A创建时发现依赖B，将早期引用放入三级缓存，B创建时发现依赖A，从缓存获取A的早期引用完成创建。

---

## 总结

Spring IoC 是 Spring 框架的基石：
- **IoC/DI**：控制权反转，容器管理依赖
- **BeanFactory**：基础容器接口
- **ApplicationContext**：扩展容器，支持更多功能
- **生命周期**：实例化→属性填充→初始化→销毁
- **注入方式**：构造器注入（推荐）、Setter注入、字段注入
