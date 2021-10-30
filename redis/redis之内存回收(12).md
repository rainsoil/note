# redis之内存回收

redis的所有数据都是存储在内存中, 在某些情况下需要对占用的内存进行内存回收. 内存回收主要分为两类: 一类是`key` 过期, 一类是使用内存达到上限(`max_memory`), 触发内存淘汰. 



## 1. 过期策略

要实现key过期, 我们有几种思路

### 1.1 定时过期(主动淘汰)

每个设置过期时间的key都需要创建一个定时器, 到过期时间就会立即清除. 该策略可以立即清除过期的数据, 对内存很友好; 但是会占用大量的CPU资源去处理过期的数据, 从而影响缓存的响应时间和吞吐量. 

### 1.2 惰性过期(被动淘汰)

只有当访问一个key 到时候, 才会判断这个key是不是过期, 过期则删除。 该策略可以最大化的节省CPU资源, 却对内存非常不友好 . 极端情况下可能会出现大量的过期的key没有被访问,从而不会被删除, 占用大量的内存. 

例如,`String`, 在 `getCommand` 里面会调用到`expireIfNeeded`. 

server.c  `expireIfNeeded(redisDb *db, robj *key)`

第二种情况, 每次写入key的时候, 发现内存不够, 调用`activeExpireCycle` 释放一部分内存. 

expire.c   `activeExpireCycle(int type)`



###  1.3  定时过期

源码: `server.h`

```c
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    dict *dict;      //所有的键值对            /* The keyspace for this DB */
    dict *expires;      // 设置了过期时间的键值对        /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```



每隔一定的时间, 会扫描一定数量的数据库的`expires` 字典中一定数量的key, 并清除其中已过期的key. 该策略是前两者的一个折中方案, 通过调整定时扫描的时间间隔和每次扫描的限定耗时, 可以在不同情况下使得CPU和内存资源达到最优的平衡效果. 

redis 中同时使用了惰性过期和定时过期这两种过期策略. 

**如果都不过期, redis内存满了怎么办? **



## 2. 淘汰策略

redis的内存淘汰策略, 是指当内存使用达到最大内存极限时, 需要使用淘汰算法来决定清理掉哪些数据, 以保证新数据的存入. 

### 2.1 最大内存设置

redis.conf 参数配置：

```properties
# maxmemory <bytes>
```

如果不设置`maxmemory` 或者设置为0, 64位系统不限制内存， 32位系统最多使用3GB内存. 

动态修改: 

```bash
redis> config set maxmemory 2GB

```

达到最大内存怎么办? 

###  2.2 淘汰策略

https://redis.io/topics/lru-cache

**redis.conf**

```properties
# maxmemory-policy 
# volatile-lru -> Evict using approximated LRU among the keys with an expire set. 
# allkeys-lru -> Evict any key using approximated LRU. 
# volatile-lfu -> Evict using approximated LFU among the keys with an expire set. 
# allkeys-lfu -> Evict any key using approximated LFU. 
# volatile-random -> Remove a random key among the ones with an expire set. 
# allkeys-random -> Remove a random key, any key. 
# volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
# noeviction -> Don't evict anything, just return an error on write operations.
```



先从算法上看: 

`LRU`: `Least Recently Used`, 最近最少使用, 判断最近被使用的时间， 目前最远的数据优先被淘汰. 

`LFU`: `Least Frequently Used` , 最不常用, 4.0 版本新增

`random`: 随机删除

| 策略              | 含义                                                         |
| :---------------- | :----------------------------------------------------------- |
| `volatile-lru`    | 根据 LRU 算法删除设置了超时属性（expire）的键，直到腾出足够内存为止。如果没有 可删除的键对象，回退到 noeviction 策略。 |
| `allkeys-lru`     | 根据 LRU 算法删除键，不管数据有没有设置超时属性，直到腾出足够内存为止。 |
| `volatile-lfu`    | 在带有过期时间的键中选择最不常用的。                         |
| `allkeys-lfu`     | 在所有的键中选择最不常用的，不管数据有没有设置超时属性。     |
| `volatile-random` | 在带有过期时间的键中随机选择。                               |
| `allkeys-random`  | 随机删除所有键，直到腾出足够内存为止。                       |
| `volatile-ttl`    | 根据键值对象的 ttl 属性，删除最近将要过期数据。如果没有，回退到 noeviction 策略。 |
| `noeviction 默`   | 默认策略，不会删除任何数据，拒绝所有写入操作并返回客户端错误信息（error）OOM command not allowed when used memory，此时 Redis 只响应读操作。 |

如果没有符合前提条件的key被淘汰, 那么`volatile-lru`、`volatile-random` 、 `volatile-ttl` 相当于`noeviction`（不做内存回收）

动态修改淘汰策略: 

```bash
redis> config set maxmemory-policy volatile-lru

```

建议使用`volatile-lru`, 在保证正常服务的情况下, 优先删除最近最少使用的key

### 2.3 LRU 淘汰原理

如果基于传统的`LRU` 算法实现redis `LRU` 会有什么问题? 

需要额外的数据结构存储, 消耗内存. 

`redis LRU` 对传统的`LRU` 算法进行了改良, 通过随机采样来调整算法的精度. 

如果淘汰策略是`LRU`,则根据配置的采样值`maxmemory_samples`（默认是5个）, 随机从数据库中选择m个key, 淘汰其中热度最低的key对应的缓存数据, 所以采样参数m 配置的数值越大, 就越能精确的查找到待淘汰的缓存数据, 但是也消耗更多的CPU计算, 执行效率降低. 



