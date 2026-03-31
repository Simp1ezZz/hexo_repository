---
title: MySQL执行计划：EXPLAIN深度解析与慢查询优化
date: 2024-04-18 00:45:43
tags:
  - 文章
categories: 学习
---


## 前言

在实际的开发工作中，我们经常会遇到查询性能问题。仅仅通过观察SQL语句很难判断其执行效率，而`EXPLAIN`命令则为我们提供了一种查看查询执行计划的有效手段。本文将深入探讨`EXPLAIN`的输出格式、各个字段的含义，以及如何根据执行计划进行慢查询优化。

## EXPLAIN基础

### 基本语法

```sql
EXPLAIN [FORMAT=JSON] sql_statement;
-- 或者
EXPLAIN ANALYZE sql_statement;  -- MySQL 8.0.18+
```

### 基础示例

```sql
EXPLAIN SELECT * FROM users WHERE age > 25 AND city = 'Beijing';
```

输出：

```
+----+-------------+-------+------------+-------+---------------+--------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key    | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | users | NULL       | range | idx_age       | idx_age | 5       | NULL |  152 |    10.00 | Using index |
+----+-------------+-------+------------+-------+---------------+--------+---------+------+------+----------+-------------+
```

## 输出字段详解

### 1. id

**含义**：SELECT标识符，表示查询中SELECT的顺序。

```sql
EXPLAIN SELECT * FROM (SELECT id FROM users WHERE id > 10) t WHERE t.id > 5;
```

结果中id值越大，优先级越高；如果id相同，则按照从上到下的顺序执行。

### 2. select_type

| 值 | 含义 |
|---|---|
| SIMPLE | 简单SELECT，不使用子查询或UNION |
| PRIMARY | 最外层SELECT |
| SUBQUERY | 子查询中的第一个SELECT |
| DERIVED | 派生表（FROM子句中的子查询） |
| UNION | UNION中的第二个或后面的SELECT |
| UNION RESULT | UNION结果集 |

```sql
-- DERIVED示例
EXPLAIN SELECT * FROM (SELECT MAX(id) as max_id FROM users GROUP BY status) t;
```

### 3. table

表示正在访问的表名，如果是派生表或子查询，显示为`<derivedN>`或`<subqueryN>`，其中N是ID。

### 4. type（重要）

type字段表示访问类型，从最优到最差的排序：

```
system > const > eq_ref > ref > range > index > ALL
```

| 类型 | 说明 | 场景 |
|---|---|---|
| **system** | 表只有一行 | 系统表 |
| **const** | 通过索引一次命中 | 主键或唯一索引的等值查询 |
| **eq_ref** | 唯一索引扫描 | 使用主键或唯一索引关联查询 |
| **ref** | 非唯一索引扫描 | 普通索引的等值查询 |
| **range** | 索引范围扫描 | BETWEEN、IN、>、<等 |
| **index** | 全索引扫描 | 只是扫描索引树 |
| **ALL** | 全表扫描 | 最差情况，需要优化 |

```sql
-- const示例：主键等值查询
EXPLAIN SELECT * FROM users WHERE id = 1;

-- eq_ref示例：关联查询
EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id;

-- ref示例：普通索引查询
EXPLAIN SELECT * FROM users WHERE age = 25;

-- range示例：范围查询
EXPLAIN SELECT * FROM users WHERE id BETWEEN 1 AND 100;

-- ALL示例：全表扫描（需要优化）
EXPLAIN SELECT * FROM users WHERE name = '张三';
```

### 5. possible_keys和key

- **possible_keys**：可能使用的索引（显示可能被使用的索引）
- **key**：实际使用的索引（显示被实际使用的索引）

```sql
EXPLAIN SELECT * FROM users WHERE age = 25 AND city = 'Beijing';
-- possible_keys: idx_age, idx_city
-- key: idx_age  -- MySQL选择了idx_age
```

如果possible_keys和key都为空，说明没有可用的索引。

### 6. key_len

表示索引使用的字节数，用于判断复合索引使用了多少列。

```sql
-- users表结构：
-- id INT PRIMARY KEY,
-- name VARCHAR(50),
-- age INT,
--复合索引 idx_name_age (name(20), age)

EXPLAIN SELECT * FROM users WHERE name = '张三' AND age = 25;
-- key_len = 20*3+4 = 64 (VARCHAR(50)使用3字节编码，保留20字符)
```

### 7. ref

表示索引列的比较操作类型：

- `const`：使用常量值比较
- `func`：使用函数
- `column_name`：使用某列的值
- `NULL`：没有使用索引

```sql
EXPLAIN SELECT * FROM users WHERE id = 1;  -- ref: const
EXPLAIN SELECT * FROM users WHERE age = (SELECT MAX(age) FROM users);  -- ref: func
```

### 8. rows

表示MySQL估算的要扫描的行数，不是返回的行数。这个值越小越好。

```sql
EXPLAIN SELECT * FROM users WHERE age > 20;
-- rows: 1523  -- 估算需要扫描1523行
```

### 9. filtered

表示符合WHERE条件的行数占扫描行数的百分比，值越大越好。

```sql
EXPLAIN SELECT * FROM users WHERE age > 25 AND city = 'Beijing';
-- filtered: 10.00  -- 约10%的行符合条件
```

### 10. Extra（重要）

包含不适合在其他列中显示的额外信息，常见值：

| 值 | 含义 | 优化建议 |
|---|---|---|
| Using filesort | 需要额外的排序操作 | 考虑添加索引 |
| Using temporary | 使用临时表 | 考虑优化查询或增加内存 |
| Using index | 覆盖索引扫描 | 性能好 |
| Using index condition | 索引下推 | 性能好 |
| Using where | 使用WHERE过滤 | 正常 |
| Using join buffer | 使用连接缓冲区 | 考虑优化连接顺序 |
| Impossible WHERE | WHERE条件永假 | 检查SQL逻辑 |
| Select tables optimized away | 优化器优化掉了表 | 正常 |

#### Using filesort

```sql
-- 没有合适索引时，会出现Using filesort
EXPLAIN SELECT * FROM users ORDER BY age;

-- 优化：添加索引
ALTER TABLE users ADD INDEX idx_age (age);
EXPLAIN SELECT * FROM users ORDER BY age;  -- Extra变为Using index
```

#### Using temporary

```sql
-- DISTINCT、GROUP BY、UNION等可能产生临时表
EXPLAIN SELECT DISTINCT city FROM users GROUP BY status;

-- 优化：添加索引
ALTER TABLE users ADD INDEX idx_city_status (city, status);
```

#### Using index（覆盖索引）

```sql
-- 查询的列都在索引中，不需要回表
EXPLAIN SELECT id, age FROM users WHERE age > 25;

-- 无法使用覆盖索引（需要回表）
EXPLAIN SELECT * FROM users WHERE age > 25;
```

#### Using index condition（索引下推）

```sql
-- MySQL 5.6+ 引入的优化
EXPLAIN SELECT * FROM users WHERE age > 25 AND name LIKE '%张%';
-- Extra: Using index condition
```

## 慢查询优化实战

### 案例1：全表扫描优化

**问题SQL**：
```sql
SELECT * FROM orders WHERE status = 'completed' AND DATE(created_at) = '2024-01-01';
```

**问题分析**：
```sql
EXPLAIN SELECT * FROM orders WHERE status = 'completed' AND DATE(created_at) = '2024-01-01';
-- type: ALL (全表扫描)
-- Extra: Using where
```

**优化方案**：

1. 避免在索引列上使用函数：
```sql
-- 修改前
WHERE DATE(created_at) = '2024-01-01'

-- 修改后
WHERE created_at >= '2024-01-01 00:00:00' AND created_at < '2024-01-02 00:00:00'
```

2. 添加复合索引：
```sql
ALTER TABLE orders ADD INDEX idx_status_created (status, created_at);
```

3. 优化后验证：
```sql
EXPLAIN SELECT * FROM orders
WHERE status = 'completed'
AND created_at >= '2024-01-01 00:00:00'
AND created_at < '2024-01-02 00:00:00';
-- type: range
-- key: idx_status_created
```

### 案例2：分页查询优化

**问题SQL**：
```sql
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;
```

**问题分析**：
深度分页需要扫描大量数据，效率极低。

**优化方案**：

1. **使用延迟关联**：
```sql
-- 修改前：偏移量越大越慢
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- 修改后：使用延迟关联
SELECT t.* FROM orders t
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 10) AS tmp
ON t.id = tmp.id;
```

2. **使用书签式分页**：
```sql
-- 修改前
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- 修改后：记住上次查询的最大ID
SELECT * FROM orders
WHERE id > 1000000
ORDER BY id
LIMIT 10;
```

