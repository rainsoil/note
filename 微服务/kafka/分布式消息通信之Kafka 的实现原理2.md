#  分布式消息通信之Kafka 的实现原理2

## 分区的副本机制

我们已经知道kafka的每个topic 都可以分为多个partition, 并且多个partition 会均匀分布在集群中的各个节点下.虽然这种方式能够有效的对数据进行分片, 但是对于每个partition而言, 都是单点的. 当其中某个partition不可用的时候, 那么这部分消息就没办法消费. 所以kafka为了提高partition 的可靠性而提供了副本的概念(Replica),通过副本机制来实现冗余备份. 

每个分区可以有多个副本, 并且在副本集合中会存在一个leader 的副本, 所有的读写请求都是由leader 副本来进行处理. 剩余的其他副本都作为follower 副本. follower 副本会从leader 副本同步消息日志. 这个有点类似于zookeeper 中的leader 和follower 的概念. 但是具体的实现方式还是有比较大的差异的. 所以我们可以认为，副本集会存在一主多从的关系. 

一般情况下,同一个分区的多个副本会被均匀的分配到集群中的不同broker 上. 当leader 副本所在的broker 出现故障后, 可以重新选举新的leader 副本继续对外提供服务. 通过这样的副本机制来提高kafka 集群的可用性. 



###  创建一个带副本机制的topic

通过下面的命令去创建带2个副本的topic

> sh kafka-topics.sh --create --zookeeper 192.168.86.128:2181 --replication-factor 3 --partitions 3 --topic secondTopic

然后我们可以在/tmp/kafka-logs 路径下看到对应topic 的副本信息, 我们通过一个图形的方式来表达. 

**针对secondTopic 这个topic 的3个分区对应的3个副本**

