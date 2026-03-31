---
title: Spring Boot Actuator：监控与健康检查
date: 2023-11-07 11:53:18
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring Boot,Actuator,健康检查,监控,端点
description: 深入理解Spring Boot Actuator监控机制，自定义端点与安全配置
cover:
---

## 前言

Spring Boot Actuator 提供了强大的应用监控功能，可以查看应用的健康状态、指标信息、配置详情等。本文深入解析 Actuator 的使用与扩展。

## 快速开始

### 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- 如需完整监控功能 -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### 暴露端点

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: always  # 显示详细健康信息
```

### 访问端点

```
GET /actuator/health      # 健康检查
GET /actuator/info        # 应用信息
GET /actuator/metrics    # 指标列表
GET /actuator/metrics/jvm.memory.used  # 特定指标
GET /actuator/prometheus  # Prometheus格式指标
GET /actuator/env         # 环境变量
GET /actuator/beans       # Bean列表
GET /actuator/caches      # 缓存信息
```

---

## 内置端点

### 健康检查端点

```json
GET /actuator/health

{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "6.0.5"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 512GB,
        "free": 256GB,
        "threshold": 10GB
      }
    }
  }
}
```

### 自定义健康指标

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        try {
            // 检查自定义服务
            boolean serviceAvailable = checkExternalService();

            if (serviceAvailable) {
                return Health.up()
                    .withDetail("service", "available")
                    .withDetail("responseTime", "100ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "unavailable")
                    .withDetail("error", "Connection timeout")
                    .build();
            }
        } catch (Exception e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### DiskSpaceHealthIndicator

```java
// 内置磁盘空间检查
// 阈值配置
management:
  health:
    diskspace:
      enabled: true
      threshold: 10GB  # 小于此值则DOWN
```

---

## 指标监控

### micrometer 指标

```java
@RestController
public class OrderController {

    private final MeterRegistry meterRegistry;

    public OrderController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    @PostMapping("/order")
    public Order createOrder(@RequestBody OrderRequest request) {
        // 计数
        Counter counter = meterRegistry.counter("orders.created");
        counter.increment();

        // 计时
        Timer timer = meterRegistry.timer("orders.process.time");
        return timer.record(() -> {
            return orderService.create(request);
        });
    }

    @GetMapping("/order/{id}")
    public Order getOrder(@PathVariable Long id) {
        // Gauge 动态值
        meterRegistry.gauge("orders.active", activeOrders.size());
        return orderService.findById(id);
    }
}
```

### @Timed 注解

```java
@Service
public class UserService {

    @Timed(value = "user.create.time", percentile = 0.95)
    public User createUser(User user) {
        return userRepository.save(user);
    }
}
```

---

## 自定义端点

### @Endpoint

```java
@Component
@Endpoint(id = "custom")
public class CustomEndpoint {

    @ReadOperation
    public Map<String, Object> info() {
        Map<String, Object> info = new HashMap<>();
        info.put("version", "1.0.0");
        info.put("timestamp", System.currentTimeMillis());
        return info;
    }

    @WriteOperation
    public String reloadCache(@Selector String cacheName) {
        cacheManager.clear(cacheName);
        return "Cache " + cacheName + " cleared";
    }

    @DeleteOperation
    public String clearAllCaches() {
        cacheManager.clearAll();
        return "All caches cleared";
    }
}
```

### @WebEndpoint

```java
@Component
@WebEndpoint(id = "webCustom")
public class WebCustomEndpoint {

    @GetMapping("/actuator/webCustom/info")
    public Map<String, Object> info() {
        return Map.of("web", "endpoint");
    }
}
```

### @JmxEndpoint

```java
@Component
@JmxEndpoint(id = "jmxCustom")
public class JmxCustomEndpoint {

    @ReadOperation
    public String info() {
        return "JMX endpoint info";
    }
}
```

---

## 健康检查聚合

### 自定义健康聚合

```java
@Component
public class CustomHealthAggregator implements HealthAggregator {

    @Override
    public Status aggregate(Collection<HealthComponent> components) {
        // 逻辑：任何一个DOWN则DOWN
        //      任何一个UNKNOWN且无DOWN则UNKNOWN
        //      否则UP
        boolean hasDown = components.stream()
            .anyMatch(c -> c.getStatus().equals(Status.DOWN));
        if (hasDown) {
            return Status.DOWN;
        }

        boolean hasUnknown = components.stream()
            .anyMatch(c -> c.getStatus().equals(Status.UNKNOWN));
        if (hasUnknown) {
            return Status.UNKNOWN;
        }

        return Status.UP;
    }
}
```

### Liveness 与 Readiness

```java
// Spring Boot 2.6+ Liveness/Readiness
management:
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

// Kubernetes 探针
// LivenessProbe: 应用是否存活
// ReadinessProbe: 应用是否就绪
```

---

## 安全配置

### 保护敏感端点

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/actuator/info").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
            )
            .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

### 敏感信息过滤

```yaml
management:
  endpoint:
    env:
      access: readonly
    beans:
      access: readonly
    configprops:
      access: readonly
    heapdump:
      access: none
    threaddump:
      access: none
```

---

## 集成 Prometheus

### 配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: prometheus
  metrics:
    tags:
      application: ${spring.application.name}
  prometheus:
    metrics:
      export:
        enabled: true
```

### Prometheus 查询

```promql
# JVM内存使用
jvm_memory_used_bytes{area="heap"}

# HTTP请求数
http_server_requests_seconds_count{uri="/api/users"}

# 请求延迟P99
histogram_quantile(0.99, rate(http_server_requests_seconds_bucket{uri="/api/users"}[5m]))
```

---

## Spring Boot Admin

### 简介

Spring Boot Admin 是一个开源的监控管理平台，基于Actuator端点。

### 服务端

```java
@SpringBootApplication
@EnableAdminServer
public class AdminServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(AdminServerApplication.class, args);
    }
}
```

### 客户端配置

```yaml
spring:
  boot:
    admin:
      client:
        url: http://localhost:8080
        instance:
          prefer-ip: true
```

---

## 常见面试题

**Q1：Spring Boot Actuator的作用？**

> 答：提供应用监控端点，包括健康检查、指标信息、配置查看、日志级别调整等功能。用于生产环境监控和问题排查。

**Q2：如何自定义健康检查？**

> 答：实现HealthIndicator接口，重写health()方法返回Health对象。

**Q3：Liveness和Readiness探针的区别？**

> 答：Liveness探针检查应用是否存活（重启后恢复）；Readiness探针检查应用是否就绪（可以接收流量）。

**Q4：Actuator端点如何保护？**

> 答：通过Spring Security配置访问权限，或禁用不需要的端点。

**Q5：如何集成Prometheus？**

> 答：引入micrometer-registry-prometheus依赖，配置endpoints.web.exposure.include包含prometheus。

---

## 总结

Spring Boot Actuator是生产监控利器：
- **端点**：health、metrics、info、prometheus等
- **自定义端点**：@Endpoint、@WebEndpoint
- **健康检查**：HealthIndicator
- **指标**：micrometer
- **安全**：Spring Security集成
