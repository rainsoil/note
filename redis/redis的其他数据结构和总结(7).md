# 其他数据结构

##  1. BitMaps

`BitMaps` 是在字符串类型上面定义的操作. 一个字节由8个二进制组成

![image-20200328161332775](http://files.luyanan.com//img/20200328161334.png)

>  set k1 a 

获取 value 在 offset 处的值（a 对应的 ASCII 码是 97，转换为二进制数据是 01100001）

修改二进制数据（b 对应的 ASCII 码是 98，转换为二进制数据是 01100010）

```bash
setbit k1 6 1
setbit k1 7 0
get k1
```



统计二进制位中 1 的个数

```bash
bitcount k1

```

获取第一个 1 或者 0 的位置

```bash
bitpos k1 1
bitpos k1 0
```

BITOP 命令支持 AND 、 OR 、 NOT 、 XOR 这四种操作中的任意一种参数：

 BITOP AND destkey srckey1 … srckeyN ，对一个或多个 key 求逻辑与，并将结果保存到 destkey 

BITOP OR destkey srckey1 … srckeyN，对一个或多个 key 求逻辑或，并将结果保存到 destkey 

BITOP XOR destkey srckey1 … srckeyN，对一个或多个 key 求逻辑异或，并将结果保存到 destkey 

BITOP NOT destkey srckey，对给定 key 求逻辑非，并将结果保存到 destkey



###  应用场景

用户访问统计

在线用户统计



##  2.Hyperloglogs

`Hyperloglogs`  提供了一种不太准确的基数统计方法, 比如统计网站的UV,存在一定的误差, 

##  3. Streams

5.0 推出的数据类型。支持多播的可持久化的消息队列，用于实现发布订阅功能，借 鉴了 kafka 的设计。



# 总结

##  1.数据结构总结



| 对象         | 对象type属性值 | type命令输出 | 底层可能的数据结构                                           | object_encoding                      |
| ------------ | -------------- | ------------ | ------------------------------------------------------------ | ------------------------------------ |
| 字符串对象   | OBJ_STRING     | `string`     | OBJ_ENCODING_INT<br />OBJ_ENCODING_EMBSTR<br />OBJ_ENCODING_RAW | int<br />embstr<br />raw             |
| 列表对象     | OBJECT_LIST    | `list`       | OBJ_ENCODING_QUICKLIST                                       | quicklist                            |
| 哈希对象     | OBJ_HASH       | `hash`       | OBJ_ENCODING_ZIPLIST<br />OBJ_ENCODING_HT                    | `zuplist`<br />`hashtable`           |
| 集合对象     | OBJ_SET        | `set`        | OBJ_ENCODING_INTSET<br />OBJ_ENCODING_HT                     | `intset`<br />`hashtable`            |
| 有序集合对象 | OBJ_ZSET       | `zset`       | OBJ_ENCODING_ZIPLIST<br />OBJ_ENCODING_SKIPLIST              | `ziplist`<br />`skiplist（包含 ht）` |



##  2.编码转换特点

| 对象       | 原始编码                                                     | 升级编码           |      |
| ---------- | ------------------------------------------------------------ | ------------------ | ---- |
| 字符串对象 | INT                                                          | embstr             | raw  |
| 字符串对象 | 整数并且小于long 2^63-1                                      | 超过44个字节被修改 |      |
| 哈希对象   | ziplist                                                      | hashtable          |      |
| 哈希对象   | 键和值的长度不超过64字节,键值对个数不超过512, 同时满足       |                    |      |
| 列表对象   | quicklist                                                    |                    |      |
| set对象    | intset                                                       | hashtable          |      |
| set对象    | 元素都是整数类型, 元素个数小于512个, 同时满足                |                    |      |
| zset对象   | ziplist                                                      | spiklist           |      |
| zset对象   | 元素个数不超过128个, 任何一个member 的长度不超过64, 同时满足 |                    |      |

