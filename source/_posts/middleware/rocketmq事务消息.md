# RocketMQ事务消息：原理、实战与常见问题

## 前言

分布式事务一直是微服务架构中的难题。在电商系统中，订单服务创建订单、库存服务扣减库存、积分服务增加积分，这些操作需要保证原子性。RocketMQ的事务消息为这类场景提供了解决方案。本文将深入剖析RocketMQ事务消息的原理、使用方式以及常见问题。

## 分布式事务问题

### 本地事务的问题

```java
// 简单的本地事务可以保证一致性
@Transactional
public void createOrder(Order order) {
    orderMapper.insert(order);        // 订单入库
    stockMapper.decreaseStock(order.getProductId(), order.getCount()); // 扣库存
    // 两步在同一个事务中，要么都成功，要么都回滚
}

// 但如果库存服务是独立的微服务呢？
// order-service 和 stock-service 是两个独立的应用
// 各自有自己的数据库
// 本地事务无法跨服务保证原子性！
```

### 分布式事务解决方案对比

```
┌─────────────────────────────────────────────────────────────────┐
│                   分布式事务解决方案对比                          │
├───────────────┬─────────────────────────────────────────────────┤
│  2PC/3PC      │ 强一致，但性能差，不适合高并发                     │
│  TCC          │ 性能好，但业务侵入性大，需要改造                  │
│  本地消息表    │ 简单易实现，但需要额外部署                       │
│  消息事务      │ RocketMQ特有，业务侵入性小                      │
│  Saga         │ 适合长事务，各参与者正向操作                     │
└───────────────┴─────────────────────────────────────────────────┘
```

## RocketMQ事务消息原理

### 执行流程

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  Producer   │       │   Broker    │       │  Consumer   │
└─────────────┘       └─────────────┘       └─────────────┘
       │                    │                    │
       │  1.发送半消息        │                    │
       │------------------->│                    │
       │  2.半消息成功        │                    │
       │<-------------------|                    │
       │                    │                    │
       │  3.执行本地事务      │                    │
       │       │             │                    │
       │       ▼             │                    │
       │  ┌─────────┐        │                    │
       │  │数据库    │        │                    │
       │  │本地事务  │        │                    │
       │  └─────────┘        │                    │
       │       │             │                    │
       │  4.提交/回滚          │                    │
       │------------------->│                    │
       │                    │                    │
       │  5.发送commit/rollback│                    │
       │<-------------------|                    │
       │                    │                    │
       │  6.消息投递          │                    │
       │                    │------------------->│
```

### 半消息（Half Message）

```java
// 半消息特点：
// 1. 消费者看不到这条消息
// 2. 不会被消费
// 3. 等到事务提交后才可见

// RocketMQ实现：
// - 半消息存储在RMQ_SYS_TRANS_HALF_TOPIC
// - 原topic为 %transaction%，不影响正常消费
// - 事务提交后，消息才会投递到真正的topic
```

### 事务状态

```java
// 事务状态枚举
public enum TransactionStatus {
    COMMIT_MESSAGE,    // 提交事务，消息可被消费
    ROLLBACK_MESSAGE, // 回滚事务，消息丢弃
    UNKNOWN           // 未知状态，需要Broker回查
}
```

## 代码实现

### 1. RocketMQ依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.3</version>
</dependency>
```

### 2. 生产者配置

```java
@Configuration
public class RocketMQConfig {

    @Bean
    public TransactionMQProducer transactionMQProducer() {
        TransactionMQProducer producer = new TransactionMQProducer();
        producer.setNamesrvAddr("localhost:9876");
        producer.setProducerGroup("order-producer-group");

        // 事务回查线程池
        ExecutorService executor = new ThreadPoolExecutor(
            2, 5, 60,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(100),
            new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r);
                    t.setName("transaction-msg-check-thread");
                    return t;
                }
            }
        );
        producer.setExecutorService(executor);

        // 事务回查监听器
        producer.setTransactionListener(new OrderTransactionListener());

        return producer;
    }
}
```

