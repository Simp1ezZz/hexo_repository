---
title: MySQL索引原理：B+树结构与索引优化策略
date: 2026-04-27 10:00:00
tags:
  - Java进阶
  - 数据库
categories: 学习
keywords: MySQL,索引,B+树,索引优化,Explain
description: 深入理解MySQL索引底层实现，B+树结构与索引优化策略
cover:
---

## 前言

索引是数据库查询性能优化的核心技术。理解索引的底层结构和工作原理，是写出高效SQL的基础。本文深入解析MySQL索引的实现机制。

## 索引概述

### 什么是索引

索引是一种特殊的数据结构，能够快速定位数据，而不需要扫描全表。

```
无索引：SELECT * FROM users WHERE name = '张三'
       └──▶ 全表扫描 100万行 → 逐行比对 → 找到目标

有索引：SELECT * FROM users WHERE name = '张三'
       └──▶ 索引树查找 3次 → 直接定位 → 找到目标
```

### 索引类型

| 类型 | 说明 | 场景 |
|------|------|------|
| 主键索引 | 主键自动建立，唯一且非空 | 主键查询 |
| 唯一索引 | 值唯一，可为空 | 唯一性约束 |
| 普通索引 | 普通字段索引 | 加速查询 |
| 全文索引 | 文本内容搜索 | LIKE模糊搜索 |
| 复合索引 | 多字段组合索引 | 多条件查询 |

---

## B+树结构

### B+树 vs B树

```
B树节点结构：
┌─────────┬─────────┬─────────┬─────────┐
│  data1  │  data2  │  data3  │  data4  │
└─────────┴─────────┴─────────┴─────────┘
     ▲           ▲           ▲
     │           │           │
  子节点      子节点       子节点

B+树节点结构：
┌─────────┬─────────┬─────────┬─────────┐
│   key   │   key   │   key   │   key   │
└────┬────┴────┬────┴────┬────┴────┬────┘
     │         │         │
  子节点    子节点     子节点

叶子节点（双向链表）：
┌────┬────┬────┬────┬────┬────┐
│ 5  │ 10 │ 15 │ 20 │ 25 │ 30 │ ...
└────┴────┴────┴────┴────┴────┘
   ◀─────────────────────────▶
        双向链表，便于范围查询
```

### B+树的优势

1. **所有数据都在叶子节点**：查询稳定
2. **叶子节点链表连接**：范围查询高效
3. **非叶子节点只存key**：每个节点能容纳更多key，减少树高
4. **查询效率稳定**：都是O(log n)，不会像B树波动大

### MySQL B+树结构

```
┌──────────────────────────────────────────────────────────────┐
│                        B+树索引结构                           │
│                                                               │
│                        [根节点]                                │
│                     非叶子节点                                 │
│                   存储索引列的值                               │
│                                                               │
│         ┌─────────────┴─────────────┐                       │
│         │                           │                        │
│    [非叶子节点]               [非叶子节点]                     │
│                                                               │
│    ┌────┴────┐               ┌────┴────┐                    │
│    │         │               │         │                    │
│  [叶节点]  [叶节点]  ...    [叶节点]  [叶节点]  ...           │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              叶子节点双向链表                             │  │
│  │  [15] ──▶ [20] ──▶ [25] ──▶ [30] ──▶ [35] ──▶ ...  │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## InnoDB索引实现

### 主键索引（聚集索引）

InnoDB表数据按主键顺序存储，主键索引叶子节点存储完整行数据。

```
主键索引（聚集索引）：
┌─────────────────────────────────────────────────────────┐
│  B+树                                                   │
│                                                         │
│  [15] ──▶ [15, row_data_1]                            │
│  [20] ──▶ [20, row_data_2]                            │
│  [25] ──▶ [25, row_data_3]                            │
│  [30] ──▶ [30, row_data_4]                            │
│                                                         │
│  叶子节点存储完整行数据（聚簇在一起）                      │
└─────────────────────────────────────────────────────────┘
```

### 辅助索引

辅助索引叶子节点存储主键值，而非完整数据。

```
辅助索引（以name列为例）：
┌─────────────────────────────────────────────────────────┐
│  B+树（name索引）                                      │
│                                                         │
│  [张三] ──▶ [主键=101]                                 │
│  [李四] ──▶ [主键=102]                                 │
│  [王五] ──▶ [主键=103]                                 │
│                                                         │
│  叶子节点存储主键值，查询后需回表                        │
└─────────────────────────────────────────────────────────┘

