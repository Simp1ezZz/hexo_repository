---
title: SpringCloud微服务：Nacos、Feign、Gateway核心原理
date: 2023-09-04 08:51:38
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: SpringCloud,Nacos,Feign,Gateway,微服务
description: 深入理解SpringCloud微服务架构核心组件，服务注册发现与网关原理
cover:
---

## 前言

Spring Cloud 是微服务架构的核心解决方案，提供了服务注册发现、负载均衡、API网关、配置中心等功能。本文深入解析 Nacos、Feign、Gateway 等核心组件的原理。

## 微服务架构概述

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         微服务架构                                  │
│                                                                   │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                    │
│  │ Gateway │ ──▶ │  用户   │ ──▶ │  订单   │                    │
│  │  网关   │     │  服务   │     │  服务   │                    │
│  └────┬────┘     └────┬────┘     └────┬────┘                    │
│       │               │               │                           │
│       │               ▼               ▼                           │
│       │          ┌─────────────────────────┐                     │
│       │          │       Nacos 注册中心       │                    │
│       │          │   (服务注册与配置中心)     │                     │
│       │          └─────────────────────────┘                     │
│       │               │               │                           │
│       │               ▼               ▼                           │
│       │          ┌─────────┐     ┌─────────┐                     │
│       │          │  商品   │     │  支付   │                     │
│       │          │  服务   │     │  服务   │                     │
│       │          └─────────┘     └─────────┘                     │
│       │                                                        │
│       ▼                                                        │
│  ┌─────────┐                                                   │
│  │ Config  │                                                   │
│  │ 配置中心│                                                   │
│  └─────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Nacos 服务注册与发现

### 核心概念

| 概念 | 说明 |
|------|------|
| 服务注册 | 服务提供者向Nacos注册自己的信息 |
| 服务发现 | 消费者从Nacos获取服务提供者列表 |
| 健康检查 | Nacos定期检查服务实例是否可用 |
| 命名空间 | 隔离不同环境的服务 |
| 分组 | 按业务线或功能分组管理 |

### 服务注册

```yaml
# application.yml
spring:
  application:
    name: user-service
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: dev
        group: DEFAULT_GROUP
        metadata:
          version: v1
```

### 服务发现

```java
@RestController
public class UserController {

    @Autowired
    private NamingService namingService;

    @GetMapping("/services")
    public List<Instance> getServices() throws NacosException {
        // 获取所有服务实例
        return namingService.selectInstances("order-service", true);
    }

    @GetMapping("/service/{serviceName}")
    public Instance getService(@PathVariable String serviceName) throws NacosException {
        // 获取一个健康实例（负载均衡）
        return namingService.selectOneHealthyInstance(serviceName);
    }
}
```

### OpenFeign 服务调用

```java
// 1. 定义接口
@FeignClient(name = "order-service", path = "/orders")
public interface OrderClient {

    @GetMapping("/user/{userId}")
    List<Order> getOrdersByUserId(@PathVariable("userId") Long userId);

    @PostMapping
    Order createOrder(@RequestBody Order order);
}

// 2. 启用Feign
@SpringBootApplication
@EnableFeignClients
public class UserApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }
}

// 3. 使用
@Service
public class UserService {

    @Autowired
    private OrderClient orderClient;

    public List<Order> getUserOrders(Long userId) {
        return orderClient.getOrdersByUserId(userId);
    }
}
```

### Feign 原理

```
┌─────────────────────────────────────────────────────────────────┐
│                         Feign 调用流程                             │
│                                                                   │
│  1. @EnableFeignClients 扫描 @FeignClient 注解                  │
│                             │                                     │
│  2. 为每个 FeignClient 生成 JDK 动态代理                         │
│                             │                                     │
│  3. ClientImpl 执行 HTTP 请求                                    │
│     ├─ LoadBalancerFeignClient                                   │
│     │     └─ RibbonLoadBalancerClient                           │
│     │           └─ 从注册中心获取实例列表                         │
│     └─ OkHttpClient / HttpURLConnection                         │
│                                                                   │
│  4. 解码响应，返回结果                                            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Nacos 配置中心

### 共享配置

```yaml
# bootstrap.yml
spring:
  application:
    name: user-service
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        shared-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true
          - data-id: database.yaml
            group: DEFAULT_GROUP
            refresh: true
```

### 动态刷新配置

```java
@RestController
@RefreshScope  // 支持动态刷新
public class UserController {

    @Value("${app.max-users:100}")
    private int maxUsers;

    @GetMapping("/config")
    public String getConfig() {
        return "maxUsers: " + maxUsers;
    }
}
```

### 配置优先级

```
1. 命名空间 + 分组 + data-id
2. 共享配置（shared-configs）
3. 扩展配置（extension-configs）
4. 本地配置文件
```

---

## Spring Cloud Gateway

### 核心概念

| 组件 | 说明 |
|------|------|
| Route | 路由定义，包含ID、URI、断言、过滤器 |
| Predicate | 断言，用于匹配HTTP请求 |
| Filter | 过滤器，请求/响应拦截处理 |

### 路由配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service  # lb表示负载均衡
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1  # 去掉第一层路径

        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
            - Method=GET
          filters:
            - StripPrefix=1
            - RequestRateLimiter=10,20  # 限流
```

