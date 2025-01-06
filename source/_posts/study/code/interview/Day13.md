---

title: 面试训练营Day13

date: 2025/01/05 20：46：25

categories:

- [学习成长, 编程, 面试训练营]

tags:

---
**2025-01-05**🌱上海: ☀️   🌡️+13°C 🌬️→13km/h

## **Redis 中如何实现分布式锁？**

### 总结分析

在 Redis 中实现分布式锁，常见做法是将 set ex nx 命令与 lua 脚本组合使用。这一方式能确保多个客户端不会同时获取同一资源锁，还能保障安全解锁以及在意外情况下自动释放锁 。

### 扩展知识

为了能够更好的了解分布式锁的实现原理及为什么实现这些功能进行逐步分析

### 手写一个分布式锁

首先分析一下分布式锁实现都需要满足什么，可以基于JUC中的AQS规范进行参考

#### 分布式锁所需满足的条件

- **独占性**

- 任何时刻只能且仅有一个线程持有

- **高可用**

- 在redis集群环境下，不能因为某一个节点挂了而出现获取锁和释放锁失败的情况
- 在高并发请求下，性能依旧ok

- **防死锁**

- 杜绝死锁，必须要有超时控制或者撤销操作，有个兜底终止跳出方案

- **不乱抢**

- 防止张冠李戴，不能私下释放了别人的锁，只能自己加的锁自己释放

- **重入性**

- 同一个节点的同一个线程获得锁之后，它可以再次获取这个锁

#### 逐步分析

一般来说，对于分布式锁我们一般用setnx命令进行操作，但是针对于高并发场景，setnx并不全面（考虑可重入性）。

