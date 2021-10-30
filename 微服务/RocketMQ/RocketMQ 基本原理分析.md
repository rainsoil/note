# RocketMQ 基本原理分析

##  1. 思考一下消息中间件的设计

### 1.1 可以先从基本的需求开始考虑

- 最基本的是要能支持消息的发送和接收, 需要涉及网络通信就一定就会涉及到NIO.
- 消息中心的消息存储(持久化/非持久化)
- 消息的序列化和非序列化
- 是否跨语言
- 消息的确认机制,如何避免消息重发. 

### 1.2 高级功能

- 消息的有序性
- 是否支持事务消息
- 消息收发的性能,对高并发大数据量的支持
- 是否支持集群
- 消息的可靠性存储
- 是否支持多协议



## 2. MQ消息存储选择

从主流的几种MQ 消息队列采用的存储方式来看,主要会有三种:

1. 分布式KV存储, 比如ActiviteMQ 中采用的是lebelDB、Redis 这种存储方式对于消息读写能力要求不高的情况下可以使用. 
2. 文件系统存储,常见的kafka、RocketMQ、RabbitMQ 都是采用消息刷盘到所部署的机器上的文件系统来做持久化, 这种方案适用于对于有高吞吐量的消息中间件,因为消息刷盘是一种高效率、高可靠、高性能的持久化方式, 除非磁盘出现故障,否则一般是不会出现无法持久化的问题. 
3. 关系型数据库,比如ActiviteMQ 可以采用mysql作为消息存储,关系型数据库在单表数据量达到千万级的情况下IO性能会出现瓶颈,所以ActiviteMQ 并不适合高吞吐量的消息队列场景. 

总的来说, 对于存储效率,文件系统要优于分布式LV存储,分布式LV存储要优于关系型数据库. 

## 3. RocketMQ的发展历史

RocketMQ 是一个由阿里巴巴开源的消息中间件, 2012年开源, 2017年成为apache 顶级项目. 

​    它的核心设计借鉴了kafka,所以我们在了解RocketMQ 的时候, 会发现很多和kafka 相同的特性。 同时, RocketMQ 在某些功能上和kafka 又有较大的差异,接下来我们就去了解RocketMQ.

    1. 支持集群模式、负载均衡、水平扩展能力. 
       2. 亿级别消息堆积能力.
       3. 采用零拷贝的原理、顺序写盘,随机读
       4. 底层通信框架采用Netty NIO
       5. NameServer 代替zookeeper,实现服务寻址和服务协调
       6. 消息失败重试机制, 消息可查询. 
       7. 强调集群无节点, 可扩展,任意一点高可用, 水平可扩展 . 
       8. 经过多次双11的考验. 



## 4. RocketMQ 的架构