### 动态路由

```java
@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("user-service", r -> r
            .path("/api/user/**")
            .filters(f -> f.stripPrefix(1))
            .uri("lb://user-service"))
        .route("order-service", r -> r
            .path("/api/order/**")
            .filters(f -> f.stripPrefix(1))
            .uri("lb://order-service"))
        .build();
}
```

### 断言（Predicate）

```yaml
predicates:
  # 路径断言
  - Path=/api/user/**

  # 方法断言
  - Method=GET,POST

  # Header断言
  - Header=X-Request-Id, \d+

  # Query断言
  - Query=username

  # 组合断言（且）
  - Path=/api/** AND Method=GET
```

### 过滤器（Filter）

```java
// 全局过滤器
@Component
public class AuthFilter implements GlobalFilter {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");

        if (token == null || !tokenService.validate(token)) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        return chain.filter(exchange);
    }
}

// 局部过滤器
@Bean
public RouteLocator routes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route("add-header", r -> r
            .path("/api/**")
            .filters(f -> f.addRequestHeader("X-Gateway", "SpringCloud"))
            .uri("lb://user-service"))
        .build();
}
```

### Gateway 原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    Gateway 请求处理流程                             │
│                                                                   │
│  1. 请求进入 GatewayHandlerMapping                               │
│                             │                                     │
│  2. 匹配 Route（断言）                                           │
│     └─ Path=/api/user/** → user-service Route                    │
│                             │                                     │
│  3. GatewayWebHandler 执行过滤器链                               │
│     ├─ GlobalFilter (AuthFilter)                                 │
│     ├─ GatewayFilter (AddRequestHeader)                         │
│     └─ GatewayFilter (StripPrefix)                              │
│                             │                                     │
│  4. ProxyRouteFilter 转发到后端服务                              │
│     └─ lb://user-service (负载均衡)                             │
│                             │                                     │
│  5. 响应经过过滤器链返回                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 负载均衡

### Ribbon

```yaml
# 配置文件方式
user-service:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule
    # RandomRule - 随机
    # RoundRobinRule - 轮询
    # WeightedResponseTimeRule - 响应时间加权
    # BestAvailableRule - 最空闲连接
    # RetryRule - 重试
```

### Spring Cloud LoadBalancer

```java
@Configuration
public class LoadBalancerConfig {

    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory factory) {

        String name = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);

        return new RandomLoadBalancer(
            factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}
```

---

## 服务熔断与限流

### Resilience4j

```yaml
resilience4j:
  circuitbreaker:
    instances:
      userService:
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
        permittedNumberOfCallsInHalfOpenState: 3
```

```java
@Service
public class UserService {

    @CircuitBreaker(name = "userService", fallbackMethod = "fallback")
    public User getUserById(Long id) {
        return userClient.getUserById(id);
    }

    public User fallback(Long id, Exception e) {
        return User.defaultUser();
    }
}
```

### 限流

```java
@Component
public class RateLimitFilter implements GlobalFilter {

    private final RedisRateLimiter rateLimiter = new RedisRateLimiter(100, 200);

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String key = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();

        return rateLimiter.isAllowed(key, "100").flatMap(response -> {
            if (response.isAllowed()) {
                return chain.filter(exchange);
            }
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        });
    }
}
```

---

## 常见面试题

**Q1：Nacos与Eureka的区别？**

> 答：Nacos支持服务注册发现和配置管理，CP/AP模式可切换。Eureka只支持服务发现，只支持AP模式。Nacos功能更全面，社区活跃度更高。

**Q2：Feign的工作原理？**

> 答：@EnableFeignClients扫描@FeignClient，生成JDK动态代理。代理在方法调用时，通过LoadBalancer选取实例，构建HTTP请求发送。

**Q3：Gateway的断言和过滤器区别？**

> 答：断言用于匹配HTTP请求（路径、方法、Header等），决定路由到哪个服务。过滤器在请求前后进行拦截处理（日志、认证、限流等）。

**Q4：如何实现动态路由？**

> 答：Nacos配置中心存储路由规则，Gateway监听配置变化，动态更新路由表。

**Q5：熔断和限流的区别？**

> 答：熔断在服务故障时快速失败，避免雪崩。限流在流量过大时拒绝请求，保护系统不被压垮。

---

## 总结

SpringCloud微服务架构核心组件：
- **Nacos**：服务注册发现 + 配置中心
- **Feign**：声明式HTTP客户端
- **Gateway**：API网关 + 路由 + 过滤器
- **LoadBalancer**：客户端负载均衡
- **Resilience4j**：熔断、限流、降级
