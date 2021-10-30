# redis的持久化机制

https://redis.io/topics/persistence

redis速度快 , 很大一部分原因是因为他所有的数据都存储在内存中, 如果断电或者宕机, 都会导致内存中的数据丢失. 为了实现重启后数据不丢失, redis提供了两种数据持久化方案, 一种是`RDB` 快照(`Redis DataBase`), 另外一种是`AOF（Append Only File）`.

## 1. `RDB`

`RDB`是redis默认的持久化方案, 当满足一定条件后, 会把当前内存中的数据写入到磁盘中, 生成一个快照文件`dump.rdb`. redis 重启会通过加载`dump.rdb` 文件恢复数据. 

那什么时候写入`dump.rdb` 文件呢? 



###  1.1 `RDB`触发

#### 1.1.1 自动触发

##### 1.1.1.1 配置规则触发

`redis.conf` 配置文件中的`SNAPSHOTTING` 定义了触发把数据保存到磁盘的触发频率. 

如果不需要`RDB` 方案, 注释`save` 或者配置成空字符串""

```properties
save 900 1 # 900 秒内至少有一个 key 被修改（包括添加）
save 300 10 # 400 秒内至少有 10 个 key 被修改
save 60 10000 # 60 秒内至少有 10000 个 key 被修改
```

> 注意上面的配置是不冲突的, 只要任一一个满足都会触发的. 

`RDB` 文件位置和目录

```properties
# 文件路径，
dir ./
# 文件名称
dbfilename dump.rdb
# 是否是 LZF 压缩 rdb 文件
rdbcompression yes
# 开启数据校验
rdbchecksum yes
```



| 参数           | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| dir            | rdb 文件默认在启动目录下（相对路径）<br />config get dir 获取 |
| dbfilename     | 文件名称                                                     |
| rdbcompression | 开启压缩可以节省存储空间，但是会消耗一些 CPU 的计算时间，默认开启 |
| rdbchecksum    | 使用 CRC64 算法来进行数据校验，但是这样做会增加大约 10%的性能消耗，如果希望获取到最 大的性能提升，可以关闭此功能。 |



#####  1.1.1.2  `shutdonw` 触发, 

保证服务器正常关闭

##### 1.1.1.3  `flushall`触发

``RDB`文件里面是空的, 没什么意义



#### 1.1.2  手动触发

如果我们需要重启服务或者迁移数据, 这个时候就需要手动触发`RDB`快照保存。redis 提供了两条命令:

1. `save`

    `save`在生成快照的时候会阻塞当前的redis服务器, redis 不能处理其他的命令。 如果内存中的数据比较多, 会造成redis 长时间的阻塞。 生产环境不建议使用这个命令. 

   为了解决这个问题, redis 提供了第二种方式

2. `bgsave`

    执行`bgsave` 的时候, redis会在后台异步进行快照操作, 快照同时还可以响应客户端的请求. 

   具体操作是redis 进程`fork` 操作系统子进程(`copy-on-write`),`RDB` 持久化过程由子进程负责, 完成后自动结束. 他不会记录`fork` 之后后续的命令. 阻塞只发生在`fork` 阶段, 一般时间很短. 

   用`lastsave` 命令可以查看最近一次生成快照的时间





### 1.2 `RDB`数据的恢复(演示)

####  1.2.1 `shutdown ` 持久化

 添加键值

```bash
redis> set k1 1
redis> set k2 2
redis> set k3 3
redis> set k4 4
redis> set k5 5
```



停止服务器 ，触发`save`

```bash
redis> shutdown

```

备份:`dump.rdb` 文件

```bash
cp dump.rdb dump.rdb.bak
```



启动服务器

```bash
/usr/local/soft/redis-5.0.5/src/redis-server /usr/local/soft/redis-5.0.5/redis.conf
```

数据都在

```bash
redis> keys *

```



####  1.2.2 模拟数据丢失

模拟数据丢失,触发`save`

```bash
redis> flushall

