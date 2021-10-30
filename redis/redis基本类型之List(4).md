#  Redis基本类型之List

## 1存储类型

存储有效的字符串(从左到右),元素可以重复, 可以充当队列和栈的角色。 

![image-20200328135937464](http://files.luyanan.com//img/20200328135942.png)



##  2基本命令

| 命令                                          | 作用                                                         | 返回                                                         |
| --------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| lpush key value [value„]                      | 将一个或者多个value值插入到表头,从左往右插入                 | 返回新列表长度                                               |
| rpush key value [value„]                      | 将一个或者多个值插入到表尾(最右边), 从左往右插入             | 新列表长度                                                   |
| lrange key start stop                         | 获取执行key的指定区间的元素                                  | 返回指定的列表                                               |
| lindex key index                              | 获取列表key的下标为index的元素                               | 返回下表的元素，不存在则返回`nil`                            |
| llen key                                      | 获取列表 key的长度                                           | 返回长度,key不存在, 则返回0                                  |
| lrem key count value                          | 根据参数count的值，移除列表中与参数value相等的元素，<br />count>0 ，从列表的左侧向右开始移 除； <br />count<0从列表的尾部开始移除；<br />count=0 移除表中所有与value相等的值。 | 数值，移除的元素个数                                         |
| lset key index value                          | 将列表key 下标为Index的值设置为value                         | 成功返回`ok`, key不存在或者idnex不存在返回异常               |
| `linsert keyAFTER(后) pivot value BEFORE(前)` | 将值value插入到列表key当中位于值pivot之前或之后的位置。key不存在，pivot不在列表中， 不执行任何操作 | 命令执行成功，返回新列表的长度。没有找到 pivot 返回 -1， key 不存在返回 0。 |
| RPOP key                                      | 移除列表的最后一个元素                                       | 返回值为移除的元素                                           |
| RPOPLPUSH source destination                  | 移除列表的最后一个元素，并将该元素添加到另一个列表并返回     |                                                              |
| LPOP key                                      | 移除列表的第一个元素                                         | 返回移除的元素                                               |



## 3存储(实现)原理

  在早期的版本中, 数据量较小的用`ziplist`存储, 达到临界值时转换为`linklist` 进行存储, 分别对应`OBJ_ENCODING_ZIPLIST` 和`OBJ_ENCODING_LINKLIST`

3.2版本之后, 统一用`quicklist` 来存储, `quicklist` 存储了一个双向链表, 每个节点都是一个`ziplist`



```bash
127.0.0.1:6379> object encoding queue
"quicklist"
```

### quicklist

`quicklist`(快速列表)  是`ziplist` 和`linklist`的结合体

`quicklist.h`的`head` 和`tail` 指向双向列表的表头和表尾

```c
/* quicklist is a 40 byte struct (on 64-bit systems) describing a quicklist.
 * 'count' is the number of total entries.
 * 'len' is the number of quicklist nodes.
 * 'compress' is: -1 if compression disabled, otherwise it's the number
 *                of quicklistNodes to leave uncompressed at ends of quicklist.
 * 'fill' is the user-requested (or default) fill factor. */
typedef struct quicklist {
    quicklistNode *head; // 指向双向链表的表头
    quicklistNode *tail;// 指向双向链表的表尾
    unsigned long count;  // 所有的ziplist 中一共存储了多少元素      /* total count of all entries in all ziplists */
    unsigned long len;     // 双向链表的长度, node的数量     /* number of quicklistNodes */
    int fill : 16;             /* fill factor for individual nodes */
    unsigned int compress : 16; // 压缩深度, 0 表示不压缩. /* depth of end nodes not to compress;0=off */
} quicklist;
```

redis.conf 相关参数

| 参数                            | 含义                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| list-max-ziplist-size（fill）   | 正数表示单个 ziplist 最多所包含的 entry 个数。<br /> 负数代表单个 ziplist 的大小，默认 8k。 <br />-1：4KB；-2：8KB；-3：16KB；-4：32KB；-5：64KB |
| list-compress-depth（compress） | 压缩深度，默认是 0。 <br />1：首尾的 ziplist 不压缩；2：首尾第一第二个 ziplist 不压缩，以此类推 |



`quicklistNode`  中的`*zl` 指向一个`ziplist`, 一个`ziplist` 可以存放多个元素

```c
typedef struct quicklistNode {
    struct quicklistNode *prev; // 前一个节点
    struct quicklistNode *next; // 后一个节点
    unsigned char *zl; // 指向实际的 ziplist
    unsigned int sz;   // 当前ziplist 占用多少字节          /* ziplist size in bytes */
    unsigned int count : 16;  // 当前 ziplist 中存储了多少元素, 占 16bit（下同），最大 65536 个   /* count of items in ziplist */
    unsigned int encoding : 2;  // 当前采用了LZF压缩算法压缩节点, 1:RAW,2:LZF /* RAW==1 or LZF==2 */
    unsigned int container : 2;// 2: ziplist, 未来可能支持其他数据结构  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1;// 当前ziplist是不是已经被解压出来临时使用 /* was this node previous compressed? */
    unsigned int attempted_compress : 1;// 测试使用 /* node can't compress; too small */
    unsigned int extra : 10;// 预留给未来使用 /* more bits to steal for future usage */
} quicklistNode;
```



![image-20200328145526338](http://files.luyanan.com//img/20200328145527.png)



## 4应用场景

### 用户消息时间线 `timeline`

![image-20200328145607529](http://files.luyanan.com//img/20200328145608.png)



### 消息队列

List提供了两个阻塞的弹出操作, `BLPOP /BRPOP`, 可以设置超时时间

`BLOPO`:  `：BLPOP key1 timeout`,  移除并获取列表的第一个元素, 如果列表没有元素会阻塞列表到等待超时时间或者发现可弹出的元素为止. 

`BRPOP`: `BRPOP key timeout`:  移除并获取列表的最后一个元素, 如果列表中没有元素会阻塞列表等待超时或者发现可弹出的元素为止. 





