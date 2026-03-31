# Redis数据结构：五种基本类型的底层实现

## 前言

Redis（Remote Dictionary Server）是一个开源的内存数据结构存储系统，被广泛应用于缓存、消息队列、计数器、分布式锁等场景。Redis的高性能很大程度上源于其高效的底层数据结构设计。本文将深入探讨Redis五种基本数据类型的底层实现，帮助读者理解Redis为何如此高效。

## Redis数据结构全景图

```
Redis Object (redisObject)
    │
    ├── type: 数据类型
    │       ├── STRING
    │       ├── LIST
    │       ├── HASH
    │       ├── SET
    │       └── ZSET
    │
    ├── encoding: 编码方式
    │       ├── int
    │       ├── embstr
    │       ├── raw
    │       ├── quicklist
    │       ├── hashtable
    │       ├── ziplist
    │       ├── intset
    │       └── skiplist
    │
    ├── ptr: 指向实际数据结构的指针
    │
    └── lru: 淘汰策略相关
```

## 1. STRING（字符串）

### 底层实现

Redis的STRING类型有三种编码方式：

| 编码 | 适用场景 | 说明 |
|---|---|---|
| **int** | 整数 | 存储64位有符号整数 |
| **embstr** | 短字符串 | <= 44字节的字符串 |
| **raw** | 长字符串 | > 44字节的字符串 |

### embstr vs raw

```c
// Redis 3.0之前的embstr
// embstr将redisObject和sdshdr连续存储，只分配一次内存

// Redis 3.0之后的embstr
// 保留redisObject和sdshdr分离，但使用简单动态字符串
//embstr上限从32字节调整为44字节（jemalloc内存分配单元）
```

### SDS（Simple Dynamic String）

Redis自定义的字符串结构，不同于C语言原生字符串：

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;    // 已使用长度
    uint8_t alloc;   // 总分配长度（不含header）
    unsigned char flags;  // 标志位
    char buf[];      // 实际字符串
};

// 对比C语言字符串
// C: char str[] = "hello"; // 需要遍历才能知道长度
// SDS: 可以O(1)获取字符串长度
```

### SDS优势

```c
// 1. O(1)获取长度（不遍历）
strlen(key);  // SDS直接返回len字段

// 2. 防止缓冲区溢出
// C: strcat(dest, src); // 可能溢出
// SDS: sdscat(dest, src); // 先检查容量，自动扩容

// 3. 减少内存重分配
// 空间预分配：分配时多分配一些空间
// 惰性释放：不立即释放，保留多余空间

// 4. 二进制安全
// 可以存储任意二进制数据，不受'\0'限制
```

### STRING常用命令

```bash
# 基本操作
SET name "redis"           # 设置值
GET name                    # 获取值
MSET a 1 b 2 c 3           # 批量设置
MGET a b c                  # 批量获取

# 数值操作
SET counter 100
INCR counter               # 原子递增 -> 101
INCRBY counter 5           # 递增指定值 -> 106
DECR counter               # 原子递减 -> 105
DECRBY counter 10          # 递减指定值 -> 95

# 过期时间
SETEX session 3600 "data"  # 设置值并指定过期时间
SETNX lock "unique_id"     # 不存在时设置（分布式锁常用）

# 位操作
SETBIT flags 7 1           # 设置第7位为1
GETBIT flags 7             # 获取第7位的值
BITCOUNT key               # 统计位数为1的数量
```

## 2. LIST（列表）

### 底层实现演进

```
Redis 2.8之前: ziplist（短列表）或 linkedlist（长列表）
Redis 3.2之前: ziplist 或 linkedlist（quicklist雏形）
Redis 3.2开始: quicklist（推荐默认）
```

### quicklist结构

```c
// quicklist = linkedlist of ziplists
// 每个ziplist默认最多存储8192字节
// 或每个ziplist最多包含512个元素

struct quicklist {
    quicklistNode *head;      // 头节点
    quicklistNode *tail;      // 尾节点
    unsigned long count;      // 总元素数
    unsigned long len;        // quicklistNode数量
    int fill : QL_FILL_BITS;  // 每个ziplist最大大小
    int compress : QL_COMP_BITS; // 压缩深度
    // ...
};

