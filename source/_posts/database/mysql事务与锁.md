---
title: MySQL事务与锁：MVCC、行锁、间隙锁详解
date: 2023-06-07 21:38:43
tags:
  - Java进阶
  - 数据库
categories: 学习
keywords: MySQL,事务,MVCC,行锁,间隙锁,事务隔离级别
description: 深入理解MySQL事务机制，MVCC原理与各种锁的实现
cover:
---

## 前言

事务和锁是数据库并发控制的核心机制。理解它们的原理，才能写出既高效又安全的多线程数据库操作代码。本文深入解析MySQL的事务与锁机制。

## 事务特性

### ACID特性

| 特性 | 说明 |
|------|------|
| Atomic（原子性） | 事务是最小执行单元，不可分割 |
| Consistency（一致性） | 事务执行前后，数据保持一致状态 |
| Isolation（隔离性） | 并发事务之间相互隔离 |
| Durability（持久性） | 事务提交后，结果永久保存 |

### 事务状态

```
┌─────────────────────────────────────────────────────────┐
│                    事务状态转换                             │
│                                                         │
│   开始 ──▶ 活动状态 ──▶ 部分提交 ──▶ 提交 ──▶ 终止      │
│              │                  │                        │
│              │                  │                        │
│              └──────────────────┴────────────────────▶  │
│                              │  回滚                            │
└─────────────────────────────────────────────────────────┘
```

---

## 事务隔离级别

### 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ_UNCOMMITTED | 可能 | 可能 | 可能 |
| READ_COMMITTED | 不可能 | 可能 | 可能 |
| REPEATABLE_READ | 不可能 | 不可能 | 可能 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 |

### MySQL设置隔离级别

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置会话级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局级别
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 开启事务
START TRANSACTION;

-- 提交
COMMIT;

-- 回滚
ROLLBACK;
```

---

## MVCC原理

### 版本链

每行数据都有隐藏的两列：
- `DB_TRX_ID`：最近修改的事务ID
- `DB_ROLL_PTR`：指向undo log的指针

```
┌──────────────────────────────────────────────────────────────┐
│                     版本链（多版本并发控制）                      │
│                                                               │
│  事务A修改了 age=25                                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ id=1, name='张三', age=25, trx_id=5, roll_ptr=地址1 │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ▲                                │
│                            │ roll_ptr                       │
│                            │                                │
│  事务B修改了 age=30       │                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ id=1, name='张三', age=25, trx_id=4, roll_ptr=地址2  │  │
│  └──────────────────────────────────────────────────────┘  │
│                            ▲                                │
│                            │ roll_ptr                       │
│                            │                                │
│  原始数据                                                               │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ id=1, name='张三', age=20, trx_id=3, roll_ptr=null  │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### ReadView

每个读事务开始时生成ReadView，包含：
- `m_ids`：活跃事务ID列表
- `min_trx_id`：最小活跃事务ID
- `max_trx_id`：创建ReadView时最大事务ID
- `creator_trx_id`：当前事务ID

### 快照读 vs 当前读

```sql
-- 快照读：读取历史版本，不加锁（MVCC）
SELECT * FROM users;  -- 普通SELECT

-- 当前读：读取最新数据，加锁
SELECT * FROM users FOR UPDATE;  -- 加排他锁
SELECT * FROM users LOCK IN SHARE MODE;  -- 加共享锁

-- 写操作（自动当前读）
UPDATE users SET age = 30 WHERE id = 1;  -- 排他锁
```

### 不同隔离级别的ReadView