### 3. 事务监听器

```java
@Service
public class OrderTransactionListener implements TransactionListener {

    // 存储事务上下文（本地事务执行前调用）
    // 用于后续回查时确定事务状态
    private ConcurrentHashMap<String, Boolean> localTransactionStates =
        new ConcurrentHashMap<>();

    /**
     * 执行本地事务
     * @param msgHalf  半消息
     * @param arg      业务参数
     * @return 事务状态
     */
    @Override
    public LocalTransactionState executeLocalTransaction(
            Message msgHalf, Object arg) {

        String orderId = (String) arg;
        try {
            // 1. 创建订单
            Order order = JSON.parseObject(
                new String(msgHalf.getBody()), Order.class);
            orderMapper.insert(order);

            // 2. 扣减库存（本地事务）
            stockMapper.decreaseStock(
                order.getProductId(), order.getCount());

            // 3. 记录事务状态为成功
            localTransactionStates.put(msgHalf.getTransactionId(), true);

            // 4. 返回COMMIT
            return LocalTransactionState.COMMIT_MESSAGE;

        } catch (Exception e) {
            log.error("订单创建失败: " + orderId, e);
            localTransactionStates.put(msgHalf.getTransactionId(), false);
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    /**
     * 回查事务状态（Broker调用）
     * @param msgExt  消息扩展
     * @return 事务状态
     */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msgExt) {
        String transactionId = msgExt.getTransactionId();

        // 根据transactionId查询本地事务状态
        Boolean success = localTransactionStates.get(transactionId);

        if (success != null) {
            return success
                ? LocalTransactionState.COMMIT_MESSAGE
                : LocalTransactionState.ROLLBACK_MESSAGE;
        }

        // 查不到？可能是机器重启，需要重新查库
        String orderId = new String(msgExt.getKeys());
        Order order = orderMapper.selectByOrderId(orderId);

        if (order != null) {
            // 订单存在，说明本地事务成功了
            return LocalTransactionState.COMMIT_MESSAGE;
        } else {
            // 订单不存在，可能是失败了或者还没执行
            return LocalTransactionState.UNKNOWN;
        }
    }
}
```

### 4. 发送事务消息

```java
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private TransactionMQProducer transactionMQProducer;

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private StockMapper stockMapper;

    @Override
    public void createOrder(OrderDTO orderDTO) {
        String orderId = UUID.randomUUID().toString().replace("-", "");

        // 构建订单消息
        Order order = new Order();
        order.setOrderId(orderId);
        order.setUserId(orderDTO.getUserId());
        order.setProductId(orderDTO.getProductId());
        order.setCount(orderDTO.getCount());
        order.setAmount(orderDTO.getAmount());
        order.setStatus("PENDING");

        try {
            // 发送事务消息
            // arg参数会传递给executeLocalTransaction
            Message message = new Message(
                "order-topic",           // Topic
                "create-order",           // Tag
                orderId,                 // Key
                JSON.toJSONString(order).getBytes(StandardCharsets.UTF_8)
            );

            transactionMQProducer.sendMessageInTransaction(
                message,                  // 消息
                orderId                   // arg：订单ID
            );

        } catch (MQClientException e) {
            throw new BusinessException("发送订单消息失败", e);
        }
    }
}
```

### 5. 消费者实现

```java
@Component
@Slf4j
public class OrderConsumer {

    @Autowired
    private OrderService orderService;

    @RocketMQMessageListener(
        topic = "order-topic",
        consumerGroup = "order-consumer-group",
        tag = "create-order"
    )
    public void onMessage(Message message, ConsumeContext context) {
        try {
            String body = new String(message.getBody(), StandardCharsets.UTF_8);
            Order order = JSON.parseObject(body, Order.class);

            log.info("收到订单消息: orderId={}", order.getOrderId());

            // 业务处理
            orderService.processOrder(order);

            log.info("订单处理成功: orderId={}", order.getOrderId());

        } catch (Exception e) {
            log.error("处理订单失败", e);
            // 返回RECONSUME_LATER，RocketMQ会重试
            throw new RuntimeException("处理订单失败", e);
        }
    }
}
```

