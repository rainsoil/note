##  3. Spring AMQP

### 3.1 Spring AMQP 介绍

思考： 

Java API 方式编程, 有什么问题? 

Spring 封装RabbitMQ的时候, 它做了什么呢? 

1.  管理对象(队列,交换机、绑定)
2. 封装方法(发送消息、接收消息)

​    Spring AMQP 是对Spring 基于AMQP 的消息收发解决方案, 它是一个抽象层, 不依赖特定的AMQP Broker 实现和客户端的抽象, 所以我们可以很方便的替换. 我们我们可以使用`spring-rabbit` 来实现. 

###  3.2 Spring AMQP 核心组件

#### 3.2.1 ConnectionFactory

​    Spring AMQP 的连接工厂接口, 用于创建连接, `CachingConnectionFactory` 是`ConnectionFactory`的一个实现类. 

#### 3.2.2 RabbitAdmin

​    RabbitAdmin 是`AmqpAdmin`的实现, 封装了对RabbitMQ的基础管理操作, 比如对交换机、队列、绑定的声明和删除等. 

​      编写配置类

```java
package com.rabbitmq.rabbitspring.admin;

import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
import org.springframework.amqp.support.ConsumerTagStrategy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author luyanan
 * @since 2020/1/8
 * <p></p>
 **/
@Configuration
public class AdminConfig {


    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory();

        factory.setUri("amqp://guest:guest@127.0.0.1:5672");
        return factory;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory);
        return rabbitAdmin;
    }


    @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {

        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setConsumerTagStrategy(new ConsumerTagStrategy() {
            @Override
            public String createConsumerTag(String s) {
                return null;
            }
        });
        return container;
    }

}

```



测试类

```java
package com.rabbitmq.rabbitspring.admin;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.rabbit.core.RabbitAdmin;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.context.annotation.ComponentScan;

/**
 * @author luyanan
 * @since 2020/1/8
 * <p></p>
 **/
@ComponentScan("com.rabbitmq.rabbitspring.admin")
public class AdminTest {


    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AdminTest.class);

        RabbitAdmin rabbitAdmin = applicationContext.getBean(RabbitAdmin.class);

        // 声明一个交换机
        rabbitAdmin.declareExchange(new DirectExchange("ADMIN_EXCHANGE", false, false));
        // 声明一个队列
        rabbitAdmin.declareQueue(new Queue("ADMIN_QUEUE", false, false, false));
        // 声明一个绑定
        rabbitAdmin.declareBinding(new Binding("ADMIN_QUEUE", Binding.DestinationType.QUEUE, "ADMIN_EXCHANGE", "admin", null));
    }

}

```

​    为什么我们在配置文件(Spring) 或者配置类(SpringBoot) 里面定义了交换机、队列、绑定关系,并没有直接调用Channel 的declare方法, Spring 在启动的时候就可以帮我们创建这些元数据? 这些数据就是由RabbitAdmin 完成的. 

​    RabbitAdmin 实现了InitializingBean 接口,里面由一个唯一的方法 afterPropertiesSet(),这个方法会在RabbitAdmin 的属性值设置完的时候被调用. 

​    在**afterPropertiesSet()** 方法中, 调用了一个`initialize()` 方法, 这里面创建了三个Collection,用来盛放交换机、队列、绑定关系. 

​    最后依次声明返回类型为Exchange、Queue和Binding的这些Bean, 底层还是调用了Channel 的declare方法. 

    ```java
declareExchanges(channel, exchanges.toArray(new Exchange[exchanges.size()]));
declareQueues(channel, queues.toArray(new Queue[queues.size()]));
declareBindings(channel, bindings.toArray(new Binding[bindings.size()]));
    ```



####  3.2.3 Message

 Message 是Spring AMQP 对消息的封装. 

  两个重要的属性: 

- body: 消息内容
- messageProperties: 消息属性

