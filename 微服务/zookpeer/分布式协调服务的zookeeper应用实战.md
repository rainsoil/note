date: 2019年11月8日10:49:12

分类: 

- zookeeper

tag:

-  zookeeper

---

# 分布式协调服务的zookeeper应用实战

## 集群角色

![](http://files.luyanan.com//img/20191106154252.png)

## 数据模型

zookeeper的视图结构和标准的文件系统非常类似, 每一个节点称之为ZNode, 是zookeeper的最小单元. 每个znode 上都可以保存数据以及挂载子节点. 构成一个层次化树形结构. 

### 持久节点( PERSISTENT )

创建后会一直存在zookeeper服务器上, 直到主动删除

### 持久有序节点( PERSISTENT_SEQUENTIAL )

每个节点都会为它的一级子节点维护一个顺序

### 临时节点( EPHEMERAL )

临时节点的声明周期和客户端的会话绑定在一起, 当客户端会话失效则该节点自动清理

### 临时有序节点( EPHEMERAL )

在临时节点的基础上多了一个顺序性. 

 CONTAINER  当子节点都被删除后,  CONTAINER  也随即删除,  PERSISTENT_WITH_TTL  超过TTL 未被删除,且没有子节点  PERSISTENT_SEQUENTIAL_WITH_TTL  客户端断开连接后不会主动删除Znode, 如果该Znode 没有子Znode 且在给定的TTL 时间无修改, 该Znode 将会被删除;TTL 单位是毫秒, 必须大于0 且小于或等于  EphemeralType.MAX_TTL 

### 会话

![](http://files.luyanan.com//img/20191107165150.png)

1. Client 初始化连接, 状态转为  CONNECTING ( ① )
2. Client 与Server 成功建立连接, 状态改为 CONNECTED(②) 
3. Client 丢失了与Server 的连接或者没有接收到Server 的响应, 状态改为  CONNECTING(③) 
4. Client 连上另外的Server 或连接上了之前的Server, 状态改为 CONNECTED(②) 
5. 若会话过期(是Server 负责声明会话过期, 而不是Client),状态改为 CLOSED(⑤)， 
6. Client 也可以主动关闭会话 (④)， 状态转为 CLOSED 

### stat 状态信息

每个节点除了存储数据内容以外,还存储了数据节点本身的一些状态信息, 通过get命令可以获取状态信息的详细内容. 

![](http://files.luyanan.com//img/20191107211935.png)

### 版本-保证分布式数据原子性

zookeeper为数据节点引入了版本的概念, 每个数据节点都有三类版本信息, 对数据节点任何更新操作都会引起版本号的变化. 

![](http://files.luyanan.com//img/20191107212207.png)

版本有点和我们经常使用的乐观锁类似. 这里有两个概念说一下，一个是悲观锁, 一个是乐观锁. 

#### 悲观锁: 

是数据库中一种非常典型且非常严格的并发控制策略. 假如一个事务A 正在对数据进行处理, 那么在整个处理过程中, 都会将数据处于锁定状态, 在这期间其他事务无法对数据进行更新操作 . 

#### 乐观锁

乐观锁和悲观锁正好相反, 它假定多个事务在处理过程中不会彼此影响, 因此在事务处理过程中不需要进行加锁处理, 如果多个事务对同一数据做更改, 那么在更新请求提交之前,每个事务都会首先检查当前事务读取事务后,是否有其他事务对数据进行了修改. 如果有修改, 则回滚事务再回到zookeeper. version 属性就是用来实现乐观锁机制的"写入校验"的. 

## Watcher 机制

zookeeper 提供了分布式数据的发布/订阅功能,zookeeper 允许客户端向服务端注册一个watcher 监听, 当服务端的一些指定事件触发了watcher, 那么服务器就会向客户端发送一个事件通知. 

值得注意的是, watcher 通知是一次性的,即一旦触发一次通知后,该watcher 就失效了, 因此客户端需要反复注册Wacher,即程序中在process 里面又注册了watcher , 否则, 将无法获取c3节点的创建而导致子节点变化的事件. 

## zookeeper 基于Java 访问

针对zookeeper, 比较常用的java 客户端有 zkclient、curator. 由于curator 对于zookeeper 的抽象层次比较高, 简化了zookeeper 客户端的开发量. 使得curator 逐步被广泛使用. 

1. 封装了 zookeeper client 与 zookeeper server 之间的连接处理. 
2. 提供了一套 fluent 风格的操作API. 
3. 提供了 zookeeper的各种应用场景(共享锁、leader 选举)的抽象封装. 

 ### 依赖jar

```xml
  <!-- zookeeper的jar-->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.0</version>
        </dependency>
```

### 建立连接

```java
    CuratorFramework framework = CuratorFrameworkFactory
                .builder()
                // 连接信息
                .connectString(connect)
                // 会话超时时间
                .sessionTimeoutMs(5000)
                //失败重试机制
                //ExponentialBackoffRetry
                //RetryOneTime  仅仅只重试一次
                //RetryUntilElapsed
                //RetryNTimes
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();

```

> ### 重试策略: 
>
> Curator 内部实现的几种重试策略: 
>
> -  ExponentialBackoffRetry : 重试指定的次数, 且每一次重试之后停顿的时间逐渐增加. 
> -  RetryNTimes : 指定最大重试次数的重试策略. 
> -  RetryOneTime : 仅重试一次. 
> -  RetryUntilElapsed : 一直重试直到达到规定的时间. 
>
> ### namespace
>
> 值得注意的是 session2 会话含有隔离命名空间, 即客户端对zookeeper  上数据节点 的任何操作都是相对/curator 目录进行的, 这有利于实现不同的zookeeper 的业务之间的隔离. 

### 节点的增删改查

```java

    public static void main(String[] args) throws Exception {

        String connect = "192.168.13.102:2181,192.168.13.103:2181,192.168.13.104:2181";
        String path = "/data/program";
        String value = "test";
        // 获取连接
        CuratorFramework framework = createClient(connect);
        // 创建数据节点
        createData(framework, path, value);
        System.out.println("创建的" + path + "节点,数据为:" + getData(framework, path));
        // 修改节点
        updateData(framework, path, "newTest");
        System.out.println("修改的" + path + "节点,数据为:" + getData(framework, path));

        // 删除节点
        deleteData(framework, path);
    }

    /**
     * <p>删除节点</p>
     *
     * @param framework
     * @param nodePath
     * @return {@link }
     * @author luyanan
     * @since 2019/11/6
     */
    public static void deleteData(CuratorFramework framework, String nodePath) throws Exception {
        Stat stat = new Stat();
        String value = new String(framework.getData().storingStatIn(stat).forPath(nodePath));
        framework.delete().withVersion(stat.getAversion()).forPath(nodePath);
    }


    /**
     * <p>修改节点</p>
     *
     * @param framework
     * @param nodePath  节点
     * @param value     新的值
     * @return {@link }
     * @author luyanan
     * @since 2019/11/6
     */
    public static void updateData(CuratorFramework framework, String nodePath, String value) throws Exception {

        framework.setData().forPath(nodePath, value.getBytes());
    }


    /**
     * <p>查询节点</p>
     *
     * @param nodePath
     * @return {@link String}
     * @author luyanan
     * @since 2019/11/6
     */
    public static String getData(CuratorFramework framework, String nodePath) throws Exception {

        return new String(framework.getData().forPath(nodePath));
    }

    /**
     * <p>创建节点</p>
     *
     * @param framework
     * @return {@link }
     * @author luyanan
     * @since 2019/11/6
     */
    public static void createData(CuratorFramework framework, String nodePath, String value) throws Exception {

        framework
                .create()
                //如果没有父节点则创建
                .creatingParentsIfNeeded()
                // 节点类型
                .withMode(CreateMode.PERSISTENT)
                .forPath(nodePath, value.getBytes());
    }


    public static CuratorFramework createClient(String connect) {
        CuratorFramework framework = CuratorFrameworkFactory
                .builder()
                // 连接信息
                .connectString(connect)
                // 会话超时时间
                .sessionTimeoutMs(5000)
                //失败重试机制
                //ExponentialBackoffRetry
                //RetryOneTime  仅仅只重试一次
                //RetryUntilElapsed
                //RetryNTimes
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .build();
        return framework;
    }


```

### 节点权限设置

zookeeper 作为一个分布式协调框架, 内部存储了一些分布式系统运行时的状态的数据, 比如master 选举、比如分布式锁. 对这些数据的操作会直接影响到分布式系统的运行状态。 因此, 为了保证zookeeper 中的数据的安全性, 避免误操作带来的影响. zookeeper 提供了一套ACL 权限控制机制来保证数据的安全. 

#### 权限控制的案例演示

#####  给节点赋权

```java
       // 给节点赋权
        List<ACL> aclList = new ArrayList<>();
        Id id1 = new Id("digest", DigestAuthenticationProvider.generateDigest("u1:uu"));
        Id id2 = new Id("digest", DigestAuthenticationProvider.generateDigest("u2:uu"));

        // u1 有所有的权限
        aclList.add(new ACL(ZooDefs.Perms.ALL, id1));
        // u2 有读和删除的权限
        aclList.add(new ACL(ZooDefs.Perms.READ  | ZooDefs.Perms.DELETE, id2));


        framework.create().withACL(aclList).forPath("/data");
        framework.create().withACL(ZooDefs.Ids.CREATOR_ALL_ACL).forPath("/data");
```

##### 访问授权的节点

```java
      String connect = "192.168.13.102:2181,192.168.13.103:2181,192.168.13.104:2181";

        CuratorFramework framework = CuratorFrameworkFactory.builder()
                .connectString(connect)
                // 带着用户信息创建连接
                .authorization("digest","admin:admin".getBytes())
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .namespace("curator")
                .build();

```

##### 修改已经存在节点的权限

```java
// 修改已经存在节点的权限
framework.setACL().withACL(aclList).forPath("/data");
```

####   权限模式

 权限模式分为 Schema 和授权对象, 比如 IP地址、username:passwrod,用来确定权限验证过程中使用的验证策略

1. **IP** : 通过ip 地址粒度来进行权限控制, 例如配置 [ip:192.168.0.1],或者按照网段 [ip:192.168.0.1/24]
2. **Digest** :  最常见的控制模式, 类似于 username:password,  设置的时候需要 DigestAuthenticationProvider.generateDigest() SHA- 加 密 和 base64 编码 
3. **World** : 最开放的控制模式, 这种权限控制几乎没有任何作用, 数据的访问权限对所有用户开发,  `world:anyone`
4. **Super**:  超级用户, 可以对节点做任何操作. 

#### 授权对象

指权限赋予的用户或者一个指定的实体, 不同的权限模式下，授权对象不同. 

![](http://files.luyanan.com//img/20191107221435.png)

>   Id ipId1 = new Id("ip", "192.168.190.1");
>
>  Id ANYONE_ID_UNSAFE = new Id("world", "anyone");  

####  权限

指通过权限检查后的可以被允许的操作, create/delete/read/write/admin

- create: 允许对子节点create 操作. 
- read:  允许对本节点getCliendren  和getData 操作. 
- write : 允许对本节点setData 操作. 
- delete : 允许对子节点 delete 操作. 
- admin: 允许对本节点 setAcl 操作. 



###  节点事件监听

Watcher 监听机制是Zookeeper 中非常重要的特性, 我们基于zookeeper 上创建的节点, 可以对这些节点绑定监听事件. 比如可以监听节点数据变更、节点删除、子节点状态变更等事件.通过这个事件机制, 可以基于zookeeper 实现分布式锁、集群管理等功能. 

 

| zookeeper 事件                | 事件含义                                                  |
| ----------------------------- | --------------------------------------------------------- |
| EventType.NodeCreated         | 当node-x 这个节点被创建时, 该事件被触发                   |
| EventType.NodeChildrenChanged | 当node-x 这个节点的直接子节点被创建、删除、修改的时候触发 |
| EventType.NodeDataChanged     | 当 node-x  这个节点的数据发生变更的时候,该事件被触发      |
| EventType.NodeDeleted         | 当node-x 这个节点被删除的时候,该事件被触发                |
| EventType.None                | 当zookeeper客户端的连接状态发生变更时, 被 触发            |



watcher 机制有一个特性,当数据发生改变的时候, 那么zookeeper 会产生一个watcher 事件并发送到客户端,但是客户端只会收到一次这样的通知，如果以后这个数据再发生变化, 那么之前设置watch】的客户端不会再收到消息. 因为他是一次性的, 如果要实现永久监听, 可以通过循环注册来实现. 

curator 对节点事件的监听提供了很完善的api, 接下来简单来演示一下 curator 事件监听的基本使用. 

curator 提供了三种Watcher 来监听节点的变化

- PathChildCache: 监视一个路径下孩子节点的创建、删除、更新. 
- NodeCache: 监视当前节点的创建、更新、删除并将节点的数据缓存在本地. 
- TreeCache: PathChildCache 和NodeCache 的结合体, 监视路径下的创建、更新、删除事件, 并缓存路径下的所有孩子节点的数据. 