## 事务消息底层原理

### 两阶段提交

```
┌─────────────────────────────────────────────────────────────────┐
│                    两阶段提交                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  第一阶段：预发送                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Producer ──> Broker: 发送半消息                          │   │
│  │ Producer <-- Broker: 半消息发送成功                       │   │
│  │ Producer ──> 本地数据库: 执行事务                         │   │
│  │ Producer ──> Broker: 提交/回滚事务消息                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  第二阶段：确认                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Broker ──> 消息队列: 投递消息（如果提交）                  │   │
│  │ Consumer ──> 消息队列: 消费消息                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Broker端处理流程

```java
// Broker端处理事务消息的核心逻辑

// 1. 存储半消息
public PutMessageResult putHalfMessage(MessageExtBrokerInner msgInner) {
    // 消息存储到系统topic: RMQ_SYS_TRANS_HALF_TOPIC
    // 设置消息的物理位置为-transaction- prefix
    return this.commitLog.putHalfMessage(msgInner);
}

// 2. 处理提交/回滚
public void processCommitTransaction(...) {
    if (commitOrRollback) {
        // COMMIT: 将消息从half topic转移到真正的topic
        this.commitLog.putCommitLogOffset(msg);
    } else {
        // ROLLBACK: 标记消息为已回滚，不再处理
    }
}

// 3. 回查未决事务
public void checkTransactionState(...) {
    // 定时扫描half topic中未决的消息
    // 调用Producer的checkLocalTransaction
    // 根据返回状态决定提交还是回滚
}
```

### 回查机制

```java
// Broker端回查配置
// broker.conf
transactionTimeout=60000        // 事务超时时间，默认6秒
transactionCheckInterval=10000  // 回查间隔，默认10秒
transactionCheckMax=5           // 最大回查次数

// 定时任务扫描未决消息
// 如果消息超时未确认，Broker会主动回查Producer
// Producer需要实现TransactionListener的checkLocalTransaction方法
```

## 使用注意事项

### 1. 消息超时

```java
// 如果本地事务执行时间过长，Broker会回查
// 需要合理设置超时时间

// Broker配置
transactionTimeout=60000  // 60秒超时

// Producer端
// 如果长时间未收到响应，RocketMQ会自动回查
```

### 2. 幂等性处理

```java
// 事务消息可能被重复投递
// 消费者必须保证幂等

@Component
public class OrderConsumer {

    @Autowired
    private OrderMapper orderMapper;

    public void processOrder(Order order) {
        // 幂等处理：检查订单是否已处理
        Order existing = orderMapper.selectByOrderId(order.getOrderId());
        if (existing != null) {
            log.info("订单已处理，跳过: orderId={}", order.getOrderId());
            return;
        }

        // 正常处理订单
        // ...
    }
}
```

### 3. 消息顺序

```java
// RocketMQ保证同一个Queue中的消息有序
// 但事务消息可能打乱顺序

// 场景：
// 1. 事务消息A（提交）
// 2. 事务消息B（回滚）
// 3. 事务消息B先被处理（消费失败）

// 建议：
// 1. 事务消息不要依赖顺序
// 2. 或者使用顺序消息单独处理
```

### 4. 消息丢失风险

```java
// 事务消息可能的丢失场景：

// 场景1：Broker宕机
// 解决：配置副本
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
listenPort=10911
storePathRootDir=/data/rocketmq/store
haListenPort=10912
brokerRole=SYNC_MASTER

// 场景2：回查超时
// 解决：合理设置transactionTimeout

