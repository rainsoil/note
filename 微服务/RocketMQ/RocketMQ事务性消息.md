#  RocketMQ事务性消息

​    在现在的很多分布式集群环境中, 经常会遇到本地事务无法解决的问题, 所以会引进分布式事务. 所谓的分布式事务是指分布式架构中多个服务的节点的数据一致性. 

## 1. 经典的X/OpenDTP事务模型

​    X/Open DTP(X/Open `Distributed Transaction Processing Reference Model`) 是X/Open 这个组织定义的一套分布式事务的标准， 也就是定义了规范和API接口,由各个厂商进行具体的实现. 

​    这个标准提出了使用二阶段提交(2PC-` Two-Phase-Commit`) 来保证分布式事务的完整性.后来J2EE 也遵循了X/Open DTP 规范, 设计并使用了Java 的分布式事务编程接口规范-JTA

### 1.1 `X/Open DTP` 角色

在`X/Open DTP`事务模型中,定义了三个角色: 

**AP:** `application`,应用程序,也就是业务层,哪些操作属于同一个事务,就是`AP`定义的. 

**RM**: `Resource Manager`,资源管理器, 一般是数据库, 也可以是其他资源管理器, 比如消息队列. 

**TM**:`Transaction Manage`: 事务管理器、事务协调者,负责接收来自用户程序(`AP`)发起的XA事务指令,并调度和协调参与事务的所有`RM(S数据库)` , 确保事务正确完成. 

​    在分布式系统中, 每一个机器节点虽然都能够明确知道自己在进行事务操作过程中的结果是成功还是失败, 但却无法直接获取到其他分布式节点的操作结果.因此, 当一个事务操作需要跨越多个分布式节点的时候, 为了保证事务处理的`ACID`特性, 就需要引入一个"协调者"(`TM`)来统一调度所有分布式节点的执行逻辑 , 这些被调度的分布式节点被称为`AP`.`TM`负责调度`AP`的行为, 并最终决定这些`AP` 是否要事务真正的进行提交到`RM`.

​    `XA`是`X/Open DTP` 定义的资源管理器和事务管理器之间的接口规范,`TM` 用它来通知和协调相关RM事务的开始、结束、提交和回滚.目前Oracle、Mysql、DR2 都提交了对`XA`的支持; `XA`接口是双向的系统接口, 在事务管理器(TM) 以及多个资源管理器之间形成通信的桥梁(XA不能自动提交).