![](http://files.luyanan.com//img/20200113111049.png)

集群本身没有什么特殊之处,和kafka 的整体结构相比,其中zookeeper 替换成了NameServer. 

> 在rocketMQ 的早版本(2.X) 的时候, 是没有namespace 组件的, 用的是zookeeper 做分布式协调和服务发现, 但是后期阿里数据根据实际业务需求进行改进和优化,自己研发了轻量级的namesrv,用于注册Client 服务和Broker 的请求路由,namesrv 上不做任何消息的位置存储,频繁才做zookeeper 的位置存储数据会影响整体集群性能. 

RocketMQ 由四部分组成:

1. Name Server 可集群部署, 节点之间无任何信息同步,提供轻量级的服务发现和路由. 
2. Broker(消息中间角色、负责存储消息, 转发消息). 部署相对复杂，Broker 分为Master 和Slave. 一个Master 可以对应多个Slave,但是一个Slave 只能对应一个Master. Master和Slave 的对应关系通过指定相同的BrokerName,不同的BroklerId 来定义,BrokerId0 表示Master,非0表示Slave. 
3. Producer: 生产者, 拥有相同的Producer Group 的Producer 组成一个集群, 与Name Server 集群中的其中一个节点(随机选择)建立长连接,定期 从Name Server 取Topic 路由信息, 并向提供Topic 服务的Master 建立长连接,且定时向Master 发送心跳. Producer 完全无状态,可集群部署. 
4. Consumer: 消费者,接受消息进行消费的实例,拥有相同Consumer Group 的Consumer 组成一个集群,与Name Server 集群中的其中一个节点(随机选择) 建立长连接,定期从Name Server 取Topic 路由信息,并向提供Topic 服务的Master、Slave 建立长连接,且定时向Master、Slave 发送心跳。Consumer 既可以从Master 订阅消息, 也可以从Slave 订阅消息,订阅规则由Broker 配置决定. 

 要使用RockerMQ, 至少需要启动两个进程,Name Server 、Broker , 前者是各种Topic 注册中心, 后者是真正的Broker. 



##  5.单机环境下的RocketMQ 的安装

### 5.1 下载并解压安装

1. 下载RocketMQ的安装文件  https://rocketmq.apache.org/

2. > unzip  rocketmq-all-4.6.0-bin-release.zip

   解压压缩包

###  5.2 启动Name Server 

1.  进入bin目录,运行namesrv, 启动NameServer 

   >  nohup sh mqnamesrv &

2. 默认情况下,Name Server 监听的是9876 端口

3. 查看启动日志

   > tail -1000f /root/logs/rocketmqlogs/namesrv.log

###  5.3  启动Broker

> nohup sh bin/mqbroker -n ${namesrvIp}:9876 -c /conf/broker.conf & ->[-c可以指定broker.conf配 置文件]。

默认情况下会加载 config/broker/conf

1.  启动Broker,其中-n 表示指定当前broker 对应的命名服务地址: `默认情况下,Broker 监听的是10911端口`

   > nohup sh mqbroker -n localhost:9876 &

2. 查看日志

   > tail -100f  /root/logs/rocketmqlogs/broker.log

###  5.4内存不足的问题

这是因为bin 目录下启动namesrv 和broker 的 `runbroker.sh`和`runserver.sh` 文件中默认分配的内存太大,rocketmq 比较耗内存,所以默认分配的内存比较大,而系统实际内存太小导致启动失败, 通常像虚拟机上安装的Centos 服务器内存可能是没有高的,只能调小. 实际中应该根据服务器内存情况, 配置一个合适的值. 

```text
# There is insufficient memory for the Java Runtime Environment to continue.
# Native memory allocation (mmap) failed to map 8589934592 bytes for committing
reserved memory.
# An error report file with more information is saved as:
# /data/program/rocketmq-all-4.6.0-bin-release/bin/hs_err_pid6465.log
```

#### 解决方法

修改 runbroker.sh和 runserver,sh

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512g"
Xms 是指设定程序启动时占用内存大小。一般来讲，大点，程序会启动的快一点，但是也可能会导致机器暂时
间变慢。
Xmx 是指设定程序运行期间最大可占用的内存大小。如果程序运行需要占用更多的内存，超出了这个设置值，
就会抛出OutOfMemory异常。
xmn 年轻代的heap大小，一般设置为Xmx的3、4分之一
```



### 5.5 停止服务

【sh bin/mqshutdown broker】 //停止 brokersh 

【bin/mqshutdown namesrv】 //停止 nameserver 

停止服务的时候需要注意，要先停止broker，其次停止nameserver。



### 5.6 broker.conf 文件

默认情况下,启动broker 会加载 `conf/broker.conf` 这个文件,这个文件里面就是一些常规的配置信息

**namesrvAddr**: NameServer 的地址

**brokerClusterName**: Cluster 的名称, 如果集群机器比较多, 可以分为多个cluster,每个cluster 提供给不同的业务场景使用. 

**brokerName**: broker 名称, 如果配置主从模式, master和slave 需要配置相同的名称来表明关系. 

**brokerId=0**: 在主从模式中,一个master broker 可以有多个slave,0 表示master, 大于0表示不同slave 的id

**brokerRole=SYNC_MASTER/ASYNC_MASTER/SLAVE**: 同步表示slave 和master 消息同步完成后再返回信息给客户端. 

**autoCreateTopicEnable=true**: topic 不存在的情况下自动创建



## 6. 消息发送和接收基本应用

###  6.1 添加jar 依赖

```xml

        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>4.5.2</version>
        </dependency>
```

### 6.2 生产者

```java
package com.mq;

import org.apache.rocketmq.client.exception.MQBrokerException;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;
import org.apache.rocketmq.remoting.exception.RemotingException;

import java.io.UnsupportedEncodingException;

/**
 * @author luyanan
 * @since 2020/1/13
 * <p>生产者</p>
 **/
public class Producer {

    public static void main(String[] args) throws MQClientException {
        /**
         * 生产者组,简单来说就是多个发送用一类消息的生产者称之为一个生产者组
         * RocketMQ 支持事务消息,在发送事务消息时, 如果事务消息异常(producer 挂了),
         * broker 端会来回查事务的状态,这个时候会根据group 名称来查找对应的producer 来
         * 执行相应的回查逻辑,相当于实现了producer 的高可用.
         */
        DefaultMQProducer producer = new DefaultMQProducer("producer_group");
        // 执行namesrv 服务地址, 获取broker 的相关信息
        producer.setNamesrvAddr("192.168.86.128:9876");
        producer.setVipChannelEnabled(false);
        producer.start();
        for (int i = 0; i < 100; i++) {
            try {
                // 创建一个消息实例,指定topic、tag、消息内容
                Message message = new Message("TopicTest",// topic
                        "TagA", // tag
                        ("Hello World" + i).getBytes(RemotingHelper.DEFAULT_CHARSET)// message Body
                );
                // 发送消息并且发送结果
                SendResult sendResult = producer.send(message);
                System.out.println(sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException ex) {
                    ex.printStackTrace();
                }
            }
        }
        producer.shutdown();
    }

}

```

`SendResult` 中有一个sendStatus状态, 表示消息的发送状态, 一共有四种状态. 

1. `FLUSH_DISK_TIMEOUT`: 表示没有在规定的时间内完成刷盘(需要Broker的刷新策略设置成`SYNC_FLUSH` 才会报这个错)
2. `FLUSH_SLAVE_TIMEOUT`: 表示在主备方式下, 并且Broker 设置成`成SYNC_MASTER` 方式,没有在设定时间内完成主从同步. 
3. `SLAVE_NOT_AVAILABLE`: 这个状态产生的场景和`FLUSH_SLAVE_TIMEOUT` 类似,表示在主备方式下, 并且Broker 被设置为`SYNC_MASTER` , 并且没有找到被配置为Slave 的Broker. 
4. `SEND OK`: 表示发送成功,发送成功的具体含义, 比如消息是否已经被存储到磁盘? 消息是否被同步到了Slave 上？消息在Slave 是否被写入磁盘? 需要结合所配置的刷盘策略,主从策略来定. 这个状态还可以简单理解为, 没有发生上面累出的三个问题状态就是`SEND OK`.

###  消费者

`consumerGroup`:位于同一个`consumerGroup` 中的consumer 实例和`producerGroup` 中的各个producer 实例承担的角色类似,同一个group 中可以配置多个consumer, 可以体改消费端的并发消费能力以及容灾. 

​    和Kafka 一样,多个consumer 会对消息做负载均衡, 意味着同一个topic 下的不同messageQueue 会分发给同一个group中的不同 consumer. 

​    同时, 如果我们希望消息能够达到广播的目的, 只需要把consumer 加入到不同的group就行. 

​    RocketMQ 提供了两种消息消费模式,一种是pull主动去拉, 另一种是push,被动接收. 但是实际上,RocketMQ 都是pull模式,只是push在pull模式上做了一层封装,也就是pull到消息以后触发业务消息. 

 nameServer 的地址: nameserver 地址,用于或者broker、topic. 



```java
package com.mq;

import org.apache.rocketmq.client.consumer.DefaultMQPullConsumer;
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;

import java.util.List;

/**
 * @author luyanan
 * @since 2020/1/13
 * <p>消费者</p>
 **/
public class Consumer {

    public static void main(String[] args) throws MQClientException {

        // 消费者的组名, 这个和kafka 的一样的, 需要特别注意
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_group");
        // 指定NameServer 的地址, 多个地址用, 隔开
        consumer.setNamesrvAddr("192.168.86.128:9876");

        //  设置consumer 第一次启动是从队列头部开始还是从队尾开始消费
        // 如果非第一次启动,则按照上次消费的位置继续消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        // 订阅TopicTest 下的所有tag的消息
        // * 表示不过滤,可以通过tag来过滤
        consumer.subscribe("TopicTest", "*");

        /**
         * 注册消息监听,这里有两种监听 MessageListenerConcurrently以及MessageListenerOrderly, 前者是普通监听, 后者是顺序监听.
         */
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                System.out.println("receive Message : " + list);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
    }

}

```



## 7. RocketMQ 控制台安装

​    启动好服务之后, 总得有一个可视化节界面来看看我们配置的节点把, Rocket 官方提供了一个可视化控制台,大家可以在这个地址下载. 

https://github.com/apache/rocketmq-externals

这个是RocketMQ的扩展,里面不仅包含控制台的扩展,也包含了对大数据 flume、hbase 等组件的对接和扩展. 

### 7.1  下载源码包

https://github.com/apache/rocketmq-externals/archive/master.zip

###  7.2 解压并修改配置

- `cd /${rocketmq-externals-home}/rocket-console/`

- 修改`application.properties` 文件

- 配置namesrvAddr地址，指向目标服务的ip和端口:

  `rocketmq.config.namesrvAddr=192.168.86.128:9876`

###  7.3  运行

1. ` cd /${rocketmq-externals-home}/rocket-console/`
2. `mvn spring-boot:run`

###  7.4  通过控制台创建消息

要能够发送和接受消息,需要先创建Topic, 这里的topic 和kafka 的topic 的概念是一样的. 
进入到控制台,选择Topic

![](http://files.luyanan.com//img/20200114093613.png)

![](http://files.luyanan.com//img/20200114093628.png)

readQueueNums和writeQueueNums分别表示读队列数和写队列数

writeQueueNums表示producer发送到的MessageQueue的队列个数

readQueueNumbs表示Consumer读取消息的MessageQueue队列个数，其实类似于kafka的分区的概 念 

这两个值需要相等，在集群模式下如果不相等，假如说writeQueueNums=6,readQueueNums=3, 那 么每个broker上会有3个queue的消息是无法消费的。

## 8. RocketMQ消息支持的模式

###  8.1 NormalProducer（普通）

#### 8.1.1 消息同步发送

​    普通消息的发送和接收已经前面演示过了,在上面的案例中是基于同步消息发送模式,也就是说 消息发送出去后, producer 会等到broker 回应后才能继续发送下一个消息. 

![](http://files.luyanan.com//img/20200114094305.png)

####  8.1.2 消息异步发送

​    异步发送是指发送方发送数据后,不等接收方发回响应,接着发送下个数据包的通讯方式.MQ的异步发送, 需要用户实现异步发送回调接口(SendCalllable).消息发送方在发送了一条消息后,不需要等待服务器响应即可返回,进行第二条消息发送。发送方通过回调接口接受服务器响应, 并对响应结果进行处理. 

![](http://files.luyanan.com//img/20200114094904.png)

> 异步代码改造

```java
   // 异步发送
                producer.send(message, new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        System.out.println(sendResult);
                    }

                    @Override
                    public void onException(Throwable throwable) {

                        throwable.printStackTrace();
                    }
                });