// 场景3：Producer重启
// 解决：实现checkLocalTransaction查库
```

## 与Kafka事务对比

```
┌─────────────────────────────────────────────────────────────────┐
│              RocketMQ事务 vs Kafka事务                          │
├─────────────────────┬───────────────────────────────────────────┤
│                     │  RocketMQ         │  Kafka                 │
├─────────────────────┼───────────────────┼──────────────────────┤
│ 实现方式            │ 两阶段提交+回查     │ 肘原性producer         │
│ 适用场景            │ 本地事务+消息      │ 仅Kafka内部             │
│ 回查机制            │ Broker主动回查     │ 无，需要自己实现        │
│ 业务侵入性          │ 较小              │ 较大                    │
│ 社区活跃度          │ 一般              │ 活跃                    │
└─────────────────────┴───────────────────┴──────────────────────┘

RocketMQ事务消息更适合：
- 需要跨数据库的分布式事务
- 本地事务和消息需要保证一致性
- 需要Broker主动回查的场景
```

## 常见面试题

**Q1：RocketMQ事务消息的执行流程是什么？**

> 参考答案：1）Producer发送半消息到Broker，消息此时不可见；2）Broker返回半消息发送成功；3）Producer执行本地数据库事务；4）根据事务结果，发送commit或rollback到Broker；5）如果是commit，Broker将消息投递到真正的Topic；6）Consumer消费消息。如果Commit/Rollback超时或Producer重启，Broker会回查Producer获取事务状态。

**Q2：事务消息的回查机制是什么？**

> 参考答案：回查是Broker主动发起的。当Producer发送半消息后，如果长时间没有收到commit/rollback确认，或者收到了UNKNOWN状态，Broker会调用Producer注册的TransactionListener.checkLocalTransaction()方法，查询本地事务的状态。Producer需要根据transactionId或者消息的key查库，确定事务是成功还是失败，然后返回对应的状态。

**Q3：如何保证事务消息不丢失？**

> 参考答案：1）Broker配置SYNC_MASTER+SYNC_FLUSH，保证消息刷盘；2）配置副本数replicationFactor>=2；3）Producer端开启事务，并等待broker确认；4）合理设置transactionTimeout，避免超时回查；5）实现checkLocalTransaction方法，保证回查时能正确返回状态；6）Consumer端做好幂等处理。

**Q4：事务消息的性能如何优化？**

> 参考答案：1）合理设置回查间隔和最大回查次数，避免频繁回查；2）使用连接池复用Producer；3）批量发送事务消息（但要注意事务边界）；4）异步执行本地事务；5）使用顺序消息时，减小事务范围；6）监控事务执行时间，优化本地事务性能。

## 总结

```
┌────────────────────────────────────────────────────────────────┐
│                   RocketMQ事务消息核心要点                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  原理：                                                           │
│  ├─ 半消息机制：消息先存储，对消费者不可见                          │
│  ├─ 两阶段提交：预发送 + 本地事务 + 确认/回滚                      │
│  └─ 回查机制：超时或UNKNOWN时Broker主动查询事务状态                 │
│                                                                │
│  使用要点：                                                        │
│  ├─ 实现TransactionListener接口                                   │
│  ├─ executeLocalTransaction：执行本地事务                         │
│  ├─ checkLocalTransaction：回查时查询事务状态                     │
│  ├─ Consumer端必须做幂等处理                                      │
│  └─ 合理配置transactionTimeout                                    │
│                                                                │
│  适用场景：                                                        │
│  ├─ 订单创建 + 库存扣减                                           │
│  ├─ 支付 + 积分增加                                               │
│  ├─ 订单 + 物流状态同步                                           │
│  └─ 跨服务的状态同步                                              │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

RocketMQ事务消息提供了一种轻量级的分布式事务解决方案，相比TCC等模式，业务侵入性更小。但使用时需要注意回查机制、幂等处理和消息超时等细节，才能保证系统的可靠性。