![](http://file.luyanan.com//img/20200108171217.png)



#### 3.2.4 RabbitTemplate 消息模板

​    RabbitTemplate 是AmqpTemplate 的一个实现(目前到此为止也是唯一的实现), 用来简化消息的收发,支持消息的确认(Configm)与返回(Return). 跟JDBCTemplate 一样, 它封装了建立连接、创建消息信道、收发消息、消息合适转换(ConvertAndSend ->Message)、关闭信道、关闭连接等等操作. 

 对于多个服务器连接, 可以定义多个Template, 可以注入到任何需要收发消息的地方使用. 



确认与回发

```java
 @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMandatory(true);

        // 回发
        rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message,
                                        int replyCode,
                                        String replyText,
                                        String exchange,
                                        String routingKey) {
                System.out.println("回发的消息：");
                System.out.println("replyCode: " + replyCode);
                System.out.println("replyText: " + replyText);
                System.out.println("exchange: " + exchange);
                System.out.println("routingKey: " + routingKey);

            }
        });

        rabbitTemplate.setChannelTransacted(true);
        // 确认
        rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                if (!ack) {
                    System.out.println("发送消息失败：" + cause);
                    throw new RuntimeException("发送异常：" + cause);
                }
            }
        });

        return rabbitTemplate;
    }
```



####  3.2.5 MessageListener 消息侦听

#####  MessageListener :

​    MessageListener 是Spring AMQP 异步消息投递的监听器接口, 它只有一个方法`onMessage`,用于处理消息队列送来的消息, 作用类似于Java API 中的Comsumer. 

​    

#####  MessageListenerContainer

​    MessageListenerContainer 可以理解为MessageListener 的容器, 一个Contailer 只有一个listener, 但是可以生成多个线程使用的MessageListener同时消费消息. 

​    Contailer 可以管理Listener 的生命周期, 可以用于对于消费者进行配置. 

例如: 动态添加队列、对消费者进行设置,例如ConsumerTag、Arguments、并发、消费者数量、消息确认模式等. 

```java
 @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        // 设置监听的队列
        container.setQueues(getSecondQueue(), getThirdQueue());
        // 最小消费数
        container.setConcurrentConsumers(1);
        //最大的消费者数量
        container.setMaxConcurrentConsumers(5);
        // 是否重回队列
        container.setDefaultRequeueRejected(false);

        //签收模式
        container.setAcknowledgeMode(AcknowledgeMode.AUTO);
    
        container.setExposeListenerChannel(true);
        container.setConsumerTagStrategy(new ConsumerTagStrategy() {
            @Override
            public String createConsumerTag(String s) {
                return null;
            }
        });
        return container;
    
    }
```



在SpringBoot 2.0 中新增了一个`DirectMessageListenerContainer。`

##### DirectMessageListenerContainer

Spring 整合IBM MQ、JMS、Kafka 也是这么做的. 

```java
    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory containerFactory = new SimpleRabbitListenerContainerFactory();
        containerFactory.setConnectionFactory(connectionFactory);
        containerFactory.setMessageConverter(new Jackson2JsonMessageConverter());
        containerFactory.setAcknowledgeMode(AcknowledgeMode.NONE);
        containerFactory.setAutoStartup(true);
        return containerFactory;

    }
```

 可以在消费者上指定, 当我们需要监听多个RabbitMQ 的服务器的时候, 指定不同的`MessageListenerContainerFactory`

```java
@Component
@RabbitListener(queues = "first_queue", containerFactory = "rabbitListenerContainerFactory")
public class FirstConsumer {

    @RabbitHandler
    public void process(@Payload Object message) {
        System.out.println("First_queue received mes : " + message);
    }
}

```



产生关系与继承关系

![](http://file.luyanan.com//img/20200109103854.png)

![](http://file.luyanan.com//img/20200109103906.png)





整合代码

```java
public class ContainerSender {

    public static void main(String[] args) throws URISyntaxException {
        ConnectionFactory factory = new CachingConnectionFactory(new URI(RabbitConfig.rabbitUrl));
        SimpleRabbitListenerContainerFactory containerFactory = new SimpleRabbitListenerContainerFactory();
        containerFactory.setConnectionFactory(factory);

        SimpleMessageListenerContainer container = containerFactory.createListenerContainer();
        // 不用工厂模式也可以创建
        container.setConcurrentConsumers(1);
        container.setQueueNames("BASIC_SECOND_QUEUE");
        container.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(Message message) {
                System.out.println("收到消息:" + message);
            }
        });
        container.start();

        AmqpTemplate template = new RabbitTemplate(factory);
        template.convertAndSend("BASIC_SECOND_QUEUE", "msg1");
    }

}
```



#### 3.2.6  转换器MessageConvertor

##### MessageConvertor 的作用?

​    RabbitMQ的消息在网络传输中需要转换成byte[]（字节数组）进行发送, 消费者需要对字节数组进行解析. 

​    在Spring AMQP 中, 消息会被封装为`org.springframework.amqp.core.Message` 对象. 消息的序列化和反序列化, 就是处理Message 的消息体body 对象.    

   如果消息已经是byte[] 格式,就不需要转换. 

  如果是String, 会转换成byte[].

   如果是Java 对象,会使用JDK 序列化对象转换为byte[](体积大,效果差). 

   在调用RabbitTemplate 的`convertAndSend()` 方法发送消息的时候, 会使用`MessageConvertor` 进行消息的序列化,默认使用`SimpleMessageConverter`.

​    在某些情况下,我们需要选择其他的高效的序列化工具.如果我们不想在每次发送消息的时候自己处理消息,就可以直接定义一个 `MessageConverter`.

```java
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {

        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
        return rabbitTemplate;

    }
```

##### MessageConvertor 如何工作的? 

​    调用了RabbitTemplate 的 ` convertAndSend()` 方法时会使用对应的MessageConvertor  进行消息的序列化和反序列化. 

​    序列化: Object-> json-> Message(body)-> byte[]

​    反序列化: byte[] -> Message -> json -> Object



##### 有哪些MessageConvertor?

​    在Spring 中提供一个默认的转换器: `SimpleMessageConverter`.

​    `Jackson2JsonMessageConverter`(RabbitMQ自带): 将对象转换为json, 然后再转换成字节数字进行传递. 

![](http://files.luyanan.com//img/20200109110534.png)

#####  如何自定义MessageConvertor

   例如: 我们要使用Gson 格式化消息. 

 创建一个类, 实现MessageConverter 接口，重写 toMessage()和 fromMessage() 方法。

- toMessage(): Java 对象转换为Message
- fromMessage(): Message 对象转换为Java 对象





###  3.3 Spring 集成RabbitMQ 配置解读

```xml
<rabbit:connection-factory id="connectionFactory" virtual-host="/" username="guest" password="guest" host="127.0.0.1" port="5672" />
<rabbit:admin id="connectAdmin" connection-factory="connectionFactory" />
<rabbit:queue name="MY_FIRST_QUEUE" durable="true" auto-delete="false" exclusive="false" declared-by="connectAdmin" />
<rabbit:direct-exchange name="MY_DIRECT_EXCHANGE" durable="true" auto-delete="false"
                        declared-by="connectAdmin">
<rabbit:bindings>
<rabbit:binding queue="MY_FIRST_QUEUE" key="FirstKey">
</rabbit:binding>
</rabbit:bindings>
</rabbit:direct-exchange>
<bean id="jsonMessageConverter" class="org.springframework.amqp.support.converter.Jackson2JsonMessageConverter" />
<rabbit:template id="amqpTemplate" exchange="${exchange}" connection-factory="connectionFactory" message-converter="jsonMessageConverter" />
<bean id="messageReceiver" class="com.consumer.FirstConsumer"></bean>
<rabbit:listener-container connection-factory="connectionFactory">
<rabbit:listener queues="MY_FIRST_QUEUE" ref="messageReceiver" />
</rabbit:listener-container>
```

![](http://files.luyanan.com//img/20200109111942.png)



###  3.4 SpringBoot集成RabbitMQ

在SpringBoot工程中, 为什么没有定义Spring AMQP的任何一个对象, 也能实现消息的收发? SpringBoot 做了什么? 

我们这里模拟: 

​    3个交换机与4个队列绑定. 4个消费者分别监听4个队列. 

​    生产者发送4条消息,4个队列收到5条消息. 消费者打印出5条消息. 

####  3.4.1  配置文件

 定义交换机、队列

```java
package com.rabbitmq.rabbitspring.springboot;

import org.springframework.amqp.core.*;
import org.springframework.amqp.rabbit.config.SimpleRabbitListenerContainerFactory;
import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.Executors;

/**
 * @author luyanan
 * @since 2020/1/9
 * <p></p>
 **/
@Configuration
public class RabbitConfig {


    public static final String direct_exchange = "DIRECT_EXCHANGE";

    public static final String topicexchange = "topicexchange";

    public static final String fanoutexchange = "fanout_exchange";

    public static final String firstqueue = "first_queue";

    public static final String secondqueue = "second_queue";

    public static final String thirdqueue = "third_queue";

    public static final String fourthqueue = "fourth_queue";


    // 创建三个交换机
    @Bean
    public DirectExchange directExchange() {
        return new DirectExchange(direct_exchange);
    }

    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange(topicexchange);
    }

    @Bean
    public FanoutExchange fanoutExchange() {
        return new FanoutExchange(fanoutexchange);
    }


    //  创建四个队列
    @Bean
    public Queue firstqueue() {
        return new Queue(firstqueue);
    }

    @Bean
    public Queue secondqueue() {
        return new Queue(secondqueue);
    }

    @Bean
    public Queue thirdqueue() {
        return new Queue(thirdqueue);
    }

    @Bean
    public Queue fourthqueue() {
        return new Queue(fourthqueue);
    }


    // 定义绑定关系

    @Bean
    public Binding firstBind(@Qualifier("firstqueue") Queue queue, @Qualifier("directExchange") DirectExchange directExchange) {
        return BindingBuilder.bind(queue).to(directExchange).with("best");
    }


    @Bean
    public Binding secondBind(@Qualifier("secondqueue") Queue queue, @Qualifier("topicExchange") TopicExchange topicExchange) {

        return BindingBuilder.bind(queue).to(topicExchange).with("*.test.*");
    }


    @Bean
    public Binding threadBind(@Qualifier("thirdqueue") Queue queue, @Qualifier("fanoutExchange") FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queue).to(fanoutExchange);
    }

    @Bean
    public Binding fourthBind(@Qualifier("fourthqueue") Queue queue, @Qualifier("fanoutExchange") FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queue).to(fanoutExchange);
    }


    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(ConnectionFactory connectionFactory) {
        SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(new Jackson2JsonMessageConverter());
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        factory.setAutoStartup(true);
        return factory;
    }


}

```



 #### 3.4.2 消费者

这里定义4个消费者分别消费4个队列的消息

```java
package com.rabbitmq.rabbitspring.springboot;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

/**
 * @author luyanan
 * @since 2020/1/9
 * <p></p>
 **/
@Component
@RabbitListener(queues = RabbitConfig.firstqueue, containerFactory = "rabbitListenerContainerFactory")
public class FirstConsumer {

    @RabbitHandler
    public void process(String msg) {
        System.out.println("First queue received msg:" + msg);
    }

}

```

其他消费者类似

#### 3.4.3  生产者配置文件

```java
package com.rabbitmq.rabbitspring.springboot;

import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Component;

/**
 * @author luyanan
 * @since 2020/1/9
 * <p></p>
 **/
@Configuration
public class ProducerConfig {

    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {

        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);

        rabbitTemplate.setMessageConverter(new Jackson2JsonMessageConverter());
        return rabbitTemplate;
    }

}

```



#### 3.4.4 生产者发送消息

```java
package com.rabbitmq.rabbitspring.springboot;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;

/**
 * @author luyanan
 * @since 2020/1/9
 * <p></p>
 **/
@Component
public class RabbitSender {

    private final static String directroutingkey = "java.best";

    private final static String topicroutingkey1 = "shanghai.test.teacher";

    private final static String topicroutingkey2 = "changsha.test.student";
    @Autowired
    RabbitTemplate rabbitTemplate;

    public void send() throws JsonProcessingException {

        rabbitTemplate.convertAndSend(RabbitConfig.direct_exchange, "direct msg", "北京市");
        rabbitTemplate.convertAndSend(topicroutingkey1, "topic msg: shanghai.test.teacher");
        rabbitTemplate.convertAndSend(topicroutingkey2, "topic msg: changsha.test.student");
        Map<String, Object> map = new HashMap<>();
        map.put("11", "111");
        ObjectMapper mapper = new ObjectMapper();
        String value = mapper.writeValueAsString(map);
        rabbitTemplate.convertAndSend(RabbitConfig.fanoutexchange, "", value);
    }

}

```



###  3.5 SpringBoot 参数解析

https://docs.spring.io/spring-boot/docs/2.1.6.RELEASE/reference/html/common-application-properties.html

https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html

> 注: 前缀spring.rabbitmq 全部省略



全部配置总体上分为三类: 连接类、消息消费类、消息发送类. 

> 基于SpringBoot 2.1.5

| 属性值                                   | 说明                                                         | 默认值    |
| ---------------------------------------- | ------------------------------------------------------------ | --------- |
| addres                                   | 客户端连接的地址, 有多个的时候可以使用逗号分隔,该地址可以是ip和port的结合 |           |
| host                                     | RabbitMQ的主机地址                                           | localhost |
| port                                     | RabbitMQ的端口号                                             |           |
| virtual-host                             | 连接到 RabbitMQ 的虚拟主机                                   |           |
| username                                 | 登录到 RabbitMQ 的用户名                                     |           |
| password                                 | 登录到 RabbitMQ 的密码                                       |           |
| ssl.enabled                              | 启动SSL 支持                                                 | false     |
| ssl.key-store                            | 保存SSL证书的地址                                            |           |
| ssl.key-store-password                   | ·访问SSL证书的地址使用的密码                                 |           |
| ssl.trust-store                          | SSL的可信地址                                                |           |
| ssl.trust-store-password                 | 访问SSL可信地址的密码                                        |           |
| ssl.algorithm                            | SSL算法, 默认使用Rabbit 的客户端算法库                       |           |
| cache.channel.checkout-timeout           | 当缓存已满的时候, 获取Channel 的等待时间,单位为毫秒          |           |
| cache.channel.size                       | 缓存中保存的channel 的数量                                   |           |
| cache.connection.mode                    | 连接缓存的模式                                               | CHANNEL   |
| cache.connection.size                    | 缓存的连接数                                                 |           |
| connnection-timeout                      | 连接超时参数单位为毫秒,设置为0代表无穷大                     |           |
| dynamic                                  | 默认创建一个AmqpAdmin的Bean                                  | true      |
| listener.simple.acknowledge-mode         | 容器的acknowledge模式                                        |           |
| listener.simple.auto-startup             | 启动的时候自动启动容器                                       | true      |
| listener.simple.concurrency              | 消费者的最小数量                                             |           |
| listener.simple.default-requeue-rejected | 投递消息失败时是否重新排队                                   | true      |
| listener.simple.max-concurrency          | 消费者的最大数量                                             |           |
| listener.simple.missing-queues-fata      | 容器上声明的队列不可用时是否失败                             |           |
| listener.simple.prefetch                 | 在当个请求中处理的消息个数,它应该大于等于事务数量            |           |
| listener.simple.retry.enabled            | 无论是不是重试的发布                                         | false     |
| listener.simple.retry.initial-interval   | 第一次投递和第二次投递尝试的时间间隔                         | 1000ms    |
| listener.simple.retry.max-attempts       | 尝试投递消息的最大数量                                       | 3         |
| listener.simple.retry.max-interval       | 两次尝试的最大时间间隔                                       | 10000ms   |
| listener.simple.retry.multiplier         | 上一次尝试时间间隔的乘数                                     | 1.0       |
| listener.simple.retry.stateless          | 重试是有状态的还是无状态的                                   | true      |
| listener.simple.transaction-size         | 在一个事务中处理的消息数量,为了获取最佳消化, 该值应设置为小于等于每个请求中处理的消息个数,即listener.prefetch 的值 |           |
| publisher-confirms                       | 开启 Publisher Confirm 值                                    |           |
| publisher-returns                        | 开启Publisher Return机制                                     |           |
| template.mandatory                       | 启动强制信息                                                 | false     |
| template.receive-timeout                 | receive() 方法的超时时间                                     | 0         |
| template.reply-timeout                   | sendAndReceive() 方法的超时时间                              | 5000      |
| template.retry.enabled                   | 设置为true的时候RabbitTemplate 能够实现重试                  | false     |
| template.retry.initial-interval          | 第一次发布与第二次发布消息的时间间隔                         | 1000      |
| template.retry.max-attempts              | 尝试发布消息的最大数量                                       | 3         |
| template.retry.max-interval              | 尝试发布消息的最大时间间隔                                   | 10000     |
| template.retry.multiplier                | 上一次尝试时间间隔的乘数                                     | 1.0       |