回表查询：
SELECT * FROM users WHERE name = '李四'
    │
    ▼
1. 辅助索引找到主键=102
2. 用主键回表查询完整数据
3. 返回完整行
```

### 索引结构对比

| 存储引擎 | 主键索引 | 辅助索引 |
|---------|---------|---------|
| InnoDB | 聚集索引，叶子存储行数据 | 非聚集，叶子存储主键 |
| MyISAM | 非聚集，叶子存储行指针 | 非聚集，叶子存储行指针 |

---

## 索引创建与管理

### 创建索引

```sql
-- 创建表时添加索引
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100),
    age INT,
    INDEX idx_name (name),           -- 普通索引
    UNIQUE INDEX idx_email (email),  -- 唯一索引
    INDEX idx_name_age (name, age)   -- 复合索引
);

-- 已有表添加索引
CREATE INDEX idx_name ON users(name);
ALTER TABLE users ADD INDEX idx_age (age);

-- 查看索引
SHOW INDEX FROM users;
```

### 复合索引

```sql
-- 创建复合索引 (name, age, email)
CREATE INDEX idx_composite ON users(name, age, email);

-- 查询分析
EXPLAIN SELECT * FROM users WHERE name = '张三';          -- 使用索引
EXPLAIN SELECT * FROM users WHERE name = '张三' AND age = 25;  -- 使用索引
EXPLAIN SELECT * FROM users WHERE age = 25;               -- 不使用索引
```

### 最左前缀原则

复合索引遵循最左前缀原则：

```
复合索引 (name, age, email) 结构：
┌─────────────────────────────────────────────────────────┐
│  (name='张三', age=20, email='a@x.com')                │
│  (name='张三', age=25, email='b@x.com')                │
│  (name='李四', age=20, email='c@x.com')                │
│                                                         │
│  可命中索引的查询：                                      │
│  ✓ WHERE name = '张三'                                 │
│  ✓ WHERE name = '张三' AND age = 25                    │
│  ✓ WHERE name = '张三' AND age = 25 AND email = 'a@x.com'│
│  ✗ WHERE age = 25                                     │
│  ✗ WHERE email = 'a@x.com'                            │
└─────────────────────────────────────────────────────────┘
```

---

## 索引优化策略

### 选择性

```sql
-- 高选择性字段适合建索引
SELECT COUNT(DISTINCT name) / COUNT(*) AS selectivity FROM users;

-- 值分布均匀的字段不适合建索引
-- 如：性别（只有0/1），状态字段（只有几个值）
```

### 索引覆盖

```sql
-- 覆盖索引：查询的所有字段都在索引中，无需回表
CREATE INDEX idx_name_age ON users(name, age);

-- 覆盖索引查询
EXPLAIN SELECT name, age FROM users WHERE name = '张三';
-- type: ref, Extra: Using index (覆盖索引)

-- 非覆盖索引查询
EXPLAIN SELECT * FROM users WHERE name = '张三';
-- type: ref, Extra: Using index condition (需要回表)
```

### 前缀索引

```sql
-- 对长字符串字段建前缀索引
ALTER TABLE users ADD INDEX idx_email (email(10));

-- 前缀长度选择：区分度接近完整列
SELECT
    COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) as r5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) as r10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) as r15
FROM users;
```

### 索引下推

```sql
-- 索引下推优化：在索引遍历过程中过滤数据
SET optimizer_switch = 'index_condition_pushdown=on';

-- 查询
SELECT * FROM users WHERE name LIKE '张%' AND age = 20;

