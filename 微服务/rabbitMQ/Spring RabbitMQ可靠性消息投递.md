# Spring RabbitMQ可靠性消息投递

RabbitMQ核心基础概念。

- Server:又称之为Broker，接受客户端的连接，实现AMQP实体服务。
- Connection:连接，应用程序与Broker的网络连接。
- Channel:网络信道，几乎所有的操作都在Channel中进行，Channel是进行消息读写的通道。客户端可以建立多个Channel，每个Channel代表一个会话任务。
   如果每一次访问RabbitMQ都建立一个Connection，在消息量大的时候建立TCP Connection的开销将是巨大的，效率也较低。Channel是在connection内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的channel进行通讯，AMQP method包含了channel id帮助客户端和message broker识别channel，所以channel之间是完全隔离的。Channel作为轻量级的Connection极大减少了操作系统建立TCP connection的开销。
- Message：消息，服务器和应用程序之间传送的数据，由Message Properties和Body组成。Properties可以对消息进行修饰，比如消息的优先级，延迟等高级特性，Body就是消息体内容。
- Virtual Host:虚拟地址，用于进行逻辑隔离，最上层的消息路由。一个Virtual Host里面可以有若干个Exchange和Queue，同一个Virtual Host里面不能有相同名称的Exchange或者Queue。
- Exchange：交换机，接收消息，根据路由键转发消息到绑定的队列。
- Binding:Exchange和Queue之间的虚拟连接，binding中可以包含routing key。
- Routing key:一个路由规则，虚拟机可以用它来确定如何路由一个特定消息。
- Queue：也可以称之为Message Queue(消息队列)，保存消息并将它们转发到消费者。

通过下面2张图，我们能大概能明白AMQP协议模型和消息流转过程。在Exchange和Message Queue上面还有Virtual host。记住同一个Virtual Host里面不能有相同名称的ExChange和Message Queue。



