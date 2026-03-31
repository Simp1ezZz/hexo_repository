---
title: Spring AOP：动态代理与切面编程深度解析
date: 2024-03-12 14:20:17
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring,AOP,动态代理,AspectJ,切面编程
description: 深入理解Spring AOP的底层实现，JDK动态代理与CGLIB的区别
cover:
---

## 前言

AOP（Aspect-Oriented Programming，面向切面编程）是 Spring 的另一核心功能。它可以将分散在各个模块中的重复逻辑（如日志、事务、安全）统一处理，减少代码耦合。本文深入解析 Spring AOP 的实现原理。

## AOP 核心概念

### 横切关注点

```
┌─────────────────────────────────────────────────────────────┐
│                   业务逻辑                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │业务方法1 │  │业务方法2 │  │业务方法3 │  │业务方法4 │  │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  │
│       │             │             │             │          │
│  ┌────┴─────────────┴─────────────┴─────────────┴────┐  │
│  │                    横切关注点                         │  │
│  │         日志、安全、事务、性能监控、缓存              │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### AOP 术语

| 术语 | 说明 | 类比 |
|------|------|------|
| Join Point | 连接点，程序执行的某个位置 | 方法调用、异常抛出 |
| Pointcut | 切点，匹配连接点的表达式 | 筛选"哪些"需要增强 |
| Advice | 通知/增强，拦截到的逻辑 | "做什么" |
| Aspect | 切面，切点+通知 | 模块化的横切关注点 |
| Weaving | 织入，将切面应用到目标对象 | 将通知织入连接点 |
| Target | 目标对象，被代理的对象 | 业务类 |
| Proxy | 代理对象，AOP创建的代理对象 | 增强后的对象 |

---

## AOP 实现方式

### 1. JDK 动态代理

```java
public class JdkProxy implements InvocationHandler {

    private Object target;  // 目标对象

    public JdkProxy(Object target) {
        this.target = target;
    }

    // 创建代理对象
    public Object createProxy() {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            this  // InvocationHandler
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args)
            throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = method.invoke(target, args);  // 调用目标方法
        System.out.println("After: " + method.getName());
        return result;
    }
}

// 使用
UserService userService = new UserServiceImpl();
UserService proxy = (UserService) new JdkProxy(userService).createProxy();
proxy.addUser(user);  // 通过代理调用
```

### 2. CGLIB 动态代理

```java
public class CglibProxy implements MethodInterceptor {

    private Object target;  // 目标对象

    public CglibProxy(Object target) {
        this.target = target;
    }

    public Object createProxy() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());  // 设置父类
        enhancer.setCallback(this);  // 设置回调
        return enhancer.create();  // 创建代理对象
    }

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                            MethodProxy proxy) throws Throwable {
        System.out.println("Before: " + method.getName());
        Object result = proxy.invokeSuper(obj, args);  // 调用父类方法
        System.out.println("After: " + method.getName());
        return result;
    }
}

// 使用
UserService userService = new UserService();
UserService proxy = (UserService) new CglibProxy(userService).createProxy();
proxy.addUser(user);
```

### 两种代理对比

| 特性 | JDK动态代理 | CGLIB代理 |
|------|-------------|-----------|
| 原理 | 实现接口，Proxy类生成代理 | 继承父类，ASM生成字节码 |
| 目标类要求 | 必须实现接口 | 无要求，可代理任何类 |
| 性能 | 反射调用，略慢 | 直接调用字节码生成，速度快 |
| 代理对象类型 | 实现相同接口的类 | 继承目标类的子类 |

---

## Spring AOP 注解

### @Aspect 切面类

```java
@Component
@Aspect  // 标注为切面类
@Slf4j
public class LoggingAspect {

    // 定义切点：匹配 com.example.service 包下所有方法
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void servicePointcut() {}

    // 定义切点：匹配标注 @Transactional 的方法
    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalPointcut() {}

    // 前置通知
    @Before("servicePointcut()")
    public void before(JoinPoint joinPoint) {
        log.info("方法执行前: {}", joinPoint.getSignature());
    }

