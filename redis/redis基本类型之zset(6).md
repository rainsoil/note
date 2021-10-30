# redis基本类型之zset

zset是一个有序的不重复集合



## 1操作命令



| `zadd key [NX|XX] [CH] [INCR] score member [score member…]` | ZADD 参数（options） (>= Redis 3.0.2)<br />ZADD 命令在key后面分数/成员（score/member）对前面支持一些参数，他们是：<br />XX: 仅仅更新存在的成员，不添加新成员。<br />NX: 不更新存在的成员。只添加新成员。<br />CH: 修改返回值为发生变化的成员总数，原始是返回新添加成员的总数 (CH 是 changed 的意 思)。 | 在通常情况下，ZADD返回值只计算新添加成员的数量。 |
| -------------- | -------------------------------------------- | ---- |
| ZINCRBY key increment member | 对有序集合中指定成员的分数加上增量 increment<br />可以通过传递一个负数值 increment ，让分数减去相应的值，比如 ZINCRBY key -5 member ，就是让 member 的 score 值减去 5<br />当 key 不存在，或分数不是 key 的成员时， ZINCRBY key increment member 等同于 ZADD key increment member 。<br />分数值可以是整数值或双精度浮点数。 |      |
| zrange key start stop [WITHSCORES] | 查询有序集合，指定区间的内的元素。集合成员按 score 值从小到大来排序。<br />start，stop 都是 从 0 开始。0 是第一个元素，1 是第二个元素，依次类推。<br />以 -1 表示最后一个成员，-2 表示倒数第二 个成员。WITHSCORES 选项让 score 和 value 一同返回。 | 自定区间的成员集合 |
| `zrevrange key start stop [WITHSCORES]` | 返回有序集 key 中，指定区间内的成员。<br />其中成员的位置按 score 值递减(从大到小)来排列。 其它同 zrange 命令。 | 自定区间的成员集合                               |
| `zrem key member [member…]`                                  | 删除有序集合 key 中的一个或多个成员，不存在的成员被忽略      | 被成功删除的成员数量，不包括被忽略的成员。 |
| zcard key | 获取有序集 key 的元素成员的个数                              | key 存在返回集合元素的个数， key 不存在，返回 0 |
| zrangebyscore key min max [WITHSCORES ] [LIMIT offset count] | 获取有序集 key 中，所有 score 值介于 min 和 max 之间（包括 min 和 max）的成员，有序 成员是按递增（从小到大）排序。<br />min ,max 是包括在内 ， 使用符号 ( 表示不包括。<br />min ， max 可以使用 -inf ，+inf 表示 最小和最大 limit 用来限制返回结果的数量和区间。<br />withscores 显示 score 和 value | 指定区间的集合数据 |
| zrevrangebyscore key max min [WITHSCORES ] [LIMIT offset count] | 返回有序集 key 中， score 值介于 max 和 min 之间(默认包括等于 max 或 min )的所有的成 员。<br />有序集成员按 score 值递减(从大到小)的次序排列。其他同 zrangebyscore |      |
| zcount key min max | 返回有序集 key 中， score 值在 min 和 max 之间(默认包括 score 值等于 min 或 max ) 的成员的数量 |      |



## 2存储类型

![image-20200328154604160](http://files.luyanan.com//img/20200328154605.png)

sorted set，有序的 set，每个元素有个 score。
      score 相同时，按照 key 的 ASCII 码排序。 

数据结构对比：

| 数据结构 | 是否允许重复元素 | 是否有序 | 有序实现方式 |
| -------- | ---------------- | -------- | ------------ |
| list     | 是               | 是       | 索引下标     |
| set      | 否               | 否       | 无           |
| zset     | 否               | 是       | 分值 score   |



## 3存储(实现)原理

同时满足以下条件的时候使用`ziplist` 编码

- 元素的数量小于128个
- 所有`member`的长度都小于64字节

在`ziplist` 的内部, 按照`score` 排序递增来存储。 插入的时候要移动之后的数据

对应 redis.conf 参数：

```properties
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
```

 超过阈值后, 使用`	skiplist` + `dict`存储



**问题: 什么是`skiplist`（跳跃表）**

我们先来看一下有序链表

![image-20200328155250036](http://files.luyanan.com//img/20200328155251.png)

在这样的一个链表中, 如果我们要查找某个数据, 那么需要从头开始逐个进行比较， 直到找到包含数据的那个节点, 或者找到第一个比给定数据大的节点(没找到), 也就是说, 时间复杂度为`O(n)`. 同样 , 当我们要插入新的数据的时候, 也要经历同样的查找过程, 而从确定插入的位置. 

而二分查找法只适用于有序数组, 不适用于链表 

假设我们每相邻的两个节点增加一个指针(**或者理解为有三个元素进入了第二层**), 让指针指向下下个节点. 

![image-20200328155947897](http://files.luyanan.com//img/20200328155948.png)

这样所有新增加的指针连成了一个新的链表 ,但他所包含的节点个数只有原来的一半（上图中是 7, 19, 26）。在插入一个数据的时候, 决定要放在那一层, 取决于一个算法（在 redis 中 t_zset.c 有一个 zslRandomLevel 这个方法）。

现在当我们想查找数据的时候, 可以先沿着这个新的链表进行查找。当碰到比待查找的数据大的节点的时候, 再回到原来的链表中的下一层进行查找. 比如: 当我们想查找23, 查找的路径是沿着下图中标红的指针所指向的方法进行的. 

![image-20200328160445056](http://files.luyanan.com//img/20200328160446.png)

1. 23 首先和7比较, 再和19比较, 比他们都大, 继续向后比较. 
2. 但23和26比较的时候, 比26要小, 因此回到下面的链表(原链表), 与22 比较. 
3. 23要比22大, 沿着下面的指针继续向后和26比, 23比26小, 说明待查询的数据23 在原链表中不存在. 



在这个查找的过程中, 由于新增加的指针, 我们不再需要与链表中的每个元素逐个进行比较, 需要比较的节点数大概只有原来的一半, 这就是跳跃表 . 

为什么不用`AVL`树或者红黑树? 因为`skiplist` 更加简洁. 

源码: server.h

```c
typedef struct zskiplistNode {
sds ele; /* zset 的元素 */
double score; /* 分值 */
struct zskiplistNode *backward; /* 后退指针 */
struct zskiplistLevel {
struct zskiplistNode *forward; /* 前进指针，对应 level 的下一个节点 */
unsigned long span; /* 从当前节点到下一个节点的跨度（跨越的节点数） */
} level[]; /* 层 */
} zskiplistNode;
typedef struct zskiplist {
struct zskiplistNode *header, *tail; /* 指向跳跃表的头结点和尾节点 */
unsigned long length; /* 跳跃表的节点数 */
int level; /* 最大的层数 */
} zskiplist;
typedef struct zset {
    dict *dict;
zskiplist *zsl;
} zset;
```



随机获取层数的函数

源码：t_zset.c

```c
int zslRandomLevel(void) {
int level = 1;
while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
level += 1;
return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```



##  4应用场景

### 排行榜

id 为 6001 的新闻点击数加 1：zincrby hotNews:20190926 1 n6001

获取今天点击最多的 15 条：zrevrange hotNews:20190926 0 15 withscores

![image-20200328161203888](http://files.luyanan.com//img/20200328161205.png)