struct quicklistNode {
    quicklistNode *prev;
    quicklistNode *next;
    ziplist *zl;              // 指向ziplist的指针
    unsigned int size;        // ziplist字节大小
    unsigned int count : 16;  // 元素数量
    // ...
};
```

### 为什么要用quicklist？

**ziplist的优缺点**：
- 优点：内存连续，节省指针开销，缓存友好
- 缺点：插入/删除操作可能需要整体搬迁

**linkedlist的优缺点**：
- 优点：插入/删除O(1)
- 缺点：内存碎片化，指针开销大

**quicklist的权衡**：
- 多个ziplist用linkedlist串联
- 兼顾内存效率和操作效率

### LIST常用命令

```bash
# 基本操作
LPUSH queue "task1"        # 左侧插入
RPUSH queue "task2"         # 右侧插入
LPOP queue                  # 左侧弹出
RPOP queue                  # 右侧弹出

# 范围操作
LRANGE queue 0 -1           # 获取所有元素
LINDEX queue 0              # 获取指定索引元素
LINSERT queue BEFORE "task2" "task1.5"  # 指定位置插入

# 阻塞操作（消息队列）
BLPOP queue 0               # 阻塞左侧弹出
BRPOP queue 0               # 阻塞右侧弹出

# 队列长度
LLEN queue                  # 获取列表长度
```

### 应用场景

```bash
# 1. 消息队列
LPUSH queue "message1"
LPUSH queue "message2"
BRPOP queue 0  # 阻塞获取，实现FIFO

# 2. 最新消息列表
LPUSH timeline:user:1 "new_post_id"
LTRIM timeline:user:1 0 99  # 只保留最新100条

# 3. 栈结构
LPUSH stack "a" "b" "c"
LPOP stack  # "c" - 后进先出
```

## 3. HASH（哈希）

### 底层实现

| 编码 | 条件 | 说明 |
|---|---|---|
| **ziplist** | field数量 < 512 且 所有field/value < 64字节 | 压缩列表 |
| **hashtable** | 不满足ziplist条件 | 哈希表 |

### ziplist存储HASH

```bash
# ziplist存储方式：[field1, value1, field2, value2, ...]
# 连续的key-value对，内存紧凑

HSET user:1 name "zhangsan" age "25" city "Beijing"
# 在ziplist中：[name, zhangsan, age, 25, city, Beijing]
```

### hashtable存储HASH

```c
struct dict {
    dictht ht[2];          // 两个哈希表（渐进式rehash）
    long rehashidx;        // rehash进度，-1表示未开始
    // ...
};

struct dictht {
    dictEntry **table;     // 哈希表数组
    unsigned long size;    // 哈希表大小
    unsigned long sizemask;// 掩码，用于计算索引
    unsigned long used;    // 已使用的entry数量
};

struct dictEntry {
    void *key;             // 键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    dictEntry *next;      // 指向下一个entry（链地址法）
};
```

### 渐进式rehash

```c
// Redis使用渐进式rehash，避免一次性rehash阻塞

// 步骤：
// 1. 为ht[1]分配空间
// 2. rehashidx记录进度
// 3. 每次增删改查时，额外迁移一个桶
// 4. 迁移完成后，交换ht[0]和ht[1]

// 好处：分摊计算量，避免阻塞
```

### HASH常用命令

```bash
# 基本操作
HSET user:1 name "zhangsan" age "25"
HGET user:1 name                    # 获取单个field
HMGET user:1 name age               # 批量获取field
HGETALL user:1                      # 获取所有field-value

# 计数操作
HINCRBY user:1 age 1               # 原子递增
HINCRBYFLOAT user:1 height 0.5      # 浮点递增

# 其他操作
HEXISTS user:1 name                 # 判断field是否存在
HDEL user:1 city                    # 删除field
HLEN user:1                         # field数量
HKEYS user:1                        # 获取所有field
HVALS user:1                        # 获取所有value
```

### 应用场景

```bash
# 1. 对象存储
HSET user:10001 username "zhangsan" email "zhangsan@example.com" age "28"
HGETALL user:10001

# 2. 购物车
HSET cart:user:10001 product:20001 2 product:20002 1  # 商品ID -> 数量
HINCRBY cart:user:10001 product:20001 1  # 增加数量
HGET cart:user:10001 product:20001       # 查询数量

# 3. 缓存用户会话
HSET session:abc123 userId 10001 token "xxx" expireTime "1709308800"
```

## 4. SET（集合）

### 底层实现

| 编码 | 条件 | 说明 |
|---|---|---|
| **intset** | 所有元素都是整数 且 元素数量 < 512 | 整数集合 |
| **hashtable** | 不满足intset条件 | 哈希表 |

### intset结构

```c
struct intset {
    uint32_t encoding;      // 编码方式：INTSET_ENC_INT16/32/64
    uint32_t length;        // 元素数量
    int8_t contents[];      // 元素数组（从小到大排序）
};