    // 后置通知
    @AfterReturning(pointcut = "servicePointcut()", returning = "result")
    public void afterReturning(JoinPoint joinPoint, Object result) {
        log.info("方法执行后，返回值: {}", result);
    }

    // 异常通知
    @AfterThrowing(pointcut = "servicePointcut()", throwing = "e")
    public void afterThrowing(JoinPoint joinPoint, Exception e) {
        log.error("方法异常: {}", e.getMessage());
    }

    // 最终通知
    @After("servicePointcut()")
    public void after(JoinPoint joinPoint) {
        log.info("方法执行结束");
    }

    // 环绕通知
    @Around("servicePointcut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();  // 执行目标方法
            long duration = System.currentTimeMillis() - start;
            log.info("执行耗时: {}ms", duration);
            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - start;
            log.error("执行异常，耗时: {}ms", duration);
            throw e;
        }
    }
}
```

### 切点表达式

```java
// 方法切点
execution(public * com.example.UserService.add(..))  // add方法
execution(* com.example.service.*.get*(..))  // get开头方法
execution(* com.example.service..*.delete(..))  // ..表示子包

// 类切点
within(com.example.service.UserService)  // UserService类
within(com.example.service..*)  // service包下所有类

// 注解切点
@annotation(org.springframework.transaction.annotation.Transactional)  // 标注某注解的方法
@within(org.springframework.stereotype.Service)  // 标注某注解的类

// 参数切点
args(com.example.User)  // 参数为User的方法

// 组合切点
@Pointcut("execution(* com.example.service.*.*(..)) && !execution(* com.example.service.*.get*(..))")
```

---

## Spring AOP 织入流程

```
┌─────────────────────────────────────────────────────────────┐
│                    AOP 织入流程                              │
│                                                             │
│  1. 启动配置                                                │
│     └→ @EnableAspectJAutoProxy 启用自动代理                 │
│                                                             │
│  2. 解析切面                                                │
│     └→ 扫描 @Aspect 注解的类                               │
│                                                             │
│  3. 创建代理                                                │
│     ├→ 有接口 → JDK动态代理                                │
│     └→ 无接口 → CGLIB代理                                  │
│                                                             │
│  4. 方法调用                                                │
│     └→ 代理对象 → 前置通知 → 目标方法 → 后置通知           │
└─────────────────────────────────────────────────────────────┘
```

### 代理创建时机

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)
// proxyTargetClass = true: 强制使用CGLIB
// proxyTargetClass = false: 优先JDK代理，失败用CGLIB

@Component
public class AnnotationAwareAspectJAutoProxyCreator
        extends AspectJAutoProxyCreator {

    // 在 Bean 初始化后创建代理
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 如果是需要被代理的Bean，创建代理
        if (shouldProxy(bean, beanName)) {
            return createProxy(bean);
        }
        return bean;
    }
}
```

---

## AOP 应用场景

### 1. 事务管理

```java
@Aspect
@Component
public class TransactionAspect {

    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        TransactionStatus status = null;
        try {
            // 开启事务
            DefaultTransactionDefinition def = new DefaultTransactionDefinition();
            status = transactionManager.getTransaction(def);

            Object result = joinPoint.proceed();

            // 提交事务
            transactionManager.commit(status);
            return result;
        } catch (Exception e) {
            // 回滚事务
            if (status != null) {
                transactionManager.rollback(status);
            }
            throw e;
        }
    }
}
```

### 2. 日志记录

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("execution(* com.example.controller..*.*(..))")
    public Object logController(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request = getRequest();
        log.info("请求: {} {}", request.getMethod(), request.getRequestURI());
        long start = System.currentTimeMillis();
        try {
            Object result = joinPoint.proceed();
            long duration = System.currentTimeMillis() - start;
            log.info("响应: {} ({}ms)", response.getStatus(), duration);
            return result;
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - start;
            log.error("异常: {} ({}ms)", e.getMessage(), duration);
            throw e;
        }
    }
}
```

### 3. 性能监控

```java
@Aspect
@Component
public class PerformanceAspect {

