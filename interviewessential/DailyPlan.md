# Daily Plan



10.14

了解BindingResult的使用方式,重点这种测试场景

序列化和反序列化 三种工具的使用方式: ObjectMapper Hutool FastJson2




10.15







12.12

使用Optional相关方法,如

 Optional.of(T data ) 

 Optional.ofnullable(T data) 

OrElse 方法 和 get() 方法  Optional后面可以跟 filter 和 map  这种流式方法

判断对象是否相等: Objects.eqauls(Object a , Object b);





1.6

在try两阶段期间: 

设置好要传递的参数; 

在tryInventory中记录(tryLog) && 记录库存扣减信息和类型流水表(比如collection_inventory_stream)

```xml
xxx.setIdentifier(这是从TL中获取到的token当作幂等号)//不相信前端传递的参数,只从session中获取
String orderId = DistributeID.generateWithSnowflake(BusinessCode.TRADE_ORDER, WorkerIdHolder.WORKER_ID, userId);
xxx.setOrderId(orderId)
```

 在tryOrder中这一部分记录(tryLog) && 记录订单创建基本信息和订单流水表中;

至于在confirm 和 cancel中再看









4.10

NFT项目中的压测看看;

看看对账他怎么做的,那些东西可以拿来借鉴;





涉及一下前台动态路由的让ai生成一下;

动态菜单啥的;

其他的脚本看着来: 比如python js 

背诵的时间也要有:  java react vue 重点不会的知识





> 更新: 2026-04-10 10:13:55  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/bshvvocgow10enbf>