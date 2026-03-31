---
title: Spring Boot自动装配：@EnableAutoConfiguration原理
date: 2026-04-18 10:00:00
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring Boot,自动装配,@EnableAutoConfiguration,SpringFactoriesLoader
description: 深入理解Spring Boot自动装配机制，spring.factories与自动配置原理
cover:
---

## 前言

Spring Boot 最核心的特性是"约定大于配置"，其中的自动装配机制让我们可以零配置使用各种组件。本文深入解析 Spring Boot 自动装配的底层原理。

## 自动装配 vs 手动装配

### 传统 Spring 配置

```java
@Configuration
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/test");
        config.setUsername("root");
        config.setPassword("password");
        return new HikariDataSource(config);
    }

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        return factory.getObject();
    }

    @Bean
    public MapperScannerConfigurer mapperScanner() {
        MapperScannerConfigurer configurer = new MapperScannerConfigurer();
        configurer.setBasePackage("com.example.mapper");
        return configurer;
    }
}
```

### Spring Boot 自动装配

```java
// 只需要一行配置
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=root
spring.datasource.password=password

// Spring Boot 自动完成：
// 1. 创建 DataSource
// 2. 创建 SqlSessionFactory
// 3. 扫描并注册 Mapper
```

---

## @SpringBootApplication 解析

### 组合注解

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@EnableAutoConfiguration
@ComponentScan
public @interface SpringBootApplication {
    // ...
}

// 等价于同时标注：
// @Configuration
// @EnableAutoConfiguration
// @ComponentScan
```

### @ComponentScan

```java
@ComponentScan(
    basePackages = "com.example",  // 扫描这些包
    excludeFilters = {             // 排除规则
        @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,
                             classes = ExcludeConfig.class)
    }
)
public class SpringBootApplication {
    // 主类所在的包会被默认扫描
}
```

---

## @EnableAutoConfiguration

### 注解解析

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    // 排除特定自动配置类
    Class<?>[] exclude() default {};

    // 按名称排除
    String[] excludeName() default {};

    // 环境属性前缀
    String env() default "spring.boot";
}
```

### 核心组件

```
@EnableAutoConfiguration
    │
    └── @Import(AutoConfigurationImportSelector.class)
            │
            └── SpringFactoriesLoader.loadFactoryNames()
                    │
                    └── META-INF/spring.factories
```

---

## SpringFactoriesLoader 机制

### spring.factories 文件

```properties
# 文件位置: META-INF/spring.factories

# 自动配置类
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration,\
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration

# 失败分析器
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### 加载流程

```java
public class SpringFactoriesLoader {

    // 从 spring.factories 加载配置
    public static List<String> loadFactoryNames(
            Class<?> factoryType,
            @Nullable ClassLoader classLoader) {

        String factoryTypeName = factoryType.getName();
        // 1. 从缓存获取
        // 2. 从 META-INF/spring.factories 加载
        // 3. 从 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 加载 (Spring Boot 2.7+)

        return result;
    }
}
```

### Spring Boot 2.7+ 新格式

```properties
# 新格式: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

# 每个配置类一行（Spring Boot 3.0+ 推荐）
com.example.MyAutoConfiguration
com.example.AnotherAutoConfiguration
```

---

## AutoConfiguration 注解

### @AutoConfiguration

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(
        prefix = "spring.datasource",
        name = "url",
        havingValue = "",
        matchIfMissing = true
    )
    public DataSource dataSource() {
        // 自动配置逻辑
        return new HikariDataSource();
    }
}
```

### 常用条件注解

| 注解 | 说明 |
|------|------|
| @ConditionalOnClass | 某类在classpath中 |
| @ConditionalOnMissingClass | 某类不在classpath中 |
| @ConditionalOnBean | 容器中存在某Bean |
| @ConditionalOnMissingBean | 容器中不存在某Bean |
| @ConditionalOnProperty | 配置属性满足条件 |
| @ConditionalOnWebApplication | 是Web应用 |

### 条件注解示例

```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)  // DataScource类存在
@ConditionalOnMissingBean(DataSource.class)  // 没有手动配置的DataSource
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnProperty(
        prefix = "spring.datasource",
        name = "url"
    )
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }

    @Bean
    @ConditionalOnProperty(
        prefix = "spring.datasource",
        name = "url"
    )
    @ConditionalOnMissingBean
    public DataSourceInitializer dataSourceInitializer() {
        return new DataSourceInitializer();
    }
}
```

---

## 自动装配流程