问题: 如何找出热度最低的数据? 

redis中所有对象结构都有一个`lru`字段, 且使用了`unsigned`的低24位, 这个字段用来记录对象的热度. 对象被创建时会记录`lru`值, 在被访问的时候会更新`lru` 的值,但是不是获取系统当前的时间戳, 而是设置为`全局变量`的`server.lruclock`的值. 

源码：server.h

```c
#define OBJ_SHARED_REFCOUNT INT_MAX
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

`server.lruclock`的值是怎么来的呢？ 

redis中有个定时处理的函数`serverCron`, 默认每100毫秒调用函数`updateCachedTime`  更新一次全局变量`server.lruclock`的值, 他记录的是当前`unix`的时间戳

源码：server.c

```c

/* We take a cached value of the unix time in the global state because with
 * virtual memory and aging there is to store the current time in objects at
 * every object access, and accuracy is not needed. To access a global var is
 * a lot faster than calling time(NULL).
 *
 * This function should be fast because it is called at every command execution
 * in call(), so it is possible to decide if to update the daylight saving
 * info or not using the 'update_daylight_info' argument. Normally we update
 * such info only when calling this function from serverCron() but not when
 * calling it from call(). */
void updateCachedTime(int update_daylight_info) {
    server.ustime = ustime();
    server.mstime = server.ustime / 1000;
    time_t unixtime = server.mstime / 1000;
    atomicSet(server.unixtime,unixtime);

    /* To get information about daylight saving time, we need to call
     * localtime_r and cache the result. However calling localtime_r in this
     * context is safe since we will never fork() while here, in the main
     * thread. The logging function will call a thread safe version of
     * localtime that has no locks. */
    if (update_daylight_info) {
        struct tm tm;
        time_t ut = server.unixtime;
        localtime_r(&ut,&tm);
        server.daylight_active = tm.tm_isdst;
    }
}
```

**问题: 为什么不获取精准的时间戳而是获取全局的变量呢? 不会有延迟的问题吗 ?**

这样函数`lookupKey` 中更新数据的`lru` 热度值时, 就不会每次调用系统的函数`time`, 可以提高执行效率. 

OK,当对象里面已经有了`LRU` 字段的值， 就可以评估对象的热度了. 

函数`estimateObjectIdleTime` 评估执行对象的`lru` 热度,思想就是对象的`lru`值和全局的`server.lruclock`的差值越大(越久没有得到更新), 该对象热度就越低. 

源码： `evict.c`

```c
/* Given an object returns the min number of milliseconds the object was never
 * requested, using an approximated LRU algorithm. */
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) *
                    LRU_CLOCK_RESOLUTION;
    }
}
```

`server.lruclock` 只有24位, 按秒位单位才能存储194天, 当超过24bit能表示最大时间的时候, 他会从头开始计算. 

server.h

```c
#define LRU_CLOCK_MAX ((1<<LRU_BITS)-1) /* Max value of obj->lru */

```

在这种情况下, 可能会出现对象的`lru`的值大于`server.lruclock` 的情况,如果遇到这种情况, 那么就两个相加而不是相减来求最久的值, 

为什么不适用常规的哈希呢 + 双向链表的方式实现呢? 需要额外的数据结构, 消耗资源. 而redis `lru` 算法在`sample` 为10 的情况下,已经能接近传统`LRU`算法了., 

![image-20200330220757796](http://files.luyanan.com//img/20200330220759.png)



**问题：除了消耗资源, 传统`LRU` 还有什么问题呢? **

如图, 假设A 在10秒中被访问了5次, 而B在10秒中被访问了3次。因为B 最后一次被访问的时间比A要晚, 在同等的情况下, A 反而先被回收.

![image-20200330221217158](http://files.luyanan.com//img/20200330221218.png)

**问题:要实现基于访问频率的淘汰机制, 该怎么做呢? **



### 2.4 `LFU`

`server.h`

```c
#define OBJ_SHARED_REFCOUNT INT_MAX
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```



当这24bits 用做`LFU` 时, 被分作两个部分. 

- 高16位用来记录访问时间(单位位分钟, `ldt`,`last decrement time` )
- 低8位来记录访问频率,简称counter(`logc`,`ogistic counter` )

`counter` 是用基于概率的对数计数器实现的, 8位可以表示百万次的访问频率. 

对象被读写的时候,`lfu`的值会被更新. 

`db.c`--`lookupKey`

```c
/* Update LFU when an object is accessed.
 * Firstly, decrement the counter if the decrement time is reached.
 * Then logarithmically increment the counter, and update the access time. */
void updateLFU(robj *val) {
    unsigned long counter = LFUDecrAndReturn(val);
    counter = LFULogIncr(counter);
    val->lru = (LFUGetTimeInMinutes()<<8) | counter;
}
```

增长的速率由`lfu-log-factor` 越大, `counter` 增长就越慢. 

redis.conf 配置文件

```properties
# lfu-log-factor 10

```



如果计数器只会递增不会递减, 也不能体现对象的热度. 没有被访问的时候, 计算器怎么递减呢? 

减少的值由衰减因子`lfu-decay-time`（分钟）来控制, 如果值是1的话, N 分钟没有访问就要减少N 

redis.conf 配置文件

```properties
# lfu-decay-time 1
```