```



###  8.2 OneWay

单向(OneWay) 发送特点为发送方只负责发送消息, 不等待服务器且没有回调函数触发,即只发送请求不等待应发, 效率最高. 

![](http://files.luyanan.com//img/20200114095934.png)

> producer.sendOneway(msg);

### 8.3  OrderProducer（顺序）

​     在kafka 中, 消息可以通过自定义分区策略来实现消息的顺序发送,实现原理就是把同一类消息发送到相同的分区上. 

​    在RocketMQ中,是基于多个Message Queue 来实现类似于Kafka 的分区效果的. 如果一个Topic 要发送和接收的数据量非常大,需要能支持增加并行处理的机器来提高处理速度,这时候一个Topic 可以根据需求设置一个或者多个Message Queue . Topic 有了多个Message Queue 后, 消息可以并行的向各个Message Queue 发送, 消息者也可以从多个Message Queue 读取消息并消费. 

> 要了解RocketMQ消息的顺序消费, 还的对RocketMQ的整体架构有一定的了解 

#### 8.3.1 RocketMQ 消息发送和消费的基本原理

​    这是一个比较宏观的部署架构图,Rocketmq 天然支持高可用,它可以支持多主多从的架构部署,这也是和kafka 最大的区别. 

​    原因是RocketMQ 中并没有master 选举功能,所以通过配置多个Master 节点来保证RocketMQ 的高可用, 和所有的集群角色定位一样,Master 节点负责接收事务请求,slave 节点只负责接收读请求,并且接收master 同步过来的数据和slave 保持一致. 当master 挂了以后,如果当前rocketmq 是一主多从, 就意味着无法接收发送端的消息,但是消费者仍然能够继续消费. 

​    所以配置多个主节点后, 可以保证其中一个master节点挂了之后, 另外一个master 节点仍然能够堆外提供消息发送服务. 

​    当存在多个主节点时, 一条消息只会发送到其中一个主节点, rocketmq 对于多个master节点的消息发送, 会做负载均衡,使得消息可以平衡的发送到多个master 节点上. 

​    一个消费者可以同时消费多个master 节点上的消息,在下面的这个架构图中, 两个master 节点恰好可以平均分发到两个消费者上,如果此时只有一个消费者,那么这个消费者会同时消费两个master 节点的数据. 

​    由于每个master 可以配置多个slave, 所以如果其中一个master 挂了, 消息仍然可以被消费者从slave 节点消费到,可以完美的实现rocketmq 消息的高可用. 

​    ![](http://files.luyanan.com//img/20200114103512.png)

接下来,站在topic 的角度来看看消息是如何分发和处理的, 假设有两个master 节点的集群, 创建了一个TestTopic, 并且对这个topic 创建了两个队列,也就是分区. 

​    消费者定义了两个分组, 分组的概念也是跟kafka 一样, 通过分区可以实现消息的广播. 

![](http://files.luyanan.com//img/20200114103713.png)

​    将下来, 站在topic 的角度来看看消息是如何分发和处理的,假设有两个master节点的集群, 创建了一个TestTopic,并且对这个topic 创建了两个队列,也就是分区. 

​    消费者定义了两个分组,分组的概念也是和kafka 一样, 通过分组可以实现消息的广播. 

![](http://files.luyanan.com//img/20200114104339.png)



#### 8.3.2 集群支持

RocketMQ 天生对集群的支持非常友好. 

#####   1. 单Master

优点: 除了配置简单没有什么优点

缺点: 不可靠, 该集群重启或者宕机,将导致 整个服务不可用

##### 2. 多Master 

优点: 配置简单, 性能最高

缺点: 可能会有少量消息丢失(配置相关), 单台机器重启或者宕机期间,该机器下未被消费的消息将在机器恢复前不可订阅, 影响消息实时性. 

##### 3. 多Master,多Slave(异步复制)

每个master 配置一个slave, 有多对master-slave, 集群采用异步复制的方式, 主备有短暂的消息延迟,毫秒级

优点: 性能同多master 一样,实时性高, 主备间切换应用透明, 不需要人工干预. 

缺点: Master 宕机或者磁盘损坏时会有少量的消息丢失. 

##### 4.  多Master,多Slave(同步双写)

每个Master 配一个Slave, 有多对Master-slave, 集群采用同步双写方式, 主备都写成功, 向应用返回成功. 

优点: 服务可用性与数据可用性非常高. 

缺点: 性能比异步集群略低, 当前版本主宕机备不能自动切换为主. 



需要注意的是:在RocketMQ 中, 1台机器只能要么是Master, 要么是slave, 这个在初始化的机器配置里面, 就定死了. 不会像kafka 那样存在master 动态选举的功能. 其中Master 的`brokerId=0,Slaved的brokerId>0`

有点类似于mysql的主从的概念,master 挂了以后, Slave仍然可以提供读服务,但是由于有多主的存在, 当一个master 挂了以后,可以写到其他的master上. 

### 8.4 消息发送到topic 多个MessageQueue

接下来演示一下 topic 创建多个MessageQueue 

   1. 创建一个队列,设置2个写队列和2个读队列,如果读和写的队列不一样,会存在消息无法消费的问题. 

       ![](http://files.luyanan.com//img/20200114112938.png)

        2. 构建生产者和消费者, 参考上面写的生产者和消费者的代码
    
        3. 消费者数量控制对于队列的消费情况

       1. 如果消费队列为2,启动一个消费者,那么这个消费者会消费两个队列
       2. 如果两个消费者消费这个队列,那么意味着消费会均衡平摊到这两个消费者
       3. 如果消费者数大于`readQueueNumbs`, 那么会有一些消费者消费不到消息,浪费资源. 



### 8.5 消息的顺序消费

​    首先需要保证顺序的消息要发送到用一个MessageQueue;其次, 一个MessageQueue 只能被一个消费者消费,这点是消息队列的分配机制来保证的; 最后, 一个消费者内部对一个mq 的消费者要保证是有序的. 

​    我们要做到生产者- messagequeue - 消费者,是一对一的关系.

### 8.6 自定义消息发送负责

​    通过自定义发送策略来实现消息只发送到同一个队列. 

​    因为同一个Topic 会有多个MessageQueue, 如果使用Producer 会轮流向各个Message Queue 发送消息. Consumer 在消费消息的时候, 会根据负载均衡策略, 消费被分配到Message Queue. 

​    如果不经过特定的设置, 某条消息被发往哪个Message Queue, 被哪个Consumer 消费是未知的. 

​      如果业务需要我们把消息发送到指定的Message Queue 里面, 比如把某一类型的消息都发往相同的Message Queue, 那是不是可以实现顺序消息的功能呢？ 

​     和kafka 一样,rocketMQ 也实现了消息的路由功能,我们可以自定义消息分发策略,可以实现`MessageQueueSelector` , 来实现自己的消息分发策略. 

```java
                // 自定义消息发送规则
                producer.send(message, new MessageQueueSelector() {
                    @Override
                    public MessageQueue select(List<MessageQueue> list, Message message, Object o) {
                        int key = o.hashCode();
                        int size = list.size();
                        int index = key % size;
                        return list.get(index);// list.get(0);

                    }
                }, "key_" + i);
