# Rabbitmq

[https://xie.infoq.cn/article/12c4bd997a7bd20985f2ddc00](https://xie.infoq.cn/article/12c4bd997a7bd20985f2ddc00)



![1762994808286-5c4e9dd1-3be3-4d77-9e26-b05660461826.png](./img/SrFlgMI4jyThUBPY/1762994808286-5c4e9dd1-3be3-4d77-9e26-b05660461826-916235.png)



**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">Broker 是 RabbitMQ 的服务器进程。Exchange 和 Queue 是定义在 Broker 内部的、用于实现消息路由和存储的逻辑对象。Broker 通过管理这些对象来提供完整的消息队列服务</font>**

**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);"></font>**

**<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);"></font>**

<font style="color:rgb(13, 13, 13);background-color:rgb(244, 246, 248);">可以看到在手动去ack消息的</font>

![1763016843084-7695a031-e409-4e38-9e8d-717ea4e3f479.png](./img/SrFlgMI4jyThUBPY/1763016843084-7695a031-e409-4e38-9e8d-717ea4e3f479-157191.png)

```java
@RabbitListener(queues = RabbitMQConfig.QUEUE_NAME)
public void receive(Message message, Channel channel){
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    String messageEntity = new String(message.getBody());
    log.info("消息id:{}",deliveryTag);
    log.info("消息实体: {} ",messageEntity);

    try {
        Thread.sleep(3000);
        log.info("消息消费成功");
        channel.basicAck(deliveryTag,false);
    } catch (Exception e) {
        log.error("消息消费失败：{}",e.getMessage());
        try {
            //不重新入队
            channel.basicNack(deliveryTag,false,false);
        } catch (Exception ex) {
            log.error("消息消费失败：{}",ex.getMessage());
        }
    }
```

![1763016983036-c28b3d5a-8530-4ee7-b250-bfa6a9cb14b4.png](./img/SrFlgMI4jyThUBPY/1763016983036-c28b3d5a-8530-4ee7-b250-bfa6a9cb14b4-055284.png)







出现异常直接丢不重新进入队列:

 channel.basicNack(deliveryTag,false,false);

![1763017924789-317f106e-646c-4861-8b70-b29459e81408.png](./img/SrFlgMI4jyThUBPY/1763017924789-317f106e-646c-4861-8b70-b29459e81408-621460.png)



拒绝或者ttl过期进入到死信队列

DLX

```java
@Bean
public Queue normalQueue() {
    Map<String, Object> args = new HashMap<>();
    // 指定死信交换机和路由键
    args.put("x-dead-letter-exchange", DEAD_EXCHANGE);
    args.put("x-dead-letter-routing-key", DEAD_ROUTING_KEY);
    // 设置消息过期时间 (5秒)
    args.put("x-message-ttl", 5000);
    return QueueBuilder.durable(NORMAL_QUEUE).withArguments(args).build();
}
```





![1763020954160-7a7b2cdb-b965-41ad-bad1-d4ba8f43c157.png](./img/SrFlgMI4jyThUBPY/1763020954160-7a7b2cdb-b965-41ad-bad1-d4ba8f43c157-133730.png)



## 生产级配置

### 1. Spring Boot 基础配置 (application.yml)

```yaml
spring:
  rabbitmq:
    host: 192.168.1.100
    port: 5672
    username: admin
    password: ${RABBITMQ_PASSWORD}  # 敏感信息走环境变量
    virtual-host: /prod
    # 连接池
    cache:
      channel:
        size: 50           # 每个连接的 channel 缓存数
        checkout-timeout: 5000  # 获取 channel 超时(ms)
      connection: cache       # connection 级别缓存: channel / connection
    # 连接参数
    connection-timeout: 10000  # 连接超时(ms)
    requested-heartbeat: 60    # 心跳间隔(秒), 0=禁用
    requested-channel-max: 2047
    # 消费者
    listener:
      type: simple
      simple:
        acknowledge-mode: manual   # 手动 ACK
        prefetch: 50               # QoS: 每次预取消息数
        concurrency: 5             # 最小消费者数
        max-concurrency: 20        # 最大消费者数
        default-requeue-rejected: false  # 拒绝后不重新入队(配合死信)
        retry:
          enabled: false           # 生产环境建议关闭自带重试, 用自定义重试策略
      direct:
        acknowledge-mode: manual
        prefetch: 100
    publisher-confirm-type: correlated  # 发布者确认
    publisher-returns: true             # 消息未路由到队列时返回
    template:
      mandatory: true            # 消息无法路由时触发 return 回调
      reply-timeout: 30000
```