```

停服务器

```bash
redis> shutdown
```

启动服务器

```bash
/usr/local/soft/redis-5.0.5/src/redis-server /usr/local/soft/redis-5.0.5/redis.conf
```

结果： 啥都没有

```bash
redis> keys *

```



#### 1.2.3 通过备份文件恢复数据



停止服务器

```bash
redis> shutdown

```

重命名备份文件

```bash
mv dump.rdb.bak dump.rdb

```

启动服务器

```bash
/usr/local/soft/redis-5.0.5/src/redis-server /usr/local/soft/redis-5.0.5/redis.conf

```

查看服务器

```bash
redis> keys *

```



### 1.3  `RDB` 文件的优势和劣势

####  1.3.1 优势

1. `RDB`  是一个非常紧凑(`compact`)的文件, 他保存了redis在某个时间点上的数据集, 这种文件非常适合于进行备份和灾难恢复. 
2. 生成`RDB` 文件的时候, redis主进程会`fork`一个子进程来处理所有的保存工作, 主进程不需要进行进行任何磁盘I/O 工作. 
3. `RDB`在恢复大数据集时的速度比`AOF` 的恢复速度要快. 

#### 1.3.2 劣势

1. `RDB` 方式数据没办法做到实时持久化/秒级持久化. 因为`bgsave` 每次运行都要执行`fork` 操作创建子进程, 频繁执行成本过高 . 
2. 在一定间隔时间做一次备份, 所以如果redis 意外`down` 掉的话， 就会丢失掉最后一次快照之后的所有修改(数据又丢失)

如果数据相对来说比较重要，希望将损失降到最低, 则可以使用`AOF` 方式进行持久化. 



## 2. `AOF`

`Append Only File`

`AOF`: redis默认不开启.`AOF` 采用日志的形式来记录每个写操作, 并追加到文件中. 开启后, 执行更改redis数据的命名时, 就会把命令追加到`AOF` 文件中. 

redis 重启的时候 会根据日志文件的内容把写命令从前到后执行一次以完成数据的恢复工作. 



### 2.1 `AOF` 配置

配置文件`redis.conf`

```properties
# 开关
appendonly no
# 文件名
appendfilename "appendonly.aof
```



| 参数                              | 说明                                                 |
| --------------------------------- | ---------------------------------------------------- |
| `appendonly`                      | Redis 默认只开启 RDB 持久化，开启 AOF 需要修改为 yes |
| `appendfilename "appendonly.aof"` | 路径也是通过 dir 参数配置 config get dir             |



**问题：数据都是实时持久化到磁盘吗? **

由于操作系统的缓存机制, `AOF` 数据并没有真正的写入到磁盘, 而是进入了系统的硬盘缓存. 什么时候把缓冲区的内容写入到`AOF` 文件?

`appendfsync everysec` 参数说明

AOF 持久化策略（硬盘缓存到磁盘），默认 everysec

- no 表示不执行 fsync，由操作系统保证数据同步到磁盘，速度最快，但是不太安全；
- always 表示每次写入都执行 fsync，以保证数据同步到磁盘，效率很低；
- everysec 表示每秒执行一次 fsync，可能会导致丢失这 1s 数据。通常选择 everysec ， 兼顾安全性和效率。



**文件越来越大怎么办? **

由于`AOF` 持久化是redis 不断的将命令记录到`AOF` 文件中, 随着redis 不断的进行, `AOF` 的文件会越来越大 ,文件越大, 占用服务器内存越大以及`AOF` 恢复要求的时间越长. 

例如`set test 666` ,执行了1000次, 结果都是`test = 666`

为了解决这个问题, Redis 新增了重写机制, 当`AOF` 的文件超过大小所设定的阈值的时候, redis 就会启动`AOF` 文件的内容压缩, 只保留可以恢复的数据的最小指令集. 

可以使用命令`bgrewriteaof` 来重写. 

`AOF` 文件重写并不是对原文件进行重写整理, 而是直接读取服务器现有的键值对, 然后用一条命令去代替之前记录这个键值对的多条命令, 生成一个新的文件去替换原来的`AOF` 文件. 

```properties
# 重写触发机制
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

