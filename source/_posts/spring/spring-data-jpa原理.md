---
title: Spring Data JPA：ORM映射与Repository机制解析
date: 2023-09-07 16:34:07
tags:
  - Java进阶
  - Spring
categories: 学习
keywords: Spring Data JPA,Repository,ORM,JpaRepository,自定义查询
description: 深入理解Spring Data JPA的ORM映射机制与Repository查询机制
cover:
---

## 前言

Spring Data JPA 是 Spring Data 家族中最常用的模块，简化了 JPA 的使用，提供了强大的 Repository 抽象。本文深入解析其 ORM 映射与 Repository 查询机制。

## JPA 核心概念

### ORM映射关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    JPA ORM 映射                                    │
│                                                                   │
│  Java 类          映射            数据库表                        │
│  ┌─────────┐                    ┌─────────────┐                │
│  │  类     │ ───────────────▶  │   TABLE    │                │
│  │  属性   │                    │   COLUMNS  │                │
│  │  关系   │                    │  RELATIONS │                │
│  └─────────┘                    └─────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### 实体注解

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "username", nullable = false, unique = true, length = 50)
    private String username;

    @Column(name = "email", length = 100)
    private String email;

    @Column(name = "age")
    private Integer age;

    @Enumerated(EnumType.STRING)
    @Column(name = "status")
    private UserStatus status;

    @Temporal(TemporalType.DATE)
    @Column(name = "create_date")
    private Date createDate;
}
```

---

## Spring Data JPA Repository

### 层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Repository 层次结构                              │
│                                                                   │
│  ┌─────────────────┐                                             │
│  │   Repository<T>  │  ─── 标记接口                             │
│  └─────────────────┘                                             │
│          △                                                       │
│  ┌─────────────────┐                                             │
│  │ CrudRepository  │  ─── 增删改查                               │
│  └─────────────────┘                                             │
│          △                                                       │
│  ┌─────────────────┐                                             │
│  │ PagingAndSort   │  ─── 分页排序                              │
│  └─────────────────┘                                             │
│          △                                                       │
│  ┌─────────────────┐                                             │
│  │  JpaRepository  │  ─── JPA增强（刷新、批量）                  │
│  └─────────────────┘                                             │
│          △                                                       │
│  ┌─────────────────┐                                             │
│  │ SimpleJpaRepository│  ─── 默认实现                          │
│  └─────────────────┘                                             │
└─────────────────────────────────────────────────────────────────┘
```

### 基本使用

```java
// 1. 定义 Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // 方法命名查询
    User findByUsername(String username);

    // 多条件查询
    User findByUsernameAndEmail(String username, String email);

    // 模糊查询
    List<User> findByEmailContaining(String keyword);

    // 排序查询
    List<User> findByAgeGreaterThanOrderByAgeDesc(Integer age);

    // 分页查询
    Page<User> findByStatus(UserStatus status, Pageable pageable);
}

// 2. 使用
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public User getUserByUsername(String username) {
        return userRepository.findByUsername(username);
    }

    public Page<User> getUsersByPage(int page, int size) {
        return userRepository.findAll(PageRequest.of(page, size));
    }
}
```

### 方法命名规则

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // And: 条件And
    User findByUsernameAndEmail(String username, String email);

    // Or: 条件Or
    List<User> findByUsernameOrEmail(String username, String email);

    // Is/Equals: 相等
    User findByUsernameIs(String username);
    User findByUsername(String username);  // 默认Is

    // Between: 范围
    List<User> findByAgeBetween(Integer min, Integer max);

    // LessThan/GreaterThan: 小于/大于
    List<User> findByAgeLessThan(Integer maxAge);
    List<User> findByAgeGreaterThan(Integer minAge);

    // Like/Containing/StartsWith/EndsWith: 模糊
    List<User> findByUsernameLike(String pattern);
    List<User> findByUsernameContaining(String keyword);
    List<User> findByUsernameStartsWith(String prefix);
    List<User> findByUsernameEndsWith(String suffix);

    // In: IN查询
    List<User> findByIdIn(Collection<Long> ids);

    // Null/NotNull: 空值
    List<User> findByEmailNull();
    List<User> findByEmailNotNull();

    // True/False: 布尔
    List<User> findByActiveTrue();
    List<User> findByActiveFalse();

    // IgnoreCase: 忽略大小写
    List<User> findByUsernameIgnoreCase(String username);

    // Top/First: 限制数量
    User findFirstByOrderByAgeDesc();
    List<User> findTop5ByStatus(UserStatus status);
}
```

---

## @Query 自定义查询

### JPQL 查询

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // @Query 注解
    @Query("SELECT u FROM User u WHERE u.username = ?1")
    User findByUsernameQuery(String username);

    // 占位符
    @Query("SELECT u FROM User u WHERE u.username = :username AND u.email = :email")
    User findByUsernameAndEmailQuery(@Param("username") String username,
                                     @Param("email") String email);

    // 投影返回DTO
    @Query("SELECT new com.example.UserDTO(u.id, u.username) FROM User u WHERE u.id = ?1")
    UserDTO findUserDTOById(Long id);
}
```

### 原生 SQL 查询

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // 原生SQL
    @Query(value = "SELECT * FROM users WHERE username = ?1", nativeQuery = true)
    User findByUsernameNative(String username);

    // 分页原生SQL
    @Query(value = "SELECT * FROM users WHERE status = :status LIMIT :limit OFFSET :offset",
           countQuery = "SELECT COUNT(*) FROM users WHERE status = :status",
           nativeQuery = true)
    List<User> findByStatusNative(@Param("status") String status,
                                  @Param("limit") int limit,
                                  @Param("offset") int offset);
}
```

---

## 实体关系映射

### 一对一（@OneToOne）

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "profile_id")
    private UserProfile profile;
}

@Entity
public class UserProfile {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;
}
```

