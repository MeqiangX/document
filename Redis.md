# Redis

1. 基本操作

   getkey，setkey，expire过期时间

   基本类型，string，list，set，hash，sortset

   key过期时间设置：expire：秒  pexpire：毫秒

   查看过期时间：ttl：秒 pttl：毫秒

   移除过期时间：persist

2. 缓存击穿

   访问某个数据库中不存在的key，如果为空也会设置，则每次都会打到db上，缓存就没用了

3. 持久化

4. 内存淘汰策略

   Conf中通过maxmemory \<bytes>来设置最大内存，通常会设为物理内存的3/4

   可以在conf中通过maximemory-policy参数来指定 redis内存到达限制的最大值时的内存淘汰策略，参考gc回收策略，策略如下：

   1. Volatile-lru 使用lru算法删除设置过期时间的key，最近最久未使用
   2. Allkeys-lru 使用lru算法删除任何key，通常使用该方法
   3. Volatile-random 随机删除设置过期时间的key
   4. Allkeys-random 无差别随机删除
   5. Volatile-ttl 删除即将过期的key
   6. No eviction 不删除任何key，返回一个写错误（默认选项，一般不会用）

   eg：max memory-policy allkeys-lru

5. 过期删除策略

   a、惰性删除

   > 设置过期时间后不去管它，在get的时候如果过期则删除，否则获得该key
   >
   > 优点是对cpu友好，不用通过线程去监控这个过期时间，只是在需要用到的时候才检查，缺点就是对内存不友好，假如大部分key在过期时间后没用到，则会占用大量的内存，但是却没有使用，占着茅坑不拉屎。

   b、定期删除

   > 定期对一些key进行检查，删除过期key
   >
   > 有点和惰性相反，对cpu不友好，需要通过额外的线程监控过期时间，执行删除操作，有点就是内存会及时释放

   单一的使用某一个策略都没办法满足实际需求，Redis采用惰性删除和定期删除配合使用的删除策略来满足实际需求

   惰性删除：由db.c/expireIfNeeded函数实现，所有键的读写操作都会先调用这个函数检查，过期删除，执行键不存在的操作，未过期则执行键存在的操作

   定期删除：由redis.c/activeExpireCycle函数实现，以一定的频率运行，每次取出一定数量的随机key进行检查，删除其中的过期key

   定期删除函数的运行频率，10次/秒，可通过redis.conf的hz来调整 hz 10（默认）

6. 缓存哨兵

7. 集群





> docker安装启动redis

```shell
$ docker search redis
# 寻找远程仓库的redis镜像
$ docker pull redis
# 拉取最新版本redis
$ docker run --name xq-redis -p 6379:6379 -p redis --requirepass "123456"
# 启动redis镜像
$ docker exec -it xq-redis redis-cli
# 启动客户端。 需要验证密码 auth 123456 即可
```



互联网技巧1：bitmap位图的使用

> Dau 日活统计
>
> 场景假设：产品的用户数为1亿，独立访客为5千万，如果用数据库来存储，需要对每个用户，每天的访问进行记录，如果日活很高，表会迅速扩张（千万级别），按日，周，月还需要count，sum，group，效率较低，消耗数据库资源还难以快速统计目标
>
> 位图可以理解为一个数组，那么统计日活我们可以这么统计 dau20210521[1]=1  dau20210521表示 这是20210521日期的日活统计表，1表示user_id为1的用户活跃，可以估算一下，如果数据库一行记录 user_id记录为4个字节32位，那么至少需要32位*50000000  = 200M，如果用bitmap位图来存，(位图需要考虑全部用户)1位 / 8 * 100000000 = 12M 相差很多
>
> 但是在小数据量的情况下，比如1亿的用户，10万的独立用户
>
> 数据库或者set存储 只需要。32*100000 = 390kb
>
> 而bitmap还是要存1亿个bit 12M
>
> 所以选用什么来存储还是要看具体的数据量

bitmap不是实际的数据类型，而是在字符串类型string上定义的一组向位的操作，字符串是二进制的BLOB，所能存储的最大长度是512MB=4294967296bit，所以最大能设置1024\*1024\*1024\*4 = 2^32个不同的位 

常用方法api：

1、设置key的value的 x位为0/1

```shell
$ setbit key 1 1/0
```

2、获取key的x位的数值

```shell
$ getbit key 1
```

3、获取指定key 区间内的1的个数（这里的区间是指的byte而不是位数bit，注意，1byte=8bit），如不指定则获取全部

```shell
$ bitcount key start end
```

其他应用：

属性状态存储

用户在线状态

用户签到

>  高阶用法：
>
> bitmap还有一些逻辑运算的api
>
> ```shell
> # 做多个bitmap的 and，or，not，xor 与-或-非-异或操作 并保存在destkey中
> $ bitop op destkey key1 key2....
> ```



---

互联网技巧2：基于redis的分布式锁

> redis本身是基于内存的单线程操作缓存中间件，但是客户端程序中是会出现并发，如果在java程序中 同时set key，会造成结果的随机性，这时候可以通过分布式锁来控制多个客户端实现串行执行
>
> set(key,value,"NX","PX",expiretime);
>
> set有五个参数，从前往后依次是：
>
> - key
> - value
> - \[NX/XX\]：设置模式：表示只有当key不存在的时候才设置(NX)和只有在key存在时才设置
> - \[EX/PX/EXAT/PXAT\]：设置过期时间格式- EX(秒)，PX(毫秒)，EXAT(秒时间戳)，PXAT(毫秒时间戳)
> - Expire time：具体是时间数值
>
> 这样就可以通过set来替代原先的setnx() 实现分布式环境下的加锁的问题

```java
private static final String LOCK_SUCCESS = "OK";
    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";

    /**
     * 尝试获取分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {

        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);

        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;

    }
```

设置value为何不是1呢，因为分布式环境下，如果只通过key来判断锁，就会出现，其他线程释放锁的时候，把其他机器的线程上的锁给释放了，需要通过线程的请求标识来确定加锁和解锁的是一个线程，这个requestId可以使用分布式唯一id生成，保存在线程副本变量ThreadLocal中

另外一个问题就是解锁如何实现：

和加锁一样，解锁也需要保证执行的原子性，需要先获取到锁的请求id，如果是当前线程的请求id，则执行删除，否则不做操作，如何保证这些逻辑代码的执行的原子性，就需要通过Lua脚本了，redis在执行Lua脚本的时候，会被当成一个命令执行，直到执行完成。

```java
 private static final Long RELEASE_SUCCESS = 1L;

    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;

    }
```



配合上AQS，Lock，实现自己的程序加锁逻辑，完成分布式环境下的并发安全运行



参考：

https://www.cnblogs.com/xuwc/p/14015398.html

https://mp.weixin.qq.com/s/JLEzNqQsx-Lec03eAsXFOQ

https://www.jianshu.com/p/f302aa345ca8

其他分布式锁解决方案：

基于mysql主键

基于zookeeper

Redis-<kbd><set+lua</kbd>，Redis-Redisson