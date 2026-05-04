# Redis

# 新版本如果不设置密码,使用默认的用户的时候会报错:

例如:

```yaml
data:
    redis:
      host: 127.0.0.1
      port: 32768
      username: root
##      password:
#      database: 9





```

```yaml
报错提示:

io.lettuce.core.RedisCommandExecutionException: WRONGPASS invalid username-password pair or user is disabled.

```

解决方法:去掉用户名username选项设计; 设置密码选项;

# 过期策略 VS 内存淘汰策略:

过期策略处理的是: 过期的key,redis服务器该怎么处理 (什么时候删除的问题)

内存淘汰策略: 指的是,redis内存满了,该删除那些key? (**<font style="color:rgba(0, 0, 0, 0.86);">内存满了删什么</font>**)

过期策略:

定期删除和惰性删除

# LUA脚本

## KEYS 和 ARGV 详解

### 1. **基本概念**

```lua
-- 在 Lua 脚本中
KEYS[1]   -- 第1个键名（key）
KEYS[2]   -- 第2个键名
ARGV[1]   -- 第1个参数值（value/argument）
ARGV[2]   -- 第2个参数值
```

**区别：**

* `KEYS` = Redis 的**键名**（会被 Redis 集群路由）
* `ARGV` = **普通参数值**（不参与路由）

**索引从 1 开始**（Lua 的数组习惯，不是 0）

***

### 2. **如何传递？**

**Redis 命令格式：**

**EVAL script numkeys key \[key ...] arg \[arg ...]**

```bash
EVAL "脚本内容" 键的数量 key1 key2 ... arg1 arg2 ...
                 ↑         ↑ KEYS    ↑ ARGV
```

**示例1：分布式锁**

```bash
# Redis 命令
EVAL "
  local key = KEYS[1]
  local value = ARGV[1]
  return redis.call('set', key, value, 'NX', 'EX', 30)
" 1 mylock uuid-12345
  ↑ ↑      ↑
  1个key   ARGV[1]
  KEYS[1]
```

**对应关系：**

* `1` → 告诉 Redis 后面有 1 个 key
* `mylock` → KEYS\[1]（锁的键名）
* `uuid-12345` → ARGV\[1]（锁的值）
* redis.call() 直接在服务器中执行  <font style="color:rgba(0, 0, 0, 0.86);background-color:rgba(0, 0, 0, 0.02);">所有操作都在服务器内部完成，数据在内存中直接处理,无需经过网络e</font>

***

### 3. **多个参数的例子**

**示例2：库存扣减**

```lua
-- Lua 脚本
local stockKey = KEYS[1]       -- 库存key
local orderKey = KEYS[2]       -- 订单key
local quantity = tonumber(ARGV[1])  -- 扣减数量
local userId = ARGV[2]         -- 用户ID

local stock = tonumber(redis.call('get', stockKey) or "0")
if stock >= quantity then
    redis.call('decrby', stockKey, quantity)
    redis.call('hset', orderKey, userId, quantity)
    return 1
else
    return -1
end
```

**在 Redis 中执行：**

```bash
EVAL "脚本内容" 2 stock:1001 order:20231106 5 user123
                 ↑ ↑           ↑             ↑ ↑
                 │ KEYS[1]     KEYS[2]       │ ARGV[2]
                 2个key                      ARGV[1]
```

**对应关系表格：**

| 位置 | Redis命令参数 | Lua脚本访问 | 实际值 | 含义 |
| --- | --- | --- | --- | --- |
| - | `2` | - | 2 | 告诉Redis有2个key |
| 1 | `stock:1001` | `KEYS[1]` | stock:1001 | 库存键 |
| 2 | `order:20231106` | `KEYS[2]` | order:20231106 | 订单键 |
| 3 | `5` | `ARGV[1]` | 5 | 扣减数量 |
| 4 | `user123` | `ARGV[2]` | user123 | 用户ID |

***

### 4. **在 Java 中的对应**