-- 无索引下推：先按name索引找到主键，再回表过滤age
-- 有索引下推：直接用(name, age)索引过滤后再回表
```

---

## 索引失效场景

### 1. 函数/运算

```sql
-- ✗ 索引失效
SELECT * FROM users WHERE YEAR(create_time) = 2024;
SELECT * FROM users WHERE age + 1 = 20;

-- ✓ 索引有效
SELECT * FROM users WHERE create_time >= '2024-01-01';
SELECT * FROM users WHERE age = 19;
```

### 2. 类型转换

```sql
-- ✗ 索引失效（age是INT，字符串比较会导致全表）
SELECT * FROM users WHERE age = '25';

-- ✓ 索引有效
SELECT * FROM users WHERE age = 25;
```

### 3. LIKE 前缀通配

```sql
-- ✗ 索引失效
SELECT * FROM users WHERE name LIKE '%张%';

-- △ 部分索引失效
SELECT * FROM users WHERE name LIKE '张%';  -- 可用索引
```

### 4. OR 条件

```sql
-- ✗ 索引失效（OR两端不是都建索引）
SELECT * FROM users WHERE name = '张三' OR age = 20;

-- ✓ 索引有效
SELECT * FROM users WHERE name = '张三' OR email = 'a@x.com';
-- (如果email也建了索引)
```

### 5. 范围条件

```sql
-- △ 部分索引失效（范围条件后的字段不用索引）
CREATE INDEX idx_age_name ON users(age, name);
SELECT * FROM users WHERE age > 20 AND name = '张三';
-- 只用age索引，name不走索引
```

---

## SQL执行计划分析

### EXPLAIN字段

```sql
EXPLAIN SELECT * FROM users WHERE name = '张三';

-- 输出字段解析：
-- id: 查询序号
-- select_type: 查询类型（SIMPLE/PRIMARY/SUBQUERY等）
-- table: 表名
-- type: 访问类型（ALL/index/range/ref/eq_ref/const）
-- possible_keys: 可用索引
-- key: 实际使用索引
-- key_len: 索引长度
-- ref: 索引比较的值
-- rows: 预估扫描行数
-- Extra: 额外信息（Using index/Using filesort等）
```

### type 访问类型排序

```
性能从好到差：
const  → eq_ref  → ref  → range  → index  → ALL

const: 主键/唯一索引等值查询，最多一行
eq_ref: 多表关联，主键/唯一索引关联
ref: 非唯一索引等值查询
range: 索引范围查询（>, <, BETWEEN, IN）
index: 全索引扫描
ALL: 全表扫描（最差）
```

---

## 常见面试题

**Q1：为什么InnoDB表建议自增主键？**

> 答：自增主键插入时顺序追加，只在末尾插入，不移动其他数据页，插入效率高。UUID作为主键插入时可能插入到已满的页中，导致页分裂和随机IO。

**Q2：主键索引和辅助索引的区别？**

> 答：主键索引叶子节点存储完整行数据；辅助索引叶子节点存储主键值。查询辅助索引需要先找到主键，再回表查询完整数据。

**Q3：什么是覆盖索引？**

> 答：查询的所有字段都在索引中，索引包含需要的数据，无需回表。如SELECT name FROM users WHERE name='张三'。

**Q4：最左前缀原则是什么？**

> 答：复合索引从左到右使用，查询条件必须从索引最左边开始且不跳过中间的列。索引(name, age, email)可以被name、name+age、name+age+email使用。

**Q5：什么情况索引会失效？**

> 答：使用函数/运算、类型隐式转换、LIKE前导通配符、OR条件、范围条件后的列会导致索引失效或部分失效。

---

## 总结

MySQL索引是查询优化的核心：
- **B+树**：所有数据在叶子节点，范围查询高效
- **聚集索引**：主键索引，叶子存完整数据
- **辅助索引**：叶子存主键，需回表
- **复合索引**：最左前缀原则
- **覆盖索引**：无需回表，查询效率最高