![](https://cdn.nlark.com/yuque/0/2023/png/26566882/1697012816577-f4cfb9f2-ff71-4fdb-a6b4-acec42e7ff78.png?x-oss-process=image%2Fformat%2Cwebp)

**JUC中AQS锁的规范落地参考+可重入锁考虑+Lua脚本+Redis命令一步步实现分布式锁**

#### 实现一个简单的Redis分布式锁（使用setnx通过java代码实现）

```
public String sale()
{
    String retMessage = "";
    String key = "MuziRedisLock";
    String uuidValue = IdUtil.simpleUUID()+":"+Thread.currentThread().getId();

    Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key, uuidValue);
    if(!flag)
    {
        //暂停20毫秒，进行递归重试.....
        try { TimeUnit.MILLISECONDS.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
        sale();
    }else{
        //抢锁成功的请求线程，进行正常的业务逻辑操作，扣减库存
        try
        {
            //1 查询库存信息
            String result = stringRedisTemplate.opsForValue().get("inventory001");
            //2 判断库存是否足够
            Integer inventoryNumber = result == null ? 0 : Integer.parseInt(result);
            //3 扣减库存，每次减少一个
            if(inventoryNumber > 0)
            {
                stringRedisTemplate.opsForValue().set("inventory001",String.valueOf(--inventoryNumber));
                retMessage = "成功卖出一个商品,库存剩余:"+inventoryNumber;
                System.out.println(retMessage+"\t"+"服务端口号"+port);
            }else{
                retMessage = "商品卖完了,o(╥﹏╥)o";
            }
        }finally {
            stringRedisTemplate.delete(key);
        }
    }
    return retMessage+"\t"+"服务端口号"+port;
}
```

上面代码就是一个简单通过setnx实现分布式锁的案例，但是存在问题，`递归重试，容易导致stackoverflowerror，所以不太推荐；另外，高并发唤醒后推荐用while判断而不是if`

**进一步优化**

```

public String sale()
{
    String retMessage = "";

    String key = "MuziRedisLock";
    String uuidValue = IdUtil.simpleUUID()+":"+Thread.currentThread().getId();

    //不用递归了，高并发下容易出错，用自旋替代递归方法重试调用;也不用if了，用while来替代
    while(!stringRedisTemplate.opsForValue().setIfAbsent(key, uuidValue))
    {
        //暂停20毫秒，进行递归重试.....
        try { TimeUnit.MILLISECONDS.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
    }

    //抢锁成功的请求线程，进行正常的业务逻辑操作，扣减库存
    try
    {
        //1 查询库存信息
        String result = stringRedisTemplate.opsForValue().get("inventory001");
        //2 判断库存是否足够
        Integer inventoryNumber = result == null ? 0 : Integer.parseInt(result);
        //3 扣减库存，每次减少一个
        if(inventoryNumber > 0)
        {
            stringRedisTemplate.opsForValue().set("inventory001",String.valueOf(--inventoryNumber));
            retMessage = "成功卖出一个商品,库存剩余:"+inventoryNumber;
            System.out.println(retMessage+"\t"+"服务端口号"+port);
        }else{
            retMessage = "商品卖完了,o(╥﹏╥)o";
        }
    }finally {
        stringRedisTemplate.delete(key);
    }
    return retMessage+"\t"+"服务端口号"+port;
}
```

这里我们可以通过打断点在执行过程中关闭服务模仿**机器宕机**，导致代码层面根本**走不到finally这块**，那这样就无法保证正常解锁（无过期时间该key会一致存在），同时因为key没有被删除一直存在，所以需要加入一个过期时间限定key的存在，继续优化。

#### 增加过期时间

```

public String sale()
{
    String retMessage = "";

    String key = "MuziRedisLock";
    String uuidValue = IdUtil.simpleUUID()+":"+Thread.currentThread().getId();

    //改进点：加锁和过期时间设置必须同一行，保证原子性
    while(!stringRedisTemplate.opsForValue().setIfAbsent(key,uuidValue))
    {
        //暂停20毫秒，进行递归重试.....
        try { TimeUnit.MILLISECONDS.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
    }

    stringRedisTemplate.expire(key,30L,TimeUnit.SECONDS);

    //抢锁成功的请求线程，进行正常的业务逻辑操作，扣减库存
    try
    {
        //1 查询库存信息
        String result = stringRedisTemplate.opsForValue().get("inventory001");
        //2 判断库存是否足够
        Integer inventoryNumber = result == null ? 0 : Integer.parseInt(result);
        //3 扣减库存，每次减少一个
        if(inventoryNumber > 0)
        {
            stringRedisTemplate.opsForValue().set("inventory001",String.valueOf(--inventoryNumber));
            retMessage = "成功卖出一个商品,库存剩余:"+inventoryNumber;
            System.out.println(retMessage+"\t"+"服务端口号"+port);
        }else{
            retMessage = "商品卖完了,o(╥﹏╥)o";
        }
    }finally {
        stringRedisTemplate.delete(key);
    }
    return retMessage+"\t"+"服务端口号"+port;
}
```

从上面的代码我们可以看到，在加锁的时候我们后面给锁添加了一个过期时间，但是仔细考虑，还是会存在问题就是**加锁和设置过期时间**并不是在同一行，这就说明并**没有保证原子性**，可能导致过期时间添加失败，所以还需要进行优化。

```

public String sale()
{
    String retMessage = "";

    String key = "MuziRedisLock";
    String uuidValue = IdUtil.simpleUUID()+":"+Thread.currentThread().getId();

    //改进点：加锁和过期时间设置必须同一行，保证原子性
    while(!stringRedisTemplate.opsForValue().setIfAbsent(key,uuidValue,30L,TimeUnit.SECONDS))
    {
        //暂停20毫秒，进行递归重试.....
        try { TimeUnit.MILLISECONDS.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
    }


    //抢锁成功的请求线程，进行正常的业务逻辑操作，扣减库存
    try
    {
        //1 查询库存信息
        String result = stringRedisTemplate.opsForValue().get("inventory001");
        //2 判断库存是否足够
        Integer inventoryNumber = result == null ? 0 : Integer.parseInt(result);
        //3 扣减库存，每次减少一个
        if(inventoryNumber > 0)
        {
            stringRedisTemplate.opsForValue().set("inventory001",String.valueOf(--inventoryNumber));
            retMessage = "成功卖出一个商品,库存剩余:"+inventoryNumber;
            System.out.println(retMessage+"\t"+"服务端口号"+port);
        }else{
            retMessage = "商品卖完了,o(╥﹏╥)o";
        }
    }finally {
        stringRedisTemplate.delete(key);
    }
    return retMessage+"\t"+"服务端口号"+port;
}
```

这里已经解决了加锁和设置过期时间原子性的问题，仔细考虑下还存在什么问题?

假设线程一获取到锁，但是执行业务时间＞加锁过期时间，线程一还在执行中，但是锁因为过期释放了。

线程二这时候加锁成功了，然后线程二任务执行中，线程一任务执行完成了，然后释放锁成功，但是这时候的锁已经是线程二加的锁了，导致线程二的锁被线程一释放了。所以我们需要解决一个问题就是自己只能删除自己加的锁，需要添加判断是否自己的锁进行操作。具体流程如图：

![](https://cdn.nlark.com/yuque/0/2025/png/26566882/1736071967983-0a6ba7ef-92f0-4087-943a-79400a9bf33f.png)

#### 防止误删key的问题

```
public String sale()
{
    String retMessage = "";

    String key = "MuziRedisLock";
    String uuidValue = IdUtil.simpleUUID()+":"+Thread.currentThread().getId();

    while(!stringRedisTemplate.opsForValue().setIfAbsent(key,uuidValue,30L,TimeUnit.SECONDS))
    {
        //暂停20毫秒，进行递归重试.....
        try { TimeUnit.MILLISECONDS.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
    }

    //抢锁成功的请求线程，进行正常的业务逻辑操作，扣减库存
    try
    {
        //1 查询库存信息
        String result = stringRedisTemplate.opsForValue().get("inventory001");
        //2 判断库存是否足够
        Integer inventoryNumber = result == null ? 0 : Integer.parseInt(result);
        //3 扣减库存，每次减少一个
        if(inventoryNumber > 0)
        {
            stringRedisTemplate.opsForValue().set("inventory001",String.valueOf(--inventoryNumber));
            retMessage = "成功卖出一个商品,库存剩余:"+inventoryNumber;
            System.out.println(retMessage+"\t"+"服务端口号"+port);
        }else{
            retMessage = "商品卖完了,o(╥﹏╥)o";
        }
    }finally {
        //改进点，只能删除属于自己的key，不能删除别人的
        // 判断加锁与解锁是不是同一个客户端，同一个才行，自己只能删除自己的锁，不误删他人的
        if(stringRedisTemplate.opsForValue().get(key).equalsIgnoreCase(uuidValue))
        {
            stringRedisTemplate.delete(key);
        }
    }
    return retMessage+"\t"+"服务端口号"+port;
}
```

仔细考虑下还存在什么问题？

根据我们上面保证加锁和设置过期时间的原子性分析，不难看出这里锁的判断逻辑和删除逻辑同样不是原子命令操作。这里需要怎么做呢，我们不能通过原生命令实现的话，可以通过lua脚本实现原子操作。这里说明一点**redis的原子性和mysql事务的原子性并不一致**，具体区别可以看下面分析：

|   |   |   |
|---|---|---|
|**比较项**|**Redis**|**MySQL**|
|单命令原子性|基于单线程模型，同一时刻只能执行一个命令，保证单个命令原子性。如`INCR`<br><br>命令可原子性递增键值。|通过行级锁实现某些单条 SQL 语句对特定行数据操作的原子性，如`UPDATE...WHERE`<br><br>语句。|
|事务原子性|通过`MULTI`<br><br>、`EXEC`<br><br>等命令实现。`MULTI`<br><br>后命令入队列，`EXEC`<br><br>按序执行。执行`EXEC`<br><br>前出错，整个事务不执行；执行中某个命令失败（2.6.5 及以上版本）默认不回滚整个事务，继续执行后续命令。|通过日志（重做日志、回滚日志）和锁机制实现。事务开始记录操作日志，成功则持久化并提交，出错则根据回滚日志撤销操作，保证事务原子性。|
|应用场景|单命令原子性适用于简单计数、状态标记等；事务原子性适合对多个操作原子性处理，但对事务回滚要求不严格的场景，如电商系统中同时更新商品库存和用户订单信息。|事务原子性适用于对数据一致性要求极高的场景，如银行转账、订单处理等；行级锁原子操作适用于多并发下特定行数据修改需保证原子性的场景，如多用户库存管理系统。|
|原子性范围|主要体现在单命令和简单事务上。|强调事务原子性，事务中可包含复杂多表操作。|
|回滚机制|事务执行中部分命令失败默认不回滚整个事务。|事务执行中出现错误自动回滚整个事务，确保数据一致性。|
|性能|单线程模型下原子性操作性能高，适合高并发简单操作。|事务原子性涉及日志记录和锁机制，性能相对较低。|

回归正题，那如何编写lua脚本呢？下面是我之前学习lua脚本的一些总结

[Lua脚本浅谈](https://www.yuque.com/muzijinyouergun/uy1hur/wqgmalgd2g7ka4o0 "Lua脚本浅谈")

#### lua脚本保证原子性

```
if redis.call("GET",KEYS[1]) == ARGV[1]
then
    return redis.call("DEL",KEYS[1])
else
    return 0
end
```

完整的java代码

```
 public String sale()
{
String retMessage = "";
String key = "MuziRedisLock";
String uuidValue = IdUtil.simpleUUID()+":"+Thread.currentThread().getId();

while(!stringRedisTemplate.opsForValue().setIfAbsent(key, uuidValue,30L,TimeUnit.SECONDS))
{
//暂停毫秒
try { TimeUnit.MILLISECONDS.sleep(20); } catch (InterruptedException e) { e.printStackTrace(); }
}

try
{
//1 查询库存信息
String result = stringRedisTemplate.opsForValue().get("inventory001");
//2 判断库存是否足够
Integer inventoryNumber = result == null ? 0 : Integer.parseInt(result);
//3 扣减库存
if(inventoryNumber > 0) {
stringRedisTemplate.opsForValue().set("inventory001",String.valueOf(--inventoryNumber));
retMessage = "成功卖出一个商品，库存剩余: "+inventoryNumber+"\t"+uuidValue;
System.out.println(retMessage);
}else{
retMessage = "商品卖完了，o(╥﹏╥)o";
}
}finally {
String luaScript =
"if (redis.call('get',KEYS[1]) == ARGV[1]) then " +
"return redis.call('del',KEYS[1]) " +
"else " +
"return 0 " +
"end";
stringRedisTemplate.execute(new DefaultRedisScript<>(luaScript, Boolean.class), Arrays.asList(key), uuidValue);
}
return retMessage+"\t"+"服务端口号："+port;
}
```

到这里一个分布式锁的就基本完成了，但是我们还需要继续完善下，这里想一下，我们正常使用的锁一般都有什么特性，比如`ReentrantLock`和`synchronize`，它们都具有可重入性，所以接下继续优化

#### 可重入锁

- 可重入锁又称递归锁，同一线程外层方法获取锁后，进入内层方法能自动获取同一锁对象，不会因已获锁未释放而阻塞。
- Java 里 `ReentrantLock` 和 `synchronized` 均为可重入锁，该特性可一定程度避免死锁。
- 对可重入锁进行 AQS 源码分析，有几次 `lock` 就需对应几次 `unlock`。
- 针对 `ReentrantLock`（显式锁）与 `synchronized`（隐式锁）的可重入锁计数问题，Redis 的 hash 数据类型（K,K,V）能更好解决分布式锁的可重入问题 。

那么我们就通过hash数据类型的操作进行模拟

1. `hset key field value`
2. `hset redis锁名字(MuziRedisLock) 某个请求线程的UUID+ThreadID 加锁的次数`

```
127.0.0.1:6379> HEXISTS MuziRedisLock 1111-2222
(integer) 0
127.0.0.1:6379> HSET MuziRedisLock 1111-2222 1
(integer) 1
127.0.0.1:6379> HINCRBY MuziRedisLock 1111-2222 1
(integer) 2
127.0.0.1:6379> HINCRBY MuziRedisLock 1111-2222 1
(integer) 3
127.0.0.1:6379> HGET MuziRedisLock 1111-2222
"3"
127.0.0.1:6379> HINCRBY MuziRedisLock 1111-2222 -1
(integer) 2
127.0.0.1:6379> HINCRBY MuziRedisLock 1111-2222 -1
(integer) 1
127.0.0.1:6379> HINCRBY MuziRedisLock 1111-2222 -1
(integer) 0
127.0.0.1:6379> hget MuziRedisLock 1111-2222
"0"
```

**由上总结，setnx只能解决有无的问题，但是hset不但解决有无，还可以解决可重入问题！那么接下来就是编写lua脚本**

#### 加锁lua脚本（使用hash数据类型）

**分析加锁过程**

1. **判断锁是否存在**：使用 `EXISTS KEY` 命令判断 Redis 分布式锁的键（`KEY`）是否存在。

- **不存在时**：若返回零，说明锁不存在，此时通过 `HSET MuziRedisLock 0c90d37cb6ec42268861b3d739f8b3a8:1 1` 命令，以 `HSET` 方式新建一个属于当前线程的锁，其中键为 `MuziRedisLock`，值为当前线程的 `UUID:ThreadID`（如 `0c90d37cb6ec42268861b3d739f8b3a8:1`），并将锁的相关计数等信息设为 `1`。
- **存在时**：若 `EXISTS KEY` 返回一，说明已经有锁存在，需进一步判断该锁是否为当前线程自己的。

2. **判断锁归属**：使用 `HEXISTS MuziRedisLock 0c90d37cb6ec42268861b3d739f8b3a8:1` 命令进行判断。

- **不是自己的锁**：若返回零，说明该锁不是当前线程的。
- **是自己的锁**：若返回一，说明是当前线程自己的锁，此时通过 `HINCRBY MuziRedisLock 0c90d37cb6ec42268861b3d739f8b3a8:1 1` 命令，使用 `HINCRBY` 对该锁的相关字段（以当前线程的 `UUID:ThreadID` 为字段）进行自增 `1` 次，表示锁的重入。

```
if redis.call('exists',KEYS[1]) == 0 or redis.call('hexists',KEYS[1],ARGV[1]) == 1 
then 
  redis.call('hincrby',KEYS[1],ARGV[1],1) 
  redis.call('expire',KEYS[1],ARGV[2]) 
  return 1 
else
  return 0
end
```

#### 分析解锁过程

1. **判断锁是否存在：**通过 `HEXISTS MuziRedisLock 0c90d37cb6ec42268861b3d739f8b3a8:1` 检查锁是否存在及归属。

- **若返回零**，表明无锁，程序块返回 `nil`，解锁操作无法进行。
- **若返回非零**，说明有锁且为自身持有。

2. **判断锁归属：**对于有锁且是自己的情况，调用 `HINCRBY` 命令，将对应字段值每次减 `1` 进行解锁操作，即 `HINCRBY zzyyRedisLock 0c90d37cb6ec42268861b3d739f8b3a8:1 -1`。
3. 持续执行上述解锁操作，直至字段值变为零，此时可使用 `del` 命令删除锁 `key`（`del zzyyRedisLock`），完成锁的释放，确保其他线程可获取该锁，保障分布式锁的正确性与可靠性，避免资源泄漏和死锁等问题。

```
if redis.call('HEXISTS',KEYS[1],ARGV[1]) == 0 
then
  return nil
elseif redis.call('HINCRBY',KEYS[1],ARGV[1],-1) == 0 then
  return redis.call('del',KEYS[1])
else
  return 0
end
```

整合进java代码中

```
//@Component 引入DistributedLockFactory工厂模式，从工厂获得即可
public class RedisDistributedLock implements Lock {
    private StringRedisTemplate stringRedisTemplate;

    private String lockName;//KEYS[1]
    private String uuidValue;//ARGV[1]
    private long expireTime;//ARGV[2]


    public RedisDistributedLock(StringRedisTemplate stringRedisTemplate, String lockName, String uuid) {
        this.stringRedisTemplate = stringRedisTemplate;
        this.lockName = lockName;
        this.uuidValue = uuid + ":" + Thread.currentThread().getId();
        this.expireTime = 30L;
    }

    @Override
    public void lock() {
        tryLock();
    }

    @Override
    public boolean tryLock() {
        try {
            tryLock(-1L, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        if (time == -1L) {
            String script =
                    "if redis.call('exists',KEYS[1]) == 0 or redis.call('hexists',KEYS[1],ARGV[1]) == 1 then    " +
                            "redis.call('hincrby',KEYS[1],ARGV[1],1)    " +
                            "redis.call('expire',KEYS[1],ARGV[2])    " +
                            "return 1  " +
                            "else   " +
                            "return 0 " +
                            "end";
            System.out.println("lockName:" + lockName + "\t" + "uuidValue:" + uuidValue);

            while (!stringRedisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList(lockName), uuidValue, String.valueOf(expireTime))) {
                //暂停60毫秒
                try {
                    TimeUnit.MILLISECONDS.sleep(60);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            //新建一个后台扫描程序，来坚持key目前的ttl，是否到我们规定的1/2 1/3来实现续期
            renewExpire();
            return true;
        }
        return false;
    }


    @Override
    public void unlock() {
        System.out.println("unlock(): lockName:" + lockName + "\t" + "uuidValue:" + uuidValue);
        String script =
                "if redis.call('HEXISTS',KEYS[1],ARGV[1]) == 0 then    " +
                        "return nil  " +
                        "elseif redis.call('HINCRBY',KEYS[1],ARGV[1],-1) == 0 then    " +
                        "return redis.call('del',KEYS[1])  " +
                        "else    " +
                        "return 0 " +
                        "end";
        // nil = false 1 = true 0 = false
        Long flag = stringRedisTemplate.execute(new DefaultRedisScript<>(script, Long.class), Arrays.asList(lockName), uuidValue, String.valueOf(expireTime));

        if (null == flag) {
            throw new RuntimeException("this lock doesn't exists，o(╥﹏╥)o");
        }
    }
                                      
}
```

重入性已经解决了，但是还有一个问题，就是我们之前考虑到可能出现的一种情况，业务没有执行完，锁就过期了，所以我们应该考虑如何实现续期功能。

这里我们可以参考`**看门狗**`的实现原理，通过一个定时任务不断刷新过期时间。

#### 自动续期

编写lua脚本，判断锁是否存在

```
if redis.call('HEXISTS',KEYS[1],ARGV[1]) == 1 then
  return redis.call('expire',KEYS[1],ARGV[2])
else
  return 0
end
```

java代码

```
private void renewExpire() {
    String script =
            "if redis.call('HEXISTS',KEYS[1],ARGV[1]) == 1 then     " +
                    "return redis.call('expire',KEYS[1],ARGV[2]) " +
                    "else     " +
                    "return 0 " +
                    "end";

    new Timer().schedule(new TimerTask() {
        @Override
        public void run() {
            if (stringRedisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList(lockName), uuidValue, String.valueOf(expireTime))) {
                renewExpire();
            }
        }
    }, (this.expireTime * 1000) / 3);
}
```

```
public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
    if (time == -1L) {
        String script =
                "if redis.call('exists',KEYS[1]) == 0 or redis.call('hexists',KEYS[1],ARGV[1]) == 1 then    " +
                        "redis.call('hincrby',KEYS[1],ARGV[1],1)    " +
                        "redis.call('expire',KEYS[1],ARGV[2])    " +
                        "return 1  " +
                        "else   " +
                        "return 0 " +
                        "end";
        System.out.println("lockName:" + lockName + "\t" + "uuidValue:" + uuidValue);

        while (!stringRedisTemplate.execute(new DefaultRedisScript<>(script, Boolean.class), Arrays.asList(lockName), uuidValue, String.valueOf(expireTime))) {
            //暂停60毫秒
            try {
                TimeUnit.MILLISECONDS.sleep(60);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //新建一个后台扫描程序，来检测key目前的ttl，是否到我们规定的1/2 1/3来实现续期
        renewExpire();
        return true;
    }
    return false;
}
```

#### 总结

- **锁释放问题**：仅使用 `setnx` 加锁而未释放锁，若出现异常，可能无法释放锁，因此需在代码层面的 `finally` 块中释放锁。但如果服务器宕机，代码无法走到 `finally` 块，仍无法保证解锁，所以要给锁的 `key` 设置过期时间。
- **命令原子性问题**：为保证设置锁和过期时间的原子性，`setnx` 和设置过期时间的操作必须在同一行。
- **安全解锁问题**：要确保只能自己删除自己的锁，防止误删他人的锁，可将 `unlock` 操作变为 Lua 脚本以保证原子性。
- **可重入问题**：为考虑可重入性，可用 `hset` 替代 `setnx`，并且将整个加锁过程变为 Lua 脚本保证操作的原子性。
- **续期问题**：为防止业务执行时间超过锁的过期时间，需要添加自动续期机制 。

以上只是解决了单个Redis服务器的加锁，同时还会存在集群下的网络分区问题。这里就需要了下RedLock算法了
#### Redlock 介绍

- **基本思想**：为解决上述问题，Redis 推出 Redlock，适用于集群环境。部署多个（通常 5 个）Redis 实例，客户端在多数（至少 3 个）实例上请求锁，一定时间内成功则加锁成功，可提供更高容错性。
- **实现流程**：客户端在有限时间（通常为锁的过期时间）内尝试在每个 Redis 实例上加锁，若多数（N/2 + 1）实例加锁成功，则加锁成功；否则释放已加锁实例，重新尝试 。
- **缺点**：

- **复杂性**：需多个 Redis 实例，增加系统复杂性和维护成本。
- **时间同步依赖**：依赖多个节点系统时间一致，时间不同步可能影响锁有效性。
- **不适用于高并发**：高并发场景下，访问多个实例获取锁会导致性能下降。
- **锁的续期问题**：长时间操作需手动续期锁，涉及多个实例，增加实现复杂度和风险。
## **Redis 的 Red Lock 是什么？你了解吗**

### 总结分析

- Red Lock（红锁）是一种分布式锁实现方案，用于解决分布式环境中使用 Redis 实现分布式锁的安全性问题。
- 生产环境通常采用主从 + 哨兵方式部署 Redis。
- 使用 Redis 分布式锁时，主从切换过程中，从节点可能未同步主节点的锁信息，导致新主节点无锁信息。
- 另一业务可能因误以为锁未被占而抢到锁，进而与原持有锁业务同时进入临界区操作临界资源，引发数据不一致问题。
- Redis 官方推出红锁，可避免上述状况，确保部分节点故障时不影响锁的使用和数据的正确性。

### 扩展知识

- **红锁部署要求**：使用红锁需集群部署 Redis，官方推荐至少 5 个主库实例，无需从库和哨兵，实例间无关系、无需信息交互。
- **红锁申请机制**：客户端依次向 5 个实例申请锁，成功申请数量超过半数（>=3）则红锁申请成功，否则失败。若有实例宕机，不影响申请，理论成功数可达 4 个。没有主从机制，避免同步丢失锁问题。
- **加锁流程**：

1. 客户端获取当前时间（t1）。
2. 按顺序对 N 个 Redis 节点用 set 命令加锁，设置短超时时间（远小于锁总过期时间），请求超时则立即向下一节点申请。
3. 成功从半数节点获锁后，获取当前时间 t2，计算加锁总耗时 t（t2 - t1）。若 t < 锁过期时间，加锁成功，否则失败。
4. 加锁成功执行业务，失败则向全部节点发起释放锁流程。

#### 优点

- **高容错性**：基于多个独立 Redis 主节点（至少 5 个），部分节点故障时仍能保证半数以上节点正常工作，可获取锁，系统容错能力强。
- **避免同步问题**：不依赖主从架构，无主从同步延迟导致的锁安全问题，锁可靠性和安全性高。
- **适应分布式**：适用于分布式场景，能在多节点间实现有效锁控制，满足分布式应用对锁的需求。

#### 缺点

- **复杂性高**：需部署多个独立 Redis 主节点，增加部署和维护成本，实现与运维更复杂。
- **性能开销大**：客户端向多节点依次发送加锁请求，网络通信和请求处理时间增加，高并发下性能下降、效率低。
- **依赖时钟同步**：运行依赖各节点时钟同步，时钟偏差大时会影响锁有效性，影响系统正确性和稳定性。
- **锁续期复杂**：长时间业务操作中锁续期需协调多个节点，实现难度大，易引入潜在问题 。

#### 红锁一定安全么？

![](https://cdn.nlark.com/yuque/0/2025/png/26566882/1736079114551-8e709613-c76b-4692-bfbf-92c9358f7a1e.png)

1. `Client 1`向`Lock service`请求获取锁（`get lease`），`Lock service`返回`ok`，表示`Client 1`成功获取锁。
2. `Client 1`发生了`stop-the-world GC pause`（停止世界的垃圾回收暂停），在此期间，`Lock service`认为`Client 1`的锁租赁（`lease`）已过期。
3. `Client 2`向`Lock service`请求获取锁（`get lease`），`Lock service`返回`ok`，`Client 2`成功获取锁。
4. `Client 2`向`Storage`写入数据（`write data`），`Storage`返回`ok`，表示写入成功。
5. `Client 1`的垃圾回收暂停结束后，它认为自己仍然持有锁，于是也向`Storage`写入数据，但此时可能会出现数据冲突或错误

这里我问了ai一些相关的解决方案，但是感觉并不是每个方案都合理（仅作参考）

- **延长锁有效期**：增加锁在 Redis 中的有效时长，降低因 GC 致锁过期概率，但过长会影响并发性能。
- **引入续租机制**：获取锁后启动后台线程或定时任务定期续租，如 Redisson 的 Watch Dog 机制。
- **增唯一标识校验**：加锁时生成唯一标识，执行逻辑前检查标识是否一致，不一致则重新处理。
- **优化 GC 策略**：调整 JVM GC 参数，如使用高效算法、调整堆内存等，减少 GC 暂停时间。
- **分布式事务保障**：将相关操作封装在分布式事务中，利用其原子性，异常时事务回滚。
- **利用框架高级特性**：如 Redisson 提供多种获取锁模式和重试策略，可按需配置。

然后就是还有可能发生时钟漂移问题，同样通过询问ai做了解决方案

- **协议同步**：使用 NTP 协议，通过与 NTP 服务器通信获取准确时间来调整本地时钟；对于高精度场景，采用 PTP 协议，借助硬件与软件协同实现亚微秒级同步。
- **定期校准**：即便使用同步协议，仍定期（如每天或每周）手动或自动校准时钟，保障时钟准确性。
- **硬件同步**：对时间精度要求极高的场景，利用 GPS 时钟服务器、原子钟等专业硬件设备提供精确时间信号，实现节点时间同步。
- **监控告警**：建立监控机制实时监测时钟偏差，设置阈值，超过阈值时及时告警通知管理员处理。
- **架构设计**：分布式系统设计中采用容错机制和算法，如分布式锁结合锁获取顺序等因素、分布式数据库使用基于版本号或逻辑时钟的并发控制，减少对物理时钟依赖 。

但是对于以上方案中的相关知识并不了解，又进行了网上搜查，找到一篇相关文章（**内容有点复杂，暂做了解，后续再做深入学习**）

[【分布式系统】Ch 6 同步 Synchronization （时钟同步、互斥、选举）_ntp选举算法-CSDN博客](https://blog.csdn.net/qq_41283356/article/details/124964332?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-1-124964332-blog-113695334.235^v43^pc_blog_bottom_relevance_base5&spm=1001.2101.3001.4242.2&utm_relevant_index=4)

## Redis 实现分布式锁时可能遇到的问题有哪些？

### 总结分析

1. 业务未执行完，锁已到期
2. 单点故障问题
3. 主从问题不同步问题
4. 网络分区问题
5. 时钟漂移问题
6. 锁的可重入性问题
7. 误释放锁问题

### 扩展知识

- **业务未执行完，锁已到期**：为防锁无法正常释放需设过期时间，却可能导致业务未完成锁已过期。可采用续约机制（如 Redisson 的看门狗机制），由守护线程判断业务执行情况并适时续约。同时，要合理评估设置锁的过期时间，避免影响 Redis 性能或出现锁提前过期问题。
- **单点故障问题**：Redis 单机部署时，实例宕机或不可用会使分布式锁服务无法工作，阻碍业务执行。
- **主从问题**：主从 + 哨兵部署的 Redis 中，主从复制异步，主节点获取锁后未同步到从节点就宕机，新主节点无锁数据，会导致多个客户端同时获取锁。
- **网络分区**：网络不稳定时客户端与 Redis 连接中断，未设过期时间会致锁无法释放，多个锁还可能引发死锁。
- **时钟漂移**：Redis 分布式锁依实例时间判断过期，时钟漂移可能使锁失效，可通过 NTP 服务同步所有节点系统时钟来减少影响。

以上扩展解决方案可参考以下文档的扩展分析

[Redis 的 Red Lock 是什么？你了解吗？ - 木子金又二丨的回答记录 - 面试鸭 - 程序员求职面试刷题神器](https://www.mianshiya.com/answer/1826085029072973825/question-answer/1875881520580595714?questionId=1780933295664558082)

[Redis 中如何实现分布式锁？ - 木子金又二丨的回答记录 - 面试鸭 - 程序员求职面试刷题神器](https://www.mianshiya.com/answer/1826085029072973825/question-answer/1875861401376722946?questionId=1780933295660363778)