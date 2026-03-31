# Java开发技术博客系列设计文档

**日期：** 2026-03-31
**目标读者：** 有一定基础的Java开发者（进阶/求职导向）
**文章形式：** 知识点深度解析
**总篇数：** 44篇

---

## 文章结构规范

每篇文章统一格式：
1. **开头**：简述知识点的重要性和应用场景
2. **正文**：原理深度解析 + 关键代码示例
3. **结尾**：总结要点，给出进一步学习方向

## 分类与标签规划

| 分类 | 标签 |
|------|------|
| `Java进阶` | `JVM`, `并发编程`, `集合框架`, `Spring`, `数据库`, `中间件`, `生产异常处理` |

## 发布顺序建议

JVM → 并发编程 → 集合框架 → Spring → 数据库 → 中间件 → 生产异常处理

---

## 完整文章清单

### JVM系列（5篇）

1. Java内存区域详解：堆、栈、方法区深度剖析
2. 深入理解JVM垃圾回收：GC算法与收集器对比
3. 类加载机制：双亲委派模型与自定义类加载器
4. JVM性能调优实战：参数配置与问题排查
5. Java对象的创建、布局与访问定位

### 并发编程系列（5篇）

6. synchronized底层原理：从对象头到Monitor锁
7. AQS深度解析：ReentrantLock与AbstractQueuedSynchronizer
8. Java线程池原理：ThreadPoolExecutor核心机制详解
9. volatile关键字：内存可见性与指令重排序
10. Java并发容器：ConcurrentHashMap的演进与实现

### 集合框架系列（4篇）

11. HashMap深度解析：哈希冲突、扩容与红黑树
12. ConcurrentHashMap：分段锁到CAS+synchronized的演进
13. ArrayList与LinkedList：底层结构与性能对比
14. Java集合框架整体设计：接口体系与设计模式

### Spring系列（12篇）

15. Spring IoC容器：Bean的生命周期与依赖注入原理
16. Spring AOP：动态代理与切面编程深度解析
17. Spring事务管理：传播行为与事务失效场景分析
18. Spring Boot自动装配：@EnableAutoConfiguration原理
19. Spring MVC请求处理流程：DispatcherServlet源码解析
20. Spring循环依赖：三级缓存解决方案详解
21. Spring Security核心原理：认证与授权流程详解
22. Spring Cache：缓存抽象与@Cacheable注解原理
23. Spring事件机制：ApplicationEvent与观察者模式
24. Spring Boot Actuator：监控与健康检查
25. SpringCloud微服务：Nacos、Feign、Gateway核心原理
26. Spring Data JPA：ORM映射与Repository机制解析

### 数据库系列（6篇）

27. MySQL索引原理：B+树结构与索引优化策略
28. MySQL事务与锁：MVCC、行锁、间隙锁详解
29. MySQL执行计划：EXPLAIN深度解析与慢查询优化
30. Redis数据结构：五种基本类型的底层实现
31. Redis持久化：RDB与AOF机制对比分析
32. Redis分布式锁：实现原理与Redisson使用

### 中间件系列（6篇）

33. Kafka核心原理：消息队列、分区与消费者组
34. Kafka消息可靠性：生产者ACK与消费者幂等
35. RocketMQ事务消息：分布式事务解决方案
36. Nginx负载均衡：反向代理与常用配置详解
37. MyBatis核心原理：SqlSession与动态SQL解析
38. Elasticsearch入门：倒排索引与搜索原理

### 生产异常处理系列（6篇）

39. Java线上OOM排查：堆内存溢出定位与解决
40. CPU飙高问题排查：线程栈分析与死锁定位
41. Java线程死锁：检测工具与排查流程实战
42. 接口超时与雪崩：熔断限流降级方案详解
43. 分布式系统常见异常：幂等、重试、补偿机制
44. 生产日志规范：MDC链路追踪与ELK日志分析
