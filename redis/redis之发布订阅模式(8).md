#  redis之发布订阅

##  1 列表的局限

我们知道, redis的队列可以通过`rpush`和`lpop` 来实现消息队列(队尾进,队头出),但是消费者需要不断的调用`lpop`查看list中是否有等待处理的任务(比如写一个`while`循环).为了减少通信的消耗, 可以`sleep`一段时间后再进行消费, 但是这样会有两个问题: 

1. 如果生产者生产消息的速度远大于消费者消费消息的速度, List会占用大量的内存. 

2. 消息的实时性降低. 

    `list` 还提供了一个阻塞的命令, `blpop`, 没有任何元素可以弹出的时候 , 连接会被阻塞. 

   ```bash
   blpop queue 5
   ```

基于`list` 实现的消息队列, 不支持一对多的消息分发. 



##  2发布订阅模式

除了通过`list`实现消息队列之外, `redis` 还提供了一组命令来实现发布/订阅模式

这种方式, 发送者和接收者没有任何关联(实现了解耦), 接受着也不需要尝试持续接收消息.



### 2.1 订阅频道

首先, 我们有很多的频道(`channel`), 我们也可以把这个频道理解成`queue`. 订阅者可以订阅一个或者多个频道. 消费的发布者(生产者) 可以给指定的频道发送消息. 	只要有消息达到了频道, 所有订阅了这个频道的订阅者都会收到这条消息.

需要注意的是, 发出去的消息不会被持久化, 因为他们已经被从队列中删除了,所以消费者只能收到他开始订阅了这个频道之后发布的消息了. 

下面我们来看看发布订阅命令的使用. 

**订阅者订阅消息, 可以一次订阅多个**

```bash
subscribe channel-1 channel-2 channel-3
```

发布者可以向指定的频道发布消息(并不支持一次向多个频道发布消息)

```bash
publish channel-1 2673
```

取消订阅(不能在订阅状态下使用)

```bash
unsubscribe channel-1
```



### 2.2 按规则(`pattern` ) 订阅频道

支持`?` 和`*` 占位符. `?` 代表一个字符,`*` 代表0 或者多个字符

消费端 1，关注运动信息:

```bash
psubscribe *sport
```

消费端 2，关注所有新闻：

```bash
psubscribe news*
```

消费端 3，关注天气新闻：

```bash
psubscribe news-weather

```

生产者，发布 3 条信息

```bash
publish news-sport yaoming
publish news-music jaychou
publish news-weather rain
```

![image-20200329114733925](http://files.luyanan.com//img/20200329114858.png)



**java 基于jedis实现**

**发布者**

```java
public class PublishTest {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        jedis.publish("test-123", "666");
        jedis.publish("test-abc", "pengyuyan");
    }
}

```



订阅者

MyListener

```java
public class MyListener extends JedisPubSub {
    // 取得订阅的消息后的处理
    public void onMessage(String channel, String message) {
        System.out.println(channel + "=" + message);
    }

    // 初始化订阅时候的处理
    public void onSubscribe(String channel, int subscribedChannels) {
        // System.out.println(channel + "=" + subscribedChannels);
    }

    // 取消订阅时候的处理
    public void onUnsubscribe(String channel, int subscribedChannels) {
        // System.out.println(channel + "=" + subscribedChannels);
    }

    // 初始化按表达式的方式订阅时候的处理
    public void onPSubscribe(String pattern, int subscribedChannels) {
        // System.out.println(pattern + "=" + subscribedChannels);
    }

    // 取消按表达式的方式订阅时候的处理
    public void onPUnsubscribe(String pattern, int subscribedChannels) {
        // System.out.println(pattern + "=" + subscribedChannels);
    }

    // 取得按表达式的方式订阅的消息后的处理
    public void onPMessage(String pattern, String channel, String message) {
        System.out.println(pattern + "=" + channel + "=" + message);
    }
}

```



ListenTest

```java
public class ListenTest {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        final MyListener listener = new MyListener();
        // 使用模式匹配的方式设置频道
        // 会阻塞
        jedis.psubscribe(listener, new String[]{"test-*"});
    }
}
```