![img](https:////upload-images.jianshu.io/upload_images/4636177-22901bbb566bbcbc.png?imageMogr2/auto-orient/strip|imageView2/2/w/682/format/webp)

image.png

![img](https:////upload-images.jianshu.io/upload_images/4636177-a49faed915ac3182.png?imageMogr2/auto-orient/strip|imageView2/2/w/825/format/webp)

image.png

接着我们看下面的图，这是RabbitMQ消息可靠性投递的解决方案之一。



![img](https:////upload-images.jianshu.io/upload_images/4636177-911c83939b34bf1b.png?imageMogr2/auto-orient/strip|imageView2/2/w/853/format/webp)

image.png



1.将消息落地到业务db和Message db。
 2.采用Confirm方式发送消息至MQ Broker，返回结果的过程是异步的。Confirm消息，是指生产者投递消息后，如果Broker收到消息后，会给生产者一个ACK。生产者通过ACK，可以确认这条消息是否正常发送到Broker，这种方式是消息可靠性投递的核心。
 3、4：在这里将消息分成3种状态。status=0表示消息正在投递中，status=1表示消息投递成功，status=2表示消息投递了3次还是失败。生产者接收Broker返回的Confirm确认消息结果，然后根据结果更新消息的状态。将status的状态从投递中改成投递成功即可。
 5.在消息Confirm过程中，可能由于网络闪断问题或者是Broker端出现异常，导致回送消息失败或者出现异常。这时候，就需要生产者对消息进行可靠性投递，保证投递到Broker的消息可靠不丢失。还有一种极端情况值得我们考虑，那就是网络闪断。我们的消息成功投递到Broker，但是在回送ACK确认消息时，由于网络闪断，生产者没有收到。此时我们再重新投递此消息可能会造成消费端重复消费消息了。这时候需要消费端去做幂等处理(生成全局消息ID，判断此消息是否消费过)。对于没有投递成功的消息，我们可以设置一个重新投递时间。比如一个消息在5分钟内，status状态还是0，也就是这个消息还没有成功投递到Broker端。这时候我们需要一个定时任务，每隔几分钟从Message db中拉取status为0的消息。
 6.将拉取的消息执行重新投递操作。
 7.设置最大消息投递次数。当一个消息被投递了3次，还是不成功，那么将status置为2。最后交给人工解决处理此类问题或者将消息转存到失败表。

下面讲解一下涉及到消息可靠性的知识点和一些配置了。

application-dev.properties



```csharp
#rabbtisMQ配置
spring.rabbitmq.host=127.0.0.1
spring.rabbitmq.port=5672
spring.rabbitmq.username=root
spring.rabbitmq.password=root
spring.rabbitmq.virtual-host=/
#消费者数量
spring.rabbitmq.listener.simple.concurrency=10
#最大消费者数量
spring.rabbitmq.listener.simple.max-concurrency=10
#消费者每次从队列获取的消息数量
spring.rabbitmq.listener.simple.prefetch=1
#消费者自动启动
spring.rabbitmq.listener.simple.auto-startup=true
#消费失败，自动重新入队
#重试次数超过最大限制之后是否丢弃（true不丢弃时需要写相应代码将该消息加入死信队列）
#true，自动重新入队，要写相应代码将该消息加入死信队列
#false,丢弃
spring.rabbitmq.listener.simple.default-requeue-rejected=false
#是否开启消费者重试（为false时关闭消费者重试，这时消费端代码异常会一直重复收到消息）
spring.rabbitmq.listener.simple.retry.enabled=true
spring.rabbitmq.listener.simple.retry.initial-interval=1000
spring.rabbitmq.listener.simple.retry.max-attempts=3
spring.rabbitmq.listener.simple.retry.multiplier=1.0
spring.rabbitmq.listener.simple.retry.max-interval=10000

#启动发送重试策略
spring.rabbitmq.template.retry.enabled=true
#初始重试间隔为1s
spring.rabbitmq.template.retry.initial-interval=1000
#重试的最大次数
spring.rabbitmq.template.retry.max-attempts=3
#重试间隔最多10s
spring.rabbitmq.template.retry.max-interval=10000
#每次重试的因子是1.0 等差
spring.rabbitmq.template.retry.multiplier=1.0
#
#RabbitMQ的消息确认有两种。
#一种是消息发送确认。这种是用来确认生产者将消息发送给交换器，交换器传递给队列的过程中，
# 消息是否成功投递。
#发送确认分为两步，一是确认是否到达交换器，二是确认是否到达队列。
#第二种是消费接收确认。这种是确认消费者是否成功消费了队列中的消息。
# 确认消息发送成功，通过实现ConfirmCallBack接口，消息发送到交换器Exchange后触发回调
spring.rabbitmq.publisher-confirms=true
# 实现ReturnCallback接口，如果消息从交换器发送到对应队列失败时触发
# （比如根据发送消息时指定的routingKey找不到队列时会触发）
spring.rabbitmq.publisher-returns=true
# 消息消费确认，可以手动确认
spring.rabbitmq.listener.simple.acknowledge-mode=manual
#在消息没有被路由到合适队列情况下会将消息返还给消息发布者
#当mandatory标志位设置为true时，如果exchange根据自身类型和消息routingKey无法找到一个合适的queue存储消息，
# 那么broker会调用basic.return方法将消息返还给生产者;当mandatory设置为false时，
# 出现上述情况broker会直接将消息丢弃;通俗的讲，mandatory标志告诉broker代理服务器至少将消息route到一个队列中，
# 否则就将消息return给发送者;
spring.rabbitmq.template.mandatory=true
```

要确保RabbitMQ消息的可靠要保证以下3点：
 1.publisher Confirms：要确保生产者的消息到broker的可靠性。可能会发生消息投递到broker过程中，broker挂了的情况。
 2.Exchange，Queue，Message持久化：RabbitMQ是典型的内存式消息堆积。我们需要把message存储到磁盘中。如果是未持久化的消息存储在内存中，broker挂了那么消息会丢失。
 3.consumer acknowledgement:消费者确认模式有3种：none(没有消息会发送应答),auto(自动应答),manual(手动应答)。为了保证消息可靠性，我们设置手动应答，这是为什么呢？采用自动应答的方式，每次消费端收到消息后，不管是否处理完成，Broker都会把这条消息置为完成，然后从Queue中删除。如果消费端消费时，抛出异常。也就是说消费端没有成功消费该消息，从而造成消息丢失。为了确保消息被消费者正确处理，我们采用手动应答(调用basicAck、basicNack、basicReject方法)，只有在消息得到正确处理下，再发送ACK。

RabbitMQ消息确认有2种：消息发送确认，消费接收确认。消息发送确认是确认生产者将消息发送到Exchange，Exchange分发消息至Queue的过程中，消息是否可靠投递。第一步是否到达Exchange，第二步确认是否到达Queue。

实现ConfirmCallBack接口,消息发送到Exchange后触发回调。



```tsx
    // 消息发送到交换器Exchange后触发回调
    private final RabbitTemplate.ConfirmCallback confirmCallback =
            new RabbitTemplate.ConfirmCallback() {
                @Override
                public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                    log.info("生产端confirm...");
                    log.info("correlationData=" + correlationData);
                    String messageId = correlationData.getId();
                    if (ack) {
                        //confirm返回成功,更新消息投递状态
                        brokerMessageLogMapper.updateMessageLogStatus(messageId, Constants.ORDER_SEND_SUCCESS, new Date());
                    } else {
                        // 失败则进行具体的后续操作，重试或者补偿等手段。
                        log.info("异常处理...");
                    }
                }
            };
```

实现ReturnCallBack接口，消息从Exchange发送到指定的Queue失败触发回调



```cpp
    // 如果消息从交换器发送到对应队列失败时触发
    private final RabbitTemplate.ReturnCallback returnCallback =
            new RabbitTemplate.ReturnCallback() {
                @Override
                public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                    log.info("message=" + message.toString());
                    log.info("replyCode=" + replyCode);
                    log.info("replyText=" + replyText);
                    log.info("exchange=" + exchange);
                    log.info("routingKey=" + routingKey);
                }
            };
```

消息确认机制开启，需要配置以下信息



```bash
spring.rabbitmq.publisher-confirms=true
spring.rabbitmq.publisher-returns=true
spring.rabbitmq.listener.simple.acknowledge-mode=manual
```

之前说过手动应答可以调用basicAck,basicNack,basicReject方法，下面来讲讲。

手动确认消息，当multiple为false，只确认当前的消息。当multiple为true，批量确认所有比当前deliveryTag小的消息。deliveryTag是用来标识Channel中投递的消息。RabbitMQ保证在每个Channel中，消息的deliveryTag是从1递增。



![img](https:////upload-images.jianshu.io/upload_images/4636177-2b086b54dda6ef5a.png?imageMogr2/auto-orient/strip|imageView2/2/w/447/format/webp)

image.png

当消费端处理消息异常时，我们可以选择处理失败消息的方式。如果requeue为true，失败消息会重新进入Queue，试想一下，如果消费者在消费时发生异常，那么就不会对这一次消息进行ACK，进而发生回滚消息的操作，使消息始终放在Queue的头部，然后不断的被处理和回滚，导致队列陷入死循环，为了解决这种问题，我们可以引入重试机制(当重试次数超过最大值，丢弃该消息)或者是死信队列+重试队列。
 requeue为false，丢弃该消息。



![img](https:////upload-images.jianshu.io/upload_images/4636177-51a70431b7c91ad1.png?imageMogr2/auto-orient/strip|imageView2/2/w/621/format/webp)

image.png

和basicNack用法一样。



![img](https:////upload-images.jianshu.io/upload_images/4636177-3be76bcaebead13b.png?imageMogr2/auto-orient/strip|imageView2/2/w/460/format/webp)

image.png

为了配合Return机制，我们要配置`spring.rabbitmq.template.mandatory=true`。它的作用是在消息没有被路由到合适的队列情况下，Broker会将消息返回给生产者。当mandatory为true时，如果Exchange根据类型和消息Routing Key无法路由到一个合适的Queue存储消息，那么Broker会调用Basic.Return回调给handleReturn()，再回调给ReturnCallback，将消息返回给生产者。当mandatory为false时，丢弃该消息。



```dart
    @Override
    public void handleReturn(int replyCode,
            String replyText,
            String exchange,
            String routingKey,
            BasicProperties properties,
            byte[] body)
        throws IOException {

        ReturnCallback returnCallback = this.returnCallback;
        if (returnCallback == null) {
            Object messageTagHeader = properties.getHeaders().remove(RETURN_CORRELATION_KEY);
            if (messageTagHeader != null) {
                String messageTag = messageTagHeader.toString();
                final PendingReply pendingReply = this.replyHolder.get(messageTag);
                if (pendingReply != null) {
                    returnCallback = new ReturnCallback() {

                        @Override
                        public void returnedMessage(Message message, int replyCode, String replyText, String exchange,
                                String routingKey) {
                            pendingReply.returned(new AmqpMessageReturnedException("Message returned",
                                    message, replyCode, replyText, exchange, routingKey));
                        }
                    };
                }
                else if (logger.isWarnEnabled()) {
                    logger.warn("Returned request message but caller has timed out");
                }
            }
            else if (logger.isWarnEnabled()) {
                logger.warn("Returned message but no callback available");
            }
        }
        if (returnCallback != null) {
            properties.getHeaders().remove(PublisherCallbackChannel.RETURN_CORRELATION_KEY);
            MessageProperties messageProperties = this.messagePropertiesConverter.toMessageProperties(
                    properties, null, this.encoding);
            Message returnedMessage = new Message(body, messageProperties);
            returnCallback.returnedMessage(returnedMessage,
                    replyCode, replyText, exchange, routingKey);
        }
    }
```

当消息路由不到合适的Queue，会在回调给ReturnCallck这些信息。



![img](https:////upload-images.jianshu.io/upload_images/4636177-345e272cdc10e53c.png?imageMogr2/auto-orient/strip|imageView2/2/w/913/format/webp)

image.png

如果消费端忘记了ACK，这些消息会一直处于Unacked 状态。由于RabbitMQ消息消费没有超时机制，也就是程序不重启，消息会一直处于Unacked状态。当消费端程序关闭时，这些处于Unack状态的消息会重新恢复成Ready状态。这时候会出现一种情况：当消费端程序开启时，由于Broker端积压了大量的消息，又可能会让消费端崩溃。所以我们要对消费端进行限流处理。RabbitMQ提供了一种qos(Quality of Service,服务质量保证)功能，即在非自动ACK前提下，如果一定数量的消息未被ACK前，不进行新消息的消息。

![img](https:////upload-images.jianshu.io/upload_images/4636177-75d3bd2a4021cf3e.png?imageMogr2/auto-orient/strip|imageView2/2/w/448/format/webp)

image.png



```undefined
spring.rabbitmq.listener.simple.prefetch=1
```

![img](https:////upload-images.jianshu.io/upload_images/4636177-147a1dcea6b69648.png?imageMogr2/auto-orient/strip|imageView2/2/w/946/format/webp)

image.png

下面贴消息可靠性解决方案代码了。

配置任务调度中心



```java
@Configuration
@EnableScheduling
public class TaskSchedulerConfig implements SchedulingConfigurer {

    protected ThreadPoolExecutor threadPoolExecutor;

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskExecutor());
    }

    @Bean(destroyMethod = "shutdown")
    public ThreadPoolExecutor taskExecutor() {
        ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
                .setNameFormat("task-executor-pool-%d").build();
        this.threadPoolExecutor = new ScheduledThreadPoolExecutor(10, namedThreadFactory,
                new ThreadPoolExecutor.AbortPolicy());
        return threadPoolExecutor;
    }

}
```

执行重新投递status为0的消息。这里也可以使用corn表达式设置触发任务调度的时间。关于fixedRate和fixedDelay概念总有人搞混。fixedRate任务两次执行时间间隔是任务的开始点，而fixedDelay的间隔是前次任务的结束和下一次任务开始的间隔。



```java
@Component
@Slf4j
public class RetryMessageTask {

    @Autowired
    private RabbitmqOrderSender rabbitmqOrderSender;

    @Autowired
    private BrokerMessageLogMapper brokerMessageLogMapper;

    @Scheduled(initialDelay = 5000, fixedDelay = 30000)
    public void trySendMessage() {
        log.info("定时投递status为0的消息...");
        List<BrokerMessageLog> brokerMessageLogList = brokerMessageLogMapper.listStatusAndTimeoutMessage();
        brokerMessageLogList.forEach(brokerMessageLog -> {
            if (brokerMessageLog.getTryCount() >= 3) {
                log.info("投递3次还是失败...");
                brokerMessageLogMapper.updateMessageLogStatus(brokerMessageLog.getMessageId(),
                        Constants.ORDER_SEND_FAIL,
                        new Date());
            } else {
                log.info("投递失败...");
                brokerMessageLogMapper.updateReSendMessage(brokerMessageLog.getMessageId(),
                        new Date());
                Order order = JSON.parseObject(brokerMessageLog.getMessage(), Order.class);
                try {
                    rabbitmqOrderSender.sendOrder(order);
                } catch (Exception e) {
                    log.error("重新投递消息发送异常...:" + e.getMessage());
                }
            }
        });
    }
}
```

消息生产端



```java
@Component
@Slf4j
public class RabbitmqOrderSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private BrokerMessageLogMapper brokerMessageLogMapper;

    // 消息发送到交换器Exchange后触发回调
    private final RabbitTemplate.ConfirmCallback confirmCallback =
            new RabbitTemplate.ConfirmCallback() {
                @Override
                public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                    log.info("生产端confirm...");
                    log.info("correlationData=" + correlationData);
                    String messageId = correlationData.getId();
                    if (ack) {
                        //confirm返回成功,更新消息投递状态
                        brokerMessageLogMapper.updateMessageLogStatus(messageId, Constants.ORDER_SEND_SUCCESS, new Date());
                    } else {
                        // 失败则进行具体的后续操作，重试或者补偿等手段。
                        log.info("异常处理...");
                    }
                }
            };

    // 如果消息从交换器发送到对应队列失败时触发
    private final RabbitTemplate.ReturnCallback returnCallback =
            new RabbitTemplate.ReturnCallback() {
                @Override
                public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                    log.info("message=" + message.toString());
                    log.info("replyCode=" + replyCode);
                    log.info("replyText=" + replyText);
                    log.info("exchange=" + exchange);
                    log.info("routingKey=" + routingKey);
                }
            };

    public void sendOrder(Order order) {
        log.info("生产端发送消息...");
        rabbitTemplate.setConfirmCallback(this.confirmCallback);
        rabbitTemplate.setReturnCallback(this.returnCallback);
        CorrelationData correlationData = new CorrelationData(order.getMessageId());
        rabbitTemplate.convertAndSend(MQConfig.ORDER_DIRECT_EXCAHNGE,
                MQConfig.ORDER_QUEUE,  order, correlationData);
    }
}
```

消息消费端



```kotlin
@Component
@Slf4j
public class RabbitmqOrderReceiver {

    @RabbitListener(queues = MQConfig.ORDER_QUEUE)
    public void receive(@Payload Order order, Channel channel,
                        @Headers Map<String, Object> headers,
                        Message message) throws IOException, InterruptedException {
        log.info("消费端接收消息...");
        log.info("message=" + message.toString());
        log.info("order=" + order);
        Long deliveryTag = (Long) headers.get(AmqpHeaders.DELIVERY_TAG);
        log.info("deliveryTag=" + deliveryTag);
        // 手工ack
        channel.basicAck(deliveryTag, false);
    }
}
```

当我们发送消息时，故意将Exchange设置成一个不存在的值。消息路由不到合适的Exchange，Confirm机制回送的ACK会返回false，走异常处理。这个消息的状态不会更新成1。然后定时任务会拉取status为0的消息，进行重新投递，投递了3次消息还未成功，将status置为2。



![img](https:////upload-images.jianshu.io/upload_images/4636177-ae5e7d6291d107bf.png?imageMogr2/auto-orient/strip|imageView2/2/w/882/format/webp)

image.png

接下来，我们测试一波。



```java
    @Test
    public void test() {
        Order order = new Order();
        order.setId("36");
        order.setName("cmazxiaoma测试订单-36");
        order.setMessageId(UUIDUtil.uuid());

        rabbitmqOrderService.createOrder(order);
    }
```

消息投递失败。



![img](https:////upload-images.jianshu.io/upload_images/4636177-8c0a4b2a821beef9.png?imageMogr2/auto-orient/strip|imageView2/2/w/945/format/webp)

image.png



定时任务重新投递消息失败。



![img](https:////upload-images.jianshu.io/upload_images/4636177-0589a051b0fa0973.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png

将失败的消息重新投递3次还是失败。



![img](https:////upload-images.jianshu.io/upload_images/4636177-fa06fafddedf40cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1034/format/webp)

image.png

更新Message db信息，将重新投递3次还是失败的消息状态置为2。



![img](https:////upload-images.jianshu.io/upload_images/4636177-0b277fe19a13ebe9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1118/format/webp)

image.png

接着我们把消费端手动ACK的代码注释掉，再让生产端发送消息。看看会出现什么情况。



![img](https:////upload-images.jianshu.io/upload_images/4636177-b0ec1e755a24b8f3.png?imageMogr2/auto-orient/strip|imageView2/2/w/907/format/webp)

image.png

我们会发现Queue堆积了该消息。



![img](https:////upload-images.jianshu.io/upload_images/4636177-66d586ce7e94562d.png?imageMogr2/auto-orient/strip|imageView2/2/w/870/format/webp)

image.png

我们关掉RabbitMQ Server，看看此消息是否会持久化。



```bash
[root@VM_0_11_centos log]# ps -ef|grep rabbitmq
root     13283 10291  0 13:42 pts/1    00:00:00 grep --color=auto rabbitmq
root     23051     1  1 Nov06 ?        00:09:29 /usr/lib64/erlang/erts-5.10.4/bin/beam -W w -A 64 -P 1048576 -t 5000000 -stbt db -zdbbl 128000 -K true -- -root /usr/lib64/erlang -progname erl -- -home /root -- -pa /usr/local/rabbitmq/ebin -noshell -noinput -s rabbit boot -sname rabbit@VM_0_11_centos -boot start_sasl -kernel inet_default_connect_options [{nodelay,true}] -sasl errlog_type error -sasl sasl_error_logger false -rabbit error_logger {file,"/usr/local/rabbitmq/var/log/rabbitmq/rabbit@VM_0_11_centos.log"} -rabbit sasl_error_logger {file,"/usr/local/rabbitmq/var/log/rabbitmq/rabbit@VM_0_11_centos-sasl.log"} -rabbit enabled_plugins_file "/usr/local/rabbitmq/etc/rabbitmq/enabled_plugins" -rabbit plugins_dir "/usr/local/rabbitmq/plugins" -rabbit plugins_expand_dir "/usr/local/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@VM_0_11_centos-plugins-expand" -os_mon start_cpu_sup false -os_mon start_disksup false -os_mon start_memsup false -mnesia dir "/usr/local/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@VM_0_11_centos" -kernel inet_dist_listen_min 25672 -kernel inet_dist_listen_max 25672 -noshell -noinput
[root@VM_0_11_centos log]# kill -9 23051
```



```ruby
[root@VM_0_11_centos sbin]# rabbitmq-server -detached
Warning: PID file not written; -detached was passed.
[root@VM_0_11_centos sbin]# ps -ef|grep rabbitmq
root     13500     1 31 13:44 ?        00:00:02 /usr/lib64/erlang/erts-5.10.4/bin/beam -W w -A 64 -P 1048576 -t 5000000 -stbt db -zdbbl 128000 -K true -- -root /usr/lib64/erlang -progname erl -- -home /root -- -pa /usr/local/rabbitmq/ebin -noshell -noinput -s rabbit boot -sname rabbit@VM_0_11_centos -boot start_sasl -kernel inet_default_connect_options [{nodelay,true}] -sasl errlog_type error -sasl sasl_error_logger false -rabbit error_logger {file,"/usr/local/rabbitmq/var/log/rabbitmq/rabbit@VM_0_11_centos.log"} -rabbit sasl_error_logger {file,"/usr/local/rabbitmq/var/log/rabbitmq/rabbit@VM_0_11_centos-sasl.log"} -rabbit enabled_plugins_file "/usr/local/rabbitmq/etc/rabbitmq/enabled_plugins" -rabbit plugins_dir "/usr/local/rabbitmq/plugins" -rabbit plugins_expand_dir "/usr/local/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@VM_0_11_centos-plugins-expand" -os_mon start_cpu_sup false -os_mon start_disksup false -os_mon start_memsup false -mnesia dir "/usr/local/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@VM_0_11_centos" -kernel inet_dist_listen_min 25672 -kernel inet_dist_listen_max 25672 -noshell -noinput
root     13597 10291  0 13:44 pts/1    00:00:00 grep --color=auto rabbitmq
```

执行`rabbitmqctl list_queues name messages_ready messages_unacknowledged`命令，查询Queue情况，发现Message持久化了。

![img](https:////upload-images.jianshu.io/upload_images/4636177-eff971e1acc61223.png?imageMogr2/auto-orient/strip|imageView2/2/w/899/format/webp)

image.png



![img](https:////upload-images.jianshu.io/upload_images/4636177-5876f46788c1c923.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

image.png



断开消费者程序，我们可以看到消息从Unacked状态转换成Ready了。



![img](https:////upload-images.jianshu.io/upload_images/4636177-224d1b298b21762e.png?imageMogr2/auto-orient/strip|imageView2/2/w/777/format/webp)

image.png



