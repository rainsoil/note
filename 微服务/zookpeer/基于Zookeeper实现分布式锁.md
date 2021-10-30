---
date: 2019年11月11日10点45分
---



#  基于Zookeeper 实现分布式锁

##  分布式锁的基本场景

如果在多线程并行情况下去访问某一个共享资源,比如说共享变量, 那么势必会造成线程安全问题. 那么我们可以用很多种方法来解决, 比如synchronized、比如Lock 之类的锁操作来解决线程问题, 那么在分布式架构下,涉及到多个进程访问某一个共享资源的情况下, 比如说在电商平台中商品库存问题, 在库存中只有10个的情况下进来100个用户, 如果能够避免超卖呢? 所以这个时候就要一个互斥的手段来防止彼此之间的干扰. 

然后在分布式情况下, synchronized 或者Lock 之类的锁只能控制单一进程的资源访问, 在多进程架构下, 这些API就没办法解决我们的问题了, 怎么办呢? 

## 用Zookeeper 来实现分布式锁

我们可以利用zoookeeper 的特性来实现独占锁, 就是同级节点的唯一性,多个进程往zookeeper的指定节点下创建一个相同名称的节点, 只有一个能成功, 另外一个是创建失败. 创建失败的节点全部通过zookeeper的watcher 机制来监听zookeeper 这个子节点的变化, 一旦监听到子节点的删除事件, 则再次触发所有进程去写锁. 

