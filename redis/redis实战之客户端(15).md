# redis客户端

##  1. 客户端通信原理

客户端和服务器端通过TCP 连接来进行数据交互, 服务器默认的端口号是6379, 客户端和服务端发送的命令和数据一律以`\r\n` (CRLF 回车+换行) 结尾. 

如果使用`wireshark` 对`jedis` 抓包

环境: `jedis` 连接到运行在虚拟机202的redis, 执行命令, 对`VMnet8` 进行抓包

过滤条件： `ip.dst==192.168.8.202 and tcp.port in {6379}`

`set test 2673` 转包

可以看到实际发出的数据包是: 

```bash
*3\r\n$3\r\nSET\r\n$4\r\ntest\r\n$4\r\n2673\r\n

```

`get test` 抓包

```bash
*2\r\n$3\r\nGET\r\n$4\r\ntest\r\n
```



客户端跟redis 之间, 使用一种特殊的编码格式(在AOF文件里面我们能够看到). 叫做`Redis Serialization Protocol`(redis 序列化协议). 特点: 容易实现、解析快、可读性强. 客户端发送服务端的消息需要经过编码, 服务端收到之后会按约定进行解码,反之亦然. 

基于此, 我们可以自己实现自己的一个redis客户端

1. 建立socket连接
2. `OutputStream` 写入数据(发送到服务端)
3. `InputStream` 读取数据(从服务端接口)

```java

/**
 * @author luyanan
 * @since 2020/4/2
 * <p>自己编写客户端</p>
 **/
public class MyRedisClient {


    private Socket socket;
    private OutputStream write;
    private InputStream read;


    public MyRedisClient(String host, int port) throws IOException {
        socket = new Socket(host, port);
        write = socket.getOutputStream();
        read = socket.getInputStream();
    }

    private static final String LINE_SUFFIX = "\r\n";

    public void set(String key, String val) throws IOException {
        StringBuffer sb = new StringBuffer();

        // 代表3个参数
        sb.append("*3").append(LINE_SUFFIX);
        //  第一个参数(get) 的长度
        sb.append("$3").append(LINE_SUFFIX);
        // 第一个参数的内容
        sb.append("SET").append(LINE_SUFFIX);
        // 第二个参数key的长度
        sb.append("$").append(key.getBytes().length).append(LINE_SUFFIX);
        // 第二个参数key 的内容
        sb.append(key).append(LINE_SUFFIX);
        // 第三个参数value的长度
        sb.append("$").append(val.getBytes().length).append(LINE_SUFFIX);
        // 第三个参数value 的内容
        sb.append(val).append(LINE_SUFFIX);
        write.write(sb.toString().getBytes());
        byte[] bytes = new byte[1024];
        read.read(bytes);
        System.out.println("----set------的返回结果为:" + new String(bytes));
    }


    public void get(String key) throws IOException {
        StringBuffer sb = new StringBuffer();
        // 代表2个参数
        sb.append("*2").append(LINE_SUFFIX);
        // 第一个参数(get)的长度
        sb.append("$3").append(LINE_SUFFIX);
        // 第一个参数的内容
        sb.append("GET").append(LINE_SUFFIX);
        //第二个参数的长度
        sb.append("$").append(key.getBytes().length).append(LINE_SUFFIX);
        // 第二个参数的值
        sb.append(key).append(LINE_SUFFIX);

        write.write(sb.toString().getBytes());
        byte[] bytes = new byte[1024];
        read.read(bytes);
        System.out.println("-------get------的返回结果为:" + new String(bytes));
    }

    public static void main(String[] args) throws IOException {
        MyRedisClient redisClient = new MyRedisClient("192.168.8.220", 6379);
        redisClient.set("test", "111111");
        redisClient.get("test");
    }
}
```



基于这种协议, 我们可以用java实现所有的redis操作命令。当然, 我们不需要这么做, 因为已经有很多成熟的java客户端, 实现了完整的功能和高级特性, 并且提供了很好的性能. 

https://redis.io/clients

官网推荐的java客户端主要有三个`jedis`、`redisson`和`luttuce`.

| 客户端     | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| `jedis`    | A blazingly small and sane redis java client                 |
| `redisson` | distributed and scalable Java data structures on top of Redis server |
| `luttuce`  | Advanced Redis client for thread-safe sync, async, and reactive usage. Supports Cluster, Sentinel, Pipelining, and codecs. |



