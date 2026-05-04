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



> 更新: 2025-11-13 16:02:36  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/niws8vei0xmhacmt>