### 2. 连接工厂 + 消息序列化配置

```java
@Configuration
@EnableRabbit
public class RabbitMQConfig {

    @Bean
    public MessageConverter jsonMessageConverter() {
        // 使用 Jackson 序列化, 替代默认的 Java 序列化
        Jackson2JsonMessageConverter converter = new Jackson2JsonMessageConverter();
        converter.setCreateMessageIds(true);  // 自动生成 messageId
        return converter;
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory,
                                         MessageConverter messageConverter) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(messageConverter);
        // 发布者确认回调
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("消息发送失败, correlationId:{}, cause:{}", correlationData, cause);
                // TODO: 持久化到本地表, 定时任务补偿
            }
        });
        // 消息未路由回调
        template.setReturnsCallback(returned -> {
            log.error("消息未路由到队列, exchange:{}, routingKey:{}, replyCode:{}",
                returned.getExchange(), returned.getRoutingKey(), returned.getReplyCode());
            // TODO: 记录日志或告警
        });
        return template;
    }
}
```

### 3. 生产者可靠投递

```java
@Service
public class OrderProducer {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendOrder(Order order) {
        String orderId = order.getId();
        CorrelationData correlationData = new CorrelationData(orderId);

        // 发送前先落库, 状态为"已发送未确认"
        messageLogService.save(new MessageLog(orderId, "ORDER_EXCHANGE", "order.create", "SENDING"));

        rabbitTemplate.convertAndSend(
            "order.exchange",
            "order.create",
            order,
            message -> {
                // 设置消息持久化
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                // 设置消息过期时间(兜底)
                message.getMessageProperties().setExpiration("86400000"); // 24h
                return message;
            },
            correlationData
        );
    }
}
```

### 4. 消费者幂等 + 手动 ACK

```java
@RabbitListener(queues = "order.create.queue", concurrency = "5-20")
public void handleOrderCreate(Message message, Channel channel) {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    String messageId = message.getMessageProperties().getMessageId();

    try {
        // 幂等校验: 先查 Redis, 消费过的直接 ACK
        if (Boolean.TRUE.equals(redisTemplate.hasKey("msg:" + messageId))) {
            log.info("重复消息, 跳过: {}", messageId);
            channel.basicAck(deliveryTag, false);
            return;
        }

        // 业务处理
        Order order = objectMapper.readValue(message.getBody(), Order.class);
        orderService.process(order);

        // 标记已消费 + ACK
        redisTemplate.opsForValue().set("msg:" + messageId, "1", 24, TimeUnit.HOURS);
        channel.basicAck(deliveryTag, false);

    } catch (BusinessException e) {
        // 业务异常 → 拒绝并进入死信队列
        log.error("业务异常, 进入死信: {}", e.getMessage());
        channel.basicNack(deliveryTag, false, false);
    } catch (Exception e) {
        // 系统异常 → 重新入队(最多重试 3 次, 通过 Redis 计数)
        int retryCount = getRetryCount(messageId);
        if (retryCount < 3) {
            incrementRetry(messageId);
            channel.basicNack(deliveryTag, false, true);  // requeue = true
        } else {
            log.error("重试耗尽, 进入死信: {}", messageId);
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

### 5. 延迟队列 (TTL + DLX 模式)

```java
@Configuration
public class DelayQueueConfig {