// 升级机制：当添加大整数时，扩展数组类型
// 不支持降级，节省内存
```

### intset vs hashtable

```bash
# intset示例：所有元素都是整数
SADD numbers 1 2 3 4 5
# encoding会根据最大值动态调整

# hashtable示例：包含字符串
SADD fruits "apple" "banana" "orange"
# 自动转换为hashtable编码
```

### SET常用命令

```bash
# 基本操作
SADD tags "java" "redis" "mysql"     # 添加元素
SREM tags "mysql"                      # 删除元素
SMEMBERS tags                          # 获取所有元素
SISMEMBER tags "java"                  # 判断是否存在

# 集合运算
SADD set1 1 2 3 4
SADD set2 3 4 5 6
SDIFF set1 set2                       # 差集：1,2
SINTER set1 set2                      # 交集：3,4
SUNION set1 set2                      # 并集：1,2,3,4,5,6

# 随机操作
SRANDMEMBER tags 2                    # 随机获取2个元素
SPOP tags 1                           # 随机弹出1个元素

# 计数
SCARD tags                            # 获取集合基数
```

### 应用场景

```bash
# 1. 标签系统
SADD tag:java article:1001 article:1002 article:1003
SADD tag:redis article:1001 article:1004
# 查询同时打上java和redis标签的文章
SINTER tag:java tag:redis

# 2. 关注关系
SADD following:user:1 user:2 user:3 user:4
SADD followers:user:2 user:1
# 检查user:1是否关注了user:2
SISMEMBER following:user:1 user:2

# 3. 抽奖系统
SADD lottery:2024 participants user:1 user:2 user:3 ... user:1000
# 抽取10个获奖者
SRANDMEMBER lottery:2024 10
```

## 5. ZSET（有序集合）

### 底层实现

| 编码 | 条件 | 说明 |
|---|---|---|
| **ziplist** | 元素数量 < 128 或 所有member < 64字节 | 压缩列表 |
| **skiplist + hashtable** | 不满足ziplist条件 | 跳表 + 哈希表 |

### 为什么ZSET同时使用skiplist和hashtable？

```c
// ZSET同时维护两个数据结构：
// 1. skiplist：按score排序，支持范围查询
// 2. hashtable：按member查找，O(1)查找

struct zset {
    dict *dict;              // member -> score 映射
    zskiplist *zsl;         // 跳表，按score排序
};

// 空间换时间的设计
// 两个数据结构共享数据，不额外占用内存
```

### 跳表（Skip List）结构

```c
struct zskiplistNode {
    sds ele;                 // 元素值
    double score;           // 分数
    zskiplistNode *backward;// 前驱指针
    zskiplistLevel levels[]; // 多层前向指针
};

struct zskiplistLevel {
    zskiplistNode *forward; // 当前层前向指针
    unsigned int span;      // 到下一个节点的跨度
};

struct zskiplist {
    zskiplistNode *header;  // 头节点（不存储数据）
    zskiplistNode *tail;    // 尾节点
    unsigned long length;   // 节点数量
    int level;              // 当前最大层数
};

// 跳表层数：随机1-64，概率递减
// 平均每2个节点有一层，每4个节点有2层，以此类推
// 时间复杂度：O(log n)查找/插入/删除
```

### 跳表查找过程

```
Level 3: 1 --------> 9 --------> 23 --------> NULL
           |          |          |
Level 2: 1 -> 5 -> 9 -> 15 -> 23 -> 30 -> NULL
           |    |    |    |    |    |
Level 1: 1->3->5->7->9->11->15->17->23->25->30-> NULL

查找17:
1. 从Level 3开始：1 -> 9 -> 23（17 < 23，下降到Level 2）
2. Level 2：9 -> 15（17 > 15，继续前进 -> 23，下降到Level 1）
3. Level 1：15 -> 17（找到！）
```

### ZSET常用命令

```bash
# 基本操作
ZADD leaderboard 100 "user1" 200 "user2" 150 "user3"  # 添加元素
ZSCORE leaderboard "user1"                # 获取分数
ZRANK leaderboard "user1"                  # 获取排名（从小到大）
ZREVRANK leaderboard "user1"               # 获取排名（从大到小）

# 范围查询
ZRANGE leaderboard 0 9 WITHSCORES          # 获取前10名
ZREVRANGE leaderboard 0 9 WITHSCORES      # 获取后10名
ZRANGEBYSCORE leaderboard 100 200         # 获取分数100-200的成员
ZREVRANGEBYSCORE leaderboard 200 100       # 逆序

# 计数
ZCARD leaderboard                          # 获取成员数量
ZCOUNT leaderboard 100 200                 # 统计分数区间的成员数