![](http://files.luyanan.com//img/20191111113744.png)

这种实现方式很简单, 但是会产生"惊群效应",简单来说, 就是如果存在修多的客户端在等待获取锁, 当成功获取到锁的进程释放该节点后, 所有处于等待状态的客户端都会被唤醒, 这个时候zookeeper 在短时间内发送大量子节点变更事件给所有待获取锁的客户端, 然后实际情况是只会有一个客户端获取锁. 如果在集群规模比较大的情况下, 会对zookeeper 服务器的性能产生比较大的影响. 

## 利用有序节点来实现分布式锁

我们可以通过有序节点来实现分布式锁, 每个客户端都往指定的节点下注册一个临时有序节点, 越早创建的节点, 节点的顺序编号就越小, 那么我们可以判断子节点中最小的节点设置为获取锁. 如果自己的节点不是所有子节点中最小的, 意味着还没有获取锁. 这个锁的实现和前面单节点的实现的差异在于, 每个节点只需要监听比自己小的节点, 当比自己小的节点删除以后, 客户端会受到watcher 事件, 此时再次判断自己的节点是不是所有子节点中最小的, 如果是则获取锁, 否则就不断重复这个过程, 这样就不会导致"惊群效应",因为每个客户端只需要监听一个节点. 

## curator 分布式锁的基本使用

curator 对于锁这块做了一些封装, curator 提供了 InterProcessMutex  这样一个API。除了分布式锁之外, 还提供了Leader 选举,、分布式队列等常用的功能. 

-  InterProcessMutex：分布式可重入排它锁 

-  InterProcessSemaphoreMutex：分布式排它锁 

-  InterProcessReadWriteLock：分布式读写锁 



```java
package com.zk.demo;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessLock;
import org.apache.curator.framework.recipes.locks.InterProcessMultiLock;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

/**
 * @author luyanan
 * @since 2019/11/11
 * <p>分布式锁</p>
 **/
public class LockDemo {

    public static void main(String[] args) {


        String connect = "192.168.91.128:2181";

        CuratorFramework framework = createClient(connect);

        framework.start();


        final InterProcessMutex lock = new InterProcessMutex(framework, "/locks");

        for (int i = 0; i < 10; i++) {
            new Thread(() -> {

                System.out.println(Thread.currentThread().getName() + ": 尝试获得锁");
                try {
                    lock.acquire();
                    System.out.println(Thread.currentThread().getName() + ": 已经获得锁");
                } catch (Exception e) {
                    e.printStackTrace();
                }

                try {
                    Thread.sleep(4000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                try {
                    lock.release();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + ": 释放锁");

            }, "thread-" + i).start();
        }
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
}

```

##  curator 实现分布式锁的基本原理

### 构造函数

```java
   public InterProcessMutex(CuratorFramework client, String path) {
      //  Zookeeper 利用path 创建临时顺序节点, 实现公平锁的核心
       this(client, path, new StandardLockInternalsDriver());
    }

 public InterProcessMutex(CuratorFramework client, String path, LockInternalsDriver driver) {
      // maxLeases  =1 表示可以获取分布式锁的线程数量(跨JVM)为1, 即为互斥锁
     this(client, path, "lock-", 1, driver);
    }
  InterProcessMutex(CuratorFramework client, String path, String lockName, int maxLeases, LockInternalsDriver driver) {
        this.threadData = Maps.newConcurrentMap();
        this.basePath = PathUtils.validatePath(path);
     
      // internals 的类型为 LockInternals, InterProcessMutex 将分布式锁的申请和释放操作委托给 internals 执行
      this.internals = new LockInternals(client, driver, path, lockName, maxLeases);
    }
```

###  InterProcessMutex.acquire();   获取锁

```java
// 无限等待 
public void acquire() throws Exception {
        if (!this.internalLock(-1L, (TimeUnit)null)) {
            throw new IOException("Lost connection while trying to acquire lock: " + this.basePath);
        }
    }
// 限时等待
 public boolean acquire(long time, TimeUnit unit) throws Exception {
        return this.internalLock(time, unit);
    }
```

#### InterProcessMutex.internalLock();

```java
 private boolean internalLock(long time, TimeUnit unit) throws Exception {
        Thread currentThread = Thread.currentThread();
        InterProcessMutex.LockData lockData = (InterProcessMutex.LockData)this.threadData.get(currentThread);
        if (lockData != null) {
            // 实现可重入
            // 同一个线程再次 acquire, 首先判断当前的映射表内(threadData) 是否有该线程的锁信息, 如果有, 则原子+1, 然后返回. 
            lockData.lockCount.incrementAndGet();
            return true;
        } else {
            // 映射表中没有对应的锁信息, 尝试通过LockInternals 获取锁. 
            String lockPath = this.internals.attemptLock(time, unit, this.getLockNodeBytes());
            if (lockPath != null) {
                // 成功获取锁, 记录信息到映射表
                InterProcessMutex.LockData newLockData = new InterProcessMutex.LockData(currentThread, lockPath);
                this.threadData.put(currentThread, newLockData);
                return true;
            } else {
                return false;
            }
        }
    }

// 映射表
// 记录线程与锁信息的映射关系
    private final ConcurrentMap<Thread, InterProcessMutex.LockData> threadData;

// 锁信息
// Zookeeper 中一个临时顺序节点对应一个锁, 但让锁生效激活需要排队(公平锁),下面会继续分析
   private static class LockData {
        final Thread owningThread;
        final String lockPath;
       // 分布式锁重入次数
        final AtomicInteger lockCount;

        private LockData(Thread owningThread, String lockPath) {
            this.lockCount = new AtomicInteger(1);
            this.owningThread = owningThread;
            this.lockPath = lockPath;
        }
    }
```

#### InterProcessMutex.attemptLock

```java
// 尝试获取锁, 并返回锁对应的zookeeper 临时顺序节点的路径 
String attemptLock(long time, TimeUnit unit, byte[] lockNodeBytes) throws Exception {
    
        long startMillis = System.currentTimeMillis();
    // 无限等待时, millisToWait 为null
        Long millisToWait = unit != null ? unit.toMillis(time) : null;
    // 创建ZNode 节点时的数据内容, 无关紧要, 这里为null, 采用默认值(ip地址)
        byte[] localLockNodeBytes = this.revocable.get() != null ? new byte[0] : lockNodeBytes;
    // 当前已经重试次数, 与    CuratorFramework的重试策略有关. 
    int retryCount = 0;
    // 在zookeeper 中创建的临时顺序节点的路径相当于一把待激活的分布式锁.
    // 激活条件: 同级目录下子节点, 名字排序最小(排队、公平锁) 
        String ourPath = null;
    // 是否已经持有分布式锁. 
        boolean hasTheLock = false;
    // 是否已经完成了尝试获取分布式锁的操作 
        boolean isDone = false;

        while(!isDone) {
            isDone = true;

            try {
                // 从InterProcessMutex 的构造函数可知实际driver 为StandardLockInternalsDriver 的实例. 
                // 在zookeeper 中创建顺序临时节点. 
                ourPath = this.driver.createsTheLock(this.client, this.path, localLockNodeBytes);
                // 循环等待来激活分布式锁, 实现锁的公平性
                hasTheLock = this.internalLockLoop(startMillis, millisToWait, ourPath);
            } catch (NoNodeException var14) {
                // 容错处理
                // 因为绘会话过期等原因,StandardLockInternalsDriver 因为无法找到创建的临时顺序节点而抛出NoNodeException 异常. 
                if ( client.getZookeeperClient().getRetryPolicy().allowRetry(retryCount++, System.currentTimeMillis() - startMillis, RetryLoop.getDefaultRetrySleeper()) )
                {
                    // 满足重试策略尝试重新获取锁
                    isDone = false;
                }
                else
                {
                    // 不满足重试策略则继续抛出 NoNodeException 异常. 
                    throw e;
                }
            }
        }

        if ( hasTheLock )
        {
            // 成功获取分布式锁, 返回临时顺序节点的路径, 上层将其封装成锁信息记录在映射表中, 方便锁重入
            return ourPath;
        }

    // 获取分布式锁失败, 返回null
        return null;
    }

```



####  createsTheLock

```java
   @Override
// 在zookeeper中创建临时顺序节点
    public String createsTheLock(CuratorFramework client, String path, byte[] lockNodeBytes) throws Exception
    {
        String ourPath;
        if ( lockNodeBytes != null )
        {
            // lockNodeBytes 不为null 则作为数据节点内容, 否则采用默认内容(ip地址)
            // creatingParentContainersIfNeeded: 用于创建父节点, 如果不支持 CreateMode.CONTAINER
            // 那么将采用 CreateMode.PERSISTENT
            // withProtection: 临时子节点会添加 GUID 前缀
            ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path, lockNodeBytes);
            // CreateMode.EPHEMERAL_SEQUENTIAL: 临时顺序节点, zookeeper 能保证在节点产生的顺序性.
            // 依据顺序来激活分布式锁, 从而实现了分布式锁的公平性,
        }
        else
        {
            ourPath = client.create().creatingParentContainersIfNeeded().withProtection().withMode(CreateMode.EPHEMERAL_SEQUENTIAL).forPath(path);
        }
        return ourPath;
    }
```

####  LockInternals.internalLockLoop

```java
// 循环等待来激活分布式锁,实现锁的公平性.  
private boolean internalLockLoop(long startMillis, Long millisToWait, String ourPath) throws Exception
    {
    // 是否已经持有分布式锁
        boolean     haveTheLock = false;
    // 是否需要删除子节点
        boolean     doDelete = false;
        try
        {
            if ( revocable.get() != null )
            {
                client.getData().usingWatcher(revocableWatcher).forPath(ourPath);
            }

            while ( (client.getState() == CuratorFrameworkState.STARTED) && !haveTheLock )
            {
                // 获取排序后的子节点的顺序
                List<String>        children = getSortedChildren();
                // 获取前面自己创建的临时顺序子节点的名称
                String              sequenceNodeName = ourPath.substring(basePath.length() + 1); // +1 to include the slash

                // 实现锁的公平性的核心逻辑. 
                PredicateResults    predicateResults = driver.getsTheLock(client, children, sequenceNodeName, maxLeases);
                if ( predicateResults.getsTheLock() )
                {
                    // 获取了锁, 中断循环, 继续返回上层
                    haveTheLock = true;
                }
                else
                {
                    // 没有获取锁, 监听上一临时顺序节点
                    String  previousSequencePath = basePath + "/" + predicateResults.getPathToWatch();

                    synchronized(this)
                    {
                        try 
                        {
                            // use getData() instead of exists() to avoid leaving unneeded watchers which is a type of resource leak
                            // exists  会导致资源泄露, 因此exists() 可以监听不存在的ZNode, 因为采用 getData
                            //  上一临时顺序节点如果被删除, 会唤醒当前线程继续竞争锁,正常情况下能直接获得锁, 因为锁是公平的. 
                            
                            client.getData().usingWatcher(watcher).forPath(previousSequencePath);
                            if ( millisToWait != null )
                            {
                                millisToWait -= (System.currentTimeMillis() - startMillis);
                                startMillis = System.currentTimeMillis();
                                if ( millisToWait <= 0 )
                                {
                                    // 获取锁超时, 标记删除之前创建的临时顺序节点
                                    doDelete = true;    // timed out - delete our node
                                    break;
                                }

                                // 等待被唤醒, 限时等待
                                wait(millisToWait);
                            }
                            else
                            {
                                // 等待被唤醒, 无限等待
                                wait();
                            }
                        }
                        catch ( KeeperException.NoNodeException e ) 
                        {
                            // 容错处理, 
                            // client.getData()  可能调用时抛出 NoNodeException，原因可能是锁被释放或会话过期(连接丢失)等. 
                            // 这里并没有做任何处理, 因为外层是while 循环, 再次执行driver.getsTheLock 会调用 validateOurIndex
                            // 此时会抛出NoNodeException, 从而进入下面的catch 和finally 逻辑, 重新抛出上层尝试重试获取锁并删除临时顺序节点
                            // it has been deleted (i.e. lock released). Try to acquire again
                        }
                    }
                }
            }
        }
        catch ( Exception e )
        {
            ThreadUtils.checkInterrupted(e);
            // 标记删除, 在finally 删除之前创建的临时顺序节点(后台不断尝试)
            doDelete = true;
            //  重新抛出,尝试重新获取锁. 
            throw e;
        }
        finally
        {
            if ( doDelete )
            {
                deleteOurPath(ourPath);
            }
        }
        return haveTheLock;
    }
```

####   getsTheLock

```java
 @Override
    public PredicateResults getsTheLock(CuratorFramework client, List<String> children, String sequenceNodeName, int maxLeases) throws Exception
    {
        // 之前创建的临时顺序节点在排序后的子节点中的索引
        int             ourIndex = children.indexOf(sequenceNodeName);
        // 校验之前创建的临时顺寻节点是否有效
        validateOurIndex(sequenceNodeName, ourIndex);

        // 锁公平性的核心逻辑
        // 由InterProcessMutex 的构造函数可知,maxLeases 为1, 即只有ourIndex 为0时, 线程才能持有锁, 或者说该线程创建的临时顺序节点激活了锁.
        // Zookeeper 的临时顺序节点特性能够保证跨多个JVM 的线程并发创建节点的顺序性, 越早创建临时顺序节点成功的线程会更早的激活锁或获取锁. 
       
        boolean         getsTheLock = ourIndex < maxLeases;
       // 如果 已经获取到锁, 则无需监听任何节点, 否则需要监听上一个顺序节点(ourIndex -1)
        // 因为锁是公平的, 因为无需监听除了(ourIndex -1) 以外 的所有节点, 这时为了减少羊群效应, 非常巧妙的涉及. 
        String          pathToWatch = getsTheLock ? null : children.get(ourIndex - maxLeases);

        // 返回获取锁的结果, 交由上层继续处理(添加监听等操作)
        return new PredicateResults(pathToWatch, getsTheLock);
    }


   static void validateOurIndex(String sequenceNodeName, int ourIndex) throws KeeperException
    {
        if ( ourIndex < 0 )
        {
            // 容错处理
            // 由于会话过期或连接丢失等原因, 该线程创建的临时顺序节点被zookee 服务端删除, 往外抛出NoNodeException
            // 如果在重试策略允许范围内, 则进行重新禅师获取锁, 这会重新生成临时顺序节点
           
            throw new KeeperException.NoNodeException("Sequential path not found: " + sequenceNodeName);
        }
    }
```



###  释放锁  InterProcessMutex.release();

```java
 @Override
    public void release() throws Exception
    {
        /*
            Note on concurrency: a given lockData instance
            can be only acted on by a single thread so locking isn't necessary
         */

        Thread currentThread = Thread.currentThread();
        LockData lockData = threadData.get(currentThread);
        if ( lockData == null )
        {
            //  无法从映射表中获取锁信息, 不持有锁. 
            throw new IllegalMonitorStateException("You do not own the lock: " + basePath);
        }

        int newLockCount = lockData.lockCount.decrementAndGet();
        if ( newLockCount > 0 )
        {
             // 锁是可重入的, 初始值为1 原子为-1 到0, 锁才释放. 
            return;
        }
        if ( newLockCount < 0 )
        {
           // 理论上无法执行该路径
            throw new IllegalMonitorStateException("Lock count has gone negative for lock: " + basePath);
        }
        try
        {
            //  lockData != null && newLockCount ==
0，释放锁资源
            internals.releaseLock(lockData.lockPath);
        }
        finally
        {
            // 最后从映射表中移除当前线程的锁信息. 
            threadData.remove(currentThread);
        }
    }
```

####  LockInternals.releaseLock 

```java
   final void releaseLock(String lockPath) throws Exception
    {
        client.removeWatchers();
        revocable.set(null);
       // 删除临时顺序节点, 只会触发后一顺序节点去获取锁, 理论上不存在竞争, 只排队, 非抢占, 公平锁.先到先得
        deleteOurPath(lockPath);
    }


  private void deleteOurPath(String ourPath) throws Exception
    {
        try
        {
            // 后台不断尝试删除.
            client.delete().guaranteed().forPath(ourPath);
        }
        catch ( KeeperException.NoNodeException e )
        {
            // 已经删除(可能会话过期到期),不做处理. 
            // ignore - already deleted (possibly expired session, etc.)
        }
    }
```

