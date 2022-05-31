

**R**emote **D**ictionary **S**erver 远程词典服务器，一个基于内存的键值型NoSQL数据库<br />key value **键值**数据库 **NoSql**的数据库(非结构化)

- 键值型
- 单线程，每个命令具有原子性
- 低延迟，速度快（基于内存、IO多路复用、良好的编码》》C语言编写 源码写的很好）
- 支持数据持久化（定期从内存持久化到磁盘）
- 支持主从集群、分片集群
- 支持多语言的客户端


docker exec -it redis-l8bs redis-cli -a redispw

<a name="qipqI"></a>
#### STRING



<a name="dml9e"></a>
#### KEY

<a name="u2mWB"></a>
#### HASH


<a name="Jr9aT"></a>
#### LIST

- 有序
- 元素可重复
- 插入和删除速度快
- 查询速度一般


<a name="wG3eZ"></a>
#### SET

- 无序
- 元素不可重复
- 查找快
- 支持交集，并集，差集等功能


<a name="Iu0Dy"></a>
#### SORT SET

- 可排序
- 元素不重复
- 查询速度快

因为可排序性，经常被用来实现排行榜<br />

<a name="hGUS4"></a>
#### JEDIS


jedis不是线程安全因此要用线程池JerdisPoolConfig

SpringDataRedis<br />spring封装的模块

- 提供了对不同客户端的整合（Lettuce和Jedis）
- 提供了RedisTemplate统一API来操作Redis
- 支持Redis的发布订阅模式
- 支持Redis哨兵和Redis集群
- 支持基于Lettuce的响应式编程
- 支持基于JDK,JSON,字符串,Spring对象的数据序列化和反序列化
- 支持基于Redis和JDKCollection实现

<a name="Jy4ze"></a>
#### redis的更新策略

- 删除缓存 在有查询操作的时候再新增缓存
- 保证数据库和缓存的原子性，（对于分布式系统要用TCC等分布式事务）
- 保证线程的安全（先操作数据库再删除缓存）

缓存更新的合理方案

1. 低一致性：使用redis自带的内存淘汰机制
1. 高一致性：主动更新，并以超时剔除为兜底方案

**读操作**

- 缓存命中就直接返回
- 缓存未命中则查询数据库，并写入缓存，设定超时时间

**写操作**

- 先写数据库，然后删除缓存
- 要确保数据库和缓存操作的原子性

<a name="ISomY"></a>
#### 缓存穿透
客户端请求的数据在缓存中和数据库中不存在，这样缓存永远不会生效，这些请求都会达到数据库

解决方案

- 缓存空对象（ 最好设置TTL）

1.额外的内存消耗 <br />       2.可能造成短期不一致（空KEY又有值了，但是还是会返回null）

- 布隆过滤

1.内存占用少，没有多余的key<br />       2.缺点：实现复杂，存在误判的可能<br />
<a name="O3lSo"></a>
#### 缓存雪崩
指统一时间段大量的缓存key同时失效或者redis服务宕机，导致大量请求达到数据库，带来巨大压力

解决方案

- 给不同的key添加TTL随机值
- 利用redis集群提高服务的可用性(哨兵模式 主从同步)
- 给缓存业务添加降级限流策略
- 给业务添加多级缓存（nginx层缓存，jvm缓存 等等）

<a name="oa7K8"></a>
#### 缓存击穿
缓存击穿问题也叫做热点key问题，就是一个被**高并发访问**并且**缓存重建业务较复杂**的key突然失效了，无数的请求会瞬间给数据库带来巨大的压力。

解决方案

- 互斥锁（性能不好，并发太多的话，可能都在等待互斥锁的释放---setnx ）

      -----setnx是redis的互斥锁，jedis的互斥锁是setIfAbsent<br />

- 逻辑过期（不设置TTL;设置一个expire字段）



案例<br />基于互斥锁方式解决缓存击穿的问题