    @Around("execution(* com.example.service..*.*(..))")
    public Object monitor(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().toShortString();
        long start = System.nanoTime();
        try {
            return joinPoint.proceed();
        } finally {
            long duration = System.nanoTime() - start;
            if (duration > 1_000_000) {  // > 1ms
                log.warn("慢方法: {} 耗时 {}ms", method, duration / 1_000_000);
            }
        }
    }
}
```

### 4. 权限校验

```java
@Aspect
@Component
public class SecurityAspect {

    @Before("execution(* com.example.admin..*.*(..))")
    public void checkAdmin(JoinPoint joinPoint) {
        User currentUser = SecurityContext.getCurrentUser();
        if (currentUser == null || !currentUser.isAdmin()) {
            throw new UnauthorizedException("需要管理员权限");
        }
    }
}
```

---

## JDK vs CGLIB 代理选择

### 强制使用

```java
@EnableAspectJAutoProxy(proxyTargetClass = true)  // 强制CGLIB
@EnableAspectJAutoProxy(proxyTargetClass = false)  // 优先JDK代理
```

### Spring Boot 默认行为

```java
// Spring Boot 2.0+ 默认使用 CGLIB
// 原因：即使目标类实现了接口，也使用CGLIB生成子类代理
// 这样可以代理没有接口的类，且行为更一致
```

### 代理选择规则

```
┌─────────────────────────────────────────────────────────┐
│                   代理选择流程                            │
│                                                         │
│  1. 检查 proxyTargetClass 配置                         │
│     ├→ true → 使用 CGLIB                              │
│     └→ false → 继续判断                               │
│                                                         │
│  2. 目标类是否有接口                                   │
│     ├→ 有接口 → JDK 动态代理                          │
│     └→ 无接口 → CGLIB 代理                           │
└─────────────────────────────────────────────────────────┘
```

---

## AspectJ vs Spring AOP

| 特性 | Spring AOP | AspectJ |
|------|------------|---------|
| 织入时机 | 运行时织入 | 编译时/加载时织入 |
| 代理方式 | 动态代理 | 字节码织入 |
| 切点能力 | 方法级别 | 字段、构造函数 |
| 性能 | 略低（运行时） | 高（编译时） |
| 配置 | 简单 | 复杂 |
| 依赖 | 轻量 | 需要AspectJ编译器 |

---

## 面试高频问题

**Q1：JDK动态代理和CGLIB代理的区别？**

> 答：JDK代理需要目标类实现接口，通过Proxy.newProxyInstance生成；CGLIB通过继承生成子类代理，无需接口。Spring默认对有接口的使用JDK代理，无接口的使用CGLIB。

**Q2：AOP的通知类型有哪些？**

> 答：@Before前置通知、@After后置通知、@AfterReturning返回通知、@AfterThrowing异常通知、@Around环绕通知（可以控制目标方法执行）。

**Q3：Spring AOP的织入流程？**

> 答：@EnableAspectJAutoProxy启用→扫描@Aspect类→创建代理（BeanPostProcessor）→方法调用时通过代理执行通知。

**Q4：什么情况下AOP会失效？**

> 答：方法内部调用this.xxx()不会走代理；访问private方法；同类内部方法调用。可通过ApplicationContext获取代理对象解决。

**Q5：Spring事务和AOP的关系？**

> 答：Spring事务通过AOP实现。@Transactional标注的方法会被事务切面代理，方法执行前开启事务，执行后提交，异常时回滚。

---

## 总结

Spring AOP 提供了强大的解耦能力：
- **JDK代理**：需要接口，反射调用，性能略低
- **CGLIB代理**：继承生成子类，无需接口，性能更好
- **五种通知**：Before/After/AfterReturning/AfterThrowing/Around
- **切点表达式**：execution、within、@annotation等
- **织入时机**：运行时代理，Spring BeanPostProcessor实现