```java
// 使用 Spring RedisTemplate
public Long deductStock(String stockKey, String orderKey, 
                        int quantity, String userId) {
    String script = 
        "local stockKey = KEYS[1] " +
        "local orderKey = KEYS[2] " +
        "local quantity = tonumber(ARGV[1]) " +
        "local userId = ARGV[2] " +
        "local stock = tonumber(redis.call('get', stockKey) or '0') " +
        "if stock >= quantity then " +
        "    redis.call('decrby', stockKey, quantity) " +
        "    redis.call('hset', orderKey, userId, quantity) " +
        "    return 1 " +
        "else " +
        "    return -1 " +
        "end";
    
    DefaultRedisScript<Long> redisScript = new DefaultRedisScript<>();
    redisScript.setScriptText(script);
    redisScript.setResultType(Long.class);
    
    // KEYS 放在 List 中
    List<String> keys = Arrays.asList(stockKey, orderKey);
    //                            ↑          ↑
    //                         KEYS[1]    KEYS[2]
    
    // ARGV 作为可变参数
    return redisTemplate.execute(redisScript, keys, quantity, userId);
    //                                               ↑         ↑
    //                                           ARGV[1]   ARGV[2]
}

// 调用示例
deductStock("stock:1001", "order:20231106", 5, "user123");
```

***

### 5. **为什么分 KEYS 和 ARGV？**

**Redis 集群路由需要：**

* Redis 集群模式下，不同的 key 可能在不同节点
* `KEYS` 会被解析用于路由检查（确保所有key在同一个slot）
* `ARGV` 只是普通参数，不参与路由

**示例：集群限制**

```bash
# ✅ 允许（所有key在同一个slot）
EVAL "..." 2 user:1001:lock user:1001:count , 10

# ❌ 报错（key可能在不同节点）
EVAL "..." 2 user:1001:lock user:2002:lock , 10
# Error: CROSSSLOT Keys in request don't hash to the same slot
```

**解决：使用 hash tag**

```bash
EVAL "..." 2 {user:1001}:lock {user:1001}:count , 10
             ↑ {}内的内容用于计算slot，确保在同一节点
```

***

### 6. **实战记忆技巧**

**分布式锁完整示例：**

```java
@Service
public class RedisLockService {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    // 加锁脚本
    private static final String LOCK_SCRIPT = 
        "return redis.call('set', KEYS[1], ARGV[1], 'NX', 'EX', ARGV[2])";
    //                            ↑ 锁的key  ↑ 锁的值       ↑ 过期时间
    
    // 解锁脚本（防止误删别人的锁）
    private static final String UNLOCK_SCRIPT = 
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        //                    ↑ 锁的key    ↑ 我的锁值
        "    return redis.call('del', KEYS[1]) " +
        "else " +
        "    return 0 " +
        "end";
    
    public boolean lock(String lockKey, String requestId, int expireSeconds) {
        DefaultRedisScript<String> script = new DefaultRedisScript<>();
        script.setScriptText(LOCK_SCRIPT);
        script.setResultType(String.class);
        
        String result = redisTemplate.execute(
            script,
            Collections.singletonList(lockKey),  // KEYS[1] = lockKey
            requestId,                           // ARGV[1] = requestId
            String.valueOf(expireSeconds)        // ARGV[2] = expireSeconds
        );
        
        return "OK".equals(result);
    }
    
    public boolean unlock(String lockKey, String requestId) {
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setScri
[该消息未完整显示，消息长度超出了限制]
```

# 缓存和数据库一致性问题:

1. 最简单直接的方案是:\[全部数据刷到缓存中]

* 数据库中的数据,全量刷入缓存(不设置失效时间)
* 写请求只更新数据库,不更新缓存
* 启动一个定时任务,定时把数据库的数据,更新到缓存中.

这种方式优点:

* 所有的请求都能直接命中缓存,不需要查询数据库,性能非常高

这种方式的缺点:

* 缓存利用率低: 不经常访问的数据,还一直停留在缓存中
* 数据不一致的问题: 因为定时刷新缓存,缓存和数据库存在不一致

## 缓存利用率和数据一致性的问题:

想要利用缓存利用最大化: 缓存中只保留最近访问的**热点数据**

这样优化:

1. 写请求依旧只写数据库
2. 读请求先读缓存,如果缓存不存在,则从数据库中读取,并重建缓存
3. 同时,写入缓存的数据,都设置失效时间.

这样缓存利用率提高了起来,但是**数据一致性呢?**

**数据发生更新的时候?不仅操作数据库还会要一并操作缓存**

**数据库**和缓存都更新 ,先后问题:

1. **先更新缓存?后更新数据库**
2. **先更新数据库 后更新缓存**

## <font style="color:#74B602;">先不考虑并发问题,先考虑异常情况</font>

### 先更新缓存,后更新数据库