![](http://files.luyanan.com//img/20200115140700.png)

![](http://files.luyanan.com//img/20200115140722.png)

### 1.2    2PC

####  1.2.1 第一阶段

![](http://files.luyanan.com//img/20200115140836.png)

RM 在第一阶段会做两件事情: 

1. 记录事务日志: reduo、undo
2. 返回给TM 信息 : ok、error

存在问题:如果第一阶段完成后TM宕机或者网络出现故障了,此时RM 就会一直阻塞,发生了死锁,因为没有 timeout机制,3PC就针对此问题进行了改造,加入了 timeout机制. 



#### 1.2.2 第二阶段

![](http://files.luyanan.com//img/20200115141134.png)

根据第一个阶段的返回结果进行提交或者回滚. 

### 1.3 CAP理论

CAP的含义是: 

- C: `Consistency`一致性,同一个数据的多个副本是否实时相同. 
- A：`Availability`可用性, 可用性: 一定时间内 & 系统返回一个明确的结果,则称之为该系统可用. 
- P：`Partition tolerance` 分区容错性,将同一服务分布在多个系统中, 从而保证某一个系统宕机, 让然有其他系统提供相同的服务. 

CAP 理论告诉我们, 在分布式系统中, C、A、P 三个条件中我们最多只能选择两个,那么问题来了,究竟选择哪两个条件较为合适呢? 

​    对于一个业务系统来说, 可用性和分区容错性是必须满足的两个条件,并且这两个是相辅相成的. 业务系统之所以使用分布式系统, 主要原因有两个: 

- 提升系统性能, 当业务量猛增, 单个服务器已经无法满足我们的业务需求的时候, 就需要使用分布式系统,使用多个节点提供相同的功能, 从而整体上提升系统的性能,这就是使用分布式系统的第一个原因. 
- 实现分区容错性, 单一节点或者多个节点处于相同的是网络环境下, 那么会存在一定的风险, 万一该机房断电、该地区发生自然灾害, 那么业务系统就全面瘫痪了.为了防止这一问题,采用分布式系统,将多个子系统分布在不同的地域、不同的机房, 从而保证了系统高可用. 

这说明分区容错性是分布式系统的根本, 如果分区容错性不能满足, 那使用分布式系统将失去意义. 

​    此外,可用性对业务系统也尤为重要, 在大谈用户体验的今天,如果业务系统常出现"系统异常"、响应时间过程等情况, 这使得用户对系统的好感度大打折扣, 在互联网行业竞争激烈的今天,相同领域竞争者不甚枚举,系统的间歇性不可以会立马导致用户流向竞争对手。因此, 我们只能通用牺牲一致性来换取系统的**可用性**和**分区容错**. 

### 1.4 Base 理论

CAP理论告诉我们一个悲惨但不得不接受的事实-- 我们只能在C、A、P 中选择两个条件, 而对于业务系统而言,我们往往选择牺牲一致性来换取系统的可用性和分区容错性. 不过要在这里指出的是, 所谓是"牺牲一致性"并不是完全放弃数据一致性, 而是牺牲"强一致性"换取"弱一致性". 

- `BA:Basic Availabl ` 基本可用

   整个系统在某些不可抗力的情况下, 让然能够保证"可用性", 即一定时间内然然能返回一个明确的结果.只不过"基本可用"和"高可用"的区别是:

  - “一定时间”可以适当延长,当举行大促时, 响应时间可以适当延长. 
  - 给部分用户返回一个降级页面, 给部分用户直接返回一个降级页面, 从而缓解服务器压力. 但要注意的是: 返回降级页面仍然是返回明确结果. 

- `S:Soft State`: 柔性状态,同一数据的不同副本的状态, 可以不需要实时一致. 
- `E:Eventual Consisstency`: 最终一致性, 同一数据的不同副本状态, 可以不需要实时一致, 但一定要保证经过一段时间后仍然是一致的. 


## 2. 分布式事务常见解决方案

### 2.1 最大努力通知方案

![](http://files.luyanan.com//img/20200115155523.png)

### 2.2 TCC 两阶段补偿方案

TCC 是`Try-Confirm-Cance`, 比如在支付场景中,先冻结一部分资金,再去发起支付. 如果支付成功,则将冻结的资金进行实时扣除, 如果支付失败, 则取消资金冻结. 

​    

![](http://files.luyanan.com//img/20200115155726.png)

#### 2.2.1 Try阶段

完成所以业务检查(一致性),预留业务资源(准隔离性)

#### 2.2.2Confirm阶段

确认执行业务操作, 不做任何业务检查, 只使用try 阶段预留的业务资源. 

#### 2.2.3 Cancel阶段

取消try 阶段预留的业务资源, Try出现出现阶段时, 取消所有业务资源预留请求. 

### 2.3 关于状态机

在使用最终一致性的方案时, 一定要提到的一个概念就是状态机. 

​    什么是状态机? 是一种特殊的组织代码的方式, 用这种方式能够确保你的对象随时都直到自己所处的状态以及能够做的操作。它也是一种用来进行对象行为建模的工具, 用于描述对象在他的生命周期内所经历的状态序列, 以及如何响应来自外界的各种事件. 

​    状态机这个概念大家都不陌生, 比如TCP 协议的状态机. 同时我们在编写相关业务逻辑的时候经常也会需要处理各种事件和状态的切换, 比如swith、if/else . 所以我们其实一致都在跟状态机打交道,只是可能都没有意识到而已. 在处理一些业务逻辑比较复杂的需求的时候, 可以先看看是否适用于一个有限状态机来描述, 如果可以把业务模型抽象成一个有限的状态机, 那么代码就会逻辑非常清晰, 结构特别规整. 

​    比如我们来简单描述一个订单. 

​    我们以支付为例, 一笔订单可能会有等待支付、支付中、已支付状态, 那么我们就可以先去把可能出现的状态以及状态的流程画出来。 

![](http://files.luyanan.com//img/20200115163617.png)

状态机的两个作用: 

- 实现幂等
- 通过状态驱动数据的变化
- 业务流程以及逻辑更加清晰, 特别是应对复杂的业务场景. 

## 3. 什么是幂等

   简单来说, 重复调用多次产生的业务结果与调用一次产生的业务结果相同; 在分布式架构中, 我们调用一个远程服务去完成一个操作, 除了成功和我失败以外, 还有未知状态, 那么针对这个未知状态, 我们会采用一些重试的行为. 或者在消息中间件的使用场景中, 消费者可能会重复收到消息. 对于这两种情况, 消费端或者服务端需要采取一定的手段, 也就是考虑到重发的情况下保证数据的安全性.我们一般常用的手段： 

1. 状态机实现幂等
2. 数据库唯一约束实现幂等
3. 通过`tokenId` 的方式去识别每次请求判断是否重复. 

## 4. 开源的分布式事务解决方案

### 4.1 TransactionProducer（事务消息）

RocketMQ 和其他消息中间件最大的一个区别就是支持了事务消息, 这也是分布式事务中基于消息的最终一致性方案. 

### 4.2  RocketMQ 消息的事务架构设计

1. 生产者执行本地事务, 修改订单支付状态, 并且提交事务
2. 生产者发送事务消息到broker上, 消息发送到broker 上在没有确认之前, 消息对于`consumer` 是 不可见的状态. 
3. 生产者确认事务消息, 使得发送到broker上的事务消息对于消费者可见. 
4. 消费者获取到消息进行消费,消费完之后执行ack 进行确认. 
5. 这里可能会存在一个问题 , 生产者本地事务成功后,发送事务确认消息到broker 上失败了怎么办? 这个时候意味着消费者无法正常消费到这个消息, 所以RocketMQ 提供了消息回查机制, 如果事务消息一直处于中间状态,broker 会发起重试去查询broker 上这个事务的处理状态。一旦发送事务处理成功, 则把这条消息设置为可见. 



![](http://files.luyanan.com//img/20200116094256.png)



### 4.3 事务消息的实践

通过一个下单以后扣减库存的数据一致性场景来演示RocketMQ的分布式事务特性. 

TransactionProducer

```java
package com.mq;

import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.TransactionMQProducer;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

import java.io.UnsupportedEncodingException;
import java.util.UUID;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author luyanan
 * @since 2020/1/16
 * <p>事务生产者</p>
 **/
public class TransactionProducer {

    public static void main(String[] args) throws MQClientException, UnsupportedEncodingException, InterruptedException {

        TransactionMQProducer producer = new TransactionMQProducer("tx_producer_group");
        producer.setNamesrvAddr("192.168.86.128:9876");
        //  自定义线程池
        ExecutorService executorService = Executors.newFixedThreadPool(10);
        producer.setExecutorService(executorService);
        producer.setTransactionListener(new TransactionListenerLocal());
        producer.start();
        for (int i = 0; i < 20; i++) {

            String orderId = UUID.randomUUID().toString();
            String body = "{'operation':'doOrder','orderId':'" + orderId + "'}";
            Message message = new Message("ex_topic", "TagA", orderId, body.getBytes(RemotingHelper.DEFAULT_CHARSET));
            producer.sendMessageInTransaction(message, orderId + "&" + i);
            Thread.sleep(1000);
        }
    }

}

```

TransactionListenerLocal

```java
package com.mq;

import org.apache.rocketmq.client.producer.LocalTransactionState;
import org.apache.rocketmq.client.producer.TransactionListener;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageExt;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @author luyanan
 * @since 2020/1/16
 * <p></p>
 **/
public class TransactionListenerLocal implements TransactionListener {

    private static final Map<String, Boolean> results = new ConcurrentHashMap<>();

    /**
     * <p>执行本地事务</p>
     *
     * @param message
     * @param o
     * @return {@link LocalTransactionState}
     * @author luyanan
     * @since 2020/1/16
     */
    @Override
    public LocalTransactionState executeLocalTransaction(Message message, Object o) {

        System.out.println("执行本地事务:" + o.toString());
        String orderId = o.toString();
        //模拟插入数据库操作
        boolean rs = saveOrder(orderId);

        //这个返回状态表示broker 这个事务消息是否被确认,允许给到consumer 进行消费
        // LocalTransactionState.ROLLBACK_MESSAGE 回滚
        // LocalTransactionState.UNKNOW 未知
        return rs ? LocalTransactionState.COMMIT_MESSAGE : LocalTransactionState.UNKNOW;
    }

    private boolean saveOrder(String orderId) {
        // 如果订单取模等于0, 表示成功, 否则失败
        boolean success = Math.abs(orderId.hashCode()) % 2 == 0;
        results.put(orderId, success);
        return success;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
        String orderId = messageExt.getKeys();
        System.out.println("执行事务执行状态的回查:orderId:" + orderId);

        boolean rs = Boolean.TRUE.equals(results.get(orderId));
        System.out.println("回调：" + rs);
        return rs ? LocalTransactionState.COMMIT_MESSAGE : LocalTransactionState.ROLLBACK_MESSAGE;
    }
}

```



消费者

TransactionConsumer

```java
package com.mq;

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.remoting.common.RemotingHelper;

import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.util.List;

/**
 * @author luyanan
 * @since 2020/1/16
 * <p>事务 消费者</p>
 **/
public class TransactionConsumer {
    public static void main(String[] args) throws MQClientException, IOException {

        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("tx_consumer_group");
        consumer.setNamesrvAddr("192.168.86.128:9876");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("ex_topic", "");

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                list.stream().forEach(messageExt -> {
                    String orderId = messageExt.getKeys();
                    String body = null;
                    try {
                        body = new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET);
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                    System.out.println("收到消息:" + body + "->开始扣钱库存:" + orderId);
                });
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.in.read();
    }

}

```



### 4.4  RocketMQ 事务消息的三种状态

1. `ROLLBACK_MESSAGE`:  回滚事务
2. `COMMIT_MESSAGE`: 提交事务
3. `UNKNOW`: broker 会定时的回查producer 消息状态,知道彻底成功或者失败. 

当 `com.mq.TransactionListenerLocal#executeLocalTransaction` 方法返回`ROLLBACK_MESSAGE`的时候, 表示直接回滚事务, 当返回 `COMMIT_MESSAGE`的时候直接提交事务. 

当返回`UNKNOW`的时候,broker 会在一段时间之后查`com.mq.TransactionListenerLocal#checkLocalTransaction`, 根据`com.mq.TransactionListenerLocal#checkLocalTransaction` 返回状态执行事务的操作(提交或者回滚)

​    如示例中,当返回`ROLLBACK_MESSAGE` 时消费者不会收到消息, 且不会调用回查函数, 当返回`COMMIT_MESSAGE` 时事务提交,消费者收到消息. 当返回`UNKNOW`的时候, 在一段时间之后调用回查函数, 并根据state 判断返回提交或者回滚状态,返回提交状态的消息将会被消费者消费, 所以此时消费者可以消费部分消息. 



## 5. 消息的存储和发送

​    由于分布式消息队列对于可靠性的要求比较高, 所以需要保证生产者将消息发送到broker 之后, 保证消息是不会丢失的,因此消息队列就少不了对于可靠性存储的要求. 

### 5.1 MQ消息存储选择

​    从主流的几种MQ 消息队列采用的存储方式来看, 主要会有几种: 

1. 分布式KV存储, 比如使用ActiviteMQ中采用的LebelDB、Redis, 这种存储方式对于消息读写能力要求不高的情况下可以使用. 
2. 文件系统存储, 常见的比如 kafka、RocketMQ、RabbitMQ 都是采用消息刷盘到所部署的机器上的文件系统来做持久化, 这种方案适用于对于有高吞吐量要求的消息中间件, 因为消息刷盘是一种高效率、高可靠、高性能的持久化方式, 除非磁盘出现故障, 否则一般是不会出现无法持久化问题的. 
3. 关系型数据库,比如ActiviteMQ 可以采用Mysql 作为消息存储,关系型数据库在单表数量达到千万级的情况下IO性能会出现瓶颈, 所以ActiviteMQ 并不适用于高吞吐量的消息队列场景. 

### 5.2 消息的存储结构

​    RocketMQ 就是采用文件系统的方式来存储消息, 消息的存储是由`ConsumeQueue`和`CommitLog` 配合完成的. `CommitLog` 是消息真正的物理存储文件。 `ConsumeQueue` 是消息的逻辑队列, 有点类似于数据库的索引文件, 里面存储的是指向`CommitLog`  文件中消息存储的地址. 

​    每个Topic 下的每个`Message Queue`都会对应一个`ConsumerQueue` 文件,文件的地址是`${store_home}/consumequeue/${topicNmae}/${queueId}/${filename},`, 默认路径是`/root/store`. 在rocketMQ的文件存储目录下, 可以看到这样一个结构的文件

![](http://files.luyanan.com//img/20200116112604.png)

我们只需要关心`Commitlog`、`Consumerqueue`、`Index`

#### 5.2.1 Commitlog

`commitlog`是用来存放消息的物理文件, 每个broker 上的`commitlog`文件是当前机器上的所有`consumerqueue` 共享, 不做任何区分. 

​    `commitlog`中的文件默认大小为1G，可以动态配置; 当一个文件写满以后, 会生成一个新的`commitlog`文件. 所有的topic 数据是顺序写入在`commitlog` 文件中的. 



文件名的长度为20位, 左边补零, 剩余末起始偏移量,比如00000000000000000000 表示第一个文件， 文件大小为102410241024，当第一个文件写满之后，生 成第二个文件 000000000001073741824 表示第二个文件，起始偏移量为1073741824

00000000000000000000 表示第一个文件， 文件大小为102410241024，当第一个文件写满之后，生 成第二个文件 000000000001073741824 表示第二个文件，起始偏移量为1073741824



####  5.2.2 consumeQueue

`consumeQueue` 表示消息 消费的逻辑队列, 这里面包含`messageQueue`在`commitlog` 中的真实物理地址偏移量ooffst,消息实体内容的大小和`Message Tag`的hash值。 对于实际物理存储来说, `consumerQueue` 对应每个topic 和`queueid`下的文件, 每个`consumerqueue`类型的文件也是有带大小的,每个文件默认大小为6000W字节,如果文件满了之后也会生成一个新的文件. 

####  5.2.3 IndexFile

​    索引文件, 如果一个消息包含key值的话 ,会使用`indexFile`存储消息索引.Index 索引文件提交了对`Commitlog` 进行数据检索, 提供了一种通过key 或者时间区间来查找`commitlog` 中的消息的方法. 在物理存储中, 文件名是以创建的时间戳命名的, 固定的单个`indexFile` 大小大概为400M,一个`indexFile` 可以保存2000W个索引. 

#### 5.2.4 abort

​    broker 就会在启动的时候创建一个名为abort的文件,并在`shutdown`的时候将其删除,用于标识进程是否正常退出,如果不是正常退出, 会在启动的时候做故障恢复. 



### 5.3 消息存储的消息结构

 ![](http://files.luyanan.com//img/20200116134637.png)

​    RrokerMQ 的消息存储采用的是混合型的存储结构, 也就是broker 单个实例下的所有队列公用一个日志数据文件`connitlog`. 这个是和kafka 的又一不同之处. 

​    为什么不采用kafka 的设计, 针对不同的`partition`存储一个独立的物理文件呢? 这是因为在kafka的设计中, 一旦kafka 中Topic的的`partition`数量过多, 队列文件会过多, 那么会给磁盘的IO读写造成比较大的压力, 也就造成了性能瓶颈. 所以RocketMQ 进行了优化, 消息主题统一存储在`commitlog`中. 

当然, 这种设计并不是银弹, 也是有他的优缺点: 

**优点**: 由于消息主题都是通过`commitlog` 进行读写, `comsumerQueue` 中只存储了很少的数据, 所以队列更加的轻量化, 对于磁盘的访问是串行化从而避免了磁盘的竞争. 

**缺点**: 消息写入磁盘虽然是基于顺序读写, 但是读的过程是随机的,读取一条消息会先读取`consumerQueue` , 在读`commitlog`, 会降低消息读的效率. 

### 5.4 消息发送到消息接收的整体流程

1. Producer 将消息发送到Broker 后, Broker 会采用同步或者异步的方式把消息写入到`commitlog`,RocketMQ 所有的消息都会存放在`commitlog`中, 为了保证消息存储不发生混乱, 对`commitlog` 写之前会加锁, 同时也可以使得消息能够被顺序写入到`commitlog`, 只要消息被持久化到磁盘文件`commitlog`,那么就可以保证producer 发送的消息不会丢失. 

   

   ![](http://files.luyanan.com//img/20200116135615.png)
   
2. `commitlog`持久化后, 会把里面的消息`Dispatch`到对应的`consumer queue`上, `consumer queue` 相当于kafka 的`partition`, 是一个逻辑队列, 存储了这个queue 在`commitlog` 中的其实offset、log大小和MessageTag的hashCode. 

   ![](http://files.luyanan.com//img/20200116140325.png)

3. 当消费者进行消息消费时,会先读取`consumerQueue`,逻辑消费队列`consumerQueue` 保存了指定topic 下的队列消息在`consumerLog` 中的起始物理偏移量offset、消息大小、和消息tag HashCode值. 

   ![](http://files.luyanan.com//img/20200116140504.png)

4. 直接从`consumerqueue` 中读取消息是没有数据的, 真正的消息主题是在`consumerlog`中, 所以换需要从`consumerlog` 中读取消息. 

   ![](http://files.luyanan.com//img/20200116140934.png)





## 6. 什么时候清理物理消息文件

那消息文件到底删不删? 什么时候删除呢? 

​    消息存储在`commitlog`之后, 的确是会被清理的,但是这个清理只会在以下任一条件成立后才会批量删除消息文件(`comitlog`)

1. 消息文件过期(默认72个小时),且到达清理时点(默认是凌晨4点),删除过期文件. 
2. 消息文件过期(默认72个小时),且磁盘空间达到了水位线(默认75%), 删除过期文件
3. 磁盘已经达到了必须释放的上限(85%水位线)的时候, 则开始批量清理文件(无论是否过期), 直到空间充足. 

> 注意： 若磁盘空间达到危险水位线(90%), 处于保护自身的目的, broker 会拒绝写入服务. 



