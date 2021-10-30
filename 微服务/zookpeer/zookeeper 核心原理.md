#  zookeeper 核心原理

## Zookeeper 数据的同步流程

我们知道, zookeeper 是通过三种不同的集群角色来组成整个高性能集群的, 在zookeeper 中, 客户端会随机连接到 zookeeper 集群中的一个节点, 如果是读请求, 就直接从当前节点中读取数据, 如果是写请求, 那么请求会被转发给leader 提交事务. 然后leader 会传播事务, 只要有超过半数的节点写入成功, 那么写请求就会被提交.(类似于2PC 事务).



![](http://files.luyanan.com//img/20191112135730.png)

> 那么问题来了? 
>
> 1. 集群中的leader 节点是如何选举出来的? 
> 2. leader  节点崩溃后, 整个集群无法处理写请求, 如何快速的从其他节点中选举出新的leader 呢? 
> 3. leader节点和各个follower 节点的数据一致性如何保证. 

### ZAB 协议

ZAB(Zookeeper Atomic Broadcase) 协议是为分布式协调服务Zookeeper 专门设计的一种支持崩溃恢复的原子广播协议. 在Zookeeper中, 主要依赖ZAB 协议来实现分布式数据一致性, 基于该协议, zookee实现了一种主备协议的系统架构来保持集群中各个副本之间的数据一致性. 

####  ZAB 协议介绍

ZAB 协议包含两种基本模式: 包括:

1. 崩溃恢复
2. 原子广播

当整个集群在启动时,或者当leader节点出现网络中断、崩溃等情况时, ZAB协议会进入恢复模式并选举产生新的Leader, 当leader 服务器选举出来后, 并且集群中由过半的机器和该leader 节点完成数据同步后(同步指的是数据同步, 用来保证集群中过半的机器能够和leader服务器的数据状态保持一致),ZAB 协议就会退出恢复模式. 当集群中已经有过半的Follower 节点完成了和leader 节点状态同步以后, 那么整个集群模式就进去了消息广播模式. 这个时候, 在leader 节点正常工作时, 启动一台新的服务器加入到集群, 那这个服务器会直接进入数据恢复模式,和leader节点进行数据同步. 同步完成后即可正常对外提供非事务请求的处理 .

>  需要注意的是: leader 节点可以处理事务请求和非事务请求, follower 节点只能处理非事务请求, 如果follower节点接收到事务请求, 就会把这个请求转发到  leader 节点 .

#### 消息广播的实现原理

消息广播其实就是一个简化版本的二节点提交过程。 

1. leader 接收到消息请求后, 将消息赋予一个全局唯一的64位自增id, 叫zxid, 通过zxid 的大小比较可以实现因果有序这个特征. 
2. leader 为每个follower准备了一个FIFO 队列(通过TCP协议来实现, 以实现了全局有序这个特点)将带有zxid 的消息作为一个提案(proposal) 分发给所有的follower
3. 当follower 接受到 proposal , 先把 proposal  写到磁盘, 写入成功以后再想leader 回复一个ack
4. 当leader 接收到合法数量(超过半数节点)的ACK后, leader 就会向这些follower发送 commit  命令, 同时也会在本地执行该消息. 
5. 当follower 收到消息的commit 命令后, 会提交该消息. 

![](http://files.luyanan.com//img/20191112153026.png)

> 和完整的2pc事务不一样的地方在于, zab 协议不能终止事务, follower节点要么ACK 给leader, 要么抛弃leader, 只需要保证过半的节点响应了这个消息并提交了即可.虽然在某一个时刻 follower节点和leader 节点的状态会不一致, 但是这个特性提升了集群的整体性能. 当然这种数据不一致的问题, zab协议提供了一种恢复模式来进行数据恢复. 

这里需要注意的: 

leader 的投票过程,不需要Observer 的ack,也就是Observer 不需要参与投票过程, 但是Observer 必须要同步Leader 的数据从而在处理请求的时候保证数据的一致性. 

#### 崩溃恢复的实现原理

我们知道 ZAB 协议是基于原子广播协议的消息广播过程, 在正常情况下是没有任何问题的, 但是一旦leader 节点崩溃或者由于网络问题导致Leader 服务器事务了过半的follower 的节点的联系()leader 失去与过半的follower节点联系, 可能是leader 节点和follower节点之间产生了网络分区, 那么此时的leader 已经不再是合法的leader了),那么就会进入到崩溃恢复模式. 崩溃恢复模式下ZAB协议需要做两件事情:

1. 选举出新的leader
2. 数据同步. 

我们知道, ZAB 协议的消息广播机制是简化版本的2PC协议, 这种协议只需要集群中过半的节点响应提交即可. 但是它无法处理Leader 服务器崩溃带来的数据不一致问题, 因此在ZAB 协议中添加了一个"崩溃恢复模式" 来解决这个问题. 

那么ZAB协议中的崩溃恢复需要保证, 如果一个事务Proposal 在一台机器上被处理成功, 哪怕是出现故障, 为了达到这个目的, 我们先来设想一下, 在zookeeper 中会有哪些场景导致数据不一致性, 以及针对这个场景, zab协议中的崩溃恢复应该怎么处理. 

#####  已经被处理的消息不能丢

当leader 收到合法数量follower的ACK 后, 就向各个follower 广播 commit 命令, 同时也会在本地执行 commit 并向连接的客户端返回[成功]. 但是如果在各个 follower 在收到commit 命令前leader 就挂了, 导致剩下的服务器并没有执行这条消息. 

![](http://files.luyanan.com//img/20191112161319.png)

> 图中是C2就是一个典型的例子, 在集群正常运行过程的某一个时刻, server1 是leader 服务器, 先后广播了消息P1、P2、C1、P3 和C2。 其中当leader 服务器把消息C2(commit 事务proposal2) 发出后就立即崩溃退出了, 那么针对这种情况, ZAB协议就需要确保事务Proposal2 最终能够在所有的服务器上都能被提交成功, 否则将会出现不一致. 

#####  被丢弃的下次不能再次出现

当leader 接收到消息请求生成proposal 后就挂了, 其他follower 并没有收到此proposa,因此经过恢复模式重新选了leader 后, 这条消息是被跳过的. 之前挂了的leader 重新启动并注册成了follower, 它保留了被跳过消息的proposal状态, 与整个系统的状态是不一致的. 需要将其删除. 

ZAB协议需要满足上面两种情况, 就必须设计一个leader 选举算法，能够确保已经被leader 提交的事务proposal 能够提交, 同时丢弃已经被跳过的事务Proposal.

针对这个要求:

1. 如果leader 选举算法能够保证新选举出来的leader 服务器拥有集群中所有寄去最高编号(zxid最大) 事务Proposal, 那么就可以保证这个新选举出来的leader 一定具有已经提交的提案. 因为所有提案被commit 之前必须有超过半数的follower  ACK,即必须有超过半数的节点的服务器的事务日志上有该提案的proposal. 因此, 只要有合法数量的节点正常工作, 就必然有一个节点保存了所有被Commit 消息的proposal 状态.
2.  另外一个, zxid 是64位, 高32位是epoch编号, 每经过一次leader 选举产生一个新的leader, 新的leader 会将epoch 号 +1, 低32位是 消息计数器, 每接收到一条消息这个值+1,新的leader 选举后这个值重置为0, 这样设计的好处在于老的leader 挂了以后重启, 它不会被选举为leader, 因此此时它的zxid 肯定小于当前新的leader.当老的leader 作为follower 接入新的leader 后, 新的leader 会让他将所有的拥有旧的epoch 号的未被 commit 的proposal 清除. 

### 关于zxid

前面一直提到zxid, 也就是事物id, 那么这个id 具体起什么作用呢? 以及这个id 是如何生成的呢? 简单给大家解释一下, 为了保证事务的顺序一致性, zookeeper 采用了递增的事务id 号(zxid)来标识事务, 所有的提议(proposal)都在被提出的时候加上了zxid. 实现中zxid 是一个64位的数字, 它高32位是epoch(ZAB协议通过epoch 编号来区分leader 周期变化的策略)用来标识leader 关系是否改变, 每次一个leader 被选出来后, 它都会有一个新的epoch=（原来的epoch +1）,标识当前属于哪个leader 的统治时期. 低32 用于递增计数. 

> epoch: 可以理解为当前集群所处的年代或者周期, 每个leader 就像皇帝, 都有自己的年号,所以每次改朝换代,leader 变更之后, 都会在前一个年代的基础上加上1. 这样就算旧的leader 崩溃恢复后, 也没人听她的了. 因为follower 只听从当前年代的leader 的命令. 