缓存更新成功,数据库更新失败了,缓存时最新值,数据库时旧值,

当**缓存没有失效**的时候,用户读取的是**正确**的值,当**缓存失效**,就会从数据库读到**旧值**,

重建缓存也是这个旧值.

### 先更新数据库,后更新缓存

如果数据库更新成功了,但是缓存更新失败了,数据库中是最新值,缓存中是旧的值,用户读取数据,先读取缓存,但此时是旧的值,只有等缓存失效的时候,才能读到正确的值.

也就是自己刚刚修改了数据,但看不到变更.一段数据过后,数据才变化.

## <font style="color:#74B602;">并发引发的一致性问题:</font>

### 先更新数据库,在更新缓存(都可以成功执行)

1. Thread A update database (x=1)
2. Thread B update database (X=2)
3. Thread B updata cache (X=2)
4. Thread A updata cache (X=1)

最终X值在数据库是2,在缓存中的数据为1 ,数据不一致问题!!!

### 先更新缓存,在更新数据库(都已成执行)

1. Thread Ａ　update cache(X=1)
2. Thread B  update cache(X=2)
3. Thread B update Database(X=2)
4. Thread A update Database(X=1)

最终X值在数据库中是1 在缓存时2 数据不一致问题!

## <font style="color:#74B602;">所以此时要删除缓存:</font>

删除缓存也有两种:

1. 先删除缓存,后更新数据库
2. 先更新数据库,后删除缓存

不考虑异常问题,因为只要第二部失败,都会导致数据不一致问题

这里看并发问题:

### 先删除缓存,后更新数据库

1. Thread A  update X=2 (原值 x=1)
2. Thread A 先 删除cache
3. Thread B 读cache 发现不存在,数据库种读取旧值x=1
4. Thread A 将x=2 写入数据库
5. Thread B 将x=1 写入缓存

X在缓存中是 1 ,在数据库是2 结果不一致

### <font style="color:#DF2A3F;">先更新数据库,后删除缓存</font>

1. 缓存 X 不存在(数据库 x=1)
2. Thread A 读取数据库 得到旧值 x=1
3. Thread B 更新数据库 X=2
4. Thread B 删除缓存
5. Thread Ａ将旧值写入缓存 x=1

<font style="color:rgba(0, 0, 0, 0.9);">最终 X 的值在缓存中是 1（旧值），在数据库中是 2（新值），也发生不一致。</font>

这**种概率很低**

1. 缓存刚好失效
2. 读请求+写并发
3. 更新数据库+删除缓存 ,要比读数据库+写缓存时间短

### 此时再次考虑第二步是否能够成功的问题

程序在执行过程中发生异常,简单的解决方法是: 重试

但是重试之后:

1. 立即重试大概率还会失败
2. 重试多少次合理?
3. 重试一值占用资源,无法服务其他客户端请求

### 更好的解决方案是异步重试

把重试请求写道消息队列中,然后由专门的消费者来重试,直到成功.

把操作缓存的这一步,直接放到消息队列中,由消费者来操作缓存.

* **<font style="color:rgb(34, 34, 34);">消息队列保证可靠性</font>**<font style="color:rgb(74, 74, 74);">：写到队列中的消息，成功消费之前不会丢失（重启项目也不担心）</font>
* **<font style="color:rgb(34, 34, 34);">消息队列保证消息成功投递</font>**<font style="color:rgb(74, 74, 74);">：下游从队列拉取消息，成功消费后才会删除消息，否则还会继续投递消息给消费者（符合我们重试的场景）</font>

<font style="color:rgb(74, 74, 74);"> </font>

[缓存和数据库一致性问题，看这篇就够了](https://mp.weixin.qq.com/s?__biz=MzIyOTYxNDI5OA==\&mid=2247487312\&idx=1\&sn=fa19566f5729d6598155b5c676eee62d\&chksm=e8beb8e5dfc931f3e35655da9da0b61c79f2843101c130cf38996446975014f958a6481aacf1\&scene=178\&cur_album_id=1699766580538032128#rd)

# 延迟双删

1. 第一次删除 在更新数据库之前进行操作
2. 更新数据库 : 执行数据库的写操作
3. 间隔一段时间 可以使用: ScheduleExecutorService 间隔一段时间
4. 再次删除缓存数据;


> 更新: 2026-04-11 15:50:48  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/ti4si0p1sm8oi1xe>