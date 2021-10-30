#   redis基本类型之hash

![image-20200327201703297](http://files.luyanan.com//img/20200327201704.png)

## 1存储类型

包含键值对的无序散列表, value只能是字符串, 不能嵌套其他类型. 

同样是存储字符串, Hash与String有什么区别呢? 

1. 把所有相关的值都聚集到一个key中, 节省内存空间. 
2. 只使用一个key, 减少key冲突. 
3. 当需要批量获取值的时候, 只需要使用一个命令,减少内存/IO/CPU 的消耗. 



**Hash不适用的场景: **

1. Field不能单独设置过期时间
2. 没有bit操作
3. 需要考虑数据量分布的问题(value值非常大的时候, 无法分不到多个节点)

## 2操作命令

| hset key field value                 | 设置值, 如果key不存在, 则新建, 如果field存在, 则覆盖 | 如果field不存在, 返回1, 如果存在, 覆盖并返回0 |
| ------------------------------------ | ---------------------------------------------------- | --------------------------------------------- |
| hget key field                       | 根据指定key 和field 获取值                           | 如果field不存在返回null                       |
| hmset key field value [field value„] | 将多个 field value设置到哈希表的key中                | 成功返回"ok",否则异常                         |
| hmget key field [field„]             | 获取指定key中多个 field 的值                         | 返回指定的值                                  |
| hgetall key                          | 根据key 获取所有的field和value                       | 返回列表                                      |
| hdel key field [field„]              | 根据key和field列表删除指定的值                       | 返回删除的数量                                |
| hkeys key                            | 查看指定key 的所有field                              | 返回列表                                      |
| hvals key                            | 查看指定key 的所有value值                            | 返回value值的列表                             |
| hexists key field                    | 查看指定key 的指定field是否存在                      | 存在返回1, 不存在返回0                        |







## 3存储(实现)原理

Redis的Hash本身也是一个KV的结构, 类似于java中的HashMap	

外层的哈希(Redis KV 的实现), 只用到了`hashtable`.当存储hash数据类型的时候, 我们把他叫做内层的哈希, 内层的哈希有两种数据结构的实现. 

- ziplist  `OBJ_ENCODING_ZIPLIST` 压缩列表 
- hashtable  `OBJ_ENCODING_HT` 哈希表

``` bash
127.0.0.1:6379> hset h2 f aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
(integer) 1
127.0.0.1:6379> hset h3 f aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
(integer) 1
127.0.0.1:6379> object encoding h2
"ziplist" 127.0.0.1:6379> object encoding h3
"hashtable"
```



### ziplist 压缩列表

ziplist 压缩列表是什么? 

>  /* ziplist.c 源码头部注释 */ The ziplist is a specially encoded dually linked list that is designed to be very memory efficient. It stores both strings and integer values, where integers are encoded as actual integers instead of a series of characters. It allows push and pop operations on either side of the list in O(1) time. However, because every operation requires a reallocation of the memory used by the ziplist, the actual complexity is related to the amount of memory used by the ziplist.



ziplist 是一个经过特殊编码的双向链表, 他不存储指向上一个链表节点和指向下一个链表节点的指针,而是存储上一个链表节点的长度和当前节点的长度, 通过牺牲部分的读写性能, 来换取高效的内存空间利用率, 是一种时间换空间的思想. 只用在字段个数少, 字段值小的场景中. 



### ziplist的内部结构?

ziplist.c 源码第 16 

> * <zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>

![image-20200327204806544](http://files.luyanan.com//img/20200327204826.png)

```c
typedef struct zlentry {
unsigned int prevrawlensize; /* 上一个链表节点占用的长度 */
unsigned int prevrawlen; /* 存储上一个链表节点的长度数值所需要的字节数 */
unsigned int lensize; /* 存储当前链表节点长度数值所需要的字节数 */
unsigned int len; /* 当前链表节点占用的长度 */
unsigned int headersize; /* 当前链表节点的头部大小（prevrawlensize + lensize），即非数据域的大小 */
unsigned char encoding; /* 编码方式 */
unsigned char *p; /* 压缩链表以字符串的形式保存，该指针指向当前节点起始位置 */
} zlentry;
```



编码 encoding（ziplist.c 源码第 204 行）

```c
#define ZIP_STR_06B (0 << 6) //长度小于等于 63 字节
#define ZIP_STR_14B (1 << 6) //长度小于等于 16383 字节
#define ZIP_STR_32B (2 << 6) //长度小于等于 4294967295 字节
```

![image-20200327210859431](http://files.luyanan.com//img/20200327210901.png)



### 问题: 什么实时使用 ziplist存储

当hash 的对象同时满足以下两个条件的时候, 使用`ziplist` 编码

1.  所有的键值对的键和值的字符串长度都小于等于64byte(一个英文字母一个字节)
2.  哈希对象中保存的键值对数量小于512个



> /* src/redis.conf 配置 */



```properties
hash-max-ziplist-value 64 // ziplist 中最大能存放的值长度
hash-max-ziplist-entries 512 // ziplist 中最多能存放的 entry 节点数量
```



```c
/* 源码位置：t_hash.c ，当达字段个数超过阈值，使用 HT 作为编码 */
if (hashTypeLength(o) > server.hash_max_ziplist_entries)
hashTypeConvert(o, OBJ_ENCODING_HT);
/*源码位置： t_hash.c，当字段值长度过大，转为 HT */
for (i = start; i <= end; i++) {
if (sdsEncodedObject(argv[i]) &&
sdslen(argv[i]->ptr) > server.hash_max_ziplist_value)
{
hashTypeConvert(o, OBJ_ENCODING_HT);
break;
    }
}
```

   一个哈希长度超过配置的阈值(键和值的长度有>64byte,键值对小于512个)时, 会转换成 哈希表(`hashtable`)



### hahstable(dict)

再Redis中, `hashtabl` 被称之为字典(`dictionary`),他是一个数组+链表的结构. 

源码位置：dict.h

我们知道, Redis 的KV结构是通过一个`dictEntry` 来实现的. 

Redis 又对`dictEntry` 进行了多层封装.

```c
typedef struct dictEntry {
 void *key; /* key 关键字定义 */
  union {
    void *val; uint64_t u64; /* value 定义 */
    int64_t s64; double d;
} v;
 struct dictEntry *next; /* 指向下一个键值对节点 */
} dictEntry;
```

`dictEntry`  放到了`dictht`(hashtable里面)

```c
/* This is our hash table structure. Every dictionary has two of this as we
* implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
 dictEntry **table; /* 哈希表数组 */
 unsigned long size; /* 哈希表大小 */
 unsigned long sizemask; /* 掩码大小，用于计算索引值。总是等于 size-1 */
 unsigned long used; /* 已有节点数 */
} dictht;
```



`ht` 放到了`dict` 里面. 

```c
typedef struct dict {
 dictType *type; /* 字典类型 */
 void *privdata; /* 私有数据 */
 dictht ht[2]; /* 一个字典有两个哈希表 */
 long rehashidx; /* rehash 索引 */
 unsigned long iterators; /* 当前正在使用的迭代器数量 */
} dict;
```

从最底层到最高层: `dictEntry` -> `dictht` -> `dict` ->  `OBJ_ENCODING_HT`

总结: 哈希的存储结构 

![image-20200327215019310](http://files.luyanan.com//img/20200327215020.png)

注意: `dictht`后面是NULL 说明第二个`ht` 还没有用到. `dictEntry*` 后面是NULL 说明还没有hash到这个位置, `dictEntry` 后面是NULL 说明没有发生 hash冲突. 

### 为什么要定义两个哈希表呢? ht[2]

redis的hash默认使用的是`ht[0]`,`ht[1]` 是不会初始化和分配空间的. 

哈希表`dictht` 是用链地址法来解决碰撞问题的. 在这种清空下, 哈希表之间的性能取决于他的大小(size属性), 和他所保存的节点的数量(used属性)之间的比率:

- 比率在1:1 时(一个哈希ht只存储一个节点`entry`),哈希表的性能最好
- 如果节点数量比哈希表的大小要大很多的时候(这个比例用`ratio` 表示, 5表示平均一个`ht` 存储5个`Entry` ),那么哈希表就会退化为多个链表, 哈希表本身的性能优势就不再存在. 

在这种情况下, 需要扩容, Redis里面的这种操作就叫做`rehash`. 

#### `rehash`的步骤

1. 为字符`ht[1]` 哈希表分配空间, 这个哈希表的空间大小取决于要执行的操作, 以及`ht[0]` 当前包含的键值对的数量

    扩展: `ht[1]` 的大小为第一个大于等于`ht[0].used*2`

2.  将所有的`ht[0]` 上的节点`rehash`到`ht[1]` 上, 重新计算hash 值和索引, 然后放入指定的位置. 

3. 将`ht[0]` 全部迁移到`ht[1]`之后, 释放`ht[0]` 的空间, 将`ht[1]` 设置为 `ht[0]` 表, 并创建新的 `ht[1]`,  为下次`rehash` 做准备 . 

### 什么时候触发扩容

负载因子(（源码位置：dict.c):

```c
static int dict_can_resize = 1;
static unsigned int dict_force_resize_ratio = 5;
```

`ratio = used / size`:  已经使用的节点于字节大小的比例

`dict_can_resize`为1并且`dict_force_resize_ratio` 已使用字节树和字典大小之间的比率超过`1:5`, 触发扩容. 

**扩容判断**`_dictExpandIfNeeded`（源码 dict.c）

```c
if (d->ht[0].used >= d->ht[0].size &&
(dict_can_resize ||
 d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
{
return dictExpand(d, d->ht[0].used*2);
}
return DICT_OK;
```

**扩容方法**  `dictExpand`（源码 dict.c）

```c
/* Expand or create the hash table */
int dictExpand(dict *d, unsigned long size)
{
    /* the size is invalid if it is smaller than the number of
     * elements already inside the hash table */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    dictht n; /* the new hash table */
    unsigned long realsize = _dictNextPower(size);

    /* Rehashing to the same table size is not useful. */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* Allocate the new hash table and initialize all pointers to NULL */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* Is this the first initialization? If so it's not really a rehashing
     * we just set the first hash table so that it can accept keys. */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* Prepare a second hash table for incremental rehashing */
    d->ht[1] = n;
    d->rehashidx = 0;
    return DICT_OK;
}
```



**缩容**:  源码: server.c

```c
int htNeedsResize(dict *dict) {
    long long size, used;

    size = dictSlots(dict);
    used = dictSize(dict);
    return (size > DICT_HT_INITIAL_SIZE &&
            (used*100/size < HASHTABLE_MIN_FILL));
}
```



##  4应用场景

### String

String能做的事情， hash都能做。 

### 存储对象类型的数据

比如对象或者一张表的数据, 比`String` 节省了更多的key空间, 也更加便于集中管理. 

###  购物车

![image-20200328135024356](http://files.luyanan.com//img/20200328135042.png)

key：用户 id；field：商品 id；value：商品数量。

 +1：hincr。-1：hdecr。删除：hdel。全选：hgetall。商品数：hlen。