| 参数                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| `auto-aof-rewrite-percentag e` | 默认值为 100。aof 自动重写配置，当目前 aof 文件大小超过上一次重写的 aof 文件大小的 百分之多少进行重写，即当 aof 文件增长到一定大小的时候，Redis 能够调用 bgrewriteaof 对日志文件进行重写。当前 AOF 文件大小是上次日志重写得到 AOF 文件大小的二倍（设 置为 100）时，自动启动新的日志重写过程。 |
| auto-aof-rewrite-min-size      | 默认64M, 设置允许重写的最小`aof` 文件大小, 避免到了约定百分比但尺寸仍然很小的情况下还要重写. |



**重写过程中, `AOF` 文件被修改了怎么办呢? **

![image-20200331141046763](http://files.luyanan.com//img/20200331141256.png)



另外有两个与`AOF` 相关的参数: 

- `no-appendfsync-on-rewrite`

   在`aof` 重写或者写入`rdb` 文件的时候, 会执行大量IO, 此时对于`everysec` 和`always` 的`aof` 模式来说, 执行`fsync`  会造成阻塞过长时间, `no-appendfsync-on-rewrite` 字段设置为默认设置为`no`.如果对延迟要求很高的应用, 这个字段可以设置为`yes`, 否则还是设置为`no`, 这样对持久化特性来说这是更安全的选择, 设置为`yes` 表示`rewrite` 期间对新写操作不`fsync`,暂时存在内存中,等`rewrite` 完成后在写入, 默认为`no`, 建议修改为`yes`. linux 的默认`fsync` 的策略是30秒, 所以可能会丢失30秒的数据. 

- `aof-load-truncated`

   `aof` 文件可能在尾部是不完整的, 当redis 启动的时候, `aof`文件的数据被载入到内存中. 重启可能会发生在`redis` 所在的主机操作系统宕机后, 尤其是`ext4` 文件系统没有加上`data=ordered` 选项, 出现这种现象	. redis 宕机或者出现异常终止不止会造成尾部不完整现象, 可以选择让redis退出或者导入尽可能多的数据. 如果选择的是`yes` ,当截断的`aof` 文件被导入的时候, 会自动发布一个`log`给客户端然后`load`, 如果是`no`, 用户必须手动`redis-check-aof` 修复`aof` 文件才可以, 默认为`yes`. 



### 2.2  `AOF` 数据恢复

重启redis之后就会进行`AOF` 文件的恢复.



###  2.3 `AOF` 优势和劣势

#### 2.3.1 优势

1. `AOF` 持久化的方法提供了多种的同步频率, 即时使用默认的同步频率每秒同步一次, redis 最多也就丢失一秒的数据. 

#### 2.3.2 缺点

1. 对于具有相同数据的redis, `AOF` 文件通常会比`RDB` 文件提交更大(`RDB`文件存的是数据快照)
2. 虽然`AOF` 提供了多种同步的频率, 默认情况下, 每秒同步一次的频率也是具有较高的性能. 在高并发的情况下, `RDB` 比`AOF` 具有更好的性能保证.





## 3. 两种方案比较

那么对于`AOF` 和`RDB` 这两种持久化方式, 我们应该如何做选择呢? 

如果能够忍受一小段时间内数据的丢失, 毫无疑问是`RDB` 最好了.定时生成`RDB` 快照(`snapshot`) 非常便于进行数据库备份, 并且`RDB` 恢复数据集的速度也是要比`AOF` 恢复的速度要快. 

否则就使用`AOF` 重写, 但是一般情况下不建议单独使用一种持久化方式, 而是应该两种一起用, 在这种情况下, 当redis 重启的时候会优先载入`AOF` 文件来恢复原始的数据, 因为在通常的情况下`AOF` 文件保存的数据集要比`RDB` 文件保存的数据集要完整. 