```
READ COMMITTED（每次读取都生成新ReadView）：
┌──────────────────────────────────────────────────────────────┐
│  事务A（id=5）                                              │
│  SELECT时生成ReadView：m_ids=[3,4], min=3, max=6          │
│  遍历版本链：trx_id=5<6，但trx_id=5在m_ids中，不可见        │
│  继续向下找：trx_id=4不在m_ids中，可见                       │
└──────────────────────────────────────────────────────────────┘

REPEATABLE READ（事务开始时生成ReadView，一直用同一个）：
┌──────────────────────────────────────────────────────────────┐
│  事务A（id=5）                                              │
│  事务开始时生成ReadView：m_ids=[3,4], min=3, max=6        │
│  后续SELECT都用这个ReadView                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 锁类型

### 锁的分类

```
┌──────────────────────────────────────────────────────────────┐
│                         MySQL锁分类                            │
│                                                               │
│   ┌────────────────────────────────────────────────────┐   │
│   │                     锁                              │   │
│   │  ┌──────────────┐  ┌──────────────┐             │   │
│   │  │  共享锁(S)    │  │  排他锁(X)   │             │   │
│   │  └──────────────┘  └──────────────┘             │   │
│   │                                                    │   │
│   │  ┌──────────────┐  ┌──────────────┐  ┌────────┐│   │
│   │  │  记录锁      │  │  间隙锁      │  │ 临键锁 ││   │
│   │  │  Record Lock │  │  Gap Lock   │  │Next-Key││   │
│   │  └──────────────┘  └──────────────┘  └────────┘│   │
│   │                                                    │   │
│   │  ┌──────────────┐  ┌──────────────┐             │   │
│   │  │  表锁        │  │ 意向锁       │             │   │
│   │  └──────────────┘  └──────────────┘             │   │
│   └────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### 共享锁与排他锁

```sql
-- 共享锁（S锁）：读取数据，多个事务可以同时持有
SELECT * FROM users WHERE id=1 LOCK IN SHARE MODE;
-- 其他事务也可以加共享锁读取，但不能修改

-- 排他锁（X锁）：修改数据，排他锁不能与其他锁共存
SELECT * FROM users WHERE id=1 FOR UPDATE;
-- 其他事务不能加任何锁，必须等待释放
```

### 记录锁（Record Lock）

锁定索引记录，非主键索引也会有记录锁。

```sql
-- 对id=5的记录加锁
SELECT * FROM users WHERE id=5 FOR UPDATE;
-- 锁定id=5这条记录
```

### 间隙锁（Gap Lock）

锁定索引记录之间的间隙，防止插入。

```sql
-- 锁定 id > 5 的间隙
SELECT * FROM users WHERE id > 5 AND id < 10 FOR UPDATE;
-- 锁定 (5, 10) 之间的间隙，其他事务无法插入 id 在 (5, 10) 范围内的记录
```

### 临键锁（Next-Key Lock）

记录锁 + 间隙锁，锁定一个范围。

```sql
-- 对 id=5 加锁
SELECT * FROM users WHERE id = 5 FOR UPDATE;
-- 锁定 [5, 下一个键值) 范围
-- 既锁定 id=5 记录，也锁定 (上一个键值, 5] 间隙
```

---

## 事务隔离级别与锁

### READ UNCOMMITTED

```sql
-- 无隔离，最不安全
-- 事务可以读取其他事务未提交的修改
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

START TRANSACTION;
-- 事务B修改但未提交
UPDATE users SET age = 30 WHERE id = 1;
-- 事务A可以读到 age=30（脏读）
SELECT * FROM users WHERE id = 1;  -- age=30
```

### READ COMMITTED

```sql
-- 每次读取生成新ReadView，解决脏读
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

START TRANSACTION;
-- 事务B修改并提交
UPDATE users SET age = 30 WHERE id = 1;
COMMIT;
-- 事务A读取
SELECT * FROM users WHERE id = 1;  -- age=30，可重复读
```

### REPEATABLE READ（MySQL默认）

```sql
-- 事务开始时生成ReadView，解决不可重复读
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

START TRANSACTION;
-- 事务B修改并提交
UPDATE users SET age = 30 WHERE id = 1;
COMMIT;
-- 事务A读取（使用事务开始时的ReadView）
SELECT * FROM users WHERE id = 1;  -- age=20，旧数据

-- 但REPEATABLE READ下，范围查询会有幻读
SELECT * FROM users WHERE id > 10;  -- 第一次查询
-- 事务B插入新记录 id=15 并提交
INSERT INTO users VALUES(15, '王五', 25);
COMMIT;
SELECT * FROM users WHERE id > 10;  -- 第二次查询，多了id=15（幻读）
```

### SERIALIZABLE

```sql
-- 完全串行化，最安全但性能最差
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

START TRANSACTION;
-- 自动将普通SELECT转为 SELECT ... LOCK IN SHARE MODE
SELECT * FROM users WHERE id > 10;  -- 临键锁，锁定 id>10 的范围
```

---

## 死锁与避免

### 死锁示例