```



### 8.7 如何保证消息消费顺序呢? 

​    通过分区规则可以实现同类消息在rocketmq 上的顺序存储,但是对于消费端来说,如何保证消费的顺序?

​    我们前面写的消息消费代码使用的是`MessageListenerConcurrently` 并发监听, 也就是基于多个线程并行来消费消息,这个无法保证消息消费的顺序. 

​    RocketMQ 中提供了`MessageListenerOrderly` 一个类来实现顺序消费. 

```java
   // 顺序消费
        consumer.registerMessageListener(new MessageListenerOrderly() {
            @Override
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> list, ConsumeOrderlyContext consumeOrderlyContext) {
                list.stream().forEach(m -> System.out.println(new String(m.getBody())));
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
```

顺序消费会带来一些问题： 

1. 遇到消息失败的消息, 无法跳过,当前队列消费暂停. 
2. 降低了消息处理的性能. 



### 8.8 消费端的负载均衡

​    和kafka 一样,消费端也会针对Message Queue 做负载均衡,使得每个消费者能够合理的消费多个分区的消息. 

  #####   8.8.1 消费端会通过`RebalanceService` 线程, 10秒做一次机遇topic下的所有队列负载

- 消费者遍历自己所有的topic, 依次调用`rebalanceByTopic`. 
- 根据topic 获取此topic 下的所有queue
- 选择一台broker 获取机遇group 的所有消费端(有心跳向所有broker 注册客户端信息)
- 选择队列分配策略实例 `AllocateMessageQueueStrategy` 执行分配算法

##### 8.8.2 什么时候触发负载均衡

- 消费者启动之后
- 消费者数量发生变更
- 每10秒会触发检查一次`rebalance`

##### 8.8.3 分配算法

RocketMQ 提供了6种分区的分配算法

- `AllocateMessageQueueAveragely`: 平均分配算法(默认)
- `AllocateMessageQueueAveragelyByCircle`: 环状分配消息队列
- `AllocateMessageQueueByConfig`: 按照配置来分配队列, 根据用户指定的配置来进行负载
- `AllocateMessageQueueByMachineRoom`: 按照指定机房来配置队列
- `AllocateMachineRoomNearby`: 按照就近机房来配置队列
- `AllocateMessageQueueConsistentHash`: 一致性Hash,根据消费者的cid进行



## 9. 消息的可靠性原则

​    在实际使用RocketMQ 的时候我们并不能保证每次发送的消息都刚好能被消费者一次性正常消费成功,可能会存在需要多次消费才能成功或者一直消费失败的情况,那作为发送者该做如何处理呢? 

### 9.1 消息消费端的确认机制

​    RocketMQ 提供了ack机制,以保证消息能够被正常消费. 发送方为了保证消息肯定消费成功,只有使用方明确表示消费成功,RocketMQ 才会认为消息消费成功. 中途断电、抛出异常都不会认为成功. 

```java
  consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext consumeConcurrentlyContext) {
                System.out.println("receive Message : " + list);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
```

​    所有的消费者在设置监听的时候会提供一个回调,业务实现消费回调的时候, 当回调方法中返回`ConsumeConcurrentlyStatus.CONSUME_SUCCESS`, RocketMQ 才会认为这批消息(默认是1条),是消费完成的. 如果这个时候消息消费失败, 例如数据库异常、余额扣款不足等一切业务认为消息需要重试的场景,只要返回`回ConsumeConcurrentlyStatus.RECONSUME_LATER`, RocketMQ 就会认为这批消息消费失败了. 

### 9.2 消息的衰减机制

​    为了保证消息肯定至少被消费一次,RocketMQ 会把这批消息重新发回到broker,在延迟的某个时间点(默认是10s, 业务可设置)后, 再次投递到这个`consumerGroup`. 而如果一直这样重复消费都持续到一定次数(默认16次),就会投递到DLQ死信队列,应用可以监控死信队列来做人工干预. 

 可以修改broker-a.conf 文件

> messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h



### 9.3 重试消息的处理机制

​    一般情况下,我们在实际生产中是不需要重试16次, 这样既浪费时间又浪费性能,理论上当尝试重复次数达到我们想要的结果时如果还是消费失败, 那么我们就需要将对应的消息进行记录,并且结束重试尝试.

```java
consumer.registerMessageListener((MessageListenerConcurrently) (list,
consumeOrderlyContext) -> {
for (MessageExt messageExt : list) {
if(messageExt.getReconsumeTimes()==3) {
//可以将对应的数据保存到数据库，以便人工干预
System.out.println(messageExt.getMsgId()+","+messageExt.getBody());
return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
}
}
return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