### 完整流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    Spring Boot 启动                          │
│                                                             │
│  1. SpringApplication.run()                                │
│                                                             │
│  2. 创建 ApplicationContext                                │
│                                                             │
│  3. 加载 spring.factories 中的 EnableAutoConfiguration     │
│                                                             │
│  4. 排除用户排除的配置                                      │
│                                                             │
│  5. 按 @Conditional 条件过滤                              │
│                                                             │
│  6. 注册剩余的 AutoConfiguration Bean                     │
│                                                             │
│  7. 完成 Bean 填充和初始化                                │
└─────────────────────────────────────────────────────────────┘
```

### AutoConfigurationImportSelector

```java
public class AutoConfigurationImportSelector
        implements DeferredImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // 1. 获取 EnableAutoConfiguration 注解的属性
        // 2. 加载 spring.factories 中的所有自动配置类
        // 3. 排除用户通过 exclude 排除的类
        // 4. 应用 @Conditional 条件过滤
        // 5. 返回需要导入的配置类数组
        return autoConfigurationEntry.getConfigurations().toArray(new String[0]);
    }
}
```

---

## 自定义自动配置

### 创建自动配置模块

```
my-starter/
├── src/main/java/com/example/autoconfigure/
│   ├── MyAutoConfiguration.java
│   └── MyProperties.java
├── src/main/resources/META-INF/
│   └── spring.factories
└── pom.xml
```

### 1. 定义 Properties 类

```java
@ConfigurationProperties(prefix = "myapp.service")
public class MyProperties {

    private String url = "http://localhost:8080";
    private int timeout = 5000;
    private boolean enabled = true;

    // getters and setters
}
```

### 2. 创建 AutoConfiguration

```java
@AutoConfiguration
@ConditionalOnClass(MyService.class)  // MyService类存在
@ConditionalOnProperty(prefix = "myapp.service", havingValue = "enabled", matchIfMissing = true)
@EnableConfigurationProperties(MyProperties.class)
public class MyAutoConfiguration {

    @Autowired
    private MyProperties properties;

    @Bean
    @ConditionalOnMissingBean
    public MyService myService() {
        return new MyService(properties.getUrl(), properties.getTimeout());
    }
}
```

### 3. 注册自动配置

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfigure.MyAutoConfiguration
```

### 4. 使用自定义 Starter

```properties
# application.properties
myapp.service.enabled=true
myapp.service.url=https://api.example.com
myapp.service.timeout=3000
```

```java
// 直接使用，无需额外配置
@Autowired
private MyService myService;
```

---

## Starter 机制

### Starter 结构

```
spring-boot-starter-xxx/
├── src/main/java/
│   └── ...
├── src/main/resources/
│       └── META-INF/
│           └── spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
└── pom.xml
```

### 官方 Starter 命名

| Starter | 用途 |
|---------|------|
| spring-boot-starter-web | Web开发 |
| spring-boot-starter-data-jpa | JPA数据访问 |
| spring-boot-starter-data-redis | Redis |
| spring-boot-starter-validation | 表单验证 |
| spring-boot-starter-actuator | 应用监控 |

### 自定义 Starter 命名

```
# 推荐命名
xxx-spring-boot-starter

# 例如
myapp-spring-boot-starter
```

---

## 启动 Banner 自定义

```java
// src/main/resources/banner.txt
 ____  _          _ _                   ____
| __ )| |_ _   _| | |_   ___  __ _    / ___|__ ___   _____ _ __ _ __
|  _ \| __| | | | | __| / __|/ _` |  | |   / _` \ \ / / _ \ '__| '_ \
| |_) | |_| |_| | | |_  \__ \ (_| |  | |__| (_| |\ V /  __/ |  | | | |
|____/ \__|\__,_|_|\__| |___/\__,_|   \____\__,_| \_/ \___|_|  |_| |_|

${application.title} ${application.version}
Powered by Spring Boot ${spring-boot.version}
```

---

## 常见面试题

**Q1：Spring Boot 自动装配原理？**

> 答：通过 @EnableAutoConfiguration 导入 AutoConfigurationImportSelector，扫描 spring.factories 中的 EnableAutoConfiguration 配置类，按 @Conditional 条件过滤，最终注册自动配置的 Bean。

**Q2：spring.factories 和 AutoConfiguration.imports 的区别？**

> 答：spring.factories 是传统格式，AutoConfiguration.imports 是 Spring Boot 2.7+ 推荐的新格式。两种格式可以共存，但新格式更简洁。

**Q3：@Conditional 条件注解有哪些？**

> 答：@ConditionalOnClass（类存在）、@ConditionalOnMissingClass（类不存在）、@ConditionalOnBean（Bean存在）、@ConditionalOnMissingBean（Bean不存在）、@ConditionalOnProperty（配置属性满足）、@ConditionalOnWebApplication（是Web应用）。

**Q4：如何自定义 Starter？**

> 答：创建 xxx-spring-boot-starter 模块，包含 AutoConfiguration 类和 spring.factories 注册。用户提供 pom 依赖和配置即可使用。

**Q5：自动装配的 Bean 优先级？**

> 答：用户手动配置的 Bean 优先级高于自动配置。通过 @ConditionalOnMissingBean 确保自动配置只在没有用户配置时才生效。

---

## 总结

Spring Boot 自动装配是"约定大于配置"的核心：
- **@EnableAutoConfiguration**：触发自动配置
- **SpringFactoriesLoader**：加载 spring.factories
- **@AutoConfiguration**：标记自动配置类
- **@Conditional**：条件过滤，控制生效条件
- **Starter**：封装自动配置的发布单元