```java

@Test
Result testRedisSetnx(String key) {
    String lockKey = "lock:result" + key;
    Result result = null;
    boolean isLock = tryLock(lockKey);//设置互斥锁
    if (!isLock) { //判断是否加锁成功
        try {
            Thread.sleep(5);//失败，休眠重试，休眠是为了等待其他加锁的程序执行时间
            return testRedisSetnx(key)//递归执行
            //TODO ....业务逻辑的执行 获取和封装Result对象用来返回
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        } finally {
            unLock(lockKey);//释放锁
        }
    }
    return result;
}


/**
 * 尝试加锁
 * 如果没有之前没有加锁，则可以加锁成功
 * 命令行方法名为 setnx lockKey 50
 */
private boolean tryLock(String lockKey) {
    return stringRedisTemplate.opsForValue().setIfAbsent(lockKey,"zss",50, TimeUnit.MILLISECONDS);
}


/**
 * 解锁操作既直接删除锁
 */
private void unLock(String lockKey){
    stringRedisTemplate.delete(lockKey);
}

```
**产生的并发问题**<br />setnx lock 1<br />EXPIRE lock 50<br />**以上两个操作虽然设置了互斥锁和超时时间，但是两个操作不是原子性的**

**应对并发的情况应该用原子性的操作 如下**
```java
set lock 1 EX 50 NX //设置锁
    //todo业务逻辑
DEL lock //释放锁

```
**存在的问题**<br />极端情况：如果线程持续执行了太长时间超过了锁的超时释放时间，这时候线程1锁已经释放了，但是线程1还未执行结束，那么线程2执行就会获取锁，这时候线程1执行结束了去释放锁就会释放了线程2的锁，那线程三就能获得锁了，但是此时线程2还未执行结束，这时候就会出现线程不一致问题。


```java

    @Override
    public boolean tryLock(long time) {
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        Boolean lock = redisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId + "", time, TimeUnit.MINUTES);
        return Boolean.TRUE.equals(lock);
    }

    @Override
    public void unlock() {
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        String id = redisTemplate.opsForValue().get(KEY_PREFIX + name);
        if(threadId.equals(id)){
            redisTemplate.delete(KEY_PREFIX + name);
        }
    }
```
但是即使是加上了锁标识，还是可能出现线程不安全的情况如下图<br />那就是判断锁标识和释放锁的动作要是原子性（虽然情况很极端）<br />

**解决方案**<br />Lua脚本<br />Redis提供了Lua脚本功能，在一个脚本中可以编写多个Redis命令确保多个命令执行的原子性

**语法如下**<br />EVAL "return redis.call('set',KEYS[1],ARGV[1])" 1 name ssz<br />EVAL是执行脚本的命令redis.call是redis提供的执行命令的方法<br />首先编写一个lua脚本
```lua
//获取锁中的线程提示 get key

local id = redis.call('get',KEYS[1])
//比较线程标识与锁中的标识是否一致
if(id == AVG[1]) then
  //释放锁
  return redis.call('del',KEYS[1])
end
return 0

```
基于java的代码调用API就可以了<br />
```java

    private static final String KEY_PREFIX = "lock:";
    private static final String ID_PREFIX = UUID.randomUUID().toString() + "-";

    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }


    @Override
    public boolean tryLock(long time) {
        String threadId = ID_PREFIX + Thread.currentThread().getId();
        Boolean lock = redisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId + "", time, TimeUnit.MINUTES);
        return Boolean.TRUE.equals(lock);
    }

    @Override
    public void unlock() {
        redisTemplate.execute(UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),
                ID_PREFIX + Thread.currentThread().getId());
/*        String threadId = ID_PREFIX + Thread.currentThread().getId();
        String id = redisTemplate.opsForValue().get(KEY_PREFIX + name);
        if(threadId.equals(id)){
            redisTemplate.delete(KEY_PREFIX + name);
        }*/
    }
```

高并发测试工具 JMeter


<a name="OnTN5"></a>
#### 全球ID唯一生产策略

- UUID
- REDIS 自增
- 雪花算法

```java

public class RedisIdWorker {

    private StringRedisTemplate redisTemplate;

    private static final int BEGIN_TIMESTAMP = 1640995200;
    private static final int COUNT_BITS = 32;


    public long nextId(String keyPrefix) {
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timeTamp = nowSecond - BEGIN_TIMESTAMP;
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        //自增长 incr
        long count = redisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
        //拼接返回
        return timeTamp << COUNT_BITS | count;
    }


    public static void main(String[] args) {
        LocalDateTime dateTime = LocalDateTime.of(2022, 1, 1, 0, 0, 0);
        System.out.println(dateTime.toEpochSecond(ZoneOffset.UTC));//BEGIN_TIMESTAMP 1640995200
    }
}

```
<a name="glea2"></a>
####