> Spring连接`redis`使用的是什么？ `RedisConnectionFactory` 接口支持多种实现, 例如: `JedisConnectionFactory`、`LettuceConnectionFactory` 等. 



## 2 [Jedis](https://github.com/xetorthio/jedis)

####  1.2.1 特点

jedis 是我们最熟悉和最常用的客户端. 轻量、简单、便于集成和改造. 

```java
    public static void main(String[] args) {
        Jedis jedis = new Jedis("localhost", 6379);
        jedis.set("test", "111");
        System.out.println(jedis.get("test"));
        jedis.close();
        
    }

```



`jedis` 多个线程使用一个连接的时候线程不安全. 可以使用连接池, 为每个请求创建不同的连接, 基于`Apache common pool` 实现. 跟数据库是一样的, 可以设置最大连接数等参数. `jedis` 中有多种连接池的子类. 

![image-20200402133357314](http://files.luyanan.com//img/20200402133358.png)

`jedis` 有3种工作模式: 单节点、分片、哨兵、集群

3种请求模式:`client`、`pipeline`和事务. `client`模式就是客户端发送一个命令, 阻塞等待服务端执行,然后服务返回结果。 `pipeline` 模式就是一次性发送多条命令, 最后一次性取回所有的返回结果,这种模式通过减少网络的往返时间和io读写, 大幅度提高通信性能. 第三种事务模式, `Transaction` 模式即开启redis 的事务管理, 事务模式开启后,所有的命令(除了`exec`、`discard`、`multi`和`watch`) 到达服务端以后不会立即执行, 会进入一个等待队列. 



#### 1.2.2 sentinel 获取连接原理

问题： `jedis` 连接`sentinel` 的时候, 我们配置的是全部哨兵地址, `sentinel` 是如何返回可用的`master`地址的呢? 

在构造方法:



```java
public JedisSentinelPool(String masterName, Set<String> sentinels) {
        this(masterName, sentinels, new GenericObjectPoolConfig(), 2000, (String)null, 0);
    }
```

我们继续往下看, 直到 redis.clients.jedis.JedisSentinelPool#JedisSentinelPool(java.lang.String, java.util.Set<java.lang.String>, org.apache.commons.pool2.impl.GenericObjectPoolConfig, int, int, java.lang.String, int, java.lang.String)

```java
  public JedisSentinelPool(String masterName, Set<String> sentinels, GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password, int database, String clientName) {
        this.connectionTimeout = 2000;
        this.soTimeout = 2000;
        this.database = 0;
        this.masterListeners = new HashSet();
        this.log = Logger.getLogger(this.getClass().getName());
        this.poolConfig = poolConfig;
        this.connectionTimeout = connectionTimeout;
        this.soTimeout = soTimeout;
        this.password = password;
        this.database = database;
        this.clientName = clientName;
        HostAndPort master = this.initSentinels(sentinels, masterName);
        this.initPool(master);
    }
```



我们看到调用了`HostAndPort master = this.initSentinels(sentinels, masterName);`

```java
 private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {

    HostAndPort master = null;
    boolean sentinelAvailable = false;

    log.info("Trying to find master from available Sentinels...");
   //有多个sentinel 地址转换为一个HostAndPort 对象
    for (String sentinel : sentinels) {
        // host:port 表示的sentinel 地址转换为一个HostAndPort 对象
      final HostAndPort hap = HostAndPort.parseString(sentinel);

      log.fine("Connecting to Sentinel " + hap);

      Jedis jedis = null;
      try {
          // 连接到sentinel
        jedis = new Jedis(hap.getHost(), hap.getPort());
// 根据masterName 得到一个master地址, 返回一个list, host=list[0], port=list[1]
        List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);

        // connected to sentinel...
        sentinelAvailable = true;
       
        if (masterAddr == null || masterAddr.size() != 2) {
          log.warning("Can not get master addr, master name: " + masterName + ". Sentinel: " + hap
              + ".");
          continue;
        }
         // 如果在任何一个sentinel 中找到了master, 不再遍历 sentinel
        master = toHostAndPort(masterAddr);
        log.fine("Found Redis master at " + master);
        break;
      } catch (JedisException e) {
        // resolves #1036, it should handle JedisException there's another chance
        // of raising JedisDataException
        log.warning("Cannot get master address from sentinel running @ " + hap + ". Reason: " + e
            + ". Trying next one.");
      } finally {
        if (jedis != null) {
          jedis.close();
        }
      }
    }
  // 到这里, 如果master为null, 则说明有两种情况, 一种是所有的sentinel节点都down掉了, 另外一种是master没有被存活的sentinel 监控到
    if (master == null) {
      if (sentinelAvailable) {
        // can connect to sentinel, but master name seems to not
        // monitored
        throw new JedisException("Can connect to sentinel, but " + masterName
            + " seems to be not monitored...");
      } else {
        throw new JedisConnectionException("All sentinels down, cannot determine where is "
            + masterName + " master is running...");
      }
    }

     // 如果走到这里, 说明找到了master的地址
    log.info("Redis master running at " + master + ", starting Sentinel listeners...");

     // 启动对每个sentinel 的监控为每个sentinel 都启动一个监听者 `masterListener`. MasterListener 本身是一个线程, 它会去订阅sentinel 上关于master节点地址改变的消息. 
    for (String sentinel : sentinels) {
      final HostAndPort hap = HostAndPort.parseString(sentinel);
      MasterListener masterListener = new MasterListener(masterName, hap.getHost(), hap.getPort());
      // whether MasterListener threads are alive or not, process can be stopped
      masterListener.setDaemon(true);
      masterListeners.add(masterListener);
      masterListener.start();
    }

    return master;
  }
```



#### 1.2.3 Cluster 获取连接原理

```java
  public static void main(String[] args) throws IOException {
        HostAndPort h1 = new HostAndPort("192.168.8.207", 7291);
        HostAndPort h2 = new HostAndPort("192.168.8.207", 7292);
        HostAndPort h3 = new HostAndPort("192.168.8.207", 7293);
        HostAndPort h4 = new HostAndPort("192.168.8.207", 7294);
        HostAndPort h5 = new HostAndPort("192.168.8.207", 7295);
        HostAndPort h6 = new HostAndPort("192.168.8.207", 7296);
        Set nodes = new HashSet();
        nodes.add(h1);
        nodes.add(h2);
        nodes.add(h3);
        nodes.add(h4);
        JedisCluster cluster = new JedisCluster(nodes);
        cluster.set("cluster:test", "1111");
        System.out.println(cluster.get("cluster:test"));
        cluster.close();
    }

```



问题: 使用`jedis` 连接`cluster`的时候, 我们只需要连接到任意一个或者多个`redis group` 中的实例地址, 那我们是怎么获取到需要操作的`redis master` 实例的? 

关键问题: 在于如何存储`slot`和redis 连接池的关系

1. 程序启动初始化集群环境, 读取配置文件中的节点配置, 无论是主从, 无论多少个, 只拿第一个, 获取redis 连接实例(后面有个break)

    从`JedisCluster cluster = new JedisCluster(nodes);` 进入

   到 `redis.clients.jedis.BinaryJedisCluster#BinaryJedisCluster(java.util.Set<redis.clients.jedis.HostAndPort>, int, int, org.apache.commons.pool2.impl.GenericObjectPoolConfig)`

   ```java
     public BinaryJedisCluster(Set<HostAndPort> jedisClusterNode, int timeout, int maxAttempts,
         final GenericObjectPoolConfig poolConfig) {
       this.connectionHandler = new JedisSlotBasedConnectionHandler(jedisClusterNode, poolConfig,
           timeout);
       this.maxAttempts = maxAttempts;
     }
   ```

    最后进入到`redis.clients.jedis.JedisClusterConnectionHandler#JedisClusterConnectionHandler`

   ```java
     public JedisClusterConnectionHandler(Set<HostAndPort> nodes,
                                          final GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password) {
       this.cache = new JedisClusterInfoCache(poolConfig, connectionTimeout, soTimeout, password);
       initializeSlotsCache(nodes, poolConfig, password);
     }
   
   ```

    ```java
   
     private void initializeSlotsCache(Set<HostAndPort> startNodes, GenericObjectPoolConfig poolConfig, String password) {
       for (HostAndPort hostAndPort : startNodes) {
           //  获取一个jedis实例
         Jedis jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort());
         if (password != null) {
           jedis.auth(password);
         }
         try {
             // 获取redis节点和slot 虚拟槽
           cache.discoverClusterNodesAndSlots(jedis);
             // 直接跳出循环
           break;
         } catch (JedisConnectionException e) {
           // try next nodes
         } finally {
           if (jedis != null) {
             jedis.close();
           }
         }
       }
     }
    ```

2.  用获取的redis 连接实例执行`clusterSlots()` 方法, 实际执行redis 服务端`cluster solts` 命令, 获取虚拟槽信息. 

    该集合的基本信息为`[long,long,list,list]`, 第一,二个元素为该节点负责槽点的起始位置, 第三个元素为是主节点信息, 第四个元素为主节点对应的从节点的信息. 该list 的基本信息为`[string,int,string]`. 第一个是host信息, 第二个是port信息, 第三个为唯一id. 

    ![image-20200402142431808](http://files.luyanan.com//img/20200402142434.png)

3. 获取有关节点的槽点信息后, 调用`getAssignedSlotArray(slotinfo)` 来获取所有的槽点值。

4. 再获取主节点的地址信息, 调用`generateHostAndPort(hostInfo)`, 生成一个`hostAndPort` 对象. 

5. 再根据节点地址信息来设置节点对应的`jedisPool`,即设置`Map nodes`的值

    接下来判断若此时节点信息为主节点信息, 则调用`assignSlotsToNodes` 方法, 设置每个槽点值对应的连接池信息, 即设置`Map slots `的值. 

   `redis.clients.jedis.JedisClusterInfoCache#discoverClusterNodesAndSlots`

   ```java
   
     public void discoverClusterNodesAndSlots(Jedis jedis) {
       w.lock();
   
       try {
         reset();
           // 获取节点信息
         List<Object> slots = jedis.clusterSlots();
        // 遍历3个master节点
         for (Object slotInfoObj : slots) {
             // slotInfo 槽开始，槽结束，主，从
             // {[0,5460,7291,7294],[5461,10922,7292,7295],[10923,16383,7293,7296]}
           List<Object> slotInfo = (List<Object>) slotInfoObj;
           // 如果<=2,表示没有分配slot
           if (slotInfo.size() <= MASTER_NODE_INDEX) {
             continue;
           }
          // 获取分配到当前master节点的槽, 例如7291 节点的{0,1,2,3……5460}
           List<Integer> slotNums = getAssignedSlotArray(slotInfo);
   
           // hostInfos
           int size = slotInfo.size();  // size是4，槽最小最大, 主, 从
             // 第3位和第4位是主从端口的信息
           for (int i = MASTER_NODE_INDEX; i < size; i++) {
             List<Object> hostInfos = (List<Object>) slotInfo.get(i);
             if (hostInfos.size() <= 0) {
               continue;
             }
            // 根据ip端口生成HostAndPort 对象
             HostAndPort targetNode = generateHostAndPort(hostInfos);
             // 根据HostAndPort解析出 ip:port 的 key 值，再根据 key 从缓存中查询对应的 jedisPool 实例。如果没有 jedisPool实例，就创建 JedisPool 实例，最后放入缓存中。nodeKey 和 nodePool 的关系
               setupNodeIfNotExist(targetNode);
               // 把 slot 和 jedisPool 缓存起来（16384 个），key 是 slot 下标，value 是连接池
             if (i == MASTER_NODE_INDEX) {
               assignSlotsToNode(slotNums, targetNode);
             }
           }
         }
       } finally {
         w.unlock();
       }
     }
   
   ```

   



从集群环境中存取值: 

1. 把key作为参数, 执行`CRC16` 算法, 获取key对应的`slot`值. 
2. 通过该`slot` 值, 去`slots`的map 集合中获取`jedisPool`实例. 
3. 通过`jedisPool` 实例获取`jedis` 实例, 最终完成redis数据存取工作. 



#### 1.2.4 pipeline  

 当我们使用set方法设置几万条数据的时候, 会发现非常慢, 完全没有把redis  的10万的QPS 利用起来, 但是单个命令的执行到底慢在哪里? 

#####  慢在哪里? 

redis 使用的是客户端/服务端(C/S) 模式和请求/响应协议的TCP服务器 , 这意味着通常情况下下一个请求会遵循以下步骤:

- 客户端向服务端发送一个查询请求, 并监听socket 返回,通常是以阻塞模式, 等待服务端响应. 
- 服务端处理命令, 并将结果返回给客户端. 

Redis客户端与redis服务端之间使用TCP 协议进行连接, 一个客户端可以通过一个socket 连接发起多个请求命令. 每个请求命令发出后client 通常会阻塞并等待redis服务器处理, Redis处理完成后请求命令会将结果通过响应报文返回给 Client, 因此当执行多条命令的时候都需要等待上一个命令执行完毕才执行 。 

Redis 本身提供了一些批量操作的命令, 比如`mget`、`mset`,可以减少通信的时间, 但是大部分都是不支持`multi`操作的, 例如`hash`就没有. 

 由于通信会有网络延迟, 例如`client` 和`server` 之间的包传输时间需要10毫秒, 一次交互就需要20毫秒(`RTT：Round Trip Time`), 这样的话, `client` 1秒钟也只能发送50条命令, 这显然没有充分利用redis 的处理能力. 另外一个, redis 服务端执行I/O的次数过多. 



#####  [pipeline 管道](https://redis.io/topics/pipelining)

那我们能不能像数据库的`batch`操作一样, 把一组命令组装在一起发送给redis 服务端执行,然后一次性获取返回结果呢? 这个就是`pipeline`的作用. `pipeline` 通过一个队列将所有的命令缓存起来, 然后把多条命令在一次连接中发送给服务器. 

![image-20200402145657214](http://files.luyanan.com//img/20200402145658.png)

我们先来看一下效果： 

```java
/**
 * @author luyanan
 * @since 2020/4/2
 * <p>使用pipeline 测试set</p>
 **/
public class PipelineTest {

    private static final String host = "localhost";
    private static final int port = 6379;

    private static final int count = 100000;

    public static void main(String[] args) {
        // 普通插入
        new Thread(() -> {
            long t1 = System.currentTimeMillis();
            Jedis jedis = new Jedis(host, port);
            for (int i = 0; i < count; i++) {
                jedis.set("aa" + i, "" + i);
            }
            long t2 = System.currentTimeMillis();
            System.out.println("普通插入" + count + "条数据,耗时:" + (t2 - t1) + "毫秒");

            for (int i = 0; i < count; i++) {
                jedis.get("aa" + i);
            }
            long t3 = System.currentTimeMillis();
            System.out.println("普通获取" + count + "条数据,耗时:" + (t3 - t2)+ "毫秒");
        }).start();


        // pipeline插入
        new Thread(() -> {
            long t1 = System.currentTimeMillis();
            Jedis jedis = new Jedis(host, port);
            Pipeline pipelined = jedis.pipelined();
            for (int i = 0; i < count; i++) {
                pipelined.set("pp" + i, "" + i);
            }
            pipelined.syncAndReturnAll();
            long t2 = System.currentTimeMillis();
            System.out.println("pipeline插入" + count + "条数据,耗时:" + (t2 - t1)+ "毫秒");

            for (int i = 0; i < count; i++) {
                pipelined.get("pp" + i);
            }
            pipelined.syncAndReturnAll();
            long t3 = System.currentTimeMillis();
            System.out.println("pipeline获取" + count + "条数据,耗时:" + (t3 - t2)+ "毫秒");
        }).start();
    }

}

```

查看结果

```text
pipeline插入100000条数据,耗时:1275毫秒
pipeline获取100000条数据,耗时:472毫秒
普通插入100000条数据,耗时:166122毫秒
普通获取100000条数据,耗时:170747毫秒
```



要实现`pipeline` , 既要服务端的支持, 也要客户端的支持. 对于服务端来说，需要能够处理客户端通过一个TCP连接发送起来的多个命令, 并且逐个执行命令一起返回. 

对于客户端来说, 要把多个命令缓存起来, 达到一定的条件就发送出去, 最后才处理redis的应答(这里也要注意对客户端内存的消耗)

`jedis-pipeline`的`client-buffer`限制是: 8192bytes, 客户端堆积的命令超过`8192bytes` 时, 会发送给服务端. 

源码: `redis.clients.util.RedisOutputStream#RedisOutputStream(java.io.OutputStream)`

```java
  public RedisOutputStream(final OutputStream out) {
    this(out, 8192);
  }

```

`pipeline` 对于命令条数没有限制, 但是命令可能受限于TCP包的大小. 

如果`jedis ` 发送了一组命令, 而发送请求还没有结束, redis响应的结果会放在接收缓存区内, 如果接收缓冲区满了, `jedis` 会通知`redis win=0`, 此时redis 不会在发送结果给`jedis`端, 转而把响应结果保存在redis 服务端的输出缓冲区中,. 

输出缓冲区的配置`redis.conf`

```properties
client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
```



```properties
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

| 配置                      | 作用                                                         |
| ------------------------- | ------------------------------------------------------------ |
| class                     | 客户端类型，分为三种。a）normal：普通客户端；b）slave：slave 客户端，用于复制；c） pubsub：发布订阅客户端 |
| `hard limit`              | 如果客户端使用的输出缓冲区大于，客户端会被立即关闭，0 代表不限制 |
| `soft limit soft seconds` | 如果客户端使用的输出缓冲区超过了并且持续了秒，客户端会被立即 关闭 |



每个客户端使用的输出缓存区的大小可以用`client list`命令查看

```bash
redis> client list
```

> id=5 addr=192.168.8.1:10859 fd=8 name= age=5 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=5 qbuf-free=32763 obl=16380 oll=227 omem=4654408 events=rw cmd=set



- `obl`: 输出缓冲区的长度(字节为单位, 0表示没有分配输出缓冲区)
- `oll`: 输出列表包含的对象数量(当输出缓冲区没有剩余空间时,命令回复会以字符串对象的形式被入队到这个队列里)
- `omen`: 输出缓冲区和输出列表占用的内存总量. 

##### 使用场景

`pipeline` 适用于什么场景呢？ 

如果某些操作需要马上得到redis操作是否成功的结果, 这种场景就不适合. 

有些场景, 例如批量写入数据,对于结果的实时性和成功性要求不高的情况, 就可以用`pipeline`. 



####  1.2.5  `jedis` 实现分布式锁

[原文地址](https://redis.io/topics/distlock),  [中文地址](http://redis.cn/topics/distlock.html)

分布式锁的基本特性或者要求: 

1. 互斥性：只有一个客户端能够持有锁. 
2. 不会产生死锁, 即使持有锁的客户端崩溃, 也能保证后续其他客户端可以获取锁. 
3. 只有持有这把锁的客户端才能解锁



```java
 /**
     * 尝试获取分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, int expireTime) {
        // set支持多个参数 NX（not exist） XX（exist） EX（seconds） PX（million seconds）
        String result = jedis.set(lockKey, requestId, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }
```

 参数解读: 

1. `lockKey`: 是redis key 的名称, 也就是谁添加成功了这个key就代表谁获取锁成功. 
2. `reqyesuId`: 是客户端的id,设置成`value`, 如果我们要保证只有加锁的客户端才能释放锁, 就必须获取客户端的ID(保证第三点)
3. `SET_IF_NOT_EXIST` 是我们的命令里面加上`NX`（保证第一点）
4. `SET_WITH_EXPIRE_TIME`: `PX` 代表以毫秒为单位设置key 的过期时间(保证第2点), `expireTime` 是自动释放锁的时间, 比如5000代表5秒

释放锁, 直接删除key 来释放锁, 可以吗? 就像这样

```java
public static void wrongReleaseLock1(Jedis jedis, String lockKey) {
   jedis.del(lockKey);
}
```

没有对`requestId` 进行判断, 可能会释放其他客户端持有的锁. 

先判断后删除呢? 

```java
public static void wrongReleaseLock2(Jedis jedis, String lockKey, String requestId) {
    // 判断加锁与解锁是不是同一个客户端
    if (requestId.equals(jedis.get(lockKey))) {
   // 若在此时，这把锁突然不是这个客户端的，则会误解锁
    jedis.del(lockKey);
  }
}
```

如果在释放锁的时候, 这把锁已经不属于这个客户端了(例如已经过期, 并且已经被别的客户端获取锁成功了)， 那就回出现释放了其他客户端的锁的情况. 

所以我们把判断客户端是否相等和删除key的操作放在`lua`脚本里面执行, 

```java
 /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }

```



这个是jedis 里面分布式锁的实现. 



## 3 [`Luttuce`](https://lettuce.io/)

#### 1.3.1 特点

与`jedis`相比, `Luttuce`则完全克服了其线程不安全的缺点：`Luttuce` 是一个可伸缩的线程安全的redis客户端, 支持同步、异步和响应式模式(`Reactive`).多线程可以共享一个连接实例,而不必担心多线程并发的问题. 

同步调用：

```java
 public static void main(String[] args) {

        RedisClient redisClient = RedisClient.create("redis://127.0.0.1:6379");
        // 线程安全的长连接, 连接丢失的时候会自动创建
        StatefulRedisConnection<String, String> connect = redisClient.connect();
        // 获取同步执行命令， 默认超时时间, 默认超时时间为60s
        RedisCommands<String, String> commands =
                connect.sync();
        // 发送get请求,
        commands.set("sync_test", "1111");
        String value = commands.get("sync_test");
        //关闭连接
        connect.close();
        //关闭客户端
        redisClient.shutdown();

    }
```



异步调用

```java
public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {

        RedisClient redisClient = RedisClient.create("redis://127.0.0.1:6379");
        // 线程安全的长连接, 连接丢失的时候会自动创建
        StatefulRedisConnection<String, String> connect = redisClient.connect();

        // 获取异步执行命名
        RedisAsyncCommands<String, String> commands = connect.async();
        // 发送get请求,
        commands.set("async_test", "1111");
        RedisFuture<String> future = commands.get("sync_test");
        String value = future.get(60, TimeUnit.SECONDS);
        //关闭连接
        connect.close();
        //关闭客户端
        redisClient.shutdown();

    }
```



它是基于`Netty`框架构建的, 支持redis 的高级功能, 比如`pipeline`、发布订阅、事务、`sentinel`、集群, 支持连接池. 

`Luttuce` 是`Spring Boot` 2.X 默认的客户端, 替换了`jedis`. 集成之后不需要单独使用它, 直接调用`Spring`的`RedisTemplate` 操作, 连接和创建和关闭也是不需要我们自己操心. 



## 4   [`Redisson`](https://redisson.org/)

文档地址 : https://github.com/redisson/redisson/wiki/%E7%9B%AE%E5%BD%95    

####  1.4.1 本质

`Redisson` 是一个在redis的基础上实现的java驻内存数据网格(`In-Memory Data Grid`), 提供了分布式和可扩展的java数据结构. 



####  1.4.2 特点

基于Netty实现,采用非阻塞io, 性能高. 

支持异步请求. 

支持连接池、`pipeline`、`LUA Scripting`、`Redis Sentinel`、`Redis Cluster`. 

不支持事务, 官方建议使用`LUA Scripting` 代替事务.

主从、哨兵、集群都支持, Spring 也可以配置和注入`RedissonClient`



####  1.4.3 实现分布锁

在`Redisson` 里面提供了简单的分布式锁的实现. 

![image-20200402204005932](http://files.luyanan.com//img/20200402204026.png)

```java
public class LockTest {

    private static RedissonClient redissonClient;

    static {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379");
        redissonClient = Redisson.create(config);
    }

    public static void main(String[] args) throws InterruptedException {


        RLock lock = redissonClient.getLock("updateAccount");

        if (lock.tryLock(100, 10, TimeUnit.SECONDS)) {
            System.out.println("获取锁");
        }
        lock.unlock();
        redissonClient.shutdown();
    }

}
```

在获得`RLock`  之后, 有个`tryLock` 方法, 里面有3个参数: 

1. `waitTime`: 获取锁的最大等待时间, 超过这个时间就不再尝试获取锁了. 
2. `leaseTime`: 如果没有调用`unlock` 方法 ,超过了这个时间就自动释放锁. 
3. `timeunit`: 释放时间的单位. 



`Redisson` 的分布式锁是怎么实现的呢? 

在加锁的时候,在redis 写入了一个Hash , key是锁的名字， `field` 是线程名称, 	`value` 是1(表示锁的冲入次数)

![image-20200402210251089](http://files.luyanan.com//img/20200402210252.png)

源码： 

`tryLock()——tryAcquire()——tryAcquireAsync()——tryLockInnerAsync()`

```java

    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "return redis.call('pttl', KEYS[1]);",
                    Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
```

最终调用的是一段`lua` 脚本,里面有一个参数,两个参数的值. 

| 占位      | 填充                    | 含义             | 实际值                                   |
| --------- | ----------------------- | ---------------- | ---------------------------------------- |
| `KEYS[1]` | `getName()`             | 锁的名称(key)    | `updateAccount`                          |
| `ARGV[1]` | `internalLockLeaseTime` | 锁释放时间(毫秒) | 10000                                    |
| `ARGV[2]` | `getLockName(threadId)` | 线程名称         | `b60a9c8c-92f8-4bfe-b0e7-308967346336:1` |

```lua
// KEYS[1] 锁名称 updateAccount
// ARGV[1] key 过期时间 10000ms
// ARGV[2] 线程名称
// 锁名称不存在
if (redis.call('exists', KEYS[1]) == 0) then
  // 创建一个 hash，key=锁名称，field=线程名，value=1
  redis.call('hset', KEYS[1], ARGV[2], 1);
  // 设置 hash 的过期时间
   redis.call('pexpire', KEYS[1], ARGV[1]);
return nil;
end;
// 锁名称存在，判断是否当前线程持有的锁
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
  // 如果是，value+1，代表重入次数+1
  redis.call('hincrby', KEYS[1], ARGV[2], 1);
  // 重新获得锁，需要重新设置 Key 的过期时间
  redis.call('pexpire', KEYS[1], ARGV[1]);
return nil;
end;
 // 锁存在，但是不是当前线程持有，返回过期时间（毫秒）
 return redis.call('pttl', KEYS[1]);
```



释放锁, 源码:

`unlock——unlockInnerAsync`

```java
 protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "if (redis.call('exists', KEYS[1]) == 0) then " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; " +
                "end;" +
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; "+
                "end; " +
                "return nil;",
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));

    }
```

| 占位      | 填充                       | 含义             | 实际值                                   |
| --------- | -------------------------- | ---------------- | ---------------------------------------- |
| `KEYS[1]` | `getName()`                | 锁名称           | `updateAccount`                          |
| `KEYS[2]` | `getChannelName()`         | 频道名称         | `redisson_lock__channel:{updateAccount}` |
| `ARGV[1]` | `LockPubSub.unlockMessage` | 解锁的时候的消息 | 0                                        |
| `ARGV[2]` | `internalLockLeaseTime`    | 释放锁的时间     | 10000                                    |
| `ARGV[3]` | `getLockName(threadId)`    | 线程名称         | `b60a9c8c-92f8-4bfe-b0e7-308967346336:1` |

​	



```java
// KEYS[1] 锁的名称 updateAccount
// KEYS[2] 频道名称 redisson_lock__channel:{updateAccount}
// ARGV[1] 释放锁的消息 0
// ARGV[2] 锁释放时间 10000
// ARGV[3] 线程名称
// 锁不存在（过期或者已经释放了）
if (redis.call('exists', KEYS[1]) == 0) then
  // 发布锁已经释放的消息
  redis.call('publish', KEYS[2], ARGV[1]);
  return 1;
end;
// 锁存在，但是不是当前线程加的锁
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then
  return nil;
end;
// 锁存在，是当前线程加的锁
// 重入次数-1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
// -1 后大于 0，说明这个线程持有这把锁还有其他的任务需要执行
if (counter > 0) then
  // 重新设置锁的过期时间
  redis.call('pexpire', KEYS[1], ARGV[2]);
  return 0;
else
  // -1 之后等于 0，现在可以删除锁了
  redis.call('del', KEYS[1]);
  // 删除之后发布释放锁的消息
  redis.call('publish', KEYS[2], ARGV[1]);
  return 1;
end;
// 其他情况返回 nil
return nil;
```



这个是`Redisson` 里面分布式锁的实现, 我们在调用的时候非常简单. 

`Redisson`跟`Jedis` 定位不同, 它不是一个单纯的`Redis` 客户端, 而是基于redis 实现的分布式服务, 如果有需要用到一些分布式的数据结构, 我们我们可以基于`Redisson`的分布式队列实现分布式事务,就可以引入`Redisson` 的依赖实现. 



