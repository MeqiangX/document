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