    // 正常交换机 + 队列, 消息 TTL 后进入死信
    @Bean
    public Queue delayQueue() {
        Map<String, Object> args = new HashMap<>();
        args.put("x-dead-letter-exchange", "dlx.exchange");
        args.put("x-dead-letter-routing-key", "dlx.routing.key");
        // 不在队列级别设 TTL, 而在消息级别设不同 TTL
        return QueueBuilder.durable("delay.queue").withArguments(args).build();
    }

    // 发送延迟消息
    public void sendDelay(Object msg, long delayMs) {
        rabbitTemplate.convertAndSend("delay.exchange", "delay.routing", msg, m -> {
            m.getMessageProperties().setExpiration(String.valueOf(delayMs));
            return m;
        });
    }
}
```

### 6. 队列声明最佳实践

```java
@Configuration
public class QueueConfig {

    // 正常队列: 持久化 + 惰性队列(大消息场景)
    @Bean
    public Queue mainQueue() {
        return QueueBuilder.durable("order.create.queue")
            .withArgument("x-queue-mode", "lazy")       // 惰性队列, 消息写磁盘
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .withArgument("x-dead-letter-routing-key", "order.create.dlx")
            .build();
    }

    // 交换机
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange", true, false);  // durable, not autoDelete
    }

    // 绑定
    @Bean
    public Binding binding() {
        return BindingBuilder.bind(mainQueue()).to(orderExchange()).with("order.#");
    }

    // 死信队列
    @Bean
    public Queue dlxQueue() {
        return QueueBuilder.durable("order.dlx.queue").build();
    }

    @Bean
    public DirectExchange dlxExchange() {
        return new DirectExchange("dlx.exchange", true, false);
    }

    @Bean
    public Binding dlxBinding() {
        return BindingBuilder.bind(dlxQueue()).to(dlxExchange()).with("order.create.dlx");
    }
}
```

### 7. 死信队列消费 + 告警

```java
@RabbitListener(queues = "order.dlx.queue")
public void handleDLX(Message message, Channel channel) {
    String messageId = message.getMessageProperties().getMessageId();
    log.warn("进入死信队列, messageId:{}", messageId);

    try {
        // 记录死信日志, 可做人工介入或二次处理
        dlxMessageService.save(messageId, new String(message.getBody()));
        // 通知告警(钉钉/邮件)
        alertService.send("【RabbitMQ死信】消息进入死信队列: " + messageId);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        log.error("死信处理失败", e);
    }
}
```

### 8. 关键生产参数总结

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `acknowledge-mode` | manual | 手动确认, 避免消息丢失 |
| `prefetch` | 50-100 | 过小→吞吐低, 过大→消费不均 |
| `concurrency` | 5-20 | 根据队列积压动态调整 |
| `publisher-confirm-type` | correlated | 发送端确认 |
| `delivery-mode` | PERSISTENT | 消息持久化 |
| `mandatory` | true | 捕获路由失败的消息 |
| `default-requeue-rejected` | false | 拒绝后走死信, 避免死循环 |
| `x-queue-mode` | lazy | 大量积压时减少内存占用 |
| `x-max-length` | 100000 | 队列最大长度, 防止内存溢出 |

### 9. 常见面试问题

- **为什么用 RabbitMQ 而不是 Kafka？** RabbitMQ 支持复杂路由(Exchange 多种类型)、延迟队列、优先级队列, 适合业务系统; Kafka 适合大数据流、日志场景, 吞吐量更高但路由能力弱
- **如何保证消息不丢失？** 生产者端 publisher-confirm + 落库; Broker 端持久化+集群镜像; 消费者端手动 ACK
- **如何保证消息不重复消费？** 业务层做幂等(唯一键/Redis/数据库唯一索引)
- **消息积压怎么办？** 临时增加消费者实例 + 提高 prefetch; 排查消费慢的原因(慢 SQL、外部调用超时等)
- **死信队列的作用？** 接收被拒绝/超时的消息, 方便问题排查和数据补偿

