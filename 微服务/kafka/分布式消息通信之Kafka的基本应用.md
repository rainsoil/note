# 分布式消息通信之Kafka的基本应用

## 消息中间件的背景分析

###  场景分析

我们可以使用阻塞队列+线程池的方式来实现生产者和消费者的模式 . 比如在一个应用中, A 方法调用B方法去执行一些任务处理, 我们可以同步调用, 但是如果这个时候请求比较多的情况下, 同步调用会比较耗时会导致请求阻塞. 我们会使用阻塞队列+线程池的方式来实现异步任务的处理. 

![](http://files.luyanan.com//img/20191224155658.png)

那么问题来了, 在分布式系统用中, 两个服务之间需要通过这种异步的队列的方式来处理任务, 那么单进程级别的队列就无法解决这个问题了. 

因此, 引入了消息中间件, 也就是把消息交给第三方的服务, 这个服务能够实现数据的存储以及传输, 使得在分布式架构下实现跨进程的远程消息通信. 

所以,简单来说: 消息中间件是指利用高效的消息传输机制进行平台无关的数据交流, 并且基于数据通信来是进行分布式系统的集成. 

###  思考一下消息中间件的设计

####  可以先从基本的需求开始考虑

- 最基本的是要能支持消息的发送和接受, 需要设计到网络通信就一定会涉及到NIO
- 消息中心的消息存储(持久化/ 非持久化)
- 消息的序列化和非序列化
- 是否跨语言
- 消息的确认机制, 如何避免重复消费

####  高级功能

- 消息的有序性
- 是否支持事务消息
- 消息收发的性能,对高并发大数据的支持
- 是否支持集群
- 消息的可靠性存储
- 是否支持多协议. 

这个思考的过程其实就是做需求的整理, 然后再使用已有的技术体系进行技术的实现. 而我们所目前阶段去了解的, 无非就是别人根据实际需求进行实现后, 我们如何使用它们提供的API 进行应用而已,但是有了这样一个全局的思考, 那么对于后续的学习这个技术本身而言, 也显得很容易了. 



###  发展过程

实际上消息中间件的发展也是挺有意思的, 我们直到任何一个技术的出现都是为了解决实际问题，**这个问题是通过一种通过通用的软件"总线" 也就是一种通信系统, 解决应用程序之间繁重的信息通讯工作**. 最早的小白鼠就是金融领域, 因为在当时这个领域中, 交易员需要通过不同的终端完成交易, 每台终端显示不同的信息, 如果接入消息总线, 那么交易员只需要在一台终端上操作, 然后订阅其他终端感兴趣的消息,于是就诞生了发布订阅模型(pubsub).  同时诞生了世界上以第一个现代消息队列软件(TIB) `The information Bus`, TIB  允许开发者建立一系列规则去描述消息内容, 只要消息按照这些规则发布出去,任何消费者应用都能订阅自己感兴趣的消息. 随着TIB 带来的甜头被广泛的应用在各大领域, IBM 也开始研究开发自己的消息中间件, 3年之后IBM 的消息队列 IBM MQ 产品系列发布, 之后的一段时间MQ系列进入成了 `WebSphere MQ` 统治商业消息队列平台市场. 

包括后期微软也研发了自己的消息队列(MSMQ)

各大厂商纷纷研究自己的MQ,但是他们是以商业化的模式运营自己的MQ软件, 商业MQ 想要解决的是应用互通的问题, 而不是创建标准的接口来允许不同的MQ 互通, 所有有些大型的金融公司可能会使用多个供应商的MQ产品, 来服务企业内部的不同的应用. 那么问题来了, 如果应用已经订阅了TIB MQ的消息然后突然需要消费IBM MQ的消息, 那么整个实现过程就会很麻烦了. 为了解决这个问题, 在2001 年诞生了`Java Message Service(JMS)`, JMS 通过提供公共的java API方式, 隐藏单独MQ产品供应商的实现接口, 从而跨域了不同MQ消费和解决互通问题. 从技术层面来说, Java应用程序只需要针对JMS API 编程, 选择合适的MQ驱动即可. JMS 会处理其他部分, 这种方案其实是通过单独标准化接口来整个很多不同的接口, 效果还是不错的.  **但是碰到了互用性的问题, 两套使用不同编程语言的程序如何通过它们的异步消息传递机制相互通信呢?  这个时候就需要定义一个异步消息的通用标准. **

所以AMQP`（Advanced Message Queuing Protocol）` 高级消息队列就产生了, 它使用一套标准的底层协议, 加入了许多其他特征来支持互用性, 为现代应用丰富了消息传递需求, 针对标准编码的任何人都可以和任何AMQP 供应商提供的MQ 服务器进行交互. 

除了JMS和AMQP 规范以外, 还有一种MQTT(Message Queueing Telemetry[特莱米缺] Transport)， 他是专门为小设备设计的, 因为计算性能不高的设备不能适应AMQP的基本要求, 而如今, MQTT是互联网(IOT）生态系统中的主要成分之一. 

> 今天讲解的Kafka, 它并没有遵循上面所说的协议规范, 注重吞吐量, 类似udp和tcp. 

## Kafka 的介绍

###  什么是Kafka

Kafka是一款分布式消息发布和订阅系统, 他的特点是高性能、高吞吐量. 

最早设计的目的是作为 `LinkedIn`的活动流和运营数据的处理管道, 这些数据主要用来对用户做用户画像分析以及服务器性能的一些监控. 

所以Kafka 一开始设计的目标就是作为一个高性能、高吞吐量的消息系统, 所以适合运用在大数据传输场景. 



### Kafka 的应用场景

由于Kafka具有更好的吞吐量、内置分区、冗余和容错性的优点(Kafka每秒可以处理几十万消息), 让Kafka 成为了一个很好的大规模消息处理应用的解决方案, 所以在企业级应用场景,主要会应用于如下几个方面: 

####  行为追踪

Kafka 可以用于追踪用户浏览页面、搜索以及其他行为, 通过发布、订阅模式实时记录到对应的topic中, 通过后端大数据平台接入处理分析, 并作更进一步的实时处理和监控. 

####  日志收集

日志收集方面, 有很多比较优秀的产品, 比如 `Apache Flume`, 很多公司使用Kafka 代理日志聚合, 日志聚合表示从服务器上收集日志文件, 然后放到一个集中的平台(文件服务器)进行处理. 在实际应用开发中, 我们应用程序的log 都会输出到本地的磁盘上, 排查问题的话通过linux 命令来搞定. 如果应用程序组成了负载均衡集群, 并且集群的数量有几十台以上, 那么想通过日志快速定位到问题 , 就是很麻烦的问题. 所以一般都会做一个日志统一收集平台管理log日志用来快速查询重要应用的问题. 所以很多公司的套路都是把应用日志集中到Kafka上, 然后分别导入到ES和HSFS上, 用来做实时检索和离线统计数据备份等. 而另一方面,Kafka 本身就提供了很好的api 来集成日志并且做日志收集 . 

![](http://files.luyanan.com//img/20191226214934.png)

##  Kafka 本身的架构

一个典型的Kafka 集群包含若干 `Produce`（可以是应用节点产生的消息, 也可以是通过`Flume` 收集日志产生的事件）, 若干个Broker(Kafka支持水平扩展), 若干个Consumer Group以及一个zookeeper 集群. Kafka 通过zookeeper 管理集群配置和服务协同. Produce 使用push模式将消息发布到broker, consumer 通过监听使用pull 模式从 broker 订阅并消费消息 

多个broker 协同工作, produce和consumer 部署在各个业务逻辑中,三者通过 zookeeper 管理协调请求和转发, 这样就组成了一个高性能的分布式消息发布和订阅系统. 

 图上有一个细节是和其他mq中间件不同的点是, produce 发送消息到 broker 的过程是push, 而consumer 是从broker 消费消息的过程是pull, 主动去拉数据. 而不是broker 把数据主动发送给 consumer. 

![](http://files.luyanan.com//img/20191227105757.png)



###   名词解释

####  broker

Kafka集群包含一个或者多个服务器, 这种服务器被称为 broker. broker 端不维护数据的消费状态, 提升了性能. 直接使用磁盘进行存储, 线性读写速度快, 避免了数据在JVM内存和系统内存之间的复制, 减少了耗性能的创建对象和垃圾回收. 

#### Produce

负责发布消息到Kafka broker

#### Consumer

消息消费者,向Kafka broker 读取消息的客户端, consumer 从broker 拉取(pull) 数据并进行处理. 

#### Topic

每条发布到Kafka 集群的消息都有一个类别, 每个类别被称为Topic,(物理上不同Topic 的消息分开存储, 逻辑上一个Topic 的消息虽然保存于一个或者多个broker上, 但用户只需要指定消息的Topic 即可生产或者消费数据而不必关心数据存于何处). 

#### Partition

Partition 是物理上的概念, 每个Topic 包含一个或者多个Partition

#### Consumer Group

每个Consumer 属于一个特定的 Consumer Group(可为每个Consumer 指定group name,若不指定group name 则属于默认的group)

#### Topic & Partition

Topic 在逻辑上可以被认为是一个queue, 每条消费者都必须指定它的topic, 可以简单理解为必须指明把这条消息放进哪个queue.为了使得kafka 的吞吐率可以线性提高, 物理上把topic 分为一个或者多个Partition,  每个Partition 在物理上对应一个文件夹, 该文件夹下存储这个Partition 的所有消息和索引文件. 若创建 topic1和topic2两个topic, 且分别有13个分区和19个分区, 则整个集群上会相应生成共32个文件夹(本文所有集群共8个节点,此处topic1和topic2 replication-factor均为1). 

## Kafka 的安装部署

###  单机部署

#### 下载Kafka

https://archive.apache.org/dist/kafka/2.0.0/kafka_2.11-2.0.0.tgz

#### 安装 过程

安装过程非常简单, 只需要解压就行, 因为这个是编译好之后的可执行程序. 

> tar -zxvf kafka_2.11-2.0.0.tgz

####  配置zookeeper

因为Kafka 依赖于zookeeper 来做master 选举以及其他数据的维护, 所以需要先启动zookeeper 节点, Kafka 内置了zookeeper的服务, 所以 在bin目录下提供了这些脚本.

```text
zookeeper-server-start.sh
zookeeper-server-stop.sh
```

在config 目录下, 存在一些配置文件

```text
zookeeper.properties
server.properties

```

所以我们可以通过下面的脚来启动zk服务。当然也可以自己搭建zk 的集群来实现. 

> sh zookeeper-server-start.sh -daemon ../config/zookeeper.properties

####  启动和停止Kafka

- 修改 `server.properties` , 增加zookeeper 的配置

  > zookeeper.connect=localhost:2181

-  启动kafka

  >  ./kafka-server-start.sh -daemon ../config/server.properties

- 停止Kafka

  > sh kafka-server-stop.sh -daemon config/server.properties

### Kafka的基本操作

#### 创建Topic

> sh kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 -- partitions 1 --topic test

`Replication-factor` 表示该topic 需要在不同的broker 中保存几份, 这里设置为1, 表示在两个broker 中保存两份. 

`Partitions` 分区数

####  查看Topic

> sh kafka-topics.sh --list --zookeeper localhost:2181

#### 查看Topic的属性

> sh kafka-topics.sh --describe --zookeeper localhost:2181 --topic first_topic

####  消费消息

> sh kafka-console-consumer.sh --bootstrap-server 192.168.13.106:9092 --topic test --from-beginning

####  发送消息

> sh kafka-console-producer.sh --broker-list 192.168.244.128:9092 --topic first_topic

## 集群安装

###  环境准备

- 准备三台虚拟机
- 分别把kafka 的安装包部署在三台机器上. 

### 修改配置

以下配置修改均为 `server.properties`

- 分别修改三台机器的server.properties 配置, 同一个集群中的每个机器的id必须唯一. 

  > broker.id=0
  >
  > broker.id=1
  >
  > broker.id=2

- 修改zookeeper 的连接配置

> zookeeper.connect=192.168.86.128:2181

-  修改 listeners配置

如果配置了listeners, 那么消息生产者和消费者会使用 `listeners`的配置来进行消息的收发, 否则, 会使用 localhost

`PLAINTEXT` 表示协议, 默认是明文, 可以选择其他加密协议. 

>  listeners=PLAINTEXT://192.168.86.128:9092

- 分别启动三台服务器

  > sh kafka-server-start.sh -daemon ../config/server.properties

  