![](http://files.luyanan.com//img/20191228175148.png)



####  如何知道各个分区中对应的leader是谁呢? 

在zookeeper 的服务器上, 通过如下命令去获取对应分区的信息, 比如下面这个是获取secondTopic 的第一个分区的状态信息

> get /brokers/topics/secondTopic/partitions/1/state

{"controller_epoch":12,"leader":0,"version":1,"leader_epoch":0,"isr":[0,1]}

或通过这个命令 
> sh kafka-topics.sh --zookeeper 192.168.86.128:2181 --describe --topic
> test_partition

leader表示当前分区的leader 是哪个broker_id, 下图中,绿色线条的表示该分区中的leader 节点. 其他节点就是follower. 

![](http://files.luyanan.com//img/20191230093609.png)

> 需要注意是是: kafka 集群中的一个broker 中最多只能有一个副本,leader 副本所在的broker 节点的分区叫leader 节点, follower 副本所在的broker 节点的分区叫follower 节点. 



###  副本的leader 选举

kafka 提供了数据复制算法保证, 如果leader 副本所在的broker 节点宕机或者出现故障,或者分区的leader节点发生了故障, 这个时候怎么处理呢? 

那么, kafka 必须要保证从follower 副本中选择一个新的leader 副本. 那么kafka 是如何实现选举的呢？ 要了解leader 选举, 我们需要了解几个概念

kafka 分区下有可能有多个副本(replica)用于实现冗余, 从而进一步实现高可用. 副本根据角色的不同可分为3类: 

- **leader 副本**: 响应clients 端读写请求的副本. 

- **follower副本**: 被动的备份leader 副本中的数据, 不能响应clients 端读写请求. 

- **ISR副本**: 包含了leader 副本和所有和leader 副本保持同步的follower 副本

  >  如何判定是否与leader 同步后面会提到 每个kafka 副本对象都有两个重要的属性: LEO和HW. 注意是所有的副本, 而不是只有leader 副本. 

- **LEO**: 即日志末端位移(log end offset) , 记录了该副本底层日志(log) 中下一条消息的位移值, 注意是下一条消息. 也就是说, 如果LEO =10, 那么表示该副本中保存了10条消息, 位移值范围是[0,9].另外, leader LEO和 follower LEO的更新是有区别的. 

- **HW**: 即前面提到的水位值. 对于同一个副本对象而言, 其HW值不会大于LEO值, 小于等于HW值的所有消息都被认为是"已备份"的(relicated). 同理, leader 副本和follower 副本的HW更新是有区别的. 

  > 从生产者发出的一条消息首先会被写入分区的leader 副本,不过还需要等待ISR集合中的所有follower 副本都同步完成之后, 才能被认为已经提交. 之后才会更新分区的HW， 进而消费者可以消费到这条消息. 



###  副本协同机制

刚刚提到了, 消息的读写操作都只会由leader 节点来接受和处理. follower 副本只负责同步数据以及当leader 副本所在的broker 挂了之后, 会从follower 副本中选择新的leader . 

写请求首先由leader 副本处理,之后follower 副本会从leader 副本上拉取写入的消息, 这个过程就有一定的延迟,导致follower 副本中保存的消息略少于leader 副本,但是只要没有超过阈值就可以容忍. 但是如果一个follower 副本出现了异常,比如宕机、网络断开等原因长时间没有同步到消息, 那这个时候, leader 就会把它踢出去. kafka 通过维护ISR集合来维护一个分区副本信息. 

![](http://files.luyanan.com//img/20191230103148.png)



一个新leader 被选举并被接受客户端的消息成功写入. kafka 确保从同步副本列表中选举一个副本为leader, leader负责维护和跟踪ISR(in-Sync replicas ， 副本同步队列)中所有follower 滞后的状态. 当producer 发送一条消息到broker 后, leader写入消息并复制到所有的follower. 消息提交之后才被成功复制到所有的同步副本. 



###  ISR

ISR 表示目前"可用且消息量与leader 相差不多的副本集合. 这是整个副本集合的一个子集. "  怎么去理解可用和相差不多这两个词呢? 具体来说, ISR 集合中的副本必须满足两个条件: 

1. 副本所在节点必须维护着与zookeeper的连接
2. 副本最后一条消息的offset 与leader 副本的最后一条消息的offset 之间的差值不能超过指定的阈值(replica.lag.time.max.ms) replica.lag.time.max.ms：如果该follower 在此时间间隔内一直没有追上过leader 的所有消息, 则该follower 就会被剔除ISR列表. 
3. ISR 数据保存在Zookeeper的 /brokers/topics//partitions//state 节点中

follower 副本把leader 副本LEO 之前的日志全部同步完成时, 则认为follower 副本已经追赶上了leader 副本, 这个时候会更新这个副本的 `lastCaughtUpTimeMs` 标识, kafka 副本管理器会启动 一个副本过期检查的定时任务, 这个任务会定期检查当前时间和副本的 lastCaughtUpTimeMs 的差值是否大于参数 `replica.lag.time.max.ms`的值, 如果大于,则将这个副本剔除ISR集合. 

![](http://files.luyanan.com//img/20191230105056.png)

在ISR 中至少有一个follower 时, kafka 可以确保已经commit 的数据不丢失, 但是如果某个partition 的所有Replica 都宕机了, 就无法保证数据不丢失了. 

1. 等待ISR 中的任一个Replica "活"过来,并且选择它为leader. 
2. 选择第一个"活"过来的Replica(不一定是ISR中的) 作为leader. 

这就需要在可用性和一致性当中做出一个简单的折中

如果一定要等待ISR 中的Relica "活"过来, 那不可用的时间可能会相对较长。而且如果ISR 中的所有Replica 都无法"活"过来, 或者数据都丢失了,这个Partition 将永远不可用. 

选择第一个"活"过来的partition 作为Leader, 而这个Replica 不是ISR 中的Replica, 那即是它并不保证已经包含了所有已经commit的消息, 它也会成为leader 而作为consumer 的数据源. 

### 副本数据同步原理

了解了副本的协同过程后,还有一个最重要的机制, 就是数据的同步过程, 它需要解决: 

1. 怎么传播消息
2. 在向消息发送端返回ack 之前需要保证多少个Replica 已经接受到了这个消息. 

#### 数据的处理过程

下图中, 深红色部分表示`test_replica`分区的leader 副本, 另外另个节点上浅色部分表示 follower 副本. 

![](http://files.luyanan.com//img/20191230120229.png)



producer在发布消息到某个Partition时:

- 先通过zookeeper 找到该Partition 的leader(`get /brokers/topics//partitions/2/state`) 然后无论Topic 的Relicaiton Factor为多少(也就是该Partition 有多少个Relica),producer 只将该消息发送到该Partition的Leader,
- Leader 会将该消息写入其本地Log,每个Follower 都从Leader pull 数据. 这种方式上, Follower 存储的数据顺序与Leader 保持一致 
-  Follower 在收到该消息并写入其Log 后, 向Leader 发送ACK。 
- 一旦Leader 收到了ISR 中的所有Replica的ACK,该消息就被认为已经commit, Leader 将增加HW(HighWatermark) 并且向Producer 发送ACK。 

**LEO**： 即日志末端位移(log end  offset), 记录了该副本底层日志(log)中下一条消息的位移值. 注意是下一条消息. 也就是说, 如果LEO =10, 那么表示该副本保存了10条消息, 位移值范围是[0,9].另外, Leader LEO和follower LEO 的更新是有区别的. 

**HW**: 即上面提到的水位值(Hight Water). 对于同一个副本对象而言，其HW值不会大于LEO值, 小于等于HW值的所有消费者都被认为是"已备份"的(reolicaed).同理, leader 副本和follower 副本的HW更新是有区别的. 

通过下面这幅图来表达LEO、HW的含义. 随着Follower 副本不断和Leader副本进行数据同步, follower 副本的LEO会逐渐后以并且追赶上leader 副本, 这个追赶的判断标准化是当前副本的LEO 是否大于或者等于leader 副本的HW， 这个追赶上也会使得被剔除的follower 副本重新加入到ISR集合中. 

另外,假如说下图中的最右侧的follower 副本被踢出ISR集合, 也会导致整个分区的HW发生变化, 变成了3



![](http://files.luyanan.com//img/20191230150950.png)

####  初始状态

初始状态下, leader和follower 的HW和LEO 都是0, leader 副本会保存  remote_LEO,  表示所有follower LE0也会被初始化为0. 这个时候, producer 没有发送消息. follower 会不断的向leader 发送fetch请求,但是因为没有数据 ,这个请求会被leader 寄存, 当在指定的时间之后会强制完成请求,这个时间的配置是 `replica.fetch.wait.max.ms`. 如果在指定的时间内producer 有消息发送过来, 那么kafka 会唤醒fetch 请求， 让leader 继续处理. 

![](http://files.luyanan.com//img/20191230152031.png)

> fetch的是,当没有消息的时候会阻塞, 根据`replica.fetch.wait.max.ms`参数来设定阻塞时间

数据的同步处理会分为两种情况, 这两种情况下处理方式是不一样的. 

- 第一种是leader 处理完producer 请求之后, follower 发送一个fetch 请求过来. 
- 第二种是follower 阻塞在leader 指定的时间之内, leader 副本收到producer 的请求. 



####  第一种情况

#####  生产者发送一条消息

leader 处理完producer请求之后, follower 发送一个fetch 请求过来.状态图如下: 

![](http://files.luyanan.com//img/20191230170450.png)

leader 副本收到请求以后, 会做几件事情

1. 把消息追加到log 文件,同时更新leader 副本的LEO
2. 尝试更新leader HW值, 这个时候由于follower 副本还没有发送fetch 请求, 那么leader 的remote_LEO 仍然是0。  leader会比较自己的LEO 以及remote_LEO 的值发现最小值是0, 与HW的值相同, 所以不会更新HW。 

##### follower  fetch消息

![](http://files.luyanan.com//img/20191230170936.png)

**Follower 发送fetch 请求, leader 副本的处理逻辑是: **

1. 读取log数据, 更新 remote_LEO=0(follower还没有写入这条消息, 这个值是根据follower 的fetch 请求中的offset来决定的)
2. 尝试更新HW, 因为这个时候LEO 和remote_LEO 还是不一致, 所以仍然是HW=0
3. 把消息内容和当前分区的HW值发送给follower 副本. 

**follower 副本收到response后**

1. 将消息写入到本地log, 同时更新follower 到LEO
2. 更新follower  HW，本地的LEO和leader 返回的HW进行比较取小的值, 所以仍然是0

第一次交互结束以后, HW 仍然是0, 这个值会在下一次follower 发起fetch 请求的时候被更新. 

![](http://files.luyanan.com//img/20191230172455.png)

**fetch 发第二次fetch请求, leader 收到请求以后**

1. 读取log 数据
2. 更新remote_LEO =1, 因为这次fetch 携带的是offset 是1. 
3. 更新当前分区的HW,这个时候leader_LEO 和remote_LEO 都是1, 所以HW 的值也更新为1. 
4. 把数据和当前分区的HW的值返回给follower 副本, 这个时候如果没有数据, 则返回为空. 

**follower 副本收到response 以后**

1. 如果有数据则写入本地日志, 并且更新LEO
2. 更新follower 的HW的值

到目前为止, 数据的同步就完成了, 意味着消费者能够消费offset  = 1 这条消息. 

##### 第二种情况

前面说过, 由于leader 副本暂时没有数据过来, 所以follower 的fetch 会被阻塞, 直到等待超时或者leader 接收到新的数据。 当leader 收到请求以后会唤醒处于阻塞的fetch 请求 . 处理过程基本上跟前面说的一致. 

1. leader 将消息写入到本地日志, 更新leader 的LEO
2. 唤醒follower 的fetch 请求
3. 更新HW

kafka 使用HW和LEO 的方式来实现副本数据的同步, 本身是一个很好的设计, 但是这个地方就存在一个数据丢失的问题, 当然这个丢失只出现在特定的情况下。我们回想一下， HW的值是在新一轮 fetch 中才会被更新, 我们分析一下这个过程为什么会出现数据丢失. 

## 数据丢失的问题

> 前提
>
> `min.insync.replicas=1` 设定ISR中的最小副本数是多少? 默认为1(在server.properties中配 置), 并且acks 参数设置为-1(表示需要所有副本都确认)时, 此参数才生效. 

表达的含义是, 至少需要多少个副本同步才能表示消息是提交的.所以当  `min.insync.replicas=1` 的时候, 一旦消息被写入leader 端log 即被认为是"已提交",而延迟一轮 fetch rpc更新HW值的设计使得follower HW值是异步延迟更新的, 倘若这这个过程中leader 发生了变更, 那么称为新的leader 的follower 的HW值就有可能是过期的, 使得clients 端认为是成功提交的消息被删除. 

![](http://files.luyanan.com//img/20191231095546.png)



###  producer 的ack

acks 配置表示producer 发送消息到broker 上以后的确认值, 有三个可选项

- 0 :表示producer 不需要等待broker 的消息确认, 这个选项时延最小但是同时风险最大(因为当server 宕机时, 数据会丢失)
- 1: 表示producer 只需要获得kafka 集群中的leader 节点确认即可, 这个选择时延较小同时确保了leader 节点确保接受成功. 
- all(-1): 需要ISR 中所有的replica 给与接受确认, 速度最慢, 安全性最高. 但是由于ISR 可能会缩小到仅包含一个Replica, 所以设置参数为all 并不一定能避免数据丢失. 

###  数据丢失的解决方案

在kafka 0.11.0.0 版本之后, 引入一个leader  epoch 来解决这个问题, 所谓的leader epoch 实际上是一对值(epoch offset),epoch 表示leader 的版本号, 从0开始递增, 当leader 发生过变更, epoch 就+1, 而offset 则是对应这个epoch 版本的leader 写入第一条数据的offset, 比如

(0,0),(1,50) 表示第一个leader 从offset  = 0写入消息,一共写了50条. 第二个leader 版本号是1, 从offset = 50 开始写, 这个信息会持久化在对应的分区的本地磁盘上, 文件名是  ` /tmp/kafkalog/topic/leader-epoch-checkpoint`. 

leader broker 中会保存这样一个缓存, 并且定时写入到 `checkpoint` 文件中. 

当leader 写log时它会尝试更新整个缓存, 如果这个leader 首次写消息, 则会在缓存中增加一个条目; 否则就不做更新. 而每次副本重新成为leader时会查询这部分缓存, 获取出对应leader 版本的offset. 

我们基于同样的情况来分析, follower 宕机并且恢复之后, 有两种情况,如果这个时候leader 副本没有挂, 也就是意味着没有发生leader 选举,那么follower 恢复之后并不会去截断自己的日志, 而是先发送一个`OffsetsForLeaderEpochRequest`请求给到leader 副本, Leader 副本收到请求之后返回自己的LEO

如果follower 副本的leaderEpoch 和leader副本的epoch 相同, leader 的LEO 只可能的大于或者等于follower 副本的LEO的值, 所以这个时候不会发生截断. 

如果follower 副本和leader 副本的epoch 的值并不相同, 那么leader副本会查找follower 副本传过来的epoch+1在本地文件中存储的StartOffset 返回的follower 副本, 也就是新leader副本的LEO. 这样也避免了数据丢失的问题. 

如果leader 副本宕机了重新选举新的leader, 那么原本的follower 副本就会变成leader, 意味着epoch 从0变成了1, 使得原本follower 副本中的LEO 的值得到了保留. 

##  Leader 副本的选举过程

1. kafkaController 会监听zookeeper的/broker/ids 节点路径, 一旦发生有broker 挂了, 执行下面的逻辑,这里暂时不考虑KfkaController 挂了的情况, KafkaController 挂了，各个broker 会重新leader 选举出新的KafkaController. 

2. leader 副本在该broker 上的分区就要重新进行leader 选举, 目前的选举策略是:

   1. 优先从ISR 列表中选出第一个作为leader 副本, 这个叫优先副本, 理想情况下优先副本就是该分区的leader 副本. 
   2. 如果ISR列表为空, 则查看该topic 的`unclean.leader.election.enable` 配置.  `unclean.leader.election.enable` 配置为true 则代表允许选用非ISR 列表的副本作为leade, 那么此时就意味着数据可能丢失. 为false的话, 则不允许, 直接抛出`NoReplicaOnlineException` 异常, 造成leader 副本选举失败. 
   3. 如果上述的配置为true, 则从其他副本中选出一个作为leader 副本, 并且ISR 列表中只包含该leader 副本, 一旦选举成功则将选举后的leader 和ISR 的其他副本信息写入该分区的对应的zk路径上. 

   

   

## 消息的存储

​    消息发送端发送消息到broker 后, 消息说如何持久化的呢? 那么接下来去分析下消息的存储. 

   首先我们需要了解的是, kafka 是使用日志文件的方式来保存生产者和消费者发送的消息, 每条消息都有一个offset 值来表示它在分区中的偏移量. kafka 中存储的一般都是海量的消息数据，为了避免日志文件过大，log并不是直接对应在一个磁盘上的日志文件, 而是对应磁盘的一个目录, 这个目录的命名为 `<topic_name>_<partition_id>`

### 消息的文件存储机制

   一个topic 的多个partition 在物理磁盘上的保存路径, 路径保存在` /tmp/kafka-logs/topic_partition` ,包含日志文件、索引文件和时间索引文件和时间索引文件. 

![](http://files.luyanan.com//img/20191231112447.png)

​    kafka 是通过分段的方式将log 分为多个`LogSegment`, `LogSegment` 是一个逻辑上的概念, 一个`LogSegment` 对应磁盘上的一个日志文件和一个索引文件, 其中日志文件是用来记录消息的. 索引文件是用来保存消息的索引的. 那么这个 `LogSegment` 是什么呢? 

#### LogSegment

​    假设kafka 是以partition 为最小存储单位的, 我们可以想象一下 当kafka producre 不断发送消息, 必然会引起partition 的无限扩展, 这样对应消息文件的维护以及被消息的消息的清理带来非常大的挑战, 所以kafka 以 segment 为单位又把partition 进行细分, 每个partition 相当于一个巨型文件被平均分配到多个大小相等的segment 的数据文件中(每个segment 文件中的消息并不一定相等),  这种特性方便已经被消费的消息的清理,提高磁盘的利用率. 

- `log.segment.bytes=107370 ` (设置分段大小), 默认是1gb, 我们把这个值调小以后, 可以看到日志分段的效果. 
- 抽取其中3个分段进行分析. 

![](http://files.luyanan.com//img/20191231113703.png)

​     segment  file 由2大部分组成, 分别为 index file和data file, 此2个文件一一对应, 成对出现,后缀".index"和".log" 分别表示为segment 索引文件和数据文件. 

​    segment 文件命名规则: partition 全局的第一个 segment 从0开始, 后续每个segment 文件名为上一个segment 文件最后一条消息的offset 值进行递增。数据最大为64位long 大小, 20位数据字符长度, 没有数字用0 填充. 

### 查看Segment 文件命名规则

####  通过下面这条命令可以看到kafka 消息日志的内容. 

```bash
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test0/00000000000000000000.log --print-data-log

```

​     假如第一个log文件的最后一个offset为:5376,所以下一个segment的文件命名为: 00000000000000005376.log。对应的index为00000000000000005376.index

#### segment 中index和log 的对应关系

从所有分段中, 找一个分段进行分析

​    为了提高查找消息的性能, 为每一个日志文件添加2个索引文件: `OffsetIndex 和 TimeIndex`,分别对应.index和.timeindex. timeindex 索引文件格式: 它是映射时间戳和相对offset

​    查看索引文件内容

```bash
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test0/00000000000000000000.index --print-data-log
```

![](http://files.luyanan.com//img/20191231115734.png)

如图所示. index 中存储了索引以及物理偏移量.log 存储了消息的内容. 索引文件的元数据执行对应数据文件中message的物理偏移量. 举个简单的例子, 以[4053,80899]为例, 在log文件中, 对应的是第4053条记录, 物理偏移量(position)为80899, position 是ByteBuffer 的指针位置. 

###  在Partition 中如何通过offset查找message

查找的算法是:

1. 根据offset 的值, 查找segment 段中的index索引文件. 由于索引文件命名是以上一个文件的最后一个offset 进行命名的, 所以, 使用二分法 查找算法能够根据offset 快速定位到指定的索引文件. 
2. 找到索引文件后, 根据offset 进行定位, 找到索引文件中符合范围的索引(kafka采用稀疏索引的方式来提高查找性能)
3. 得到position 后, 再找到对应的log文件中，从position处开始查找offset 对应的消息, 将每条消息offset 与目标offset 进行比较, 直到找到消息. 

比如说,我们要查找offset = 2490 这条消息, 那么先找到`00000000000000000000.index`,然后找到`[2487,49111]` 这个索引, 再回到log文件中, 根据49111 这个position 开始查找, 比较每条消息的offset 是否大于等于2490, 最后查找到对应的消息以后返回. 

### Log文件的消息内容分析

前面我们通过kafka 的命名,可以查看二进制的日志文件信息, 一条消息, 会包含很多字段

> offset: 5371 position: 102124 CreateTime: 1531477349286 isvalid: true keysize: -1 valuesize: 12 magic: 2 compresscodec: NONE producerId: -1 producerEpoch: -1 sequence: -1 isTransactional: false headerKeys: [] payload: message_5371

offset 和position 这两个前面已经讲过了, createTime 表示创建时间、、keysize和valuesize 表示key 和value 的大小, compresscodec 表示压缩编码、payload 表示消息的具体内容



###  日志的清除策略以及压缩策略

####  日志清除策略

前面提到过, 日志的分段存储, 一方面能够减少单个文件内容的大小, 另一方面, 方便kafka 进行日志的清理. 日志的清理策略有两个: 

1. 根据消息的保留时间, 当消息在kafka 中保存的时间超过了指定的时间, 就会触发清理过程. 
2. 根据topic 存储的数据大小, 当topic 所占的日志文件大于一定的阈值, 则开始删除最旧的消息. kafka 会启动一个后台线程, 定时检查是否存在可以删除的消息. 

通过`log.retention.bytes和log.retention.hours` 这两个参数来设置, 当其中任意一个达到要求, 都会执行删除. 

默认的保留时间是7天. 



####  日志压缩策略

​    kafka 还提供了日志压缩"Log Compaction" 功能, 通过这个功能可以有效地减少日志文件的大小, 缓解磁盘紧张的情况, 在很多应用场景中, 消息的key 和value的值之间的对应关系是不断变化的, 就像数据库的数据会不断被修改一样. 消息者只关心key 对应的最新的value 值. 因此我们可以开启kafka 的日志压缩功能, 服务端会在后台启动Cleaner 线程, 定期将相同的key 进行合并, 只保留最近的value值, 日志的压缩原理是: 

![](http://files.luyanan.com//img/20191231140527.png)



##  磁盘存储的性能问题

###  磁盘存储的性能优化

​    我们现在大部分企业仍然使用的是机械结构的磁盘, 如果把消息以随机的方式写入到磁盘,那么首先磁盘要做的就是寻址, 也就是定位到数据所在的物理地址, 在磁盘上就要找到对应的柱面、磁头以及对应的扇区; 这个过程相对于内存来说会消耗大量的时间. 为了规避随机读写带来的时间消耗, Kafka 采用顺序写的方式存储数据. 即使是这样, 但是频繁的io操作仍然会造成磁盘的性能瓶颈. 

###  零拷贝

消息从发送到落地保存, Broker 维护的消息日志本身就是文件目录, 每个文件都是二进制保存, 生产者和消费者使用相同的格式来出处理. 在消费者获取消息时,服务器先从磁盘读取数据到内存, 然后把内存中是数据原封不动的通过socket 发送给消费者。虽然这个操作描述起来很简单, 但实际上经历了很多步骤. 

操作系统将数据从磁盘读入到内核空间的页缓存. 

- 应用程序将数据从内核空间读入到用户空间缓存中. 
- 应用程序将数据写回到内核空间到socket 缓存中. 
- 操作系统将数据从socket 缓存区复制到网卡缓存区, 以便将数据经网络发出. 

![](http://files.luyanan.com//img/20191231141954.png)



通过"零拷贝"技术, 可以去掉这些没必须的数据复制操作,同时也会减少上下文切换次数. 现在的linux 操作系统提供一个优化的代码路径, 用于将数据从页缓存传输到socket ; 在linux 中, 是通过sendfile 系统调用来完成。 Java 提供了访问这个系统调用的方法 `FileChannel.transferTo API`.

使用 `sendfile` , 只需要一次拷贝就行, 允许操作系统将数据直接从页缓存发送到网络上, 所以在这个优化的路径中, 只有最后一步将数据拷贝到网卡缓存中是需要的. 

![](http://files.luyanan.com//img/20191231142722.png)



###  页缓存

页缓存是操作系统实现的一种主要的磁盘缓存, 但凡涉及到缓存的, 基本都是为了提升io性能, 所以页缓存是用来减少磁盘io操作的. 

磁盘高速缓存有两个重要因素: 

- 第一,访问磁盘的速度要远低于访问内存的速度, 若从处理器L1和L2 高速缓存访问速度更快. 
- 第二, 数据一旦被访问, 就有可能短时间内再次被访问。 正是由于基于访问内存比磁盘块的多， 所以磁盘的内存缓存将给系统存储性能带来质的飞跃. 

当一个进程准备读取磁盘上的文件内容时,操作系统会先查看待读取的数据所在的页(page)是否在页缓存(pagecache)中, 如果存在(命中)则直接返回数据, 从而避免了对物理磁盘的io操作 ;如果没有命中, 则操作系统会向磁盘发起读取请求并将读取的数据页存入页缓存, 之后再将数据返回给进程. 

​    同样, 如果一个进程需要将数据写入磁盘, 那么操作系统也会检测数据对应的页是否在页缓存, 如果不存在, 则会先在页缓存中添加相应的页, 最后将数据写入对应的页。被修改过后的页也就变成了脏页, 操作系统会在合适的时间把脏页中的数据写入磁盘,保证数据的一致性. 

​    Kafka 中大量使用了页缓存, 这是kafka 实现高吞吐量的重要因素之一。虽然消息都是先被写入页缓存,然后由操作系统负责具体的刷盘任务,但是kafka 中同样提供了同步刷盘以及间断性强制刷盘(fsync),可以通过`log.flush.interval.messages `和` log.flush.interval.ms`参数来控制. 

同步刷盘能够保证消息的可靠性, 避免因为宕机导致页缓存数据还未完成同步时造成的数据丢失, 但是实际使用上, 我们没必要去考虑这样的因素以及这种问题带来的损失, 消息可靠性可以由多副本机制来解决. 同步刷盘会带来性能的影响, 刷盘的操作由操作系统去完成即可. 



##  Kafka 消息的可靠性

没有一个中间件能够做到百分之百的完全可靠, 可靠性更多的还是基于几个9的衡量标准. 比如4个9、5个9.软件系统的可靠性只能无限的去接近100%, 但不可能达到100%. 所以kafka 如何实现最大可能的可靠性呢? 

1. 分区副本,你可以创建更多的分区来提供可靠性, 但是分区数过多也会带来性能上的开销. 一般来说, 3个副本就能满足大部分场景的可靠性要求. 
2. acks, 生产者发送消息的可靠性. 也就是我要保证这个消息一定是到了broker 并且完成了多副本的持久化, 但这种要求也会带来性能上的开销. 它由几个可选项: 
   - 1,生产者把消息发送到leader 副本, leader 副本在成功写入到本地日志之后就告诉生产者消息提交成功,但是如果ISR 集合中的follower 副本还没来得及同步leader 副本的消息, leader 挂了之后, 就会造成消息的丢失. 
   -  -1 ,消息不仅仅写入到了leader 副本, 并且还被ISR集合中所有副本同步完成之后才告诉生产者已经提交成功,这个时候即使leader挂了也不会造成数据丢失. 
   - 0, 表示producer 不需要等待broker 的消息确认, 这个选项时延最小但同时风险最大(因为当server 宕机时, 数据将会丢失)
3. 保障消息到了broker 之后, 消费者也需要有一定的保证, 因为消费者也可能出现某些问题导致消息没有消费到. 
4. `enable.auto.commit` 默认为true, 也就是自动提交offset. 自动提交是批量执行的, 有一个时间窗口, 这种方式会带来重复提交或者消息丢失的问题. 所以对于高可靠性要求的程序, 需要使用手动提交. 对于高可靠性要求的应用来说, 宁愿重复消费也不应该因为消费异常而导致消息丢失. 