```
事务A：                                    事务B：
BEGIN;                                    BEGIN;
UPDATE users SET age=20 WHERE id=1;  -- 锁定id=1
                                          UPDATE users SET age=30 WHERE id=2;  -- 锁定id=2
UPDATE users SET age=25 WHERE id=2;  -- 等待id=2锁释放
                                          UPDATE users SET age=35 WHERE id=1;  -- 等待id=1锁释放
                                          -- 死锁：互相等待对方释放锁
```

### 避免死锁

```sql
-- 1. 固定顺序访问资源
UPDATE users SET age=20 WHERE id=1;
UPDATE users SET age=30 WHERE id=2;
-- 总是先更新id=1，再更新id=2

-- 2. 减少锁的持有时间
-- 尽早提交事务
-- 避免在事务中做过多操作

-- 3. 使用低隔离级别
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 4. 检测死锁
SHOW ENGINE INNODB STATUS;
-- MySQL会自动检测死锁，回滚代价最小的事务
```

---

## 乐观锁与悲观锁

### 悲观锁

```sql
-- 假设并发冲突，每次访问都加锁
SELECT * FROM users WHERE id=1 FOR UPDATE;  -- 排他锁

-- Java实现
@Transactional
public void updateUser(Long id, User newUser) {
    User user = userRepository.findById(id).orElseThrow();
    user.setAge(newUser.getAge());
    userRepository.save(user);
}
```

### 乐观锁

```sql
-- 假设并发冲突少，读取时不加锁，更新时检查版本
-- 表中需要有 version 字段

-- 更新时检查版本
UPDATE users SET age=30, version=version+1
WHERE id=1 AND version=5;  -- 版本匹配才更新

-- Java实现
@Version
private Long version;
```

---

## MVCC与ReadView代码模拟

```java
public class MVCCSimulation {

    // 模拟ReadView
    static class ReadView {
        List<Long> activeTransactions;  // 活跃事务ID列表
        long minTransactionId;           // 最小活跃事务ID
        long maxTransactionId;           // 最大事务ID
        long currentTransactionId;       // 当前事务ID

        public boolean isVisible(long trxId) {
            // 1. 创建者事务
            if (trxId == currentTransactionId) {
                return true;
            }
            // 2. 事务ID小于最小活跃事务（在ReadView生成前已提交）
            if (trxId < minTransactionId) {
                return true;
            }
            // 3. 事务ID在活跃列表中（未提交）
            if (activeTransactions.contains(trxId)) {
                return false;
            }
            // 4. 其他情况不可见
            return false;
        }
    }

    // 模拟undo log版本链查找
    public Object getVersion(Object key, ReadView readView) {
        // 从最新版本开始遍历
        Version current = getLatestVersion(key);

        while (current != null) {
            if (readView.isVisible(current.trxId)) {
                return current.data;  // 找到可见版本
            }
            current = current.previousVersion;  // 继续向下找
        }
        return null;
    }
}
```

---

## 常见面试题

**Q1：MVCC原理？**

> 答：多版本并发控制，通过版本链和ReadView实现。每个事务读取时生成ReadView，通过版本链遍历找到可见的数据版本。READ COMMITTED每次读取生成新ReadView，REPEATABLE READ事务开始时生成ReadView。

**Q2：临键锁、间隙锁、记录锁的区别？**

> 答：记录锁锁定索引记录；间隙锁锁定记录之间的间隙；临键锁是记录锁+间隙锁的组合，锁定一个范围。

**Q3：脏读、不可重复读、幻读的区别？**

> 答：脏读读到其他事务未提交的数据；不可重复读同一事务两次读取结果不同；幻读同一事务两次查询结果集不同。

**Q4：MySQL如何解决幻读？**

> 答：REPEATABLE READ隔离级别下，使用临键锁锁定查询范围，防止其他事务插入新记录。但可能产生间隙锁影响并发性能。

**Q5：乐观锁和悲观锁的使用场景？**

> 答：悲观锁适合高并发写操作频繁的场景。乐观锁适合读多写少、冲突不多的场景，通过版本号或CAS实现。

---

## 总结

MySQL事务与锁是并发控制的核心：
- **ACID**：原子性、一致性、隔离性、持久性
- **MVCC**：多版本并发控制，通过版本链和ReadView实现
- **ReadView**：快照读的关键，决定哪个版本可见
- **锁类型**：共享锁、排他锁、记录锁、间隙锁、临键锁
- **隔离级别**：READ UNCOMMITTED/READ COMMITTED/REPEATABLE READ/SERIALIZABLE
