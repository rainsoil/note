# redis基本类型之set

## 1存储类型:

`String`类型 无序不重复集合， 最多可以存储`2^32-1`(40亿左右)



## 2操作命令

| sadd key member [member…] | 将一个或者多个member 元素添加到集合key中                     | 返回添加元素的个数                       |
| ------------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| smembers key              | 获取集合中所有的成员元素                                     |                                          |
| sismember key member      | 判断 member 是集合key 的成员                                 | 是则返回1, 不是则返回0                   |
| scard key                 | 获取集合中成员的个数                                         | 成员个数                                 |
| srem key member [member…] | 删除集合中一个或者多个member 元素,不存在的成员则被忽略       | 返回成功删除的成员的个数                 |
| srandmember key [count]   | 随机返回一个元素, 元素不会被删除<br />提供了 count时，count 正数, 返回包含 count 个数元素的集合， 集合元素各不相同。<br />count 是负数，返回一个 count 绝对 值的长度的集合， 集合中元素可能会重复多次。 | 一个元素；多个元素的集合                 |
| spop key [count]          | 随机从集合中删除一个元素, count 是删除的元素个数。           | 被删除的元素，key 不存在或空集合返回 nil |



## 3存储(实现)原理

Redis 使用 `intset` 或者`hashtable`存储`set`, 如果元素都是整型类型， 则使用`intset`存储, 如果不是整数类型，则使用`hashtable` (数组+链表的结构)存储.

问题: KV怎么存储set的元素? key是元素的值? value为null. 

如果元素个数超过512个, 也会用`hashtable`  存储 .

配置文件 redis.conf

> set-max-intset-entries 512



```bash
127.0.0.1:6379> sadd iset 1 2 3 4 5 6
(integer) 6
127.0.0.1:6379> object encoding iset "intset" 127.0.0.1:6379> sadd myset a b c d e f
(integer) 6
127.0.0.1:6379> object encoding myset "hashtable"
```





##  4应用场景

### 抽奖

随机获取元素



### 点赞、签到、打卡

![image-20200328152356354](http://files.luyanan.com//img/20200328152357.png)

这条微博的 ID 是 t1001，用户 ID 是 u3001。

 用 like:t1001 来维护 t1001 这条微博的所有点赞用户。 

点赞了这条微博：sadd like:t1001 u3001 

取消点赞：srem like:t1001 u3001 

是否点赞：sismember like:t1001 u3001 

点赞的所有用户：smembers like:t1001 

点赞数：scard like:t1001 比关系型数据库简单许多。

### 商品标签

用 tags:i5001 来维护商品所有的标签。

![image-20200328152527621](http://files.luyanan.com//img/20200328152529.png)

sadd tags:i5001 画面清晰细腻 

sadd tags:i5001 真彩清晰显示屏 

sadd tags:i5001 流畅至极