# 分布式消息通信之Kafka 的实现原理

##  消息中间件能做什么? 

消息中间件主要解决的就是分布式系统之间消息的传递问题, 它能够屏蔽各种平台以及协议之间的特性,实现应用程序之间的协同. 举个简单的例子, 就拿一个电商平台的注册功能来简单分析一下,用户注册这一个服务,不仅仅只是insert 一条数据库里面就完事了, 还需要发送激活邮件、发送新人红包或者积分、发送营销短信等一些列操作. 假如说这里面的每一个操作都需要消耗1s , 那么整个注册过程就需要耗时4s 才能响应给用户.

![](http://files.luyanan.com//img/20191227142436.png)

但是我们从注册这个服务可以看到, 每一个子操作都是相对独立的. 同时, 基于领域划分后, 发送激活邮件、发送营销短信、赠送积分以及红包都属于不同的子域. 所以我们可以读这些子操作进行实现异步化执行, 类似于多线程并行处理的概念. 

如何实现异步化呢? 用多线程去实现吗? 多线程当然可以实现, 只是, 消息的持久化、消息的重发这些条件, 多线程并不能满足, 所以需要借助一些开源中间件来解决. 而分布式消息队列就是一个非常好的解决方法, 引入分布式消息队列以后, 架构图就变成了这样了(下图是异步消息队列的场景). 通过引入分布式队列， 就能够大大提升程序的处理效率, 并且还解决了各个模块之间的耦合问题. 

>  这个是分布式消息队列的第一个解决场景: 异步处理

![](http://files.luyanan.com//img/20191227143130.png)

我们再来展开一种场景,通过分布式消息队列来实现流量整形, 比如在电商平台的秒杀场景, 流量会非常大, 通过消息队列的方式就可以很好的缓解高流量的问题. 

![](http://files.luyanan.com//img/20191227143344.png)

- 用户提交过来的请求, 先写入到消息队列. 消息队列是有长度的, 如果消息队列超过指定的长度, 则直接抛弃. 
- 秒杀的具体核心处理业务, 接受消息队列中消息进行处理, 这里的消息处理能力取决于消费端本身的吞吐量. 

当然, 消息中间件还有更多的应用场景, 比如在弱一致性事务模型中, 可以采用分布式消息队列的实现最大能力通知方式来实现数据的最终一致性问题等等. 

##  Java 中使用Kafka 进行通信. 

###  依赖

```xml
   <dependency>
            <groupId>org.apache.kafka</groupId> 
            <artifactId>kafka-clients</artifactId>
            <version>2.0.0</version>
        </dependency>
```

###  发送端代码

```java
package com.mq.kafka.demo;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.IntegerSerializer;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

/**
 * @author luyanan
 * @since 2019/12/27
 * <p>发送端</p>
 **/
public class Producer extends Thread {


    private final KafkaProducer<Integer, String> producer;


    private final String topic;

    public Producer(String topic) {
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.86.128:9092");
        properties.put(ProducerConfig.CLIENT_ID_CONFIG, "producer");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        producer = new KafkaProducer<Integer, String>(properties);
        this.topic = topic;
    }


    @Override
    public void run() {


        int num = 0;
        try {
            while (num < 50) {
                String msg = "test  msg : " + num;
                producer.send(new ProducerRecord<Integer, String>(topic, msg)).get();
                TimeUnit.SECONDS.sleep(2);
                System.out.println("发送消息" + msg);
                num++;
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }


    public static void main(String[] args) {
        new Producer("test").start();
    }
}

```

###  消费端代码


```java
package com.mq.kafka.demo;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.IntegerDeserializer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

/**
 * @author luyanan
 * @since 2019/12/27
 * <p>消费端</p>
 **/
public class Consumer extends Thread {

    private final KafkaConsumer<Integer, String> consumer;


    private final String topic;

    public Consumer(String topic) {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.86.128:9092");
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);//  设置offset 自动提交
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test");
        properties.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 1000);
        properties.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // 当前groupid来说，消息的offset从最早的消息开始消费
        consumer = new KafkaConsumer<Integer, String>(properties);
        this.topic = topic;
    }

    @Override
    public void run() {

        while (true) {
            consumer.subscribe(Collections.singleton(this.topic));
            ConsumerRecords<Integer, String> records = consumer.poll(Duration.ofSeconds(1));
            records.forEach(record -> {
                System.out.println("key:" + record.key() + "--value: " + record.value() + "--offset: " + record.offset());
            });
        }
    }

    public static void main(String[] args) {
        new Consumer("test").start();

    }
}

```



###  异步发送

Kafka 对于消息的发送，可以支持同步和异步, 前面演示的案例中,我们是基于同步发送消息, 同步会需要阻塞, 而异步不需要等待阻塞的过程. 

从本质上来说, Kafka 都是采用异步的方式来发送消息到broker, 但是kafka 并不是每次都发送消息都会直接发送到broker, 而是把消息放到了一个发送队列中, 然后通过一个后台线程不断从队列取出消息进行发送,发送成功后悔触发`callback`。 Kafka 客户端会积累一定量的消息统一组装成一个批量消息发送出去, 触发条件是前面提到的 `batch.size`和 `linger.ms`. 

而同步发送的方法, 无非就是通过 `future.get()` 来等待消息的发送返回结果, 但是这种方法会严重影响消息发送的性能. 

```java

    @Override
    public void run() {
        int num = 0;
        try {
            while (num < 50) {
                String msg = "test  msg : " + num;
                producer.send(new ProducerRecord<>(topic, msg), new Callback() {
                    @Override
                    public void onCompletion(RecordMetadata recordMetadata, Exception e) {

                        System.out.println("callback-> offset: " + recordMetadata.offset() + "--partition: " + recordMetadata.partition());
                    }
                });


                TimeUnit.SECONDS.sleep(2);
                num++;
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```



###  batch.size

生产者发送多个消息到broker 的同一个分区的时候, 为了减少网络请求带来的性能开销, 通过批量的方式来提交信息, 可以通过这个参数来控制批量提交的字节数大小, 默认大小是 16384byte, 也就是16kb, 意味着当一批消息大小达到指定的`batch.size` 的时候会统一发送. 

###  linger.ms

Producer 默认会把两次发送时间间隔内收到的所有Requests  进行一次聚合然后进行发送, 以此提高吞吐量, 而`linger.ms`  及时为每次发送到broker 的请求增加一些delay, 以此来聚合更多的Message 请求. 这个有点像TCP里面的Nagle算法, 在TCP 协议的传输中, 为了减少大量小数据包的发送,采用了Nagle 算法, 也就是基于小包的等-停协议. 

> batch.size和linger.ms 这两个参数是Kafka 性能优化的关键参数, 当二者都配置的时候, 只要满足其中一个要求, 都会发送请求到Broker. 

##  一些基础配置分析

###  group.id

consumer group 是Kafka 提供的可扩展且具有容错性的消费机制. 既然是一个组, 那么组内必然可以有多个消费者或者消费者实例, 他们共享一个公共 的ID， 即Group ID. 组内的所有消费者协调在一起来消费订阅主题(subscribed topics) 的所有分区(partition).  当然, 每个分区只能由一个消费者组内的一个 consumer 来消费。 如下图所示： 分别有三个消费者, 属于两个不同的group, 那么对于`firstTopic` 这个topic 来说, 这两个组的消费者都能同时消费这个topic 的消息, 对于此时的架构来说, 这个`firstTopic` 就类似于ActiveMQ 中的topic 概念. 如下图所示, 如果这三个消费者都属于同一个group, 那么此时`firstTopic` 就是一个Queue 的概念。 

![](http://files.luyanan.com//img/20191227165011.png)

![](http://files.luyanan.com//img/20191227165050.png)

###  enable.auto.commit

消费者消费消息之后自动提交, 只有当消息提交后, 该消息才不会被再次接收到， 还可以配合`auto.commit.interval.ms` 控制自动提交的频率. 

当然, 我们也可以通过`consumer.commitSync()` 的方式实现手动提交. 

### auto.offset.reset

这个参数是针对新的group Id 中的消费者而言, 当有新的groupId 的消费者来消费指定的topic 时, 对于该参数的配置, 会有不同的语义. 

`auto.offset.reset=latest`的情况下, 新的消费者将会从其他消费者最后消费的 offset 处开始消费Topic 下的消息

`auto.offset.reset= earliest` 的情况下, 新的消费者会从该topic 最早的消息进行消费. 

`auto.offset.reset=none`的情况下, 新的消费者加入后, 如果之前不存在offset,则会直接抛出异常. 

###  max.poll.records

此设置限制每次调用poll返回的消息数，这样可以更容易的预测每次poll 间隔要处理的最大值，通过调整此值, 可以减少poll 间隔. 



##  Spring 整合Kafka

SpringBoot 的版本和Kafka的版本有一个对照表格，如果没有按照正确的版本引入, 那么会存在版本问题导致  `ClassNotFound` 的问题, 具体请参考

>  https://spring.io/projects/spring-kafka

### Jar 依赖

```xml
    <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
```

###  生产者

```java
package com.kafka;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

/**
 * @author luyanan
 * @since 2019/12/27
 * <p>生产者</p>
 **/
@Component
public class KafkaProducer {


    @Autowired
    private KafkaTemplate kafkaTemplate;

    public void send(String msg) {
        kafkaTemplate.send("spring-test", "key", "msgData:" + msg);
    }
}

```



### 消费者

```java
package com.kafka;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import java.util.Optional;

/**
 * @author luyanan
 * @since 2019/12/27
 * <p>消费者</p>
 **/
@Component
public class KafkaConsumer {


    @KafkaListener(topics = {"spring-test"})
    public void listener(ConsumerRecord record) {

        Optional<Object> optional = Optional.ofNullable(record.value());
        if (optional.isPresent()) {
            System.out.println("接受到的消息为: " + optional.get());

        }
    }

}

```
###  配置文件


```properties
spring.kafka.bootstrap-servers=192.168.86.128:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.consumer.group-id=spring-test-group
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=true
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.listener.missing-topics-fatal=false

```


### 测试

```java
package com.kafka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ConfigurableApplicationContext;

@SpringBootApplication
public class SpringKafkaApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(SpringKafkaApplication.class, args);
        KafkaProducer producer = applicationContext.getBean(KafkaProducer.class);
        for (int i = 0; i < 30; i++) {
            producer.send(i + "");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }


}

```



##  原理分析

从前面的整个演示过程来看, 只要不是超大规模的使用Kafka, 那么基本上就没什么大问题, 否则, 对于Kafka 本身的运维本身的挑战难度很多, 同时,针对每一个参数的调优也显得很重要. 

### 关于Topic和Partition

####  Topic

在Kafka中, Topic 是一个存储消息的逻辑概念, 可以认为是一个消息集合, 每条消息发送到Kafka 集群的消息都有一个类别。物理上说, 不同的topic 是分开存储的. 

每个topic 可以有多个生产者向它发送消息, 也可以有多个消费者去消费其中的消息. 

![](http://files.luyanan.com//img/20191227200345.png)

#### Partition

每个topic 可以划分为多个分区(每个Topic 至少有一个分区), 同一个topic 下的不同分区包含的消息是不同的, 每个消息在被添加到分区时, 都会被分配一个 offset(称之为偏移量), 它是消息在此分区中的唯一编号, kafka通过offset 保证消息在分区内的顺序, offset的顺序不跨分区, 即Kafka 只能保证在同一个分区内的消息是有序的 

下图中, 对于名字为`test`的 topic ,做了三个分区, 分别是p0,p1,p2

> 每条消息发送到broker的时候, 会根据partition 的规则选择存储到哪一个partition. 如果partition 规则设置合理, 那么所有的消息都会均匀的分布在不同的partition中, 这样就类似数据库的分库分表的概念,把数据做了分片处理.

![](http://files.luyanan.com//img/20191228091254.png)

#### Topic和Partition 的存储

Partition 是以文件的形式存储在文件系统中,必须创建一个名为`firstTopic`的topic,其中有三个partition, 那么在kafka 的数据目录(/tmp/kafka-logs) 中就有三个目录, firstTopic-0-3,  命名规则是 `<topic_name>-<partition_id>`

> sh kafka-topics.sh --create --zookeeper 192.168.11.156:2181--replication-factor 1 --partitions 3 --topic firstTopic



##  关于消息分发

### Kafka 消息分发策略

消息是kafka 中最基本的数据单位, 在kafka中, 一条消息是由 key、value 两部分构成, 在发送一条消息时,我们可以指定这个key, 那么producer 会根据key和partition 机制来判断当前这条消息应该发送并且存储到哪个partition,  我们可以根据需要进行扩展producer 的partition. 

###  代码演示

####  自定义partition

```java
package com.mq.kafka.demo;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;

import java.util.List;
import java.util.Map;
import java.util.Random;

/**
 * @author luyanan
 * @since 2019/12/28
 * <p>自定义Partition</p>
 **/
public class MyPartitioner implements Partitioner {

    private Random random = new Random();

    @Override
    public int partition(String s, Object o, byte[] bytes, Object o1, byte[] bytes1, Cluster cluster) {

        //  获取集群中指定topic 的所有分区信息
        List<PartitionInfo> partitionInfos = cluster.partitionsForTopic(s);

        int size = partitionInfos.size();
        int partitionNum = 0;
        if (o == null) {
            partitionNum = random.nextInt(size);
        } else {
            partitionNum = Math.abs(o1.hashCode()) % size;
        }
        System.out.println("key->" + o + ":value->" + o1 + ":send to partition->" + partitionNum);
        return partitionNum;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> map) {

    }
}

```



#### 发送端代码添加到自定义分区

```java
package com.mq.kafka.demo;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.IntegerSerializer;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

/**
 * @author luyanan
 * @since 2019/12/28
 * <p>自定义Partition</p>
 **/
public class KafkaProducerDemo {


    private final String topic;

    private final boolean isAysnc;


    private final KafkaProducer<Integer, String> producer;


    public KafkaProducerDemo(String topic, boolean isAysnc) {
        this.topic = topic;
        this.isAysnc = isAysnc;

        Properties config = new Properties();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.86.128:9092");
        config.put(ProducerConfig.CLIENT_ID_CONFIG, "kafkaProduceDemo");
        config.put(ProducerConfig.ACKS_CONFIG, -1);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, IntegerSerializer.class.getName());
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        config.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyPartitioner.class.getName());
        producer = new KafkaProducer<Integer, String>(config);
    }
}

```



### 消息的默认分发机制

默认情况下, kafka 采用的是hash 取模的分区算法, 如果key为null, 则会随机分配一个分区, 这个随机是在这个参数 `metadata.max.age.ms` 的时间范围内随机选择一个. 对于这个时间段内, 如果key 为null, 则只会发送到唯一的分区, 这个值默认的情况下是10分钟更新一次. 

关于`Metadata`, 简单理解就是Topic/Partition和Broker 的映射关系, 每一个topic 的每一个partition , 需要知道对应的broker 列表是什么?  leader 是谁？ follower 是谁. 这些信息都是存储在`Metadata` 这个类中. 

###  消费端如何消费指定的分区

通过下面的代码, 就可以消费指定该topic 下的0号分区, 其他分区的数据就无法接受

```java
          TopicPartition topicPartition = new TopicPartition(this.topic, 0);
          consumer.assign(Arrays.asList(topicPartition));
```



## 消息的消费原理

### Kafka消息消费原理演示

在实际生产过程中, 每个topic 都会有多个partition, 多个partition 的好处在于, 一方面能够对broker 上的数据进行分片有效减少了消息的容错从而提升了io性能. 另一方面, 为了提高消费端的消费能力, 一般会通过多个consumer 去消费同一个topic, 也就是消费端的负载均衡机制, 也就是我们接下来要了解的, 在多个partition以及多个consumer 的情况下,消费者是如何消费消息的. 

同时，我们知道kafka 存在consumer group 的概念, 也就是group.id 一样的consumer 属于一个`consumer group`, 组内的所有消费者协调在一起来消费订阅主题的所有分区. 当然每一个分区只能由同一个消费组内的consumer 来消费, 那么在同一个consumer group 里面的consumer 是怎么去分配该消费哪个分区里的数据呢? 如下图所示:3个分区、3个消费者, 那么哪个消费者消费哪个分区? 

![](http://files.luyanan.com//img/20191228113737.png)

对于上面这个图来说, 这3个消费者会分别消费test 这个 topic 的3个分区, 也就是每个consumer 消费一个partition. 

####   代码演示(3个partition 对应3个consumer)

- 创建一个带3个分区的topic
- 启动3个消费者消费同一个topic, 并且这3个consumer 属于同一个组
- 启动发送者进行消费发送

>  演示结果：consumer1会消费partition0分区、consumer2会消费partition1分区、consumer3会消费 partition2分区 如果是2个consumer消费3个partition呢？会是怎么样的结果？

####   代码演示(3个partition 对应2个consumer)

- 基于上面演示的案例的topic 不变
- 启动2个消费者消费该topic
- 启动发送者进行发送消息

> 演示结果: consumer1会消费partition0和partition1分区, consumer2 会消费partition2分区. 

####  代码演示(3个partition  对应4个或者4个以上的consumer)

> 演示结果: 仍然只有3个consumer 对应3个partition, 其他的consumer 无法消费消息. 
>
> 通过这个演示的过程，希望引入接下来需要了解的kafka的分区策略(`Partition Assignment Strategy`)





###  consumer和partition的数量建议

- 如果consumer 比partition多,是浪费, 因为kafka 的设计时是在一个partition 上是不允许转发的, 所以consumer 数不要大于partition数
- 如果consumer数比partition少, 一个consumer 会对应几个partition, 这里主要合理分配consumer数和partition数,否则会导致partition 里面的数据被取的不均匀. 最好partition 数目是consumer 数目的整数倍, 所以partition 的数目很重要, 比如取24, 就很容易设定consumer 的数目. 
- 如果consumer 上多个partition 读取数据, 不保证数据间的顺序性, kafka 只保证在一个partition 上数据是有序的,但是多个partition, 根据你读的顺序会有所不同. 
- 增减consumer ,broker\partition 会导致 `rebalance`, 所以`rebalance` 后consumer 对应的 partition 会发生变化. 



### 思考: 什么时候会触发这个策略呢? 

当出现这几种情况时, kafka 会进行一次分区分配操作, 也就是 kafka consumer 的 rebalance

1. 同一个consumer group 内新增了消费者
2. 消费者离开当前所属的consumer  group,比如主动停机或者宕机
3. topic 新增加了分区(也就是分区数量发生了变化)

kafka consumer 的rebalance 机制规定了一个 consumer group 下的 所有consumer 如何达成一致来分配订阅topic 的每个分区, 而具体如何执行分区策略,就是前面提到过的两种内置的分区策略. 而kafka 对于分配策略这块, 提供了可拔插的实现方式, 也就是说, 除了这两种之外, 我们还可以创建自己的分区机制. 

###  什么是分区分配策略

通过前面的案例演示, 我们应该能猜到, 同一个group 中的消费者对于一个topic 中的多个partition, 存在一定的分区分配策略. 

在kafka 中, 存在三种分区分配策略, 一种是Range(默认), 另一种是RoundRobin(轮询), StickyAssignor(粘性). 在消费端中的`consumerConfig` 中, 通过这个属性来指定分区分配策略.

>  public static final String PARTITION_ASSIGNMENT_STRATEGY_CONFIG = "partition.assignment.strategy";



####  RangeAssignor（范围分区）

Range 策略是对每个主题而言的, 首先对同一个主题里面的分区按照序号进行排序, 并对消费者按照字母顺序进行排序. 

>  假设n = 分区数/消费者数量
>
> m= 分区数%消费者数量
>
> 那么前m个消费者每个分配n+1个分区, 后面的(消费者数量-m) 个消费者每个分配n个分区. 

假设我们有10个分区, 3个消费者, 排完序的分区会是0,1,2,3,4,5,6,7,8,9; 消费者线程排完序将会是C1-0,C2-0,C3-0； 然后将partition的个数除以消费者线程的总数来决定每个消费者线程消费几个分区. 如果除不尽, 那么前面几个消费者将会多消费一个分区. 在我们的例子中,我们有10个分区， 3个消费者, 10/3 =3 而且除不尽,那么消费者线程C1-0 就会多消费一个分区. 

**结果看起来是这样的: **

- C1-0 将消费0,1,2,3 分区
- C2-0 将消费4,5,6 分区
- C3-0 将消费7,8,9 分区

**假设我们有11个分区, 那么最后分区分配的结果看起来是这样的:**

- C1-0 将消费0,1,2,3 分区
- C2-0 将消费4,5,6 分区
- C3-0 将消费7,8,9 分区

**假设我们有2个主题(T1,T2), 分别有10个分区, 那么最后分区分配的结果是这样的**: 

- C1-0 将消费T1主题的0,1,2,3 分区以及T2主题的0,1,2,3分区

- C2-0 将消费T1主题的4,5,6,分区和T2主题的4,5,6分区

- C3-0将消费T1主题的7,8,9分区和T2主题的7,8,9分区

  > 可以看出, C1-0 消费者线程比其他消费者线程多消费了2个分区, 这就是`Range strategy`的一个很明显的弊端. 



#### RoundRobinAssignor（轮询分区）

轮询分区策略是把所有的protition 和所有的consumer 线程都列出来, 然后按照hashcode 进行排序. 最后通过轮询算法分配partition 给消费者线程. 如果所有的consumer 实例的订阅是相同的, 那么partition 会均匀分布. 

在我们的例子里面, 假如按照hashcode 排序完的top-partition 组依次为T1-5, T1-3, T1-0, T1-8, T1- 2, T1-1, T1-4, T1-7, T1-6, T1-9，我们的消费者线程排序为C1-0, C1-1, C2-0, C2-1，最后分区分配的结果 为：

- C1-0 将消费 T1-5, T1-2, T1-6 分区； 
- C1-1 将消费 T1-3, T1-1, T1-9 分区；
- C2-0 将消费 T1-0, T1-4 分区； 
- C2-1 将消费 T1-8, T1-7 分区；

使用轮询分区策略必须满足两个条件: 

1. 每个主题的消费者实例具有相同数量的流
2. 每个消费者订阅的主题必须是相同的. 

####  StrickyAssignor 分配策略

kafka 在0.11X 版本中支持了 `StrickyAssignor`,翻译过来叫 粘滞策略,他主要有两个目的:

- 分区的分配尽可能的均匀
- 分区的分配尽可能的和上次分配保持相同. 



当两者发生冲突的时候, 第一个目标优先于第二个目标. 鉴于这两个目标, StrickyAssignor 分配策略的具体实现要比RangeAssignor和RoundRobinAssi gn or 这两种分配策略要复杂的多, 假设我们有这样一个场景: 

> 假设消费组有3个消费者：C0,C1,C2，它们分别订阅了4个Topic(t0,t1,t2,t3),并且每个主题有两个分 区(p0,p1),也就是说，整个消费组订阅了8个分区：tOpO 、 tOpl 、 tlpO 、 tlpl 、 t2p0 、 t2pl 、t3p0 、 t3pl 
>
> 那么最终的分配场景结果为 
>
> CO: tOpO、tlpl 、 t3p0 
>
> Cl: tOpl、t2p0 、 t3pl 
>
> C2: tlpO、t2pl 
>
> 这种分配方式有点类似于轮询策略，但实际上并不是，因为假设这个时候，C1这个消费者挂了，就势必会造成 重新分区（reblance），如果是轮询，那么结果应该是
>
>  CO: tOpO、tlpO、t2p0、t3p0 
>
> C2: tOpl、tlpl、t2pl、t3pl
>
>  然后，strickyAssignor它是一种粘滞策略，所以它会满足`分区的分配尽可能和上次分配保持相同`，所以 分配结果应该是 
>
> 消费者CO: tOpO、tlpl 、 t3p0、t2p0 
>
> 消费者C2: tlpO、t2pl、tOpl、t3pl 
>
> 也就是说，C0和C2保留了上一次是的分配结果，并且把原来C1的分区分配给了C0和C2。 这种策略的好处是 使得分区发生变化时，由于分区的“粘性，减少了不必要的分区移动
>
> 



###  谁来执行 rebalance 以及管理consumer 的group 呢? 

kafka 提供了一个角色: `coordinator` 来执行对consumer group 的管理. 当 consumer group 的第一个consumer 启动的时候, 它会去和kafka server 确定谁是他们组的`coordinator`.之后该group 内的所有成员都和该`coordinator` 进行协调通信. 



### 如何确定coordinator

consumer group 如何确定自己的`coordinator` 是谁呢? 消费者向kafka 集群中的任意一个broker 发送一个GroupCoordinatorRequest 请求, 服务端会返回一个负载最小的broker 节点的id, 并将该broker 设置为coordinator

###  JoinGroup 的过程

在rebalance 之前, 需要保证coordinator 是已经确定好的, 整个 rebalance 的过程分为两个步骤:  Join和Sync

#### Join

表示加入到consumer group 中, 在这一步, 所有的成员都会向 coordinator 发送 joinGroup的请求. 一旦所有成员都发送了joinGroup 请求, 那么coordinator 会选择一个 consumer 担任leader角色, 并把组成员信息和订阅信息发送到消费者. 

leader 选举算法比较简单, 如果消费组内没有leader, 那么第一个加入消费组的消费者就是消费者的leader,如果这个时候leader 消费者推出了消费组, 那么重新选举一个leader, 这个选举很随意, 类似与随机算法. 

![](http://files.luyanan.com//img/20191228162955.png)

- protocol_metadata: 序列化后的消费者的订阅消息
- leader_id: 消费组中的消费者, coordinator  会选择一个作为leader, 对应的就是member_id
- member_metadata : 对应消费者的订阅信息
- members: consumer group 中全部的消费者的订阅信息
- generation_id: 年代信息, 类似于zookeeper 的epoch 是一样的, 对于每一轮rebalance, generation_id 都会递增. 主要用来保护consumer group. 隔离无效的offset, 也就是上一轮的consumer 成员无法提交offset 到新的 consumer group. 

每个消费者都可以设置自己的分区分配策略, 对于消费组而言, 会从各个消费者上报过来的分区分配策略中选举一个彼此都赞同的策略来实现整体的分区分配, 这个"赞同" 的策略是: 消费组内的各个消费者会通过投票来决定. 

- 在joingroup 阶段, 每个consumer 都会把自己支持的分区分配策略发送到coordinator 
- coordinator  收集到所有消费者的分配策略, 组成一个候选集. 
- 每个消费者需要从候选集中找出一个自己支持的策略,并且为这个策略投票. 
- 最终计算候选集中各个策略的选票数, 票数最多的就是当前消费组的分配策略. 

###  Synchronizing Group State阶段

完成分区分配之后, 就进入了Synchronizing Group State阶段, 主要逻辑是向GroupCoordinator 发送SyncGroupRequest 请求, 并且处理 SyncGroupResponse 响应. 简单来说, 就是leader 将消费者对应的partition 分配方案同步给 consumer group 中的所有 consumer. 

![](http://files.luyanan.com//img/20191228164242.png)

每个消费者都回向coordinator   发送syncgroup 请求, 不过只有leader 节点会发送分配方案, 其他消费者只是打打酱油而已. 当leader 把方案发给 coordinator    之后, coordinator    会把结果设置到SyncGroupResponse 中, 这样所有的成员都知道自己应该消费哪个分区. 

> consumer group 的分区分配方案是在客户端执行的, kafka 将这个权利下发给客户端主要是因为这样有很好的灵活性. 

### 总结

我们再来总结一下consumer group rebalance 的过程

1. 对于每个consumer group 子集, 都会在服务端对应一个GroupCoordinator 进行管理. GroupCoordinator 会在zookeeper 上添加watcher, 当消费者加入或者退出consumer  group的时候, 会修改zookeeper 上修改的数据, 从而触发 GroupCoordinato 开始Rebalance 操作. 
2. 当消费者准备加入某个 consumer group 或者GroupCoordinator发生故障的时候, 消费者并不知道 GroupCoordinator 在网络中的位置, 这个时候就需要确定 GroupCoordinator, 消费者会向集群中的任意一个broker 节点发送ConsumerMetadataRequest 请求,收到请求的broker 会返回一个response 作为响应. 其中包含管理当前ConsumerGroup的GroupCoordinator.
3. 消费者会根据broker 的返回信息, 连接到GroupCoordinator , 并且发送HeartbeatRequest. 发送心跳的目的是为了保证 GroupCoordinator  这个消费者是正常在线的. 当消费者在指定的时间内没有发送心跳请求，则 GroupCoordinator  会出发 rebalance 操作. 
4. **发起join group 请求的两种情况**
   1. 如果GroupCoordinator  返回的心跳包数据包含异常,说明GroupCoordinator  因为前面说的集中情况导致了rebalance 操作, 那这个时候，consumer 会发起 join group操作. 
   2. 新加入的consumer group 的consumer 确定好了GroupCoordinator  之后. 消费者会向GroupCoordinator  发起join group 请求, GroupCoordinator  会收集全部消费者信息之后, 来确认可用的消费者, 并从中选取一个消费者成为group leader. 并把相应的信息(分区分配策略、leader_id...) 封装成一个response返回给所有的消费者. 但是只有group leader 会受到当前consumer group 中的所有消费者信息, 当消费者确定自己是 group leader后, 会根据消费者信息以及选定分区分配策略进行分区分配. 
   3. 接着进入Synchronizing Group State 阶段, 每个消费者会发送SyncGroupRequest 请求到GroupCoordinator  , 但是只有group leader 的请求会存在分区分配结果, GroupCoordinator   会根据group leader 的分区分配结果形成 SyncGroupResponse 返回给所有的consumer
   4. consumer 根据分配结果, 执行相应的操作. 

到这里来说, 我们已经知道了消息的发送分区策略, 以及消费者的分区消费策略和rebalance. 对于应用层面来说, 还有一个最重要的东西还没有讲解, 就是offset, 它类似于一个游标, 表示当前消费的消息的位置. 



### 如何保存消费端的消费位置

####  什么是offset

前面在讲解partition的时候, 提到过offset, 每个topic 可以划分多个分区(每个topic 至少有一个分区). 同一个topic 下的不同分区包含的消息是不同的.每个消息在被添加到分区的时候, 都会被分配一个offset(称之为偏移量), 它是消息在此分区中的唯一编号， kafka通过offset保证消息在分区内的顺序, offset 的顺序不跨分区, 即kafka 只保证在同一个分区的消息是有序的. 对于应用层的消费来说,每次消费一个消息并且提交后, 会保存当前消费到的最近的一个offset, 那么offset 保存到哪里呢? 

![](http://files.luyanan.com//img/20191228173121.png)

####  offset 在哪里维护

在kafka 中,提供了一个**consumer_offsets_* 的一个topic，把offset信息写入到这个topic中**, consumer_offset_* 保存了每个consumer group 某个时刻提交的offset 信息. 

根据前面我们演示的案例, 我们设置了一个`KafkaConsumerDemo`的group id . 首先我们需要找到这个 consumer_group 保存在哪个分区. 

> properties.put(ConsumerConfig.GROUP_ID_CONFIG,"KafkaConsumerDemo");

计算公式: 

> 1. Math.abs(“groupid”.hashCode())%groupMetadataTopicPartitionCount ; 由于默认情况下 groupMetadataTopicPartitionCount有50个分区，计算得到的结果为:35, 意味着当前的 consumer_group的位移信息保存在__consumer_offsets的第35个分区
>
> 2. 执行如下命令, 可以查看当前consumer_group中的offset 位移提交的信息
>
>    ```bash
>    kafka-console-consumer.sh --topic __consumer_offsets --partition 15 --
>    bootstrap-server 192.168.86.128:9092
>    --formatter
>    'kafka.coordinator.group.GroupMetadataManager$OffsetsMessageFormatter'
>    ```
>
>    
>
> 3. 从输出结果中,我们就可以知道这个topic 的offset的位移日志. 
