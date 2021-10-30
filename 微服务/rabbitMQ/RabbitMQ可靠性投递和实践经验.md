# RabbitMQ可靠性投递和实践经验

##  1. 可靠性投递

在RabbitMQ里面提供了很多保证可靠投递的机制, 这也是RbabitMQ的一个特性. 

​    我们在讲可靠性投递的时候,必须要明确一个问题,因为效率与可靠性是无法兼得的, 如果要保证每一个环节都成功, 势必会对消息的收发效率造成影响. 所以如果是一些业务实时一致性不是特别高的场景, 可以牺牲一些可靠性换取效率. 

​    比如发送通知或者记录日志的这种场景, 如果用户没有收到通知,不会造成业务影响,只要再次发送就可以了. 

​    我们再来回顾一下RabbitMQ的工作模型 

![](http://files.luyanan.com//img/20200109155845.png)

   在我们使用RabbitMQ 收发消息的时候, 有几个主要环节:

1. 代表消息从生产者发送Broker, 生产者把消息发到Broker 之后, 怎么知道自己的消息有没有被Broker 成功接收呢? 

2. 代表消息从Exchange 路由到Queue  

     Exchange 是一个绑定列表,如果消息没有办法路由到正确的队列,会发生什么事情? 怎么处理呢? 

3. 代表消息在Queue 中存储. 

    队列是一个独立运行的服务,有自己的数据库(Mnesina),它是真正用来存储消息的. 如果还没有消费者来消息, 那么消息要一直存储到队列里面. 如果队列出了问题, 消息肯定会丢失. 怎么保证消息在队列稳定的存储呢? 

4. 代表消费者订阅Queue 并消费消息呢? 

     队列的特性是什么? FOFI. 队列里面的消息是一条一条投递的,也就是说, 只有上一条消息被消费者接收以后, 才能把这一条消息从数据库删除,继续投递下一条消息. 那么问题来了, Broker 怎么知道消费者已经接收到了消息呢? 



### 1.1 消息发送到RabbitMQ 服务器

​     第一个环节是生产者发送消息到Broker. 可能以为网络或者Broker 的问题导致消息发送失败,生产者不能确定Broker 有没有正常的接收. 

​    在RabbitMQ 里面提供了两种机制 服务端确认机制,也就是在生产者发送消息给RabbitMQ的服务端的时候, 服务端会通过某种方式返回一个应答,只要生产者收到了这个应答, 就知道消息发送成功了. 

​    第一种是Transaction(事务),第二种Confirm(确认)模式。 



#### 1.1.1 Transaction(事务)模式

​    事务模式怎么使用呢? 

  我们通过一个channel.txSelect() 的方法把信道设置成事务模式,然后就可以发布消息给RabbitMQ了,如果`channel.txCommit()` 方法调用成功, 就说明事务提交成功, 则消息一定达到了RabbitMQ中. 

​    如果在事务提交执行以前由于RabbitMQ 异常崩溃或者其他原因导致抛出异常, 这个时候我们便可以将其捕获, 进而通过执行`channel.txRollback()` 方法来实现事务回滚. 

     ```java
    
    public static void main(String[] args) throws NoSuchAlgorithmException, KeyManagementException, URISyntaxException, IOException, TimeoutException {


        ConnectionFactory factory = new ConnectionFactory();
        factory.setUri(RabbitMQConfig.rabbitMQUrl);


        // 建立连接
        Connection connection = factory.newConnection();
        //创建消息通道
        Channel channel = connection.createChannel();
    
        String msg = "Hello World";
        // 声明交换机
        channel.queueDeclare("TRANSACTION_QUEUE", false, false, false, null);
    
        try {
            channel.txSelect();
            channel.basicPublish("", "TRANSACTION_QUEUE", null, msg.getBytes());
            channel.txCommit();
            System.out.println("消息发送成功");
        } catch (IOException e) {
            channel.txRollback();
            System.out.println("消息发送失败, 回滚");
        }
        channel.close();
        connection.close();
        
    }
     ```

![](http://files.luyanan.com//img/20200110093320.png)

​                                    AMQP 协议抓包示意

​    在事务模式里面, 只有收到了服务器端的`Commit-OK`的指令, 才能提交成功. 所以可以解决生产者和服务端确认的问题. 但是事务模式有一个特点, 它是阻塞的, 一条消息没有发送完毕,不能发送下一条消息,它会榨干RabbitMQ服务器的性能. 所以不建议大家在生产环境中使用. 

​    SpingBoot 中的配置

> rabbitTemplate.setChannelTransacted(true);

​    那么有没有其他可以保证消息被Broker 接收,但是又不大量消耗性能的方式呢? 这个就是第二种模式, 叫做确认(Confirm)模式. 



#### Confirm(确认)模式

​    确认模式有三种, 一种是普通确认模式. 

​    这生产者这边通过调用 `channel.confirmSelect()` 方法将信道设置为Confirm 模式, 然后发送消息. 一旦消息被投递到所有匹配的队列之后, RabbitMQ就会发送一个确认(Basic.ACK) 给生产者,也就是调用 `channel.waitForConfirms()` 返回true, 这样生产者就知道消息被服务端接受了. 

​    这种发送一条消息确认一条消息的方式效率还是不太高,所以我们还有一种批量确认的方式. 批量确认就是在开启Confirm 模式后, 只要`channel.waitForConfirmsOrDie();` 方法没有抛出异常,就代表哦消息都被服务端接受了. 

​    批量确认的方式比单条确认的方式效率要高,但是也有两个问题,第一个就是批量的数量的确认,对于不同的业务, 到底发送多少条消息确认一次? 数量太少, 效率提升不上去. 数量多的话, 又会带来另外一个问题. 比如我们发1000条消息才确认一次, 如果前面999条消息都被服务端接受了,如果第1000条消息被拒绝了,那么前面所有的消息都要被重发. 

   有没有一种方式,可以一边发送一边确认呢? 这个就是异步的确认模式. 

​    异步确认模式需要添加一个`ConfirmListener`, 并且用一个`SortedSet` 来维护没有被确认的消息. 

​    Confirm 模式是在Channel 上开启的,因为`RabbitTemplate` 对Channel 进行封装,叫做`ConfimrCallback`.

​    

```java
rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
@Override
public void confirm(CorrelationData correlationData, boolean ack, String cause) {
if (!ack) {
System.out.println("发送消息失败：" + cause);
throw new RuntimeException("发送异常：" + cause);
}
}
});
```



### 1.2 消息从交换机路由到队列

​    第二个环节就是消息从交换机路由到队列. 在什么情况下, 消息会无法路由到正确的队列呢? 可能是因为路由键错误或者队列不存在. 

​    我们这里有两种方式处理无法路由的消息,一种就是让服务端重发给生产者, 一种是让交换机路由到另一个备份的交换机,

   消息回发的方式: 使用mandatory 参数和 ReturnListener（在 Spring AMQP 中是 ReturnCallback）。

```java
rabbitTemplate.setMandatory(true);
rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback(){
    public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey){
System.out.println("回发的消息：");
System.out.println("replyCode: "+replyCode);
System.out.println("replyText: "+replyText);
System.out.println("exchange: "+exchange);
System.out.println("routingKey: "+routingKey);
}
});
```

   消息路由到备份交换机的方式,在创建交换机的时候, 从属性中指定备份交换机. 

```java
Map<String,Object> arguments = new HashMap<String,Object>();
arguments.put("alternate-exchange","ALTERNATE_EXCHANGE"); // 指定交换机的备份交换机
channel.exchangeDeclare("TEST_EXCHANGE","topic", false, false, false, arguments);
```

>  注意区别: 队列可以指定死信交换机,交换机可以指定备份交换机. 



### 1.3 消息在队列中存储. 

   第三个环节是消息对队列中存储,如果没有消费者的话,队列一直存在在数据库中. 

​    如果RabbitMQ 的服务或者硬件发生故障,比如系统宕机、重启、关闭等等, 可能会导致内存中的消息丢失, 所以我们要本消息本身和元数据(队列、交换机、绑定) 都保存到磁盘

解决方案:

#### 1.3.1 队列持久化

```java
@Bean("Queue")
public Queue Queue() {
// queueName, durable, exclusive, autoDelete, Properties
return new Queue("TEST_QUEUE", true, false, false, new HashMap<>());
}
```

#### 1.3.2 交换机持久化

```java
@Bean("Exchange")
public DirectExchange exchange() {
// exchangeName, durable, exclusive, autoDelete, Properties
return new DirectExchange("TEST_EXCHANGE", true, false, new HashMap<>());
}
```

#### 1.3.3 消息持久化

```java
MessageProperties messageProperties = new MessageProperties();
messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
Message message = new Message("持久化消息".getBytes(), messageProperties);
rabbitTemplate.send("TEST_EXCHANGE", "test", message);		
```



#### 1.3.4 集群

如果只有一个RabbitMQ的节点, 即使交换机、队列、消息做了持久化, 如果服务崩溃或者硬件发生故障,RabbitMQ 的服务一样是不可用的, 所以为了提高MQ服务的可用性, 保障消息的传输, 我们需要有多个RabbitMQ 的节点 . 



### 1.4 消息投递到消费者

​    如果消费者受到消息后没来得及处理即发生异常,或者处理过程中发生了异常, 会导致失败。 服务端应该以某种方式得知消费者对消息的接收情况,并决定是否重新投递这条消息给其他消费者. 

​    RabbitMQ 提供了消费者的消息确认机制(message acknowledgement), 消费者可以自动或者手动的发送ACK 给服务端. 

​    没有收到ACK 的消息, 消费者断开连接后, RabbitMQ 会把这条消息发送给其他消费者, 如果没有其他消费者, 消费者重启后会重新消费这条消息,重复执行业务逻辑. 

​    消费者在订阅队列时,可以指定 `autoAck`参数, 当`autoAck` 参数等于false的时候, RabbitMQ 会等待消费者显式的回复确认信号后才从队列中移除消息. 

​    如何设置手动Ack? 

​    SimpleRabbitListenerContainer 或者 SimpleRabbitListenerContainerFactory

> factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);

`application.peroperties`

```properties
spring.rabbitmq.listener.direct.acknowledge-mode=manual
spring.rabbitmq.listener.simple.acknowledge-mode=manual
```

注意这三个值的区别:

 - NONE: 自动ACK
 - MANUAL: 手动ACK
 - AUTO: 如果方法未抛出异常,则发送ACK

当抛出`AmqpRejectAndDontRequeueException` 异常的时候,则消息会被拒绝, 则不重新入队. 当抛出`ImmediateAcknowledgeAmqpException` 异常, 则消费者会发送ACK, 其他的异常,则消息会拒绝, 且 `requeue=true` 会重新入队. 

在SpringBoot 中, 消费者又怎么调用ACK,或者说怎么获得Cahnenl 的参数呢? 

```java
public class SecondConsumer {
@RabbitHandler
public void process(String msgContent,Channel channel, Message message) throws IOException {
System.out.println("Second Queue received msg : " + msgContent );
channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
}
}
```



​     如果消息无法处理或者消费失败, 也有两种拒绝的方式,`Basic.Reject()` 拒绝单条, `Basic.Nack()` 批量拒绝. 如果`requeue` 参数设置为true ,可以把这条消息重新存入队列, 以便发给下一个消费者(当然, 只有一个消费者的时候, 这种方式可能会出现无限循环重复消费的情况, 可以投递到新的队列或者只打印异常日志).

​    思考: 服务端收到了ACK 或者Nack , 生产者会知道吗? 即使消费者没有接受到消息, 或者消费时出现了异常,生产者也是完全不知情的. 

​    例如: 我们寄出去一个快递，是怎么知道收件人有没有收到的? 因为有物流追踪和签收反馈,所以寄件人可以知道. 

   在没有用上电话的年代, 我们寄出去一封信, 是怎么知道收信人有没有收到信件? 只有收到回信,才知道寄出去的信被收到了. 

   所以,这个是生产者最终确定消费者有没有消费成功的两种方式: 

1. 消费者收到消息,处理完毕后, 调用生产者的API
2. 消费者收到消息, 处理完毕后, 发送一条响应给生产者. 



### 1.5 消费者回调

#### 1.5.1 调用生产者API

​    例如: 提单系统给其他系统分别发送了碎屏保信息后， 其他系统必须在处理完消息后调用提单系统提供的API,来修改提单系统中的数据. 只要API没有被调用,数据状态没有被修改,提单系统就认为下游系统没有收到这条消息. 

#### 1.5.2 发送响应消息给生产者

​    例如：商业银行与人民银行二代支付通信, 无论是人行收到了商业银行的消息还是商业银行收到了人行的消息, 都必须发送一条响应消息(叫做回执报文)。 



![](http://files.luyanan.com//img/20200110140724.png)



### 1.6  补偿机制

   如果生产者的API就是没有被调用, 也就是没有收到消费者的响应消息, 怎么办? 

​    不要着急, 可能是消费者处理时间太长或者网络超时. 

​    生产者与消费者之间应该约定一个超时时间,比如5分钟,对于超过这个时间没有得到响应的消息, 可以设置一个定时重发的机制, 但是要控制发送的间隔和控制次数, 比如每隔2分钟发送一次, 最多重发三次,否则会造成消息堆积. 

   重发可以通过消息落库+定时任务来实现. 

​    重发, 是否发送一模一样的消息. 

​    ATM机上运行的系统叫C端(ATMC),前置系统叫P端(ATMP), 它接受ATMC 的消息, 再转发给卡系统或者核心系统. 

    - 如果客户存款, 没有收到核心系统的应答, 不知道有没有记账成功,最多发送5条存款确认报文, 因为已经吞钞了, 所以要保证成功. 
    - 如果客户取款, ATMC未得到应答,最多发送5次存款冲正报文, 因为没有吐钞, 所以要保证失败。

### 1.7 消息幂等性

​    如果消费者每一个接收生产者的消息都成功了, 只是在响应或者调用API的时候出了问题, 会不会出现消息的重复处理? 例如: 存款100元,ATM重发了5次, 核心系统一共处理了6次,余额会增加了600元. 

​    所以, 为了避免相同消息的重复处理, 必须要采取一定的措施,RabbitMQ 服务端是没有这种控制的(同一批的消息有个递增的DeliveryTag), 它不知道你是不是就是要把一条消息发送两次, 只能在消费端控制. 

​    如何避免消息的重复消费? 

​    消息出现重复可能会有两个原因:

1. 生产者的问题,环节1重复发送消息, 比如在开启了Confirm 模式但未收到确认,消费者重复投递. 
2. 环节4出现了问题, 由于消费者未发送ACK 或者其他原因, 消息重复投递. 
3. 生产者代码或者网络问题 . 

   对于重复发送的消息, 可以对每一条消息生成一个唯一的业务ID, 通过日志或者消息落库来做重复控制. 



### 1.8 最终一致

​    如果确实是消费者宕机了或者代码出现了BUG 导致无法正常消息, 在我们尝试多次重发以后, 消息最终也没有得到处理,怎么办? 

​    例如存款的场景, 客户的钱已经被吞了, 但是余额没有增加, 这个时候银行出现了脏款, 应该怎么处理呢? 如果客户没有主动通知银行, 这个问题是怎么发现的? 银行最终是怎么把这个账务做平的?

   在我们的金融系统中, 都会有双方对账或者多放对账的操作, 通常是在一天的业务结束之后, 第二天营业之前. 我们会约定一个标准, 比如ATM 跟核心系统对账, 肯定是以核心系统为准.ATMC 获取到核心的对账文件, 然后解析,登记成数据, 然后跟自己记录的流水比较, 找出核心有, ATM没有的, 或者ATM有,核心没有的. 或者两边都有大那是金额不一致的数据. 

   对账以后,我们再手工平账.比如取款记了账但是没吐钞的, 做一笔冲正.存款吞了钞但是没记账的, 那么把钱退给客户, 要么补一笔账. 

### 1.9 消息的顺序性

  消息的顺序性指的是消费者消费消息的顺序跟生产者生产消息的顺序是一致的. 

   例如:商户信息同步到其他系统,有三个业务操作: 1. 新增门店 2: 绑定产品； 3: 激活门店. 这种情况下消息消费顺序不能颠倒(门店不存在时无法绑定产品和激活). 

   又比如: 1. 发送微博 2:发表评论;3:: 删除微博,  顺序不能颠倒. 

   在RabbitMQ 中, 一个队列有多个消费者时，由于不同的消费者消费消息的速度是不一样的, 顺序无法保证。 只有一个队列仅有一个消费者的情况下才能保证顺序消费(不同的业务消息发送到不同的专用队列上).

## 2  集群和高可用

### 2.1 为什么要做集群? 

​    集群主要用以实现高可用于负载均衡. 

- 高可用: 如果集群中的某些MQ 服务器不可用, 客户端还可以连接其他MQ 服务器. 
- 负载均衡: 在高并发的场景下, 单台MQ 服务器能处理的消息是有限的. 可以分发给多台MQ服务器. 

​    RabbitMQ 有两种集群模式: 普通集群模式和镜像队列模式. 

### 2.2 RabbitMQ 如何支持集群? 

   应用做集群,需要面对数据同步和通信的问题.因为Erling天生具备分布式的特性, 所以RabbitMQ 天然支持集群, 不需要通过引入ZK或者数据库来实现数据同步. 

  RabbitMQ  通过`/var/lib/rabbitmq/.erlang.cookie` 来验证身份, 需要在所有节点上保持一致. 



### 2.3 RabbitMQ的节点类型?

 集群有两种节点类型,一种是磁盘节点(Disc Node),一种是内存节点(RAM Node). 

- 磁盘节点 : 将元数据(包括队列名称属性、交换机的类型名字属性、绑定、vhost)放在磁盘中. 
- 内存节点: 将元数据放在内存中. 



> 内存节点会将磁盘节点下的地址存放在磁盘中(不然重启就没办法同步数据了), 如果是持久化的消息, 会同时存在在内存和磁盘
>
> 集群中至少需要一个磁盘节点来持久化元数据,否则全部内存节点崩溃时, 就无法同步元数据. 未指定类型的情况下, 默认为磁盘节点. 
>
> 我们一般把应用连接到内存节点(读写快), 磁盘节点用来备份. 

 集群通过25672 端口两两通信, 需要开放防火墙的端口. 

需要注意是的: RabbitMQ 集群无法搭建在广域网上, 除非使用federation 或者 shovel等插件(没必要, 在同一个机房做集群). 

集群的配置步骤

1. 配置hosts
2. 同步`erlang.cookie`
3. 加入集群(join cluster)

###  2.4普通集群

普通集群模式下, 不同的节点之间只会相互同步元数据

![](http://file.luyanan.com//img/20200110161020.png)

疑问: 为什么不直接把队列的内容(消息) 在所有节点上复制一份呢? 

  主要是出于存储和同步数据的网络开销的考虑,如果所有节点都存储相同的数据,就无法达到线性的增加性能和存储容量的目的(堆机器).

​    假如生产者连接的是节点3, 要将消息通过交换机A 路由到队列1, 最终消息还是会转发到节点1上存储, 吨位队列1的内容只是在节点1上. 

​    同理,如果消费者连接的是节点2, 要从队列1上拉取消息, 消息会从节点1转发到节点2上。其他节点起到一个路由的作用,类似于指针 . 

​    普通集群模式不能保证队列的高可用性, 因为队列内容不会复制, 如果节点失效将导致相关队列不可用, 因此我们需要第二种集群模式. 





### 2.5 镜像集群

第二种集群模式叫做镜像对垒. 

 镜像队列模式下, 消息内容会在镜像节点间同步, 可用性更高. 不过也有一定的副作用, 系统性能会降低, 节点过多的情况下同步的代价比较大. 

​    

### 2.6  高可用

​    集群搭建成功后, 如果有多个内存节点, 那么生产者和消费者应该连接到哪个节点呢? 如果在我们的代码中根据一定的策略来选择要使用的服务器, 那每个地方都要修改,客户端的代码就会出现很多的重复, 修改起来也比较麻烦. 

![](http://file.luyanan.com//img/20200110171720.png)

​    所以需要一个负载均衡的组件(例如HAProxy、LVS、Nginx) ,由负载的组件来做路由。这个时候, 只需要俩呢及到负载组件的IP地址就可以了. 

​    ![](http://file.luyanan.com//img/20200110171833.png)

负载分为四层负载和四层负载. 

**四层负载**:工作在OSI模式的第四层, 即传输层(TCP位于第四层),它是根据IP端口进行转发(LVS 支持四层负载). RabbltMQ 是TCP的5672端口. 

**七层负载**: 工作在第七层,应用层(HTTP层位于第七层). 可以根据请求资源类型分配到后端服务器(Nginx支持七层负载, HAProxy 支持四层负载和七层负载). 

   但是如果这个负载的组件也挂了呢? 客户端就无法连接到任意一台MQ 的服务器了.所以负载软件本身也需要做一个集群.新的问题又来了, 如果有两台负载的软件,客户端应该连哪个呢? 

​     负载之上再负载? 陷入死循环了. 这个时候我们就要换个思路了. 

   我们应该需要这样一个组件: 

1. 它本身有路由(负载)功能, 可以监控集群中节点的状态(比如监控HAProxy), 如果某个节点出现异常或者发生了故障，就把它剔除掉. 
2. 为了提高可用性,它也可以部署多个服务, 但是只有一个自动选举出来的Master 服务器(叫做主路由器), 通过广播心跳消息实现. 
3. Master 服务器对外一个虚拟的IP, 提供各种网络功能. 也就是谁抢占到了VIP, 就由谁对外提供网络服务. 应用端只需要连接到这一个IP就行了. 



这个协议叫做做 VRRP 协议（虚拟路由冗余协议 Virtual Router Redundancy Protocol），这个组件就是 Keepalived，它具有 Load Balance 和 High Availability 的功能。

  



## 3. 实践经验总结

### 3.1 资源管理

到底是在消费者创建还是在生产者创建呢? 

​    如果A项目和B项目有互相发送和接收消息,应该创建几个vhost，几个exchange呢? 

交换机和队列, 实际上是作为资源, 由运维管理员创建的. 



### 3.2  配置文件与命名规范

1. 元数据的命名集中放在properties 文件中,不需要硬编码 . 如果有多个系统,可以配置多个`xxx_mq.properties`. 

2. 命名体现元数据类型

   - 虚拟机命名: `_VHOST`
   - 交换机命名:`XXX_Exchange`
   - 队列命名:`XXX_QUEUE`.

3. 命名体现数据来源和去向

    例如： 销售系统发往产品系统的交换机: SALE_TO_PRODUCT_EXCHANGE。做到 见名知义，不用去查文档（当然注释是必不可少的）。

### 3.3 调用封装

在项目中,可以对Template 做进一步封装,简化消息的发送., 

例如:如果交换机、路由键是固定的, 封装之后就只需要一个参数: 消息内容. 

 另外,如果想要平滑的迁移不同的MQ(如果有这种需求),也可以再做一层简单的封装. 



### 3.4 信息落库+定时任务

​    将需要发送的消息保存在数据库中,可以实现消息的可追溯和重复控制,需要配合定时任务来实现. 

1. 将需要发送的消息登记在消息表中. 
2. 定时任务一分钟或者半分钟扫描一次,将未发送的消息发送到MQ 服务器,并且修改状态为已发送. 
3. 如果需要重发消息, 将制定消息的状态修改为未发送即可. 

副作用:降低效率,浪费存储空间. 



### 3.5 生产环境运维监控

虽然RabbitMQ 提供了一个简单的管理界面,但是如果对于系统性能、高可用和其他参数有一定定制化的监控需求的话,我们就需要通过其他的方式来实现监控了. 

​    主要关注:磁盘、内存和连接数. 



### 3.6 日志追踪

​    RabbitMQ 可以通过`Firehose`  功能来记录消息的流入流出的情况, 用于调试、排错. 

​    它是通过创建一个TOPIC 类型的交换机(`amq.rabbitmq.trace`), 把生产者发送给Broker 的消息或者Broker 发送给消费者的消息发送到这个默认的交换机上来实现的. 

​    另外RabbitMQ 也提供了一个个 Firehose 的 GUI 版本，就是 Tracing 插件. 

​     启动Tracing 插件管理界面右侧选项卡会多一个Tracing, 可以添加相应的策略. 

​    RabbitMQ 还提供了其他的插件来增强功能. 

https://www.rabbitmq.com/firehose.html

https://www.rabbitmq.com/plugins.html

###  3.7 如何减少连接数

   在发送大批量消息的情况下, 创建和释放连接依然有不小的开销. 我们可以跟接收方约定批量消息的格式,比如支持JSON 数组的格式,通过合同消息内容,可以减少生产者/消费者与Broker 的连接. 

