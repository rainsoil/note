#  Zookeeper原理之Leader选举源码分析

## Zookeeper 的一致性

### Zookeeper 的来源

对于zookeeper的一致性问题, 我们再从来源问题梳理一遍一致性的问题. 

我们知道zookeeper的来源, 是来自于Google chubby, 为了分布式环境下, 如何从多个server 中选举出master server. 那么这么多个server 就需要涉及到一致性问题, 这个一致性体现的是多个server 就master 这个投票在分布式环境下达成一致性. 简单来说, 就是最终听谁的. 但是在网络环境中由于网络环境的不可靠性, 会存在消息丢失或者被篡改等问题. 所以如何在这样一个环境中快速并且正确的在多个server 中对某一个数据达成一致性并且保证无论发生任何异常, 都不会破坏整个系统一致性呢? 

所以Lampot大神设计了一套Paxos算法, 多个server 基于这个算法就可以达成一致性。而Google chubby 就是基于paxos 算法的实现, 用来实现分布式锁服务. 并且提供了master 选举的服务. 

### Paxos 在chubby 中的应用

也许大家会有疑问, Chubby 于paxos 算法有什么关系呢? Chubby 本来应该被设计成一个包含Paxos 算法的协议库, 使得应用程序可以基于这个库方便的使用paxos 算法, 但是它并没有这么做, 而是把chubby 设计成了一个需要访问中心化节点的分布式锁服务. 既然是 一个服务, 那么它肯定需要是一个高可靠的服务. 所以Chubby 被构建成了一个集群, 集群中存在一个中心节点(master), 采用paxos协议, 通过投票的方式来选举一个获取过半皮票数的服务器作为master, 在chubby集群中, 每个服务器都会维护一份数据的副本, 在实际运行的过程中, 只有master  服务器能执行事务操作, 其他服务器都是使用paxos 协议从master 节点同步最新的数据, 而zookeeper 是chubby 的开源实现, 所以实现原理和chubby 基本是一致性的. 

### zookeeper 的一致性是什么情况呢?

zookeeper的一致性, 体现的是什么一致性呢? 

根据前面讲的zab 协议的同步流程, 在zookeeper 集群内部的数据副本同步,是基于过半提交的策略,意味着他是最终一致性, 并不满足强一致性的要求. 

其实正确来说, zookeeper是一个顺序一致性模型. 由于zookeeper 设计出来是提供分布式锁服务, 那么意味着它本身需要实现顺序一致性 （ http://zookeeper.apache.org/doc/r3.5.5/zookeeperProgrammers.ht ml#ch_zkGuarantees ） 

顺序一致性是在分布式环境下实现分布式锁的基本要求, 比如当一个多个程序来争抢锁, 如果ClientA 获得锁后,后续所有来挣钱锁的程序看到的锁的状态都应该是被ClientA 锁定了, 而不是其他状态. 

### 什么是顺序一致性

在讲顺序一致性之前, 咱们先思考一个问题, 假如说zookeeper是 一个最终一致性模型, 那么它会发生什么情况? 

ClientA/B/C 假设只串行执行, ClientA 更新zookeeper 上的一个值x. ClientB 和ClientC 分别读取集群中的不同副本, 返回的x 的值是不一样的. ClientC 的读取操作是发生在clientB 之后, 但是却读到了过期的值. 很明显, 这是一种弱一致性模型. 如果用它来实现锁机制是有问题的. 

