# RabbitMQ工作模型与java编程

##  1. MQ入门

### 1.1 MQ的诞生历程

我们要去用MQ, 先来了解一下MQ是怎么诞生的,这样对于它解决了什么问题理解会更加深刻.大家知不知道世界上第一个MQ叫什么名字,是什么时候诞生的? 

​    1983年的时候, 有个在MIT工作的印度小伙伴突发奇想,以前我们的软件相互通信,都是点对点的,而且要实现相同的协议, 能不能有一种专门用来通信的中间件,就想主板(BUS)一样,把不同的软件集成起来呢? 于是它搞了一家公司(Tekneron),开发了世界上第一个消息队列软件The Information Bus(TIB). 最开始的时候, 它被高盛这些公司用在了金融交易里面。 因为TIB 实现了发布订阅(Publish/Subscrbe)模型, 信息的生产者和消费者完全解耦,这个特性引起了电信行业特别是新闻机构的注意. 1994年路透社收购了Teknekron. 

![](http://files.luyanan.com//img/20200107105026.png)

 TIB 的成功马上引起了业界大佬IBM的注意, 他们研发了自己的IBM MQ(IBM Wesphere). 后面微软也加入了这场战斗, 研发了MSMQ. 这个时候,每个厂商的产品是孤立的. 大家都有自己的技术壁垒. 比如一个应用订阅了IBM MQ的消息, 如果又要订阅MSMQ的消息, 因为协议、API不同,又要重复去实现,为什么大家都不愿意去创建标准接口,来实现不同的MQ产品的互通呢? 跟现在微信里面不能打开淘宝页面是一个道理(商业竞争).

​    JDBC 协议大家非常熟悉吧? J2EE 制定了JDBC的规范,那么各个数据库厂商自己去实现协议, 提供jar, 在java里面即可以使用相同的API做操作不同的数据库了. MQ产品的问题也是一样的，2001年的时候,SUM公司发布了JMS 规范,它想要在各大厂商的MQ上面统一包装一层java 的规范,大家都只需要针对API变成就可以了,不需要关注使用了什么样的消息中间件,只要选择合适的MQ驱动.但是JMS 只适用于java语言,它是跟语言绑定的,没有从根本上解决这个问题(只是一个API)。

​    所以在06年的时候,AMQP规范发布了.它是跨语言和跨平台的, 真正的促进了消息队列的繁荣发展. 

​    07年的时候, Rabbit技术公司基于AMQP 开发了BabbitMQ 1.0.为什么要用Erlang语言呢? 因为Erlang 是作者Matthias 擅长的开发语言. 第二个家就是Erlang是为了电话交换机编写的语言, 天生适合分布式和高并发. 

![](http://files.luyanan.com//img/20200107113557.png)

​    为什么要取Rabbit Technologies 这个名字呢? 因为兔子跑得快，而且繁殖起来很疯狂. 

​    从最开始用于金融行业里面,现在RabbitMQ 已经在世界各地的公司中遍地开花. 国内的绝大部分大厂都在用BabbitMQ，包括头条、美团、滴滴、去哪儿、艺龙、淘宝等. 

#### 1.1.2 什么是MQ(Message Queue)

 MQ的本质是什么呢? 

​    消息队列,又叫消息中间件.是指用高效可靠的消息传递机制进行与平台无关的数据交流,并基于数据通信来进行分布式系统的集成. 通过提供消息传递和消息队列模型,可以在分布式坏境下扩展进程的通信(维基百科).

​    基于以上的描述(MQ是用来解决通信的问题), 我们知道, MQ的几个主要特点: 

1. 是一个独立运行的服务。生产者发送消息,消费者接受消息, 需要先跟服务器建立连接. 
2. 采用队列作为数据结构,有先进先出的特点. 
3. 具有发布订阅的模型,消费者可以获取自己需要的消息. 

​    ![](http://files.luyanan.com//img/20200107135046.png)

​    我们可以把RabbitMQ比喻成邮局和邮差, 它是用来帮我们存储和转发消息的. 

   问题: 如果仅仅是解决消息消费的问题, java里面已经有了那么多的队列的实现,为什么不用他们了呢? 这个问题的答案就跟有了HashMap 之后, 为什么还要Redis 作缓存是一样的. 



![](http://files.luyanan.com//img/20200107135232.png)

因为Queue 不能跨进程, 不能在分布系统中使用,并且没有持久化机制等等. 



#### 1.1.3 为什么要使用MQ呢?

​    我们已经知道MQ是什么了, 那在什么地方可以用MQ,或者说，为什么要使用MQ呢? 这是一个很常见的面试题,如果你的项目中使用了MQ,还不知道这个问题的答案,说明你自己从来没有思考总结过,因为这个项目是别人架构设计的,你可能只是做了一些维护的工作.有一天你自己去做架构的时候, 你搞一个MQ进去,理由就是以前的项目也是这么干的,这就很危险了.



#####  1.1.3.1 实现异步通信

​    同步的通信的什么样呢?

  发出一个调用请求后, 在没有得到结果之前, 就不返回. 由调用者主动等待这个调用的结果.

   而异步是相反的, 调用在发出之后, 这个调用就直接返回了, 所以没有返回结果. 也就是说, 当一个异步过程调用发出后, 调用者不会马上得到结果.而是在调用发出后, 被调用者通过状态, 通知来通知调用者,或通过回调函数来处理这个调用.

​    ![](http://files.luyanan.com//img/20200107135909.png)

![](http://files.luyanan.com//img/20200107135921.png)

举个例子:

   大家都用过手机银行的跨行转账功能。大家用APP的转账功能 的时候, 有一个实时模式, 有一个非实时模式. 

​    实时转账实际上是异步通信, 因为这里面涉及的机构比较多, 调用链路比较长, 本行做了一系列处理之后, 转发给银联或者人民银行的支付系统,再转发给接受行, 接受行处理以后再原路返回. 

​    所以转账以后会有一行小字提示: 具体到账时间以对方行处理为准. 也就是说转出行只保证这个转账的消息发出。那为什么到账时间又那么快呢? 很多时间我们转账之后, 不用几秒钟对方就收到了,是因为大部分的MQ都有一个低延迟的特性, 能够在短时间内处理非常多的消息. 

​     很多理财软件体现也是一样的, 先提交申请, 到账时间不定. 这个是用MQ实现系统间异步通信的一个场景. 

#####  1.1.3.2 实现系统解耦

​    第二个主要的功能是用来实现系统解耦.既然说到解耦,那我们先来了解一下耦合的概念. 

​    耦合是系统内部或者系统之间存在相互作用,相互影响和相互依赖. 

​    在我们的分布式系统中, 一个业务流程涉及到多个系统的时候, 他们之间就会形成一个依赖关系. 

​    ![](http://files.luyanan.com//img/20200107140915.png)

   比如我们以12306网站退票为例,在传统的通信方式中, 订单系统发生了退货的动作, 那么要依次调用所有下游系统的API,比如调用库存系统的API恢复库存,因为这张火车票还要释放出去给其他乘客购买;调用支付系统的API,不管是支付宝还是微信还是银行卡,要把手续费扣掉之后, 原路返回给消费者. 调用通知系统API通知用户退货成功. 

```java
// 伪代码
public void returnGoods(){
stockService.updateInventory ();
payService.refund();
noticeService.notice();
}
```

​    这个过程是串行执行的, 如果在恢复库存的时候发生了异常, 那么后面的代码都不会执行. 由于这一系列的动作, 恢复库存、资金退还、发送通知,本质上没有一个严格的先后顺序,也没有直接的依赖关系, 也就是说, 只要用户提交了退货的请求, 后面的这些动作都是要完成的.库存有没有恢复成功, 不影响资金的退还和发送通知. 

​    如果把串行改成并行, 我们有什么思路?

​    (多线程）

   多线程或者线程池是可以实现的, 但是每一个需要并行执行的地方都引入线程,又会带来线程或者线程池的管理问题. 

​    所以, 这种情况下, 我们可以引入MQ实现系统之间依赖关系的解耦合. 

​    引入MQ后: 

​                    ![](http://files.luyanan.com//img/20200107141930.png)

​    订单系统只需要把退货的消息发送到消息队列上, 由各个下游的业务系统自己创建队列, 然后监听队列消费消息. 

​    在这种情况下,订单系统里面不需要配置其他系统的IP、端口号、接口地址了. 因为它不需要关心消费者在网络上的什么位置, 所以下游系统该IP没有任何影响. 甚至不需要关系消费者有没有消费成功, 它只需要把消费者发到消息队列的服务器上就可以了. 

​    这样, 我们就实现了系统之间依赖关系的解耦. 



##### 1.1.3.3  实现流量削峰

​    第三个主要的功能是,实现流量削峰. 

   在很多的电商系统里面, 有一个瞬间流量达到峰值的情况, 比如京东的618、淘宝的双11、小米抢购.普通的硬件服务器肯定支撑不了这种百万或者千万级别的并发量, 就像2012年的小米一样, 动不动就服务器崩溃. 

​    如果通过堆硬件的方式去解决, 那么在流量峰值过去之后就会出现巨大的资源浪费, 那要怎么办呢? 如果说要保护我们的应用服务器和数据库, 限流也是可以的,但是这样又会导致订单的丢失, 没有达到我们的目的. 

​    为了解决这个问题, 我们就可以引入MQ,MQ既然的队列, 一定有队列特性，我们知道队列的特定是什么?

>  先入先出(FIFO)

​    这样我们就可以先把所有的流量都承接下来, 转换成MQ消息发送到消息队列服务器上, 业务层就可以根据自己的消费速率去处理这些消息,处理完成后再返回结果.就像我们在火车站排队一样, 大家只能一个一个单独买票,不会因为人多就导致售票员忙不过来. 如果要处理的快一点, 大不了多开几个窗口(增加几个消费者).

​     这个是我们利用MQ实现流量削峰的一个案例. 

  

总结起来: 

-  对于数据量大或者处理耗时长的操作, 我们可以引入MQ实现异步通信, 减少客户端的等待, 提升响应速度. 
- 对于改动影响大的系统之间, 可以引入MQ实现解耦, 减少系统之间的直接依赖. 
- 对于会出现瞬间的流量峰值的系统, 我们可以引入MQ实现流量削峰, 达到保护应用和数据库的目的. 



​    所以对于一些特定的业务场景, MQ对于优化我们的系统还是有很大的帮助的, 那么大家想一下, 把传统的RPC通信改成MQ通信会不会带来一些问题呢? 

#### 1.1.4 实现消息队列带来的问题

​    系统可用性降低: 原来是两个节点的通信, 现在还需要独立运行一个服务,如果MQ 服务器或者通信网络出现一些问题, 就会导致请求失败. 

​    系统复杂度提高: 为什么说复杂, 第一个就是你必须理解相关的模型和概念,才能正确的配置和使用MQ.第二个就是使用MQ发送消息必须要考虑消息丢失和消息重复消息的问题. 一旦消费没有被正确的消费,就会带来数据一致性问题. 

​    所以我们在做系统架构的时候一定要根据实际情况来分析, 不要因为我们说了这么多MQ能解决的问题, 就盲目的引入MQ.

###  1.2 RabbitMQ 简介

####  1.2.1 基本特性

   官网: https://www.rabbitmq.com/getstarted.html

- **高可靠:** RabbitMQ提供了多种多样的特性让你在可靠性和性能之间做出权衡, 包括持久化、发送应答、发布确认以及高可用性. 
- **灵活的路由:** 通过交换机(Exchange) 实现消息的灵活路由. 
- **支持多客户端:** 对主流开发语言(Python、Java、Ruby、C#、JavaScript、GO、Elixir、Objective-C、Swift等) 都有客户端实现
- **集群和扩展性:** 多个节点组成一个逻辑的服务器, 实现负载均衡. 
- **高可用队列:** 通过镜像队列实现队列中数据的复制. 
- **权限管理:** 通过用户与虚拟机实现权限管理. 
- **插件系统:** 支持各种丰富的插件扩展,同时也支持自定义插件.
- **与Spring集成:** Spring 对AMQP 进行了封装. 

#### 1.2.2 AMQP协议

#####  1.2.2.1 总体介绍

http://www.amqp.org/sites/amqp.org/files/amqp.pdf

AMQP: 高级消息队列协议, 是一个工作于应用层的协议, 最新的版本是1.0版本. 

![](http://files.luyanan.com//img/20200107145823.png)

​    除了RabbitMQ之外, AMQP的实现还有 OpenAMQ、Apache Qpid、Redhat Enterprise MRG、AMQP Infrastructure 、ØMQ、Zyre。

​    除了AMQP之外, RabbitMQ支持多种协议. STOMP、MQTT、HTTP 和Websocket. 

​    可以使用 WireShark 等工具对 RabbitMQ 通信的 AMQP 协议进行抓包。

##### 1.2.2.2 工作模型

由于RabbitMQ 实现了AMQP协议, 所以BabbitMQ 的工作模式也是基于AMQP 的. 理解这种图片至关重要. 

![](http://files.luyanan.com//img/20200107150655.png)



######  1. Broker

​     我们要使用RabbitMQ 来收发消息, 必须安装一个RabbitMQ的服务, 可以安装在window上也可以安装在linux上, 默认是5672端口号。这台BabbitMQ 的服务器我们把它叫做Broker, 中文翻译是代理/中介. 因为MQ 服务器帮助我们做的事情就是存储、转发消息.

######  2. Connection

​     无论是生产者发送消息或者消费者接收消息, 都必须要跟broker 之间建立一个连接, 这个连接是一个TCP的长连接. 

###### 3. Channel

​    如果所有的生产者发送消息和消费者接收消息, 都直接创建和释放TCP连接的话, 对于Broker 来说肯定会造成很大的性能损耗,因为TCP 连接是非常宝贵的资源,创建和释放也要消耗时间. 

​    所有在AMQP 里面引入了Channel 的概念,它是一个虚拟的连接.我们把它翻译成通道,或者消费信道. 这样我们就可以在保持的TCP长连接里面去创建和释放Channel, 大大减少了资源消耗. 另外一个需要注意的是,Channel 是RabbitMQ 原生API里面最重要的编程接口,也就是我们定义交换机、队列、绑定关系、发送消息、消费消息,调用的都是Channel 接口上的方法. 

​    https://stackoverflow.com/questions/18418936/rabbitmq-and-relationship-between-channel-and-connection

###### 4. Queue

​    现在我们已经连接到Broker了, 可以收发消息了.在其他一些MQ里面,比如ActiviteMQ和Kafka, 我们的消息都是发送到队列上. 

​    队列是真正用来存储消息的, 是一个独立运行的进程, 有自己的数据库(Mnesia).

​    消费者获取消息的模式有两种模式,一种是push模式, 只要生产者发到服务器上, 就马上推送到消费者. 另一个是pull模式, 消息存放在服务端, 只有消费端主动获取才能拿到消息. 消费者需要写 一个while循环不断的从队列获取消息吗? 不需要,我们可以基于事件机制,实现消费者对队列的监听. 

​    由于队列有FIFO的特性, 只有确定前一条消息被消费者接收之后, 才会把这条消息从数据库删除,继续投递下一条消息. 

###### 5. Exchange

​    在RabbitMQ 里面永远不会出现消息直接发送到队列的情况. 因为在AMQP 里面引入了交换机(Exchange) 的概念, 用来实现消息的灵活路由. 

​    交换机是一个绑定列表, 用来查找匹配的绑定关系. 

​    队列使用绑定建(Binging Key) 跟交换机建立绑定关系. 

​    生产者发送的消息需要携带路由键(Routing Key), 交换机收到消息时会会根据它保存到绑定列表,决定将消息路由到哪些与他绑定的队列上. 

​    注意: 交换机与队列、队列与消费者都是多对多的关系. 



######  6. Vhost

​    我们每个需要实现基于RabbitMQ的异步通信系统, 都需要在服务器上创建自己要用到的交换机、队列和他们的绑定关系. 如果某个业务系统不想跟别人混用一个系统,怎么办?  再采购一台硬件服务器单独安装一个RabbitMQ 服务? 这种方式成本太高了. 在同一个硬件服务器上安装多个BabbitMQ的服务呢? 比如再运行一个5673的端口? 没有必要,因为RabbitMQ 提供了虚拟主机Vhost. 

​    Vhost 除了可以提高硬件资源的利用率之外, 还可以实现资源的隔离和权限的控制.它的作用类似于编程语言中的namespace 和package,不同的Vhost 中可以有同名的Exchange 和Queue, 他们是完全透明的. 

​    这个时候,我们可以为不同的业务系统创建不同的用户(User), 然后给这些用户分给Vhost权限.比如给风控系统的用户分配风控系统的Vhost的权限, 这个用户就可以访问里面的交换机和队列. 给超级管理员分配所有Vhost 的权限. 

​    我们说到RabbitMQ 引入Exchange 是为了实现消息的灵活路由,到底有哪些路由方式呢? 



##### 1.2.2.3 路由方式

 ######  直连Direct

​    队列与直连的交换机绑定,需指定一个精确的绑定键.

​     生产者发送消息时会携带一个路由键.只有当路由键与其中的某个绑定键完全匹配的时候,这条消息才会从交换机路由到满足路由关系的此队列上. 

![](http://files.luyanan.com//img/20200107162455.png)

> 例如: channel.basicPublish(“MY_DIRECT_EXCHANGE”,”spring”,”msg 1”); 只有第一个队列能收到消息

######  主题Topic

队列与主图类型的交换机绑定时, 可以在绑定键中使用通配符.两个通配符:

- `#` 0个或者多个单词
- `*` 不多不少一个单词

单词(word) 指的是用英文的点 "." 隔开的字符. 例如 `abc.def` 是两个单词. 

![](http://files.luyanan.com//img/20200107163142.png)

 解析: 第一个队列支持路由键以 `spring`开头的消息路由, 后面可以有单词, 也可以没有. 

​         第二个队列支持路由键以`netty` 开头,  而且后面是一个单词的消息路由. 

​         第三个队列支持路由键以`mysql` 结尾, 而且前面是一个单词的消息路由. 

> 例如: 
>
> channel.basicPublish("MY_TOPIC_EXCHANGE","spring.fjd.klj","msg 2"); 只有 第一个队列能收到消息。 channel.basicPublish("MY_TOPIC_EXCHANGE","spring.jvm", "msg 3"); 第 一 个队列和第三个队列能收到消息。



######  广播Fanout

​    主题类型的交换机与队列绑定的时候,不需要指定绑定键. 因此生产者发送消息到广播类型的交换机上, 也不需要携带路由键. 消息达到交换机的时候, 所有与之绑定了的队列, 都会收到相同的消息的副本. 

​    ![](http://files.luyanan.com//img/20200107165221.png)

> 例如: channel.basicPublish("MY_FANOUT_EXCHANGE", "", "msg 4"); 三个队列都会 收到 msg 4。



### 1.3 基本使用

####  1.3.1  安装

这里使用Docker 进行安装

1.  获取镜像

   ```shell
   #指定版本，该版本包含了web控制页面
   docker pull rabbitmq:management
   ```

2. 运行镜像

   ```shell
   #方式一：默认guest 用户，密码也是 guest
   docker run -d --hostname my-rabbit --name rabbit -p 15672:15672 -p 5672:5672 rabbitmq:management
   
   #方式二：设置用户名和密码
   docker run -d --hostname my-rabbit --name rabbit -e RABBITMQ_DEFAULT_USER=user -e RABBITMQ_DEFAULT_PASS=password -p 15672:15672 -p 5672:5672 rabbitmq:manageent 
   ```

3. 访问url

   ```tex
   http://localhost:15672/
   ```



#### 1.3.2 Java API编程

#####  1.3.2.1  添加依赖

```xml
  <dependency>
            <groupId>com.rabbitmq</groupId>
            <artifactId>amqp-client</artifactId>
            <version>5.6.0</version>
        </dependency>
```



##### 1.3.2.2 生产者

```java
package com.mq.rabbit;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * @author luyanan
 * @since 2020/1/8
 * <p>生产者</p>
 **/
public class MyProducer {

    /**
     * <p>交换机</p>
     *
     * @author luyanan
     * @since 2020/1/8
     */
    private final static String EXCHANGE_NAME = "SIMPLE_EXCHANGE";


    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        // 连接ip
        factory.setHost("192.168.86.128");

        // 端口号
        factory.setPort(5672);
        //定义虚拟机
        factory.setVirtualHost("/");

        // 用户
        factory.setUsername("guest");
        // 密码
        factory.setPassword("guest");

        // 建立连接
        Connection connection = factory.newConnection();
        //创建消息通道
        Channel channel = connection.createChannel();
        // 发送消息
        for (int i = 0; i < 10; i++) {
            String msg = "Hello World  ->" + i;
            //  String exchange, String routingKey, BasicProperties props, byte[] body
            channel.basicPublish(EXCHANGE_NAME, "test", null, msg.getBytes());
        }
        channel.close();
        connection.close();
    }

}

```



##### 1.3.2.3 消费者

```java
package com.mq.rabbit;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * @author luyanan
 * @since 2020/1/8
 * <p>消费者</p>
 **/
public class MyConsumer {
    /**
     * <p>交换机</p>
     *
     * @author luyanan
     * @since 2020/1/8
     */
    private final static String EXCHANGE_NAME = "SIMPLE_EXCHANGE";

    /**
     * <p>队列</p>
     *
     * @author luyanan
     * @since 2020/1/8
     */
    private final static String SIMPLE_QUEUE = "SIMPLE_QUEUE";

    public static void main(String[] args) throws IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        // 连接ip
        factory.setHost("192.168.86.128");

        // 端口号
        factory.setPort(5672);
        //定义虚拟机
        factory.setVirtualHost("/");

        // 用户
        factory.setUsername("guest");
        // 密码
        factory.setPassword("guest");

        // 建立连接
        Connection connection = factory.newConnection();
        //创建消息通道
        Channel channel = connection.createChannel();
        // 声明交换机
        // String exchange, String type, boolean durable, boolean autoDelete, Map<String, Object> arguments
        channel.exchangeDeclare(EXCHANGE_NAME, "direct", false, false, null);
        // 声明队列
// String queue, boolean durable, boolean exclusive, boolean autoDelete, Map<String, Object> arguments

        channel.queueDeclare(SIMPLE_QUEUE, false, false, false, null);
        System.out.println("waiting for message...");
        // 绑定队列和交换机
        channel.queueBind(SIMPLE_QUEUE, EXCHANGE_NAME, "test");
        // 创建消费者
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg = new String(body, "UTF-8");
                System.out.println("Received message : '" + msg + "'");
                System.out.println("consumerTag : " + consumerTag);
                System.out.println("deliveryTag : " + envelope.getDeliveryTag());
            }
        };
        while (true) {
            // 开始获取消息
            channel.basicConsume(SIMPLE_QUEUE, true, consumer);

        }
    }
}

```

#####  1.3.2.4  参数详解

1. 声明交换机

   - String  type: 交换机的类型,direct、topic、fanout 中的一种. 
   - boolean durable: 是否持久化,代表交换机在服务器重启后是否还存在

2. 声明队列的参数

   - boolean durable: 是否持久化,代表队列在服务器重启后是否还存在. 

   - boolean exclusive: 是否排他性队列. 排他性队列只能声明它的`connection` 中使用(可以在同一个connection的不同channel中使用),连接断开时自动删除. 

   - boolean autoDelete: 是否自动删除. 如果为true, 至少有一个消费者连接到这个队列,之后所有与这个队列连接的消费者都断开时, 队列会自动删除. 

   - Map<String,Object> arguments: 队列中的其他属性., 例如![](http://files.luyanan.com//img/20200108101236.png)

       

     | 属性                      | 含义                                          |
     | ------------------------- | --------------------------------------------- |
     | x-message-ttl             | 队列中消息的存活时间, 单位是毫秒              |
     | x-expires                 | 队列在多久没消费者访问以后会被删除            |
     | x-max-length              | 队列的最大消息数                              |
     | x-max-length-bytes        | 队列的最大容量, 单位Byte                      |
     | x-dead-letter-exchange    | 队列的死信交换机                              |
     | x-dead-letter-routing-key | 死信交换机的路由键                            |
     | x-max-priority            | 队列中消息的最大优先级,消息的优先级不能超过它 |

     

     

3. 消息属性 BasicProperties

    以下列举了一些主要的参数

   ![](http://files.luyanan.com//img/20200108101800.png)

   

| 参数                       | 释义                           |
| -------------------------- | ------------------------------ |
| Map<String,Object> headers | 消息的其他自定义参数           |
| Integer deliveryMode       | 2 持久化,其他: 瞬态            |
| Integer priority           | 消息的优先级                   |
| String correlationId       | 关联ID,方便RPC相应的与请求关联 |
| String replyTo             | 回调队列                       |
| String expiration          | TTL,消息过期时间, 单位毫秒     |





#### 1.3.3 UI管理界面的使用

​    RabbitMQ可以通过命令(RabbitMQ Cli)、HTTP API管理, 也可以通过可视化的界面去管理,这个网页就是managment 插件. 

#####  1.3.3.1 启用管理插件

1. Windows启动管理插件

   > cd C:\Program Files\RabbitMQ Server\rabbitmq_server-3.6.6\sbin
   >
   > rabbitmq-plugins.bat enable rabbitmq_management

2. Linux 启动管理插件

   > cd /usr/lib/rabbitmq/bin ./rabbitmq-plugins enable rabbitmq_management

##### 1.3.3.2 管理界面访问端口

​    默认端口号为15672, 默认用户是guest,密码是 guest

​    guest 用户默认只能在本地访问,远程用户需要创建其他的用户. 



#####  1.3.3.3  虚拟机

 在Admin 选项卡中

![](http://files.luyanan.com//img/20200108103842.png)

默认的虚拟机是/,可以创建自定义的虚拟机

##### 1.3.3.4 Linux 创建RabbitMQ 用户、权限

假如创建用户 admin，密码 admin,授权访问所有的vhost

```bash
firewall-cmd --permanent --add-port=15672/tcp
firewall-cmd --reload
rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator
rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
```



##  2.RabbitMQ 进阶知识

### 2.1 TTL(Time To Live)

####  2.1.1  消息的过期时间

 有两种设置方式: 

1. 通过队列属性设置消息过期时间

    所有队列中的消息超过时间未被消费, 都会过期. 

   ```java
       @Bean("ttlQueue")
       public Queue ttlQueue() {
           Map<String, Object> map = new HashMap<>();
           // 队列中的消息未被消费10秒后过期
           map.put("x-message-ttl", 10000);
           // 队列30秒没有使用以后会被删除
           map.put("x-expire", 30000);
           return new Queue("TTL_QUEUE", true, true, false, map);
       }
   ```

   

2. 设置单条消息的过期时间

   在发送消息的时候指定消息属性

   ```java
     AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(TtlSender.class);
           RabbitAdmin rabbitAdmin = applicationContext.getBean(RabbitAdmin.class);
           RabbitTemplate rabbitTemplate = applicationContext.getBean(RabbitTemplate.class);
           MessageProperties messageProperties = new MessageProperties();
           // 消息的过期属性(单位ms)
           messageProperties.setExpiration("4000");
           messageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT);
           Message message = new Message("这条消息4秒后过期".getBytes(), messageProperties);
           rabbitTemplate.send("TTL_EXCHANGE", "ttl", message);
   
           // 随队列的过期时间属性,单位ms
           rabbitTemplate.convertAndSend("TTL_EXCHANGE", "ttl", "这条消息");
   ```

     

 同时指定了Message TTL 和Queue TTL,则小的那个时间生效. 



###  2.2 死信队列

#### 2.2.1  消息在某些情况下会变成死信(Dead letter)

   队列在创建的时候可以指定一个死信交换机DLX(Dead Letter Exchange).死信交换机绑定的队列被为死信队列DLQ(Dead Letter Queue),DLX 实际上也就是普通的交换机,DLQ 也是普通的队列(例如替补球员也是普通的球员). 

​    ![](http://files.luyanan.com//img/20200108113112.png)



#### 2.2.2 什么情况下会变成死信? 

1. 消息被消费者拒绝并且未设置重回队列: (NACK || Reject ) && requeue == false
2. 消息过期.
3. 队列达到最长长度,超过了Max Length(消息数) 或者Max Length Bytes (字节数), 最先入队的消息会被发送到

####  2.2.3 死信队列如何使用

##### 2.2.3.1 声明原交换机,原队列相互绑定

​    队列中的消息10秒后过期,因为没有消费者,会变成死信。指定原队列的死信交换机 

```java
  private final static String ORI_USER_EXCHANGE = "ORI_USER_EXCHANGE";


    private final static String ORI_USER_QUEUE = "ORI_USER_QUEUE";

    //  死信交换机
    private final static String DEAD_LETTER_EXCHANGE = "DEAD_LETTER_EXCHANGE";

    // 私信队列

    private final static String DEAD_LETTER_QUEUE = "DEAD_LETTER_QUEUE";

    @Bean("oriUserExchange")
    public DirectExchange exchange() {
        return new DirectExchange(ORI_USER_EXCHANGE, true, false, new HashMap<>())
    }

    @Bean("oriUserQueue")
    public Queue queue() {
        Map<String, Object> map = new HashMap<>();
        map.put("x-message-ttl", 10000); // 10 秒钟后成为死信
        map.put("x-dead-letter-exchange", "DEAD_LETTER_EXCHANGE"); // 队列中的消息变成死信后，进入死信交换机
        return new Queue(ORI_USER_QUEUE, true, false, false, map);
    }


    @Bean
    public Binding binding(@Qualifier("oriUserQueue") Queue queue, @Qualifier("oriUserExchange") DirectExchange exchange) {
        return BindingBuilder.bind(queue).to(exchange).with("ori.user");
    }
    
```



#####  2.2.3.2 声明死信交换机、死信队列, 相互绑定

```java

    @Bean("deadLetterExchange")
    public TopicExchange deadLetterExchange() {
        return new TopicExchange(DEAD_LETTER_EXCHANGE, true, false, new HashMap<>());
    }


    @Bean("deadLetterQueue")
    public Queue deadLetterQueue() {
        return new Queue(DEAD_LETTER_QUEUE, true, false, false, new HashMap<>());
    }

    @Bean
    public Binding bingDead(@Qualifier("deadLetterQueue") Queue queue, @Qualifier("deadLetterExchange") TopicExchange topicExchange) {

        // 无条件路由
        return BindingBuilder.bind(queue).to(topicExchange).with("#");
    }	

```



##### 2.2.3.3  最终消费者监听死信队列

##### 2.2.3.4 生产者发送消息



![](http://files.luyanan.com//img/20200108140243.png)



###  2.3 延迟队列

​    我们在实际业务中有一些需要延迟发送消息的场景,例如:

1. 家里有一台智能热水器,需要在30分钟后启动
2. 未付款的订单, 15分钟后关闭. 

RabbitMQ 本身不支持延时队列,总的来说有三种实现方案: 

1. 先存储到数据库, 用定时任务扫描. 
2. 利用RabbitMQ的死信队列(Dead Letter Queue) 实现. 
3. 利用`rabbitmq-delayed-message-exchange` 插件. 

#### 2.3.1 TTL + DLX 的实现

 基于消息TTL,我们来看一下如何利用死信队列(DLQ) 实现延迟队列

 总体步骤: 

1. 创建一个交换机
2. 创建一个队列,与上述交换机绑定,并且通过属性指定队列的死信交换机. 
3. 创建一个死信交换机
4. 创建一个死信队列
5. 将死信交换机绑定到死信队列
6. 消费者监听死信队列.

​    消息的流转流程: 

   生产者-> 原交换机-> 原队列(超过TTL之后)-> 死信交换机-> 死信队列-> 最终消费者. 



**使用死信队列实现延迟消息的缺点:**

1. 如果统一用队列来设置消息的TTL,当梯度非常多的情况下, 比如1分钟、2分钟、5分钟、10分钟、20分钟、30分钟... 需要创建很多交换机和队列来路由消息. 
2. 如果单独设置消息的TTL, 则可能造成队列中的消息阻塞,前一条消息没有出队(没有被消费), 后面的消息无法投递(比如前一条消息的过期TTL是30min, 第二条消息的TTL 是10min. 10min后, 即使第二条消息应该投递了,但是由于第一条消息还未出队,所以无法投递)
3. 可能存在一定的时间误差



#### 2.3.2 基于延迟队列插件的实现(linux)

​    在RabbitMQ  3.5.7 以及以后的版本提供了一个插件(`rabbitmq-delayed-message-exchange`) 来实现延迟队列功能. 同时插件依赖 Erlang/OPT 18.0以及以上. 

​    插件源码地址: 

https://github.com/rabbitmq/rabbitmq-delayed-message-exchange

插件下载地址: 

https://bintray.com/rabbitmq/community-plugins/rabbitmq_delayed_message_exchange

#####   2.3.2.1 进入插件目录

```bash
whereis rabbitmq 
cd /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.12/plugins

```

##### 2.3.2.2 下载插件

```bash
wget
https://bintray.com/rabbitmq/community-plugins/download_file?file_path=rabbitmq_delayed_message_exchange-0.0.1.ez
```

```bash
mv download_file?file_path=rabbitmq_delayed_message_exchange-0.0.1.ez
rabbitmq_delayed_message_exchange-0.0.1.ez
```

![](http://files.luyanan.com//img/20200108142813.png)

##### 2.3.2.3 启动插件

> rabbitmq-plugins enable rabbitmq_delayed_message_exchange

#####  2.3.2.4 停用插件

> rabbitmq-plugins disable rabbitmq_delayed_message_exchange

##### 2.3.2.5 插件的使用

​    通过声明一个`x-delayed-message` 类型的Exchange来使用`delayed-messaging` 特性. `x-delayed-message` 是插件提供的类型,并不是rabbitmq本身的(区别与direct、topic、fanout、headers).

![](http://files.luyanan.com//img/20200108151743.png)



    ```java
    
    //  死信交换机
    private final static String DEAD_LETTER_EXCHANGE = "DEAD_LETTER_EXCHANGE";
    
    // 私信队列
    
    private final static String DEAD_LETTER_QUEUE = "DEAD_LETTER_QUEUE";
    
    public TopicExchange exchange() {
        Map<String, Object> map = new HashMap<>();
        map.put("x-delayed-type", "direct");
        return new TopicExchange(DEAD_LETTER_EXCHANGE, true, false, map);
    
    }
    
    ```

 生产者: 

​    消息属性中指定 `x-delay` 参数

```java
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(TtlSender.class);
        RabbitAdmin rabbitAdmin = applicationContext.getBean(RabbitAdmin.class);
        RabbitTemplate rabbitTemplate = applicationContext.getBean(RabbitTemplate.class);
        MessageProperties messageProperties = new MessageProperties();


// 延迟的间隔时间, 目标时间减去当前时刻
        messageProperties.setHeader("x-delay", "目标时刻" - System.currentTimeMillis());
        Message message = new Message("这条消息4秒后过期".getBytes(), messageProperties);
        rabbitTemplate.send("DELAY_EXCHANGE", "#", message);
```



### 2.4 服务端流控(Flow Control)

https://www.rabbitmq.com/configure.html

https://www.rabbitmq.com/flow-control.html

https://www.rabbitmq.com/memory.html

https://www.rabbitmq.com/disk-alarms.html

​    当RabbitMQ 生产MQ消息的速度远大于消费消息的速度时,会产生大量的消息堆积, 占用系统资源, 导致机器的性能下降. 我们想要控制服务端接受消息的数量, 应该怎么做呢? 

​    队列有两个控制长度的属性:

   - x-max-length： 队列中最大存储最大消息数, 超过这个数量, 队头的消息会被丢弃. 

   - x-max-length-bytes: 队列中存储的最大消息容量(单位bytes), 超过这个容量, 队头的消息会被丢弃. 

![](http://files.luyanan.com//img/20200108155125.png)

需要注意的是: 设置队列长度只能消息堆积的情况下有意思,而且会删除先入队的消息,不能真正的实现服务端限流. 

​    有没有其他办法实现服务端限流吗? 

####  2.4.1 内存控制

​    RabbitMQ 会在启动时检测机器的物理内存数值.默认当MQ 占用40%以上内存时, MQ会主动抛出一个内存警告并阻塞所有连接(Connections). 可以通过修改rabbitmq.config 文件来调整内存阈值,默认值是0.4,如下所示: 

> [{rabbit, [{vm_memory_high_watermark, 0.4}]}].

​    也可以用命令动态设置,如果设置为0, 则所有的消息都不能发布. 

> rabbitmqctl set_vm_memory_high_watermark 0.3

####  2.4.2 磁盘控制

​    另一种方式是通过磁盘来控制消息的发布. 当磁盘空间低于指定的值时(默认50MB)，触发流控措施. 

​    例如: 指定为磁盘的30% 或者2GB.

https://www.rabbitmq.com/configure.html

> disk_free_limit.relative = 3.0
>
>  disk_free_limit.absolute = 2GB



### 2.5 消费端限流

https://www.rabbitmq.com/consumer-prefetch.html

​    默认情况下, 如果不进行配置, RabbitMQ 会尽可能的把队列中的消息发送到消费者. 因为消费者会在本地缓存消息, 如果消息数量过多,可能回导致OOM 或者影响其他进程的正常运行. 

​    在消费者处理能力有限, 例如消费者数量太少, 或者单条消息的处理时间过长的情况下, 如果我们希望在一定数量的消息消费完之前, 不再推送消息过来, 就要用到消费段的流量控制措施. 

​    可以基于Consumer 或者channel 设置`prefetch count`的值,含义为 Consumer 端的最大的`unacked messages` 数目. 当超过这个数值的消息未被确认, RabbitMQ 会停止投递新的消息给该消费者. 

> channel.basicQos(2); // 如果超过 2 条消息没有发送 ACK，当前消费者不再接受队列消息 channel.basicConsume(QUEUE_NAME, false, consumer);

SimpleMessageListenerContainer

> container.setPrefetchCount(2);

SpringBoot 配置

> spring.rabbitmq.listener.simple.prefetch=2

举例: channel 的 `prefetch count` 设置为5, 当消费者有5条消息没有给Broker 发送ACK后, RabbitMQ 不再给这个消费者投递消息. 