### 一对多（@OneToMany）

```java
@Entity
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees;
}

@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
}
```

### 多对多（@ManyToMany）

```java
@Entity
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany
    @JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

---

## EntityManager 与懒加载

### 懒加载异常

```java
@Service
public class UserService {

    @Transactional(readOnly = true)
    public User getUserWithProfile(Long id) {
        User user = userRepository.findById(id).orElseThrow();
        // 懒加载的 profile 在 session 关闭后访问会 LazyInitializationException
        System.out.println(user.getProfile().getBio());  // 异常！
        return user;
    }
}
```

### 解决懒加载

```java
// 方案1：EAGER 抓取
@OneToOne(mappedBy = "user", fetch = FetchType.EAGER)
private UserProfile profile;

// 方案2：JOIN FETCH
@Query("SELECT u FROM User u LEFT JOIN FETCH u.profile WHERE u.id = ?1")
User findByIdWithProfile(Long id);

// 方案3：@EntityGraph
@EntityGraph(attributePaths = {"profile"})
User findById(Long id);

// 方案4：Open Session in View（不推荐）
// 配置文件中开启
spring.jpa.open-in-view=true
```

---

## JpaRepository 高级特性

### 扩展 SimpleJpaRepository

```java
// 自定义 Repository 实现
public class CustomUserRepositoryImpl implements CustomUserRepository {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<User> findUsersCustom(Long age) {
        return em.createQuery("SELECT u FROM User u WHERE u.age > :age", User.class)
            .setParameter("age", age)
            .getResultList();
    }
}

// 组合接口
public interface UserRepository extends JpaRepository<User, Long>, CustomUserRepository {
}

// 使用
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<User> findOldUsers() {
        return userRepository.findUsersCustom(50);
    }
}
```

### 乐观锁与版本控制

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version  // 乐观锁
    private Long version;

    private String name;
}
```

---

## Specification 查询

### 动态查询

```java
// 定义 Specification
public class UserSpecification {

    public static Specification<User> hasUsername(String username) {
        return (root, query, cb) -> {
            if (username == null) return null;
            return cb.equal(root.get("username"), username);
        };
    }

    public static Specification<User> hasAgeGreaterThan(Integer age) {
        return (root, query, cb) -> {
            if (age == null) return null;
            return cb.greaterThan(root.get("age"), age);
        };
    }

    public static Specification<User> hasStatus(UserStatus status) {
        return (root, query, cb) -> {
            if (status == null) return null;
            return cb.equal(root.get("status"), status);
        };
    }
}

// 使用
@Service
public class UserService {

    @Autowired
    private UserRepository userRepository;

    public List<User> searchUsers(String username, Integer age, UserStatus status) {
        Specification<User> spec = Specification
            .where(UserSpecification.hasUsername(username))
            .and(UserSpecification.hasAgeGreaterThan(age))
            .and(UserSpecification.hasStatus(status));

        return userRepository.findAll(spec);
    }
}
```

---

## 审计功能

### 启用审计

```java
@SpringBootApplication
@EnableJpaAuditing
public class Application {}

// 实体使用
@Entity
@EntityListeners(AuditingEntityListener.class)
public class User {

    @CreatedDate
    @Column(name = "create_time", updatable = false)
    private LocalDateTime createTime;

    @LastModifiedDate
    @Column(name = "update_time")
    private LocalDateTime updateTime;

    @CreatedBy
    @Column(name = "creator")
    private String creator;

    @LastModifiedBy
    @Column(name = "modifier")
    private String modifier;
}

// 配置谁创建/修改
@Component
public class AuditConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(ctx -> ctx.getAuthentication())
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName)
            .orElse("system");
    }
}
```

---

## 常见面试题

**Q1：JPA 和 Hibernate 的关系？**

> 答：JPA 是 Java EE 规范（ javax.persistence ），定义了 ORM 接口。Hibernate 是 JPA 的实现之一。Spring Data JPA 进一步封装，提供了 Repository 抽象。

**Q2：fetchType.LAZY 和 EAGER 的区别？**

> 答：LAZY 懒加载，在真正访问关联对象时才加载。EAGER 立即加载，加载主对象时同时加载关联对象。懒加载可能导致懒加载异常。

**Q3：@Query 和方法名查询哪个更好？**

> 答：方法名查询适合简单场景，类型安全。@Query 适合复杂查询，支持 JPQL 和原生 SQL，更灵活。

**Q4：为什么需要 @Version？**

> 答：@Version 实现了乐观锁机制，防止并发更新导致的数据覆盖问题。更新时自动检查版本号，不匹配则抛异常。

**Q5：Spring Data JPA 如何实现动态条件查询？**

> 答：使用 JPA Criteria API 或 Spring Data JPA 的 Specification，可以动态拼接查询条件。

---

## 总结

Spring Data JPA 简化了数据访问层：
- **Repository**：强大的查询抽象，支持方法命名查询和@Query
- **ORM映射**：@Entity、@Column、@OneToMany等注解
- **关系映射**：一对一、一对多、多对多
- **Specification**：动态查询
- **审计**：@CreatedDate、@LastModifiedDate
