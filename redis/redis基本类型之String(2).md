#  Redis String类型

Redis 最基本的数据类型就是String, 为什么叫`Binary-safe strings`呢? 

###  1 存储类型

可以用来存储字符串、整数、浮点数

### 2 操作命令

| set key value                                            | 设置值                                                       |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| get key                                                  | 取值                                                         |
| mset key1 value1 key2 value2                             | 设置多个值,(批量操作,原子性)                                 |
| setnx key vaue                                           | 设置值, 如果key存在, 则不成功,基于此可以实现分布式锁, 用 del key 释放锁, |
| set key value [expiration EX seconds PX milliseconds][NX | 使用多参数的方式执行, set lock1 1 EX 10 NX                   |
| incr key <br />incrby key 100                            | （整数）值递增                                               |
| decr key<br/> decrby key 100                             | （整数）值递减                                               |
| set f 2.6 <br />incrbyfloat f 7.3                        | 浮点数增量                                                   |
| mget key1 key2                                           | 获取多个值                                                   |
| strlen key                                               | 获取值长度                                                   |
| append key value                                         | 字符串追加内容                                               |
| getrange key 0 8                                         | 获取指定范围的字符                                           |

### 3存储实现原理

#### 4数据模型

​    set Hello World 为例, 因为Redis 是KV的数据库, 他是通过`hashtable` 实现的(我们把这个叫做外层的哈希).所以每个键值对都会有一个`dictEntry`(源码位置:dict.h),里面指向了key和value 的指针. next 指向下一个 `dictEntry`

```c
typedef struct dictEntry {
    void *key; // key 关键字定义
    union {
        void *val; // value定义
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // 指向下一个键值对节点
} dictEntry;
```

![image-20200327181223803](http://files.luyanan.com//img/20200327181225.png)

key是字符串, 但是redis 并没有直接使用C的字符数组, 而是存储在自定义的SDS中. 

​    value 既不是直接作为字符串存储,也不是直接存储在SDS中, 而是存储在`redisObject`中。 实际上五种常用的数据类型的任何一种,都是通过`redisObject`来存储的. 



##### redisObject

`redisObject`  定义在`src/server.h`文件中

```c
#define OBJ_SHARED_REFCOUNT INT_MAX
typedef struct redisObject {
    unsigned type:4;// 对象的类型, 包括OBJ_STRING、OBJ_LIST、OBJ_HASH、OBJ_SET、OBJ_ZSET
    unsigned encoding:4;// 具体的数据结构
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
                           // 24位, 对象最后一次被明明程序访问的时间,与内存回收有关. 
    int refcount; // 引用计数, 当refcount 为0 的时候,表示该对象已经不被任何对象引用, 则可以进行垃圾回收了. 
    void *ptr; // 指向对象实际的数据结构
} robj;
```



可以使用`type`命令来查看对外的类型

##### 内部编码

![image-20200327182955064](http://files.luyanan.com//img/20200327182956.png)



字符串类型的内部编码有三种: 

1. int, 存储8个字节的长整型(long , 2^63-1)
2. embstr, 代表`embstr`格式的SDS(Simple Dynamic String 简单动态字符串),存储小于44个字节的字符串
3. raw, 存储大于44个字节的字符串(3.2版本之前是39字节. )

```c
/* object.c */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
```





### 5常见问题

#### 问题1: 什么是SDS

Redis 中字符串的实现, 

在3.2以后的版本中, SDS 又有多种数据结构(sds.h), `sdshdr5`、`sdshdr8`、`sdshdr16`、`sdshdr32`、`sdshdr64`,用于存储不同长度的字符串, 分别代表 `2^5=32byte`， `2^8=256byte`，`2^16=65536byte=64KB`，`2^32byte=4GB`

```c
/* sds.h */
struct __attribute__ ((__packed__)) sdshdr8 {
uint8_t len; /* 当前字符数组的长度 */
uint8_t alloc; /*当前字符数组总共分配的内存大小 */
unsigned char flags; /* 当前字符数组的属性、用来标识到底是 sdshdr8 还是 sdshdr16 等 */
char buf[]; /* 字符串真正的值 */
};
```

#### 问题2： 为什么Redis 要使用SDS 实现字符串

我们知道, C语言本身没有字符串类型(只能使用字符数组char[]来实现)

1. 使用字符数组必须先给目标数组分配足够的空间,否则可能会溢出. 
2. 如果要获取字符长度, 必须遍历字符数组, 时间复杂度为O(n)
3. C字符串长度的变更会对字符数组做内存重分配. 
4. 通过从字符串开始到结尾碰到的第一个 '\0'来标记字符结束 , 因此不能保存图片、音频、视频、压缩文件等二进制(bytes)保存的内容, 二进制不安全. 

SDS的特点: 

1. 不用担心内存溢出问题, 如果需要会对SDS进行扩容. 
2. 获取字符串长度时间复杂度为O(1), 因为定义了 len属性
3. 通过"空间预分配"（ sdsMakeRoomFor）和“惰性空间释放”,防止多次重分配内存. 
4. 判断是否结束的标志是len属性(它同样以'\0'结尾是因为这样就可以使用 C语言中函数库操作字符串的函数了），可以包含'\0'。



| C字符串                                    | SDS                                    |
| ------------------------------------------ | -------------------------------------- |
| 获取字符串长度的时间复杂度为O(n)           | 获取字符串的时间复杂度为O(1)           |
| API是不安全的, 可能会造成缓存区溢出        | API是安全的, 不会造成缓存区溢出        |
| 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串N次最多需要执行N次内存重分配 |
| 只能保存文本数据                           | 可以保存文本或者二进制数据             |
| 可以使用`<string.h>` 库中的全部函数        | 可以使用`<string.h>` 库中的部分函数    |



#### 问题3 embstr和raw的区别？

`embstr`的使用只分配一次内存空间(因为RedisObject和SDS 是连续的) , 而`raw` 需要分配两次内存空间(分别是RedisObject和SDS 分配空间)

因此与`raw` 相比, `embstr`的好处在于创建时少分配一次内存空间, 删除时少释放一次空间, 以及对象的所有数据连在一起, 寻找方便. 

而`embstr`的坏处也很明显, 如果字符串的长度增加需要重新分配内存的时候, 整个`RedisObject`和`SDS` 都需要重新分配空间, 因为`Redis` 中的 `embstr` 实现为只读. 



####  问题4. int和embstr 什么时候转化为raw?

当int数据不再是整数，或者大小超过了long的范围(2^63-1=9223372036854775807)的时候,自动转化为`raw`

```bash
127.0.0.1:6379> set k1 1
OK
127.0.0.1:6379> append k1 a
(integer) 2
127.0.0.1:6379> object encoding k1
"raw"
```



####  问题5: 明明没有超过阈值, 为什么变成了`raw`

```bash
127.0.0.1:6379> set k2 a
OK
127.0.0.1:6379> object encoding k2
"embstr" 127.0.0.1:6379> append k2 b
(integer) 2
127.0.0.1:6379> object encoding k2
"raw"
```

对于`embstr` ,由于其实现是只读的, 因此在对 `embstr` 对象进行修改时, 都会先转化为`raw` 再进行修改. 

因为只要改变`embstr` 对象, 修改之后的对象一定是`raw`的, 不论是否达到了44个字节 

#### 问题6: 当长度小于阈值时, 会还原吗? 

关于Redis内部编码的转换, 都符合以下规律: 编码转换在Redis 写完数据时完成, 且转换过程不可逆, 只能从小内存编码转换向大内存编码(但是不包括重新set)

####  问题7: 为什么要对底层的数据结构进行一层包装呢? 

通过封装,可以根据对象的类型动态的选择存储结构和可以使用的命令, 实现节省内存和优化查询速度. 



###  6应用场景

#### 缓存

例如: 热点数据的缓存(例如报表、明星出轨),对象缓存、全页缓存

可以提升热点数据的访问速度



#### 数据共享分布式

因为Redis是分布式独立的服务,可以在多个应用之间共享. 

例如: 分布式session

```xml
<dependency>
<groupId>org.springframework.session</groupId>
<artifactId>spring-session-data-redis</artifactId>
</dependency>
```



####  分布式锁

String 类型的 `setnx` 方法, 只有在不存在的时候才会添加成功, 返回true

http://redisdoc.com/string/set.html 建议用参数的形式

```java
public Boolean getLock(Object lockObject){
jedisUtil = getJedisConnetion();
boolean flag = jedisUtil.setNX(lockObj, 1);
if(flag){
   expire(locakObj,10);
}
return flag;
}
public void releaseLock(Object lockObject){
    del(lockObj);
}
```



####  全局id

INT 类型，`INCRBY`，利用原子

> incrby userid 1000



####  计数器

int类型, `INCR` 方法, 

例如: 文章的阅读量, 微博点赞数, 允许一定的延迟, 先写入到redis 再定时同步到数据库



#### 限流

int类型, `INCR` 方法, 

以访问者的ip和其他信息作为key,访问一次增加一次计数, 超过次数则返回false, 

#### 位统计

String 类型的 `BITCOUNT`

字符是以8位二进制存储的

```bash
set k1 a
setbit k1 6 1
setbit k1 7 0
get k1
```

a 对应的 ASCII 码是 97，转换为二进制数据是 01100001

 b 对应的 ASCII 码是 98，转换为二进制数据是 01100010



因为bit非常节省空间(1 MB=8388608 bit), 可以用来做大数据量的统计, 例如: 在线用户统计、留存用户统计. 

```bash
setbit onlineusers 0 1
setbit onlineusers 1 1
setbit onlineusers 2 0
```

##### 支持按位与、按位或等等操作

```bash
BITOP AND destkey key [key ...] ，对一个或多个 key 求逻辑并，并将结果保存到 destkey 。
BITOP OR destkey key [key ...] ，对一个或多个 key 求逻辑或，并将结果保存到 destkey 。
BITOP XOR destkey key [key ...] ，对一个或多个 key 求逻辑异或，并将结果保存到 destkey 。
BITOP NOT destkey key ，对给定 key 求逻辑非，并将结果保存到 destkey 。
```



\# 计算出 7 天都在线的用户

```bash
BITOP "AND" "7_days_both_online_users" "day_1_online_users" "day_2_online_users" ... "day_7_online_users"
```