![](http://files.luyanan.com//img/20191113100127.png)

顺序一致性提供了更强的一致性保证, 我们来观察下面这个图, 从时间轴来看, B0 发生在A0之前, 读取的值是0, B2 发生在A0之后，读取到的x 的值为1, 而读操作B1/C0/C1 和写操作A0 在时间轴上有重叠, 因此他们可能读的旧的值为0, 也有可能读取到新的值1, 但是在强顺序一致性模型中, 如果B1得到的x的值为1, 那么C1看到的值也一定为1. 

![](http://files.luyanan.com//img/20191113100912.png)

需要注意的是,由于网络的延迟以及系统本身执行请求的不确定性, 会导致请求发起的客户端不一定会在服务端执行的早。最终以服务端执行的结果为准. 

简单来说: 顺序一致性是针对单个操作, 单个数据对象. 属于CAP中C 这个范畴. 一个数据被更新后, 能够立马被后续的操作读到. 

但是zookeeper的 顺序一致性实现是缩水版本, 在下面的这个网页中,可以看到官网对于一致性的这块做了解释. 

http://zookeeper.apache.org/doc/r3.5.5/zookeeperProgrammers.html#ch_zkGuarantees

zookeeper 不保证在每个实例中, 两个不同的客户端具有相同的zookeeper 数据视图, 由于网络延迟等因素, 一个客户端可能会在另外一个客户端收到更改通知之前执行更新, 

考虑到2个客户端A 和B 的场景, 如果A 把znode /a 的值从0设置为1, 然后告诉客户端B 读取/a, 则客户端B 可能读取到旧的值0, 具体取决于他连接到的服务器, 如果客户端A 和B 要读取 必须要读取到想听的值, 那么ClientB 在读取操作之前要执行sync方法. 

除此之外, zookeeper 基于zxid 以及阻塞队列的方式来实现请求的顺序一致性. 如果client 连接到一个最新的follower 上, 那么它read 读取到了最近的数据, 然后client 由于网络原因重新连接到zookeeper 节点, 而这个时候连接到一个还没有完成数据同步的follower节点, 那么这一次读取到的数据不就是旧的数据了吗? 实际上zookeeper 处理了这种情况, client 会记录自己已经读取到的最大的zxid, 如果client 重连到server 发现client 的zxid 比自己的大. 连接会失败. 

###  Single System Image 的理解  

 zookeeper 官网还说它保证了 "Single System Image ", 其解释为 “ A client will see the same view of the service regardless of the server that it connects to ”. 实际上看来这个解释还是有一点误导性的. 其实由上面的zxid 原理就可以看出来, 它表达的意思是 client 只要连接过一次zookeeper, 就不会有历史的倒退. 

https://github.com/apache/zookeeper/pull/931

## Leader选举的原理

接下来我们基于源码来分析leader 选举的整个过程, 

Leader 选举存在两个阶段, 一个是服务器启动时的leader 选举, 另一个是运行过程中leader 节点宕机导致的leader 选举。 

> 在开始分析选举的原理之前, 先了解几个重要的参数。
>
> #### 服务器id(myid)
>
> ​    比如有三台服务器, 编号分别是1,2,3
>
> ​    编号越大在选择算法中的权重越大
>
> ​    zxid 事务id
>
> 值越大说明数据越新,在选举算法中的权重也越大
>
> ####   逻辑时钟(epoch - logicalclock)
>
> 或者叫投票的次数, 同一轮投票过程中的逻辑时钟值是相同的. 每投完一票这个数据就会增加, 然后与接收到的其他服务器返回的投票信息中的数值相比,根据不同的值做出不同的判断. 
>
> ####  选举状态
>
> ​    LOOKING: 竞选状态。 
>
> ​    FOLLOWING: 随从状态, 同步leader 状态, 参与投票
>
> ​    OBSERVING : 观察状态, 同步leader 状态, 不参与投票. 
>
> ​      LEADING :  领导者状态

### 服务器启动的时候leader 选举

每个节点启动的时候状态都是LOOKING ,处于观望状态, 接下来就进行选主流程. 

若进行leader选举, 则至少需要两台机器, 这里选取3台机器组成的服务器集群为例. 在集群初始化阶段,当有一台服务器server1 启动时, 其单独无法进行和完成leader 选举。 当第二台服务器server2 启动的时候, 此时两台机器可以相互通信,每台机器都试图找到leader, 于是进入leader 选举过程. 选举过程如下:

1. 每个server 发出一个投票, 由于是初始情况, server1 和server2 都会将自己作为leader 服务器来进行投票, 每次投票都会包含所推荐的服务器的myid 和zxid、rpoch,使用(myid,zxid,epoch)来标识, 此时server1 的投票为(1,0),server2 的投票为(2,0),然后各自将这个投票发送给集群中的其他机器. 

2. 接受来自各个服务器的投票, 集群的每个服务器收到投票后, 首先判断该投票的有效性, 如检查是否为本轮投票(epoch)、是否来自 LOOKING  状态的服务器. 

3. 处理投票, 针对每一个投票, 服务器都需要将别人的投票来自己的投票进行PK,PK 规则如下: 

   1. 优先比较 epoch
   2. 其次检查zxid, zxid 比较大的服务器优先为leader
   3. 如果zxid 相同,  那么比较myid, myid 较大的服务器作为leader 服务器. 

    对于server1 而言,它的投票是(1,0),  接受server2 的投票 为(2,0), 首先会比较两者的zxid,均为0, 再比较myid, 此时server2 的myid 最大, 于是更新自己的投票为(2,0), 然后重新开始投票, 对于server2而言, 其无需更新自己的投票, 只是再次向集群中所有寄去发出上一个投票信息即可. 

4. 统计投票

      每次投票后, 服务器都会统计投票信息, 判断是否已经有过接受到相同的投票信息, 对于server1 、server2而言, 都统计出集群中已经有2台机器接受了(2,0) 的投票信息, 此时便认为已经选出了leader. 

5. 改变服务器状态

     一旦确定了leader, 每个服务器都会更新自己的状态, 如果是follower, 那么就变更为following, 如果是leader, 就变更为leading. 

###  运行过程中的leader 选举

当集群中的leader 服务器出现宕机或者不可用的情况下, 那么整个集群将无法对外提供服务, 而是进入新一轮的leader 选举, 服务器运行期间的leader 选举和启动时候的leader 选举基本过程是一致的. 

1. 变更状态

    Leader 挂后, 余下的废Observer 服务器都会将自己的服务器状态变更为LOOKING，然后进入到leader 选举过程. 

2. 每个server 会发出一个投票, 在运行期间, 每个服务器的zxid 可能不同, 此时假设server1 的zxid 为123, server3的zxid 为122, 在第一轮的投票中,server1 和server3 都投自己, 产生投票(1,123),(3,122), 然后各自将投票发送给集群中所有机器. 接受来自各个服务器的投票,与启动时过程相同. 

3. 处理投票, 与启动时过程相同, 此时， server1 将会成为leader

4. 统计投票.与启动时过程相同. 

5. 改变服务器状态

![](http://files.luyanan.com//img/20191113134052.png)

## Leader 选举的源码分析

源码分析, 最关键是要好一个入口, 对于zk 的leader 选举, 并不是由客户端触发的, 而是在启动的时候会触发一次选举. 所以我们们可以直接去看启动脚本 zkServer.sh 中的命令. 



ZOOMAIN 就是QuorumPeerMain,我们基于这个入口来看

> ```shell
> nohup "$JAVA" $ZOO_DATADIR_AUTOCREATE "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" \"-Dzookeeper.log.file=${ZOO_LOG_FILE}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \-XX:+HeapDumpOnOutOfMemoryError -XX:OnOutOfMemoryError='kill -9 %p' \-cp "$CLASSPATH" $JVMFLAGS $ZOOMAIN "$ZOOCFG" > "$_ZOO_DAEMON_OUT" 2>&1 < /dev/null &
> ```

### QuorumPeerMain 的main方法

main方法中,调用了initializeAndRun()  方法进行初始化并且运行.

```java
 protected void initializeAndRun(String[] args)
            throws ConfigException, IOException, AdminServerException {
        // 设置配置参数, 如果args 不为0, 可以基于外部的配置路径来进行解析
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        // 这里启动了一个线程, 来定时对日志进行清理
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        // 如果是集群模式, 会调用runFromConfig servers 其实就是我们在zoo.cfg 中配置的集群节点
        if (args.length == 1 && config.isDistributed()) {
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
```

#### runFromConfig(config)

从名字可以看出, 是基于配置文件来进行启动。

所以整个方法都是对参数进行解析和设置, 直接看核心的代码quorumPeer.start();,启动一个线程, 那么从这句代码可以看出来 QuorumPeerMain 其实是继承了一个Thread, 那么这里面一定有一个run方法. 

```java
public void runFromConfig(QuorumPeerConfig config)
            throws IOException, AdminServerException {
        try {
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) {
            LOG.warn("Unable to register log4j JMX control", e);
        }

        LOG.info("Starting quorum peer");
        try {
            ServerCnxnFactory cnxnFactory = null;
            ServerCnxnFactory secureCnxnFactory = null;

            if (config.getClientPortAddress() != null) {
                cnxnFactory = ServerCnxnFactory.createFactory();
                cnxnFactory.configure(config.getClientPortAddress(),
                        config.getMaxClientCnxns(),
                        false);
            }

            if (config.getSecureClientPortAddress() != null) {
                secureCnxnFactory = ServerCnxnFactory.createFactory();
                secureCnxnFactory.configure(config.getSecureClientPortAddress(),
                        config.getMaxClientCnxns(),
                        true);
            }

            quorumPeer = getQuorumPeer();
            quorumPeer.setTxnFactory(new FileTxnSnapLog(
                    config.getDataLogDir(),
                    config.getDataDir()));
            quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
            quorumPeer.enableLocalSessionsUpgrading(
                    config.isLocalSessionsUpgradingEnabled());
            //quorumPeer.setQuorumPeers(config.getAllMembers());
            quorumPeer.setElectionType(config.getElectionAlg());
            quorumPeer.setMyid(config.getServerId());
            quorumPeer.setTickTime(config.getTickTime());
            quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
            quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
            quorumPeer.setInitLimit(config.getInitLimit());
            quorumPeer.setSyncLimit(config.getSyncLimit());
            quorumPeer.setConfigFileName(config.getConfigFilename());
            quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory()));
            quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
            if (config.getLastSeenQuorumVerifier() != null) {
                quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
            }
            quorumPeer.initConfigInZKDatabase();
            quorumPeer.setCnxnFactory(cnxnFactory);
            quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
            quorumPeer.setSslQuorum(config.isSslQuorum());
            quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
            quorumPeer.setLearnerType(config.getPeerType());
            quorumPeer.setSyncEnabled(config.getSyncEnabled());
            // 投票决定方式, 默认超过半数就通过. 
            quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
            if (config.sslQuorumReloadCertFiles) {
                quorumPeer.getX509Util().enableCertFileReloading();
            }

            // sets quorum sasl authentication configurations
            quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
            if (quorumPeer.isQuorumSaslAuthEnabled()) {
                quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
                quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
                quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
                quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
                quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
            }
            quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
            quorumPeer.initialize();

            // 启动主线程. 
            quorumPeer.start();
            quorumPeer.join();
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Quorum Peer interrupted", e);
        }
    }
```

####  quorumPeer.start()

 quorumPeer.start() 方法,重写了Thread.start() 方法, 也就是在线程启动之前，会做以下操作:

1. 通过loadDataBase() 恢复快照数据

2. cnxnFactory.start() 启动zkServer, 相当于用户可以通过2181 这个端口号进行通信了. 

   ```java
    @Override
       public synchronized void start() {
           if (!getView().containsKey(myid)) {
               throw new RuntimeException("My id " + myid + " not in the peer list");
           }
           // 恢复快照数据
           loadDataBase();
           // 启动zk服务
           startServerCnxnFactory();
           try {
               adminServer.start();
           } catch (AdminServerException e) {
               LOG.warn("Problem starting AdminServer", e);
               System.out.println(e);
           }
           startLeaderElection();
           super.start();
       }
    private void startServerCnxnFactory() {
           if (cnxnFactory != null) {
               cnxnFactory.start();
           }
           if (secureCnxnFactory != null) {
               secureCnxnFactory.start();
           }
       }
   ```

   #### startLeaderElection()

   leader  选举的方法

   ```java
    synchronized public void startLeaderElection() {
           try {
               // 构建一个票据, 用于投票
               currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
           } catch (IOException e) {
               RuntimeException re = new RuntimeException(e.getMessage());
               re.setStackTrace(e.getStackTrace());
               throw re;
           }
           // 这个 getView() 返回是在配置文件中配置的server.myid=ip:port:port
           for (QuorumServer p : getView().values()) {
               // 获得当前zkserver myid 对应的ip地址
               if (p.id == myid) {
                   myQuorumAddr = p.addr;
                   break;
               }
           }
           if (myQuorumAddr == null) {
               throw new RuntimeException("My id " + myid + " not in the peer list");
           }
           // 根据electionType 匹配对应的选举算法, electionType 默认值为3, 可以在配置文件中动态生成. 
           if (electionType == 0) {
               try {
                   udpSocket = new DatagramSocket(myQuorumAddr.getPort());
                   responder = new ResponderThread();
                   responder.start();
               } catch (SocketException e) {
                   throw new RuntimeException(e);
               }
           }
           this.electionAlg = createElectionAlgorithm(electionType);
       }
   ```

#### QuorumPeer.createElectionAlgorithm

根据对应的标识创建选举算法

```java
 protected Election createElectionAlgorithm(int electionAlgorithm) {
        Election le = null;

        //TODO: use a factory rather than a switch
        switch (electionAlgorithm) {
            case 0:
                le = new LeaderElection(this);
                break;
            case 1:
                le = new AuthFastLeaderElection(this);
                break;
            case 2:
                le = new AuthFastLeaderElection(this, true);
                break;
            case 3:
                qcm = createCnxnManager();
                QuorumCnxManager.Listener listener = qcm.listener;
                if (listener != null) {
                    // 启动监听
                    listener.start();
                    // 初始化 FastLeaderElection
                    le = new FastLeaderElection(this, qcm);
                } else {
                    LOG.error("Null listener when initializing cnx manager");
                }
                break;
            default:
                assert false;
        }
        return le;
    }
```

#### **FastLeaderElection**

初始化FastLeaderElection , QuorumCnxManager 是一个很核心的对象, 用来实现 领导选举中的网络连接管理功能, 

```java
 public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
        this.stop = false;
        this.manager = manager;
        starter(self, manager);
    }
```

#### FastLeaderElection.starter

starter 方法里面, 设置了一些成员属性, 并且构建了两个阻塞队列, 分别是 sendQueue  和 recvqueue . 并且实例化了一个Messenger 

```java
 private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;

        sendqueue = new LinkedBlockingQueue<ToSend>();
        recvqueue = new LinkedBlockingQueue<Notification>();
        this.messenger = new Messenger(manager);
    }
```

#### Messenger

在 Messenger 中构建了两个线程, 一个是WorkerSender, 一个是WorkerReceiver. 这两个线程是分别用来发送和接受消息的线程. 

```java
  Messenger(QuorumCnxManager manager) {

            this.ws = new WorkerSender(manager);

            Thread t = new Thread(this.ws,
                    "WorkerSender[myid=" + self.getId() + "]");
            t.setDaemon(true);
            t.start();

            this.wr = new WorkerReceiver(manager);

            t = new Thread(this.wr,
                    "WorkerReceiver[myid=" + self.getId() + "]");
            t.setDaemon(true);
            t.start();
        }
```

####  阶段性提交

分析到这里,先做一个简单的总结, 通过一个流程图把前面部分的功能串联起来。 

![](http://files.luyanan.com//img/20191113161725.png)

####  getView() 的解析过程

```java
    public Map<Long, QuorumPeer.QuorumServer> getView() {
        return Collections.unmodifiableMap(this.quorumPeers);
    }
```

getView()  里面实际上返回是一个quorumPeers, 就是参与本次投票的成员有哪些? 这个属性在哪里赋值呢?

#### QuorumPeerMain.runFromConfig

设置了一个值为config.getServers(), 

> ```
> quorumPeer.setQuorumPeers(config.getServers());
> ```

config  这个配置信息又是通过 initializeAndRun  方法中初始化的. 

```jav
  // 设置配置参数, 如果args 不为0, 可以基于外部的配置路径来解析
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

```

#### QuorumPeerConfig.parse

这里会根据一个外部的文件去解析, 然后其中一段是这样的, 解析对应的集群信息放到servers 这个集合中. 

```java
} else if (key.startsWith("server.")) {
                int dot = key.indexOf('.');
                long sid = Long.parseLong(key.substring(dot + 1));
                String parts[] = splitWithLeadingHostname(value);
                if ((parts.length != 2) && (parts.length != 3) && (parts.length !=4)) {
                    LOG.error(value
                       + " does not have the form host:port or host:port:port " +
                       " or host:port:port:type");
                }
                LearnerType type = null;
                String hostname = parts[0];
                Integer port = Integer.parseInt(parts[1]);
                Integer electionPort = null;
                if (parts.length > 2){
                	electionPort=Integer.parseInt(parts[2]);
                }
                if (parts.length > 3){
                    if (parts[3].toLowerCase().equals("observer")) {
                        type = LearnerType.OBSERVER;
                    } else if (parts[3].toLowerCase().equals("participant")) {
                        type = LearnerType.PARTICIPANT;
                    } else {
                        throw new ConfigException("Unrecognised peertype: " + value);
                    }
                }
                if (type == LearnerType.OBSERVER){
                    observers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
                } else {
                    servers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
                }
```



## Zookeeper 服务启动的逻辑

在讲leader 选举的时候, 有一个 cnxnFactory.start()  方法来启动zk 服务, 这块具体做了什么呢? 我们来分析看看

### QuorumPeerMain.runFromConfig

在runFromConfig中， 构建了一个ServerCnxnFactory

```java
  public void runFromConfig(QuorumPeerConfig config) throws IOException {
        try {
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) {
            LOG.warn("Unable to register log4j JMX control", e);
        }

        LOG.info("Starting quorum peer");
        try {
            ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
            cnxnFactory.configure(config.getClientPortAddress(),
                    config.getMaxClientCnxns());

            
            ...
        }
```

这个明显是一个工厂模式, 基于这个工厂类创建什么呢? 打开createFactory() 方法看看就知道了. 

### ServerCnxnFactory.createFactory()

这个方法里面是根据`ZOOKEEPER_SERVER_CNXN_FACTORY`  来决定创建NIO Server 还是Netty Server

而默认情况下, 应该是创建一个 NIOServerCnxnFactory 

```java
  static public ServerCnxnFactory createFactory() throws IOException {
        String serverCnxnFactoryName =
            System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
        if (serverCnxnFactoryName == null) {
            serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
        }
        try {
            ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
                    .getDeclaredConstructor().newInstance();
            LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
            return serverCnxnFactory;
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate "
                    + serverCnxnFactoryName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```

### quorumPeer.start();

因此, 我们再回到 quorumPeer.start(); 方法中,  cnxnFactory.start()，  应该会调用NIOServerCnxnFactory 这个类去启动一个线程. 

```java
   @Override
    public synchronized void start() {
        // 恢复快照数据
        loadDataBase();
        // 启动zk 服务
        cnxnFactory.start();
        startLeaderElection();
        super.start();
    }


```

### NIOServerCnxnFactory.start()

这里通过thread.start  启动一个线程, 那thread 是一个什么对象呢? 

```java
   @Override
    public void start() {
        // ensure thread is started once and only once
        if (thread.getState() == Thread.State.NEW) {
            thread.start();
        }
    }
```

### NIOServerCnxnFactory.configure

thread  其实构建的是一个zookeeperThread 线程, 并且线程的参数为this, 表示当前NIOServerCnxnFactory 也是实现了线程的类, 那么它必须要重写run() 方法, NIOServer 的初始化以及启动过程就完成的. 并且对2181 这个端口号进行监听, 一旦发现有请求进来, 就执行相应的处理即可. 

```java
Thread thread;
    @Override
    public void configure(InetSocketAddress addr, int maxcc) throws IOException {
        configureSaslLogin();

        thread = new ZooKeeperThread(this, "NIOServerCxn.Factory:" + addr);
        thread.setDaemon(true);
        maxClientCnxns = maxcc;
        this.ss = ServerSocketChannel.open();
        ss.socket().setReuseAddress(true);
        LOG.info("binding to port " + addr);
        ss.socket().bind(addr);
        ss.configureBlocking(false);
        ss.register(selector, SelectionKey.OP_ACCEPT);
    }
```

## 选举流程分析

接下来我们正式分析leader 选举的过程. 

```java
   @Override
    public synchronized void start() {
        // 恢复快照数据
        loadDataBase();
        // 启动zk 服务
        cnxnFactory.start();
        startLeaderElection();
        super.start();
    }
```

很明显,super.start()  表示当前类QuorumPeer 继承了线程, 线程必须要重写run() 方法, 所以我们可以在 QuorumPeer 中找到一个run方法

### QuorumPeer.run()

这段代码的逻辑比较长, 粗略看一下结构, 好像也不难 

 PeerState  有几种状态, 分别是: 

1.  LOOKING : 竞选状态
2.  FOLLOWING:  随从状态. 同步leader 状态, 参与投票
3.  OBSERVING : 观察状态, 同步leader 状态, 不参与投票. 
4.  LEADING : 领导者状态. 

对于选举来说, 默认都是  LOOKING  状态. 

只有 LOOKING  状态才会去执行选举算法, 每个服务器在启动的时候都会选择自己作为领导，然后将投票信息发送出去, 循环一直到选举出领导为止. 

```java
 @Override
    public void run() {
        setName("QuorumPeer" + "[myid=" + getId() + "]" +
                cnxnFactory.getLocalAddress());

        LOG.debug("Starting quorum peer");
        try {
            jmxQuorumBean = new QuorumBean(this);
            MBeanRegistry.getInstance().register(jmxQuorumBean, null);
            for (QuorumServer s : getView().values()) {
                ZKMBeanInfo p;
                if (getId() == s.id) {
                    p = jmxLocalPeerBean = new LocalPeerBean(this);
                    try {
                        MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                    } catch (Exception e) {
                        LOG.warn("Failed to register with JMX", e);
                        jmxLocalPeerBean = null;
                    }
                } else {
                    p = new RemotePeerBean(s);
                    try {
                        MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                    } catch (Exception e) {
                        LOG.warn("Failed to register with JMX", e);
                    }
                }
            }
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            jmxQuorumBean = null;
        }

        try {
            /*
             * Main loop
             */
            // 根据选举状态, 选择不同的处理方式
            while (running) {
                switch (getPeerState()) {
                    case LOOKING:
                        LOG.info("LOOKING");

                        // 判断是否为只读模式, 通过readonlymode.enabled 开启
                        if (Boolean.getBoolean("readonlymode.enabled")) {
                            LOG.info("Attempting to start ReadOnlyZooKeeperServer");

                            // 只读模式的启动流程
                            // Create read-only server but don't start it immediately
                            final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(
                                    logFactory, this,
                                    new ZooKeeperServer.BasicDataTreeBuilder(),
                                    this.zkDb);

                            // Instead of starting roZk immediately, wait some grace
                            // period before we decide we're partitioned.
                            //
                            // Thread is used here because otherwise it would require
                            // changes in each of election strategy classes which is
                            // unnecessary code coupling.
                            Thread roZkMgr = new Thread() {
                                public void run() {
                                    try {
                                        // lower-bound grace period to 2 secs
                                        sleep(Math.max(2000, tickTime));
                                        if (ServerState.LOOKING.equals(getPeerState())) {
                                            roZk.startup();
                                        }
                                    } catch (InterruptedException e) {
                                        LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                                    } catch (Exception e) {
                                        LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                                    }
                                }
                            };
                            try {
                                roZkMgr.start();
                                setBCVote(null);
                                // 设置当前的投票, 通过策略模式来决定当前用哪个选举算法来进行领导选举
                                setCurrentVote(makeLEStrategy().lookForLeader());
                            } catch (Exception e) {
                                LOG.warn("Unexpected exception", e);
                                setPeerState(ServerState.LOOKING);
                            } finally {
                                // If the thread is in the the grace period, interrupt
                                // to come out of waiting.
                                roZkMgr.interrupt();
                                roZk.shutdown();
                            }
                        } else {
                            try {
                                setBCVote(null);
                                setCurrentVote(makeLEStrategy().lookForLeader());
                            } catch (Exception e) {
                                LOG.warn("Unexpected exception", e);
                                setPeerState(ServerState.LOOKING);
                            }
                        }
                        break;
                    case OBSERVING:
                        try {
                            LOG.info("OBSERVING");
                            setObserver(makeObserver(logFactory));
                            observer.observeLeader();
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                        } finally {
                            observer.shutdown();
                            setObserver(null);
                            setPeerState(ServerState.LOOKING);
                        }
                        break;
                    case FOLLOWING:
                        try {
                            LOG.info("FOLLOWING");
                            setFollower(makeFollower(logFactory));
                            follower.followLeader();
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                        } finally {
                            follower.shutdown();
                            setFollower(null);
                            setPeerState(ServerState.LOOKING);
                        }
                        break;
                    case LEADING:
                        LOG.info("LEADING");
                        try {
                            setLeader(makeLeader(logFactory));
                            leader.lead();
                            setLeader(null);
                        } catch (Exception e) {
                            LOG.warn("Unexpected exception", e);
                        } finally {
                            if (leader != null) {
                                leader.shutdown("Forcing shutdown");
                                setLeader(null);
                            }
                            setPeerState(ServerState.LOOKING);
                        }
                        break;
                }
            }
        } finally {
            LOG.warn("QuorumPeer main thread exited");
            try {
                MBeanRegistry.getInstance().unregisterAll();
            } catch (Exception e) {
                LOG.warn("Failed to unregister with JMX", e);
            }
            jmxQuorumBean = null;
            jmxLocalPeerBean = null;
        }
    }
```



###  FastLeaderElection .lookForLeader()

开始发起投票流程

```java
 public Vote lookForLeader() throws InterruptedException {
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(
                    self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
            self.start_fle = Time.currentElapsedTime();
        }
        try {
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

            int notTimeout = finalizeWait;

            synchronized (this) {
                // 更新逻辑时钟, 用来判断是否在同一轮选举周期
                logicalclock.incrementAndGet();
                // 初始化选票数据, 这里其实就是把当前节点的myid,zxid,epoch 更新到本地的成员属性
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() +
                    ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            // 异步发送选举消息
            sendNotifications();

            /*
             * Loop in which we exchange notifications until we find a leader
             */
            // 不断循环, 根据投票信息进行leader 选举

            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)) {
                /*
                 * Remove next notification from queue, times out after 2 times
                 * the termination time
                 */
                // 从 recvqueue 中获取消息
                Notification n = recvqueue.poll(notTimeout,
                        TimeUnit.MILLISECONDS);

                /*
                 * Sends more notifications if haven't received enough.
                 * Otherwise processes new notification.
                 */
                // 如果没有获取到外部的投票, 有可能是集群之间的节点没有真正连接上
                if (n == null) {
                    // 判断发送队列是否由数据,如果发送队列为空,再发一次自己的选票
                    if (manager.haveDelivered()) {
                        sendNotifications();
                    } else {
                        // 再次发起集群节点之间的连接
                        manager.connectAll();
                    }

                    /*
                     * Exponential backoff
                     */
                    int tmpTimeOut = notTimeout * 2;
                    notTimeout = (tmpTimeOut < maxNotificationInterval ?
                            tmpTimeOut : maxNotificationInterval);
                    LOG.info("Notification time out: " + notTimeout);
                    ...
                }
```

### 选票的判断逻辑(核心代码)

```java
     // 判断收到的选票中的sid 和选举的leader 的sid 是否存在与我们集群锁配置的myid 范围.
                } else if (validVoter(n.sid) && validVoter(n.leader)) {
                    // 判断接受到的投票者的状态, 默认是LOOKING 状态, 说明当前发起投票的服务器也是找leader
                    /*
                     * Only proceed if the vote comes from a replica in the
                     * voting view for a replica in the voting view.
                     */
                    switch (n.state) {
                        case LOOKING:
                            // 说明当前发起投票的服务器也是在找leader
                            // 如果收到的投票的逻辑时钟大于当前的节点的逻辑时钟
                            // If notification > current, replace and send messages out
                            if (n.electionEpoch > logicalclock.get()) {
                                // 更新成新一轮的逻辑时钟
                                logicalclock.set(n.electionEpoch);
                                recvset.clear();
                                // 比较接收到的投票和当前节点的细腻些进行比较,比较的顺序
                                // epoch、zxid、myid,如果返回的为true,  在更新当前节点大票据
                                // (sid、zxid、epoch)
                                //那么下一次再发起投票的时候, 就不再选自己了
                                if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                    updateProposal(n.leader, n.zxid, n.peerEpoch);
                                } else {
                                    // 否则, 说明当前节点的票据优先级更高, 再次更新自己的票据
                                    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
                                }
                                // 再次发送消息把当前的票据发出去,告诉大家要选择n.leader  为leader
                                sendNotifications();
                            } else if (n.electionEpoch < logicalclock.get()) {
                                // 如果小于, 说明收到的票据已经过期,直接把这张票丢掉
                                if (LOG.isDebugEnabled()) {
                                    LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x" + Long.toHexString(n.electionEpoch) + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                                }
                                break;

                                // 这个判断表示收到的票据epoch 是想同的, 那么按照epoch、zxid、myid顺序进行比较,比较成功后,把对方的票据信息更新到自己的节点
                            } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                                sendNotifications();
                            }

                            if (LOG.isDebugEnabled()) {
                                LOG.debug("Adding vote: from=" + n.sid + ", proposed leader=" + n.leader + ", proposed zxid=0x" + Long.toHexString(n.zxid) + ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                            }

                            // 将收到的投票信心放入到投票的集合recvset中, 用来做最终的"过半原则" 判断
                            recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                            if (termPredicate(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch))) {

                                // 进入这个判断, 说明选票进入到了leader 选举的要求
                                // 在更新状态之前, 服务器会等待finalizeWait 毫秒时间来接受新的选票,以防止漏下来关键的选票
                                // 如果收到可能改变leader 的选票, 则重新进行计票
                                // Verify if there is any change in the proposed leader
                                while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
                                    if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
                                        recvqueue.put(n);
                                        break;
                                    }
                                }

                                /*
                                 * This predicate is true once we don't read any new
                                 * relevant message from the reception queue
                                 */
                                // 如果Notification为空,说明leader 节点是已经确定好了
                                if (n == null) {
                                    // 设置当前节点的状态(判断leader 节点不是我自己, 如果是, 直接更新当前节点的 state 为LEADING)
                                    // 否则,根据当前节点的特性进行判断,决定是FOLLOWING 还是OBSERVING
                                    self.setPeerState((proposedLeader == self.getId()) ? ServerState.LEADING : learningState());

                                    // 组装生成这次leader 选举最终投票的结果.
                                    Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
                                    // 清空 recvqueue
                                    leaveInstance(endVote);
                                    // 返回最终的票据
                                    return endVote;
                                }
                            }
                            break;
                        case OBSERVING:
                            // OBSERVING 不参与Leader 的选举
                            LOG.debug("Notification from observer: " + n.sid);
                            break;
                        case FOLLOWING:
                        case LEADING:
                            /*
                             * Consider all notifications from the same epoch
                             * together.
                             */
                            if (n.electionEpoch == logicalclock.get()) {
                                recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

                                if (ooePredicate(recvset, outofelection, n)) {
                                    self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING : learningState());

                                    Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                                    leaveInstance(endVote);
                                    return endVote;
                                }
                            }

                            /*
                             * Before joining an established ensemble, verify
                             * a majority is following the same leader.
                             */
                            outofelection.put(n.sid, new Vote(n.version, n.leader, n.zxid, n.electionEpoch, n.peerEpoch, n.state));

                            if (ooePredicate(outofelection, outofelection, n)) {
                                synchronized (this) {
                                    logicalclock.set(n.electionEpoch);
                                    self.setPeerState((n.leader == self.getId()) ? ServerState.LEADING : learningState());
                                }
                                Vote endVote = new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                            break;
                        default:
                            LOG.warn("Notification state unrecognized: {} (n.state), {} (n.sid)", n.state, n.sid);
                            break;
                    }
                } else {
                    if (!validVoter(n.leader)) {
                        LOG.warn("Ignoring notification for non-cluster member sid {} from sid {}", n.leader, n.sid);
                    }
                    if (!validVoter(n.sid)) {
                        LOG.warn("Ignoring notification for sid {} from non-quorum member sid {}", n.leader, n.sid);
                    }
                }
            }
            return null;
        } finally {
            try {
                if (self.jmxLeaderElectionBean != null) {
                    MBeanRegistry.getInstance().unregister(self.jmxLeaderElectionBean);
                }
            } catch (Exception e) {
                LOG.warn("Failed to unregister with JMX", e);
            }
            self.jmxLeaderElectionBean = null;
            LOG.debug("Number of connection processing threads: {}", manager.getConnectionThreadCount());
        }
    }

    /**
     * Check if a given sid is represented in either the current or
     * the next voting view
     *
     * @param sid Server identifier
     * @return boolean
     */
    private boolean validVoter(long sid) {
        return self.getVotingView().containsKey(sid);
    }
```



###  投票处理的流程图



![](http://files.luyanan.com//img/20191114135751.png)

###  termPredicate

这个方法是使用过半原则来判断选举是否结束的, 如果返回true, 说明能够选出leader 服务器. 

 votes  表示收到的外部选票的集合

 vote  表示当前服务器的选票

```java
  protected boolean termPredicate(HashMap<Long, Vote> votes, Vote vote) {

        HashSet<Long> set = new HashSet<Long>();

        /*
         * First make the views consistent. Sometimes peers will have
         * different zxids for a server depending on timing.
         */
        // 遍历接受到的所有票据数据
        for (Map.Entry<Long, Vote> entry : votes.entrySet()) {
            // 对选票进行归纳, 就是把所有选票数据中和当前节点的票据相同的票据进行统计.
            if (vote.equals(entry.getValue())) {
                set.add(entry.getKey());
            }
        }

        // 对票据进行判断
        return self.getQuorumVerifier().containsQuorum(set);
    }
```

### QuorumMaj.containsQuorum

```java
    public boolean containsQuorum(Set<Long> set){
        return (set.size() > half);
    }
```

这个half 的值是多少呢? 可以在 QuorumPeerConfig. parseProperties   这个方法中, 找到如下源码: 

```java

                LOG.info("Defaulting to majority quorums");
                quorumVerifier = new QuorumMaj(servers.size());
```

也就是说, 在构建QuorumMaj的时候,传递了当前集群节点的数量, 这里是3. 那么half = 3/2= 1

```java
    public QuorumMaj(int n){
        this.half = n/2;
    }
```

那么 set.size()>1 ,意味着至少有两个节点的票据是选择你当leader , 否则, 还得继续投. 



## 投票的网络通信过程

### 通信流程图

![](http://files.luyanan.com//img/20191114140808.png)

![](http://files.luyanan.com//img/20191114140956.png)

### 接受数据  Notification  和发送数据  ToSend



|字段| Notification             | ToSend |
|---| ------------------------ | ------ |
|leader| 被推荐的服务器sid | 被推荐的服务器sid |
| zxid | 被推荐的服务器当前最新的事务id | 被推荐的服务器当前的事务id |
| peerEpoch | 被推荐的服务器当前所处的epoch | 被推荐的服务器当前所处的epoch |
| electionepoch | 当前服务器所处的epoch | 当前服务器所处的epoch |
| state | 当前服务器状态 | 选举服务器当前服务器状态 |
| sid | 接受消息的服务器sid(myid) | 选举服务器的sid |



### 通信过程源码分析

#### 每个zk 服务启动后创建socket 监听

```java
 protected Election createElectionAlgorithm(int electionAlgorithm) {
        Election le = null;

        //TODO: use a factory rather than a switch
        switch (electionAlgorithm) {
            case 0:
                le = new LeaderElection(this);
                break;
            case 1:
                le = new AuthFastLeaderElection(this);
                break;
            case 2:
                le = new AuthFastLeaderElection(this, true);
                break;
            case 3:
                qcm = createCnxnManager();
                QuorumCnxManager.Listener listener = qcm.listener;
                if (listener != null) {
                    // 启动监听,listener 实现了线程,所以在run方法, 可以看到构建serverSocket 的请求
                    // 这里专门用来接受其他zkServer 的投票请求
                    listener.start();
                    // 初始化 FastLeaderElection
                    le = new FastLeaderElection(this, qcm);
                } else {
                    LOG.error("Null listener when initializing cnx manager");
                }
                break;
            default:
                assert false;
        }
        return le;
    }
```

```java

        /**
         * Sleeps on accept().
         */
        @Override
        public void run() {
            int numRetries = 0;
            InetSocketAddress addr;
            while((!shutdown) && (numRetries < 3)){
                try {
                    ss = new ServerSocket();
                    ss.setReuseAddress(true);
                    if (listenOnAllIPs) {
                        int port = view.get(QuorumCnxManager.this.mySid)
                            .electionAddr.getPort();
                        addr = new InetSocketAddress(port);
                    } else {
                        addr = view.get(QuorumCnxManager.this.mySid)
                            .electionAddr;
                    }
                    LOG.info("My election bind port: " + addr.toString());
                    setName(view.get(QuorumCnxManager.this.mySid)
                            .electionAddr.toString());
                    ss.bind(addr);
                    while (!shutdown) {
                        Socket client = ss.accept();
                        setSockOpts(client);
                        LOG.info("Received connection request "
                                + client.getRemoteSocketAddress());

                        // Receive and handle the connection request
                        // asynchronously if the quorum sasl authentication is
                        // enabled. This is required because sasl server
                        // authentication process may take few seconds to finish,
                        // this may delay next peer connection requests.
                        if (quorumSaslAuthEnabled) {
                            receiveConnectionAsync(client);
                        } else {
                            receiveConnection(client);
                        }

                        numRetries = 0;
                    }
                } catch (IOException e) {
                    LOG.error("Exception while listening", e);
                    numRetries++;
                    try {
                        ss.close();
                        Thread.sleep(1000);
                    } catch (IOException ie) {
                        LOG.error("Error closing server socket", ie);
                    } catch (InterruptedException ie) {
                        LOG.error("Interrupted while sleeping. " +
                                  "Ignoring exception", ie);
                    }
                }
            }
            LOG.info("Leaving listener");
            if (!shutdown) {
                LOG.error("As I'm leaving the listener thread, "
                        + "I won't be able to participate in leader "
                        + "election any longer: "
                        + view.get(QuorumCnxManager.this.mySid).electionAddr);
            }
        }
```



### FastLeaderElection.lookForLeader

这个方法前面分析过, 里面会调用 sendNotifications  来发送投票请求

```java
 public Vote lookForLeader() throws InterruptedException {
        try {
            self.jmxLeaderElectionBean = new LeaderElectionBean();
            MBeanRegistry.getInstance().register(self.jmxLeaderElectionBean, self.jmxLocalPeerBean);
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            self.jmxLeaderElectionBean = null;
        }
        if (self.start_fle == 0) {
            self.start_fle = Time.currentElapsedTime();
        }
        try {
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

            int notTimeout = finalizeWait;

            synchronized (this) {
                // 更新逻辑时钟, 用来判断是否在同一轮选举周期
                logicalclock.incrementAndGet();
                // 初始化选票数据, 这里其实就是把当前节点的myid,zxid,epoch 更新到本地的成员属性
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() + ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            // 异步发送选举消息
            sendNotifications();

            ...
        }
```

###  FastLeaderElection.sendqueue

sendqueue 这个队列的数据, 是通过 WorkerSender  来进行获取并发送的, 而这个 WorkerSender  线程, 在构建  fastLeaderElection 的时候会启动.

```java
 class WorkerSender extends ZooKeeperThread {
            volatile boolean stop;
            QuorumCnxManager manager;

            WorkerSender(QuorumCnxManager manager) {
                super("WorkerSender");
                this.stop = false;
                this.manager = manager;
            }

            public void run() {
                while (!stop) {
                    try {
                        // 从队列中获取ToSend 对象. 
                        ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                        if (m == null) continue;

                        process(m);
                    } catch (InterruptedException e) {
                        break;
                    }
                }
                LOG.info("WorkerSender is down");
            }

            /**
             * Called by run() once there is a new message to send.
             *
             * @param m message to send
             */
            void process(ToSend m) {
                ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), m.leader, m.zxid, m.electionEpoch, m.peerEpoch);
                // 这里就是调用 QuorumCnxManager 进行消息发送. 
                 manager.toSend(m.sid, requestBuffer);
            }
        }
```

### QuorumCnxManager.toSend

```java
 public void toSend(Long sid, ByteBuffer b) {
        /*
         * If sending message to myself, then simply enqueue it (loopback).
         */
        //  如果接收者是自己, 直接放置到接受队列
        if (this.mySid == sid) {
            b.position(0);
            addToRecvQueue(new Message(b.duplicate(), sid));
            /*
             * Otherwise send to the corresponding thread to send.
             */
        } else {
            /*
             * Start a new connection if doesn't have one already.
             */
            // 否则发送到对应的发送队列上
            ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
            // 判断当前的sid 是否已经存在与发送队列, 如果是, 则直接把已经存在的数据发送出去.
            ArrayBlockingQueue<ByteBuffer> bqExisting = queueSendMap.putIfAbsent(sid, bq);
            if (bqExisting != null) {
                addToSendQueue(bqExisting, b);
            } else {
                addToSendQueue(bq, b);
            }
            // 连接申请 调用链 connectOne->initiateConnection-> startConnection
            // startConnection 就是发送方启动入口
            connectOne(sid);

        }
    }
```

### startConnection

```java
  private boolean startConnection(Socket sock, Long sid) throws IOException {
        DataOutputStream dout = null;
        DataInputStream din = null;
        try {
            // Sending id and challenge
            dout = new DataOutputStream(sock.getOutputStream());
            dout.writeLong(this.mySid);
            dout.flush();

            din = new DataInputStream(new BufferedInputStream(sock.getInputStream()));
        } catch (IOException e) {
            LOG.warn("Ignoring exception reading or writing challenge: ", e);
            closeSocket(sock);
            return false;
        }

        // authenticate learner
        authLearner.authenticate(sock, view.get(sid).hostname);

        // If lost the challenge, then drop the new connection
        if (sid > this.mySid) {
            // 为了防止重复建立连接, 只需要sid 大的主动连接sid 小的.
            LOG.info("Have smaller server identifier, so dropping the " + "connection: (" + sid + ", " + this.mySid + ")");
            closeSocket(sock);
            // Otherwise proceed with the connection
        } else {
            // 构建一个发送线程和接受线程, 负责针对当前连接的数据传输, 
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);

            if (vsw != null) vsw.finish();

            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));

            sw.start();
            rw.start();

            return true;

        }
        return false;
    }
```

####  SendWorker 

 SendWorker 会监听对应sid 的阻塞队列, 启动的时候, 如果队列为空时会重新发送一次消息最前最后的消息, 以防止上一次处理是服务器异常退出, 造成上一条消息未处理成功; 然后就是不停监听队列, 发现有消息时调用send 方法.

####  RecvWorker 

 RecvWorker  不停监听socket 的inputstream ,读取消息放到消息接收队列中, 消息放入队列中, qcm的流程就完毕了. 

###  QuorumCnxManager.Listener 

listener 监听到客户端请求后, 开始处理消息. 

```java
   @Override
        public void run() {
            int numRetries = 0;
            InetSocketAddress addr;
            while ((!shutdown) && (numRetries < 3)) {
                try {
                    ss = new ServerSocket();
                    ss.setReuseAddress(true);
                    if (listenOnAllIPs) {
                        int port = view.get(QuorumCnxManager.this.mySid).electionAddr.getPort();
                        addr = new InetSocketAddress(port);
                    } else {
                        addr = view.get(QuorumCnxManager.this.mySid).electionAddr;
                    }
                    LOG.info("My election bind port: " + addr.toString());
                    setName(view.get(QuorumCnxManager.this.mySid).electionAddr.toString());
                    ss.bind(addr);
                    while (!shutdown) {
                        Socket client = ss.accept();
                        setSockOpts(client);
                        LOG.info("Received connection request " + client.getRemoteSocketAddress());

                        // Receive and handle the connection request
                        // asynchronously if the quorum sasl authentication is
                        // enabled. This is required because sasl server
                        // authentication process may take few seconds to finish,
                        // this may delay next peer connection requests.
                        if (quorumSaslAuthEnabled) {
                            receiveConnectionAsync(client);
                        } else {
                            // 接受客户端请求
                            receiveConnection(client);
                        }

                        numRetries = 0;
                    }
                } catch (IOException e) {
                    LOG.error("Exception while listening", e);
                    numRetries++;
                    try {
                        ss.close();
                        Thread.sleep(1000);
                    } catch (IOException ie) {
                        LOG.error("Error closing server socket", ie);
                    } catch (InterruptedException ie) {
                        LOG.error("Interrupted while sleeping. " + "Ignoring exception", ie);
                    }
                }
            }
            LOG.info("Leaving listener");
            if (!shutdown) {
                LOG.error("As I'm leaving the listener thread, " + "I won't be able to participate in leader " + "election any longer: " + view.get(QuorumCnxManager.this.mySid).electionAddr);
            }
        }
```

###  QuorumCnxManager.receiveConnection

```java
 public void receiveConnection(final Socket sock) {
        DataInputStream din = null;
        try {
            // 获取客户端的数据包
            din = new DataInputStream(new BufferedInputStream(sock.getInputStream()));

            handleConnection(sock, din);
        } catch (IOException e) {
            LOG.error("Exception handling connection, addr: {}, closing server connection", sock.getRemoteSocketAddress());
            closeSocket(sock);
        }
    }
```



###  QuorumCnxManager.handleConnection

```java
   private void handleConnection(Socket sock, DataInputStream din) throws IOException {
        Long sid = null;
        try {
            // Read server id
            sid = din.readLong();
            if (sid < 0) { // this is not a server id but a protocol version (see ZOOKEEPER-1633)
                // 获取客户端的sid,也就是myid
                sid = din.readLong();

                // next comes the #bytes in the remainder of the message
                // note that 0 bytes is fine (old servers)
                int num_remaining_bytes = din.readInt();
                if (num_remaining_bytes < 0 || num_remaining_bytes > maxBuffer) {
                    LOG.error("Unreasonable buffer length: {}", num_remaining_bytes);
                    closeSocket(sock);
                    return;
                }
                byte[] b = new byte[num_remaining_bytes];

                // remove the remainder of the message from din
                int num_read = din.read(b);
                if (num_read != num_remaining_bytes) {
                    LOG.error("Read only " + num_read + " bytes out of " + num_remaining_bytes + " sent by server " + sid);
                }
            }
            if (sid == QuorumPeer.OBSERVER_ID) {
                /*
                 * Choose identifier at random. We need a value to identify
                 * the connection.
                 */
                sid = observerCounter.getAndDecrement();
                LOG.info("Setting arbitrary identifier to observer: " + sid);
            }
        } catch (IOException e) {
            closeSocket(sock);
            LOG.warn("Exception reading or writing challenge: " + e.toString());
            return;
        }

        // do authenticating learner
        LOG.debug("Authenticating learner server.id: {}", sid);
        authServer.authenticate(sock, din);

        //If wins the challenge, then close the new connection.
        //
        // 为了防止重复建立连接, 只允许sid 大的主动连接sid 小的
        if (sid < this.mySid) {
            /*
             * This replica might still believe that the connection to sid is
             * up, so we have to shut down the workers before trying to open a
             * new connection.
             */
            SendWorker sw = senderWorkerMap.get(sid);
            if (sw != null) {
                sw.finish();
            }

            /*
             * Now we start a new connection
             */
            LOG.debug("Create new connection to server: " + sid);
            // 关闭连接
            closeSocket(sock);
            // 向sid 发起连接
            connectOne(sid);

            // Otherwise start worker threads to receive data.
        } else {
            // 同样, 构建一个SendWorker 和RecvWorker 进行发送数据和接受数据
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);

            if (vsw != null) vsw.finish();

            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));

            sw.start();
            rw.start();

            return;
        }
    }
```



## Leader 选举完成之后的处理逻辑

通过 lookForLeader  方法选举完成后, 会设置当前节点的 PeerState， , 要么为Leading, 要么就是 FOLLOWER，或者Observer, 到这里, 只是表示当前的leader 选举出来了, 但是 QuorumPeer.run  方法还没有执行完, 我们再回过头来看看后续的处理过程. 

###  QuorumPeer.run 

分别来看看 case 为Follower 和Leading  会做什么事情呢? 

```java
 @Override
    public void run() {
        setName("QuorumPeer" + "[myid=" + getId() + "]" +
                cnxnFactory.getLocalAddress());

        LOG.debug("Starting quorum peer");
        try {
            jmxQuorumBean = new QuorumBean(this);
            MBeanRegistry.getInstance().register(jmxQuorumBean, null);
            for (QuorumServer s : getView().values()) {
                ZKMBeanInfo p;
                if (getId() == s.id) {
                    p = jmxLocalPeerBean = new LocalPeerBean(this);
                    try {
                        MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                    } catch (Exception e) {
                        LOG.warn("Failed to register with JMX", e);
                        jmxLocalPeerBean = null;
                    }
                } else {
                    p = new RemotePeerBean(s);
                    try {
                        MBeanRegistry.getInstance().register(p, jmxQuorumBean);
                    } catch (Exception e) {
                        LOG.warn("Failed to register with JMX", e);
                    }
                }
            }
        } catch (Exception e) {
            LOG.warn("Failed to register with JMX", e);
            jmxQuorumBean = null;
        }

        try {
            /*
             * Main loop
             */
            // 根据选举状态, 选择不同的处理方式
            while (running) {
                switch (getPeerState()) {
                    case LOOKING:
                        LOG.info("LOOKING");

                        // 判断是否为只读模式, 通过readonlymode.enabled 开启
                        if (Boolean.getBoolean("readonlymode.enabled")) {
                            LOG.info("Attempting to start ReadOnlyZooKeeperServer");

                            // 只读模式的启动流程
                            // Create read-only server but don't start it immediately
                            final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(
                                    logFactory, this,
                                    new ZooKeeperServer.BasicDataTreeBuilder(),
                                    this.zkDb);

                            // Instead of starting roZk immediately, wait some grace
                            // period before we decide we're partitioned.
                            //
                            // Thread is used here because otherwise it would require
                            // changes in each of election strategy classes which is
                            // unnecessary code coupling.
                            Thread roZkMgr = new Thread() {
                                public void run() {
                                    try {
                                        // lower-bound grace period to 2 secs
                                        sleep(Math.max(2000, tickTime));
                                        if (ServerState.LOOKING.equals(getPeerState())) {
                                            roZk.startup();
                                        }
                                    } catch (InterruptedException e) {
                                        LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
                                    } catch (Exception e) {
                                        LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
                                    }
                                }
                            };
                            try {
                                roZkMgr.start();
                                setBCVote(null);
                                // 设置当前的投票, 通过策略模式来决定当前用哪个选举算法来进行领导选举
                                setCurrentVote(makeLEStrategy().lookForLeader());
                            } catch (Exception e) {
                                LOG.warn("Unexpected exception", e);
                                setPeerState(ServerState.LOOKING);
                            } finally {
                                // If the thread is in the the grace period, interrupt
                                // to come out of waiting.
                                roZkMgr.interrupt();
                                roZk.shutdown();
                            }
                        } else {
                            try {
                                setBCVote(null);
                                setCurrentVote(makeLEStrategy().lookForLeader());
                            } catch (Exception e) {
                                LOG.warn("Unexpected exception", e);
                                setPeerState(ServerState.LOOKING);
                            }
                        }
                        break;
                        ...
                            
                }
```

###   makeFollower 

初始化一个 Follower  对象, 构建一个 FollowerZookeeperServer ,表示follower 节点的请求处理服务

```java
    protected Follower makeFollower(FileTxnSnapLog logFactory) throws IOException {
        return new Follower(this, new FollowerZooKeeperServer(logFactory,
                this, new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
    }

```



###   follower.followLeader();  

```java
 void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {}", electionTimeTaken);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
            // 根据sid 找到对应的leader, 拿到lead 连接信息
            QuorumServer leaderServer = findLeader();
            try {
                // 连接到leader
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                // 将Follower 的zxid和myid 等信息封装好发送到leader, 同步epoch
                // 也就是意味着接下来 follower节点 只同步新的epoch 的数据信息
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                // 如果 leader 的epoch 比当前follower 节点的epoch 还小, 抛异常
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid) + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                // 和leader 进行数据同步
                syncWithLeader(newEpochZxid);
                QuorumPacket qp = new QuorumPacket();
                // 接受 leader 消息, 执行并反馈给leader, 线程在此自旋
                while (this.isRunning()) {
                    // 从 leader 读取数据包
                    readPacket(qp);
                    // 处理packet
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when following the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }

                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX((Learner) this);
        }
    }
```



###   makeLeader 

初始化一个Leader 对象, 构建一个 LeaderZookeeperServer , 用于标识leader 节点的请求处理服务

```java
 protected Leader makeLeader(FileTxnSnapLog logFactory) throws IOException {
        return new Leader(this, new LeaderZooKeeperServer(logFactory,
                this, new ZooKeeperServer.BasicDataTreeBuilder(), this.zkDb));
    }
```



###   leader.lead();

在Leader 端, 则通过lead() 来处理与follower 的交互.