# 更新分数
ZINCRBY leaderboard 50 "user1"             # 增加50分

# 删除
ZREM leaderboard "user3"                   # 删除成员
ZREMRANGEBYRANK leaderboard 0 9           # 删除排名0-9的成员
ZREMRANGEBYSCORE leaderboard 0 100        # 删除分数0-100的成员
```

### 应用场景

```bash
# 1. 排行榜
ZADD game:leaderboard 10000 "player1" 9500 "player2" 8800 "player3"
# player1上升10名
ZINCRBY game:leaderboard 500 "player1"
# 获取前10名
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# 2. 延迟队列（分数为执行时间戳）
ZADD delay:queue 1709308800 "task1" 1709308900 "task2"
# 获取已到期的任务
ZRANGEBYSCORE delay:queue 0 1709308800
ZREMRANGEBYSCORE delay:queue 0 1709308800

# 3. 时间线排序
ZADD timeline:user:1 1709308800 "post1" 1709308900 "post2"
ZREVRANGE timeline:user:1 0 9 WITHSCORES
```

## 数据类型转换

### 触发条件

```
STRING:
  - int: 值是整数，且在long范围内
  - embstr: 字符串长度 <= 44字节
  - raw: 字符串长度 > 44字节

LIST:
  - ziplist: 元素数量 < 512 且 所有元素 < 64字节
  - quicklist: 不满足ziplist条件

HASH:
  - ziplist: field数量 < 512 且 所有field/value < 64字节
  - hashtable: 不满足ziplist条件

SET:
  - intset: 所有元素都是整数 且 元素数量 < 512
  - hashtable: 不满足intset条件

ZSET:
  - ziplist: 元素数量 < 128 且 所有member < 64字节
  - skiplist: 不满足ziplist条件
```

### 自动转换示例

```bash
# HASH: ziplist -> hashtable
HSET user:1 f1 v1 f2 v2 ... (当field数量 >= 512时转换)

# SET: intset -> hashtable
SADD numbers 1 2 3 ... (当元素数量 >= 512时转换)
SADD numbers "string" (当添加非整数时立即转换)
```

## 常见面试题

**Q1：Redis的STRING类型最大能存储多少数据？**

> 参考答案：STRING类型最大能存储512MB数据。但从实际使用角度，不建议存储过大的值，因为会影响网络传输效率和内存利用率。

**Q2：为什么ZSET同时使用跳表和哈希表？**

> 参考答案：ZSET需要支持两种操作：1）按score排序的范围查询，使用跳表可以实现O(log n)的查找和范围操作；2）按member直接查找，使用哈希表可以实现O(1)的查找。同时使用两个数据结构可以兼顾两种操作效率，两个结构共享数据，不额外占用内存。

**Q3：Redis的跳表为什么不用平衡树？**

> 参考答案：跳表和平衡树（如红黑树）都能实现O(log n)的查找效率。跳表的优势在于：1）实现更简单，不需要复杂的旋转操作；2）范围查询更方便，只需找到起点然后顺序遍历；3）插入/删除不需要像平衡树那样调整树结构，只需修改相邻节点的指针；4）跳表更容易实现并发安全（分段锁）。

**Q4：Redis的ziplist有什么优缺点？**

> 参考答案：ziplist的优点是内存紧凑，连续存储，缓存友好，适合存储少量数据。缺点是：1）插入/删除操作可能需要整体搬迁数据；2）查找操作需要O(n)遍历；3）不适合存储大量数据。因此Redis对ziplist设置了触发转换的条件。

**Q5：SDS相比C语言字符串有什么优势？**

> 参考答案：SDS的优势包括：1）O(1)获取长度，而C字符串需要遍历；2）防止缓冲区溢出，SDS会先检查容量；3）减少内存重分配次数，支持空间预分配和惰性释放；4）二进制安全，不受'\0'限制。

## 总结

Redis的五种基本数据类型各有特点：

| 类型 | 底层结构 | 特点 | 典型场景 |
|---|---|---|---|
| STRING | int/embstr/raw | 简单通用 | 缓存、计数器、分布式锁 |
| LIST | quicklist | 有序、FIFO | 消息队列、最新列表 |
| HASH | ziplist/hashtable | field-value映射 | 对象存储、购物车 |
| SET | intset/hashtable | 无序、唯一 | 标签、关注、抽奖 |
| ZSET | ziplist/skiplist+dict | 有序、唯一 | 排行榜、延迟队列 |

理解这些底层实现对于更好地使用Redis和排查问题至关重要。