3. **使用EXPLAIN验证优化效果**：
```sql
EXPLAIN SELECT t.* FROM orders t
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 10) AS tmp
ON t.id = tmp.id;
```

### 案例3：JOIN查询优化

**问题SQL**：
```sql
SELECT o.*, u.name, u.email
FROM orders o, users u
WHERE o.user_id = u.id AND o.amount > 1000;
```

**问题分析**：
```sql
EXPLAIN SELECT o.*, u.name, u.email
FROM orders o, users u
WHERE o.user_id = u.id AND o.amount > 1000;
-- type: ALL (orders表全表扫描)
```

**优化方案**：

1. 确保关联列有索引：
```sql
ALTER TABLE orders ADD INDEX idx_user_id (user_id);
ALTER TABLE orders ADD INDEX idx_amount (amount);
ALTER TABLE users ADD INDEX idx_id (id);  -- 主键已有索引
```

2. 添加覆盖索引：
```sql
ALTER TABLE orders ADD INDEX idx_amount_user (amount, user_id);
```

3. 优化后的执行计划：
```sql
EXPLAIN SELECT o.*, u.name, u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.amount > 1000;
-- type: range
-- key: idx_amount_user
```

## EXPLAIN ANALYZE（MySQL 8.0.18+）

MySQL 8.0.18引入了`EXPLAIN ANALYZE`，可以实际执行查询并分析性能：

```sql
EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE o.amount > 1000;
```

输出示例：

```
-> Nested loop inner join  (cost=1234.56 rows=100) (actual time=0.123..5.678 rows=50 loops=1)
    -> Index range scan on o using idx_amount_user over (amount > 1000)  (cost=1000.00 rows=100) (actual time=0.050..2.345 rows=50 loops=1)
    -> Index lookup on u using PRIMARY (id=o.user_id)  (cost=2.34 rows=1) (actual time=0.020..0.025 rows=1 loops=50)
```

## 索引优化建议

### 1. 选择性高的列放在前面

```sql
-- 索引列选择性：id > email > status > created_at
-- 复合索引顺序应该是：idx_email_status_created (email, status, created_at)
```

### 2. 最左前缀原则

```sql
-- 有索引 idx(a, b, c)
-- 可以命中：WHERE a=1 / WHERE a=1 AND b=2 / WHERE a=1 AND b=2 AND c=3
-- 无法命中：WHERE b=2 / WHERE c=3
```

### 3. 覆盖索引减少回表

```sql
-- 查询需要的列都在索引中
CREATE INDEX idx_covering ON orders(status, user_id, amount);
SELECT status, user_id, amount FROM orders WHERE status = 'completed';
```

## 常见面试题

**Q1：EXPLAIN中type有哪些类型？哪个最优？**

> 参考答案：type的值从优到劣为：system > const > eq_ref > ref > range > index > ALL。其中system是最优的，表示只有一行数据；const是通过主键或唯一索引等值查询；eq_ref是唯一索引关联查询；ref是非唯一索引扫描；range是索引范围扫描；index是全索引扫描；ALL是最差的全表扫描。

**Q2：Using filesort是怎么产生的？如何优化？**

> 参考答案：Using filesort表示MySQL无法利用索引完成排序，需要额外的排序操作。优化方法包括：1) 添加合适的索引，让排序在索引上完成；2) 尽量使用主键或有序索引列进行排序；3) 对于复合索引，确保ORDER BY遵循最左前缀原则。

**Q3：什么是覆盖索引？有什么好处？**

> 参考答案：覆盖索引是指查询的所有列都在索引中，无需回表查询数据。好处包括：1) 减少数据访问量；2) 降低IO开销；3) 因为索引一般比数据小，可以减少内存占用；4) 索引按顺序存储，有利用于范围查询和排序。

**Q4：如何优化深度分页？**

> 参考答案：深度分页的优化方法：1) 使用延迟关联，先查询索引列，再关联查询完整数据；2) 使用书签式分页，记录上次查询的位置；3) 使用游标分页；4) 如果数据量大且很少更新，可以考虑使用ES等搜索引擎。

## 总结

`EXPLAIN`是MySQL性能优化的重要工具，通过分析其输出可以：
- 判断查询是否走索引
- 了解查询的执行顺序
- 发现潜在的性能问题
- 指导索引创建和SQL改写

建议在实际工作中，对重要查询都先使用`EXPLAIN`分析执行计划，确保查询性能符合预期。
