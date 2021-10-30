#  JVM之实战

##  1. JVM参数

### 1.1  标准参数

```bash
-version
-help
-server
-cp
```

![image-20200508202553014](http://files.luyanan.com//img/20200508202554.png)

### 1.2 `-X`参数

非标准参数,也就是在`JDK` 各个版本中可能会变动. 

```bash
-Xint 解释执行
-Xcomp 第一次使用就编译成本地代码
-Xmixed 混合模式，JVM自己来决定
```

![image-20200508205342125](http://files.luyanan.com//img/20200508205343.png)



### 1.3 `-XX` 参数

使用的最多的类型

非标准化参数,相对不稳定,主要用于JVM调优和`Debug`

- `Boolean` 类型

   格式: `-XX:[+-]`    +或者-表示启用或者禁用name属性

   比如`-XX:+UseConcMarkSweepGC`  表示启用`CMS` 类型的垃圾回收器

  ​        `-XX:+UseG1GC`     表示启用`G1` 类型的垃圾回收器

- 非`Boolean` 类型

   格式: `-XX<name>=<value>` 表示`name` 属性的值是`value`

  比如: `-XX:MaxGCPauseMillis=500`



### 1.4 其他参数

```bash
-Xms1000等价于-XX:InitialHeapSize=1000
-Xmx1000等价于-XX:MaxHeapSize=1000
-Xss100等价于-XX:ThreadStackSize=100
```

所以这块也相当于是`-XX` 类型的参数



### 1.5 查看参数

```bash
java -XX:+PrintFlagsFinal -version > flags.txt
```

![image-20200508210748393](http://files.luyanan.com//img/20200508210749.png)

![image-20200508210827009](http://files.luyanan.com//img/20200508210828.png)

值得注意的是"=" 表示默认值,":=" 表示被用户或JVM修改后的值

要想查看某个进程具体参数的值,可以使用`jinfo`

一般要设置参数, 可以先查看一下当前参数是什么?  然后进行修改 . 



### 1.6  设置参数的方式

- 开发工具中设置比如`IDEA`、`eclipse`
- 运行`jar`包的时候: `java -XX:+UseG1GC xxx.jar`
- `web`容器比如`tomcat`, 可以在脚本中进行设置
- 通过`jinfo` 实时调整某个进程的参数(参数只有被标记为`manageable`的`flags` 可以被实时修改)

### 1.7 实践和单位换算

```bash
1Byte(字节)=8bit(位)
1KB=1024Byte(字节)
1MB=1024KB
1GB=1024MB
1TB=1024GB
(1)设置堆内存大小和参数打印
-Xmx100M -Xms100M -XX:+PrintFlagsFinal
(2)查询+PrintFlagsFinal的值
:=true
(3)查询堆内存大小MaxHeapSize
:= 104857600
(4)换算
104857600(Byte)/1024=102400(KB)
102400(KB)/1024=100(MB)
(5)结论
104857600是字节单位
```



###  1.8 常用参数含义

| 参数                                                         | 含义                                                         | 说明                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| `-XX:CICompilerCount=3`                                      | 最大并行编译数                                               | 如果设置大于1,虽然编译速度会提高,但是同样影响系统稳定性, 会增加JVM崩溃的可能性 |
| `-XX:InitialHeapSize=100M`                                   | 初始化堆大小                                                 | 简写`-Xms100M`                                               |
| `-XX:MaxHeapSize=100M`                                       | 最大堆大小                                                   | 简写`-Xmx100M`                                               |
| `-XX:NewSize=20M`                                            | 设置年轻代的大小                                             |                                                              |
| `-XX:MaxNewSize=50M`                                         | 年轻代最大大小                                               |                                                              |
| `-XX:OldSize=50M`                                            | 设置老年代大小                                               |                                                              |
| `-XX:MetaspaceSize=50M`                                      | 设置方法区大小                                               |                                                              |
| `-XX:MaxMetaspaceSize=50M`                                   | 方法区最大大小                                               |                                                              |
| `-XX:+UseParallelGC`                                         | 使用`UseParallelGC`                                          | 新生代,吞吐量优先                                            |
| `-XX:+UseParallelOldGC`                                      | 使用`UseParallelOldGC`                                       | 老年代,吞吐量优先                                            |
| `-XX:+UseConcMarkSweepGC`                                    | 使用`CMS`                                                    | 老年代,停顿时间优先                                          |
| `-XX:+UseG1GC`                                               | 使用`G1GC`                                                   | 新生代、老年代,停顿时间优先                                  |
| `XX:NewRatio`                                                | 新老年代比例                                                 | 比如`-XX:Ratio=4`，则表示新生代:老年代=1:4，也就是新 生代占整个堆内存的1/5 |
| `-XX:SurvivorRatio`                                          | 两个S区和`Eden`区的比值`                                     | 比如`-XX:SurvivorRatio=8`，也就是`(S0+S1):Eden=2:8`， 也就是一个S占整个新生代的1/10 |
| `-XX:+HeapDumpOnOutOfMemoryErro`                             | 启动堆内存溢出打印                                           | 当JVM堆内存发生溢出的时候,也就是OOM, 自动生成`dump`文件      |
| `-XX:HeapDumpPath=heap.hprof`                                | 指定堆内存溢出打印目录                                       | 表示在当前目录生成一个`heap.hpreof` 文件                     |
| `XX:+PrintGCDetails - XX:+PrintGCTimeStamps - XX:+PrintGCDateStamps Xloggc:$CATALINA_HOME/logs/gc.log` | 打印出GC日志                                                 | 可以使用不同的垃圾收集器,对比查看GC 情况                     |
| `-Xss128k`                                                   | 设置每个线程的堆栈大小                                       | 经验值是3000-5000最佳                                        |
| `-XX:MaxTenuringThreshold=6`                                 | 提升老年代的最大临界值                                       | 默认值为15                                                   |
| `-XX:InitiatingHeapOccupancyPercent`                         | 启动并发`GC`周期时堆内存使用占比                             | G1之类的垃圾收集器用它来触发并发GC周期,基于整个堆的使用率,而不只是某一个内存的使用比,职位0, 则表示“一直处于GC循环”,默认值为45 |
| `-XX:G1HeapWastePercent`                                     | 允许的浪费堆空间占比                                         | 默认是10%, 如果并发标记可回收的空间小于10%, 则不会触发 `MixedGC` |
| `-XX:MaxGCPauseMillis=200ms`                                 | G1最大停顿时间                                               | 暂停时间不能太小,太小的话就会导致出现G1跟不上垃圾产生的速度,最终退化为`Full GC`, 所以对这个参数调优是一个持续的过程,逐步调整到最佳状态 |
| `-XX:ConcGCThreads=n`                                        | 并发垃圾收集器使用的线程数量                                 | 默认随着JVM运行的平台不同而不同                              |
| `-XX:G1MixedGCLiveThresholdPercent=65`                       | 混合垃圾回收周期中要包含的旧区域设置占用率阈值               | 默认占用率为65%                                              |
| `-XX:G1MixedGCCountTarget=8`                                 | 设置标记周期完成后,对存活数据上限为`G1MixedGCLIveThresholdPercent` 的旧区域执行混合垃圾回收的目标次数 | 默认8次混合垃圾回收, 混合回收的目标是要控制在此目标次数以内  |
| `- XX:G1OldCSetRegionThresholdPercent=1`                     | 描述`Mixed GC`时，`Old Region`被加入到 `CSet`中              | 默认情况下，G1只把10%的`Old Region`加入到`CSet`中            |



##  2. 常用命令

###  2.1 `jps`

查看java进程

> The jps command lists the instrumented Java HotSpot VMs on the target system. The command is limited to reporting information on JVMs for which it has the access permissions.

![image-20200508221323460](http://files.luyanan.com//img/20200508221324.png)



### 2.2 `jinfo`

1. 实时查看和调整JVM配置参数

   > The jinfo command prints Java configuration information for a specified Java process or core file or a remote debug server. The configuration information includes Java system properties and Java Virtual Machine (JVM) command-line flags.

2. 查看

   `jinfo -flag name PID` 查看某个java进程的name属性的值

   ![image-20200508221510893](http://files.luyanan.com//img/20200508221512.png)

3. 修改

    参数只有被标记为`manageable`的`flags` 可以被实时修改

   ```bash
   jinfo -flag [+|-] PID
   jinfo -flag = PID
   ```

4. 查看曾经赋过值的一些参数

   ```bash
   [root@localhost bin]# jinfo -flags  24668
   Attaching to process ID 24668, please wait...
   Debugger attached successfully.
   Server compiler detected.
   JVM version is 25.231-b11
   Non-default VM flags: -XX:CICompilerCount=2 -XX:InitialHeapSize=31457280 -XX:MaxHeapSize=482344960 -XX:MaxNewSize=160432128 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=10485760 -XX:OldSize=20971
   520 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC Command line:  -Djava.util.logging.config.file=/web/apache-tomcat-8.5.54/conf/logging.properties -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager -Djdk.tls.ephemeralDHKeySize
   =2048 -Djava.protocol.handler.pkgs=org.apache.catalina.webresources -Dorg.apache.catalina.security.SecurityListener.UMASK=0027 -Dignore.endorsed.dirs= -Dcatalina.base=/web/apache-tomcat-8.5.54 -Dcatalina.home=/web/apache-tomcat-8.5.54 -Djava.io.tmpdir=/web/apache-tomcat-8.5.54/temp
   ```

   

### 2.3 `jstat`

1. 查看虚拟机性能统计信心

   > The jstat command displays performance statistics for an instrumented Java HotSpot VM. The target JVM is identified by its virtual machine identifier, or vmid option.

2. 查看类装载信息

   ```bash
   jstat -class PID 1000 10  # 查看某个java进程的类装载信息,每1---毫秒输出一次, 共计输入10次
   ```

   ```bash
   [root@localhost bin]# jstat -class 24668 1000 10
   Loaded  Bytes  Unloaded  Bytes     Time   
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
     2542  5114.8        0     0.0       1.98
   
   ```

3. 查看垃圾收集信息

   ```bash
   jstat -gc PID 1000 10
   ```

   ```bash
   [root@localhost bin]# jstat -gc 24668 1000 10
    S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   1024.0 1024.0 1008.0  0.0   32768.0  20067.2   20480.0     9506.3   15360.0 14818.9 1792.0 1642.5      4    0.063   0      0.000    0.063
   
   ```






### 2.4 `jstack`

1. 查看线程信息

   > The jstack command prints Java stack traces of Java threads for a specified Java process, core file, or remote debug server.

2. 用法

   ```bash
   jstack PID
   ```

   ```bash
   [root@localhost bin]# jstack 1422
   2020-05-09 00:48:35
   Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.231-b11 mixed mode):
   
   "Attach Listener" #31 daemon prio=9 os_prio=0 tid=0x00007f170c001000 nid=0x5c6 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "http-nio-8080-AsyncTimeout" #29 daemon prio=5 os_prio=0 tid=0x00007f1740694000 nid=0x5ad waiting on condition [0x00007f17110b1000]
      java.lang.Thread.State: TIMED_WAITING (sleeping)
   	at java.lang.Thread.sleep(Native Method)
   	at org.apache.coyote.AbstractProtocol$AsyncTimeout.run(AbstractProtocol.java:1170)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-Acceptor-0" #28 daemon prio=5 os_prio=0 tid=0x00007f1740692000 nid=0x5ac runnable [0x00007f17111b2000]
      java.lang.Thread.State: RUNNABLE
   	at sun.nio.ch.ServerSocketChannelImpl.accept0(Native Method)
   	at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:422)
   	at sun.nio.ch.ServerSocketChannelImpl.accept(ServerSocketChannelImpl.java:250)
   	- locked <0x00000000e37f34d8> (a java.lang.Object)
   	at org.apache.tomcat.util.net.NioEndpoint$Acceptor.run(NioEndpoint.java:488)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-ClientPoller-1" #27 daemon prio=5 os_prio=0 tid=0x00007f1740675800 nid=0x5ab runnable [0x00007f17112b3000]
      java.lang.Thread.State: RUNNABLE
   	at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
   	at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
   	at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
   	at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
   	- locked <0x00000000f74cd4e0> (a sun.nio.ch.Util$3)
   	- locked <0x00000000f74cd4d0> (a java.util.Collections$UnmodifiableSet)
   	- locked <0x00000000f74cd3b8> (a sun.nio.ch.EPollSelectorImpl)
   	at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
   	at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:831)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-ClientPoller-0" #26 daemon prio=5 os_prio=0 tid=0x00007f1740674000 nid=0x5aa runnable [0x00007f17113b4000]
      java.lang.Thread.State: RUNNABLE
   	at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
   	at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:269)
   	at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:93)
   	at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:86)
   	- locked <0x00000000f74cbaf8> (a sun.nio.ch.Util$3)
   	- locked <0x00000000f74cbae8> (a java.util.Collections$UnmodifiableSet)
   	- locked <0x00000000f74cb9d0> (a sun.nio.ch.EPollSelectorImpl)
   	at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:97)
   	at org.apache.tomcat.util.net.NioEndpoint$Poller.run(NioEndpoint.java:831)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-10" #25 daemon prio=5 os_prio=0 tid=0x00007f1740671000 nid=0x5a9 waiting on condition [0x00007f17114b5000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-9" #24 daemon prio=5 os_prio=0 tid=0x00007f174066f000 nid=0x5a8 waiting on condition [0x00007f17115b6000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-8" #23 daemon prio=5 os_prio=0 tid=0x00007f174066d000 nid=0x5a7 waiting on condition [0x00007f17116b7000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-7" #22 daemon prio=5 os_prio=0 tid=0x00007f174066b000 nid=0x5a6 waiting on condition [0x00007f17117b8000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-6" #21 daemon prio=5 os_prio=0 tid=0x00007f1740669000 nid=0x5a5 waiting on condition [0x00007f17118b9000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-5" #20 daemon prio=5 os_prio=0 tid=0x00007f1740667000 nid=0x5a4 waiting on condition [0x00007f17119ba000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-4" #19 daemon prio=5 os_prio=0 tid=0x00007f1740665000 nid=0x5a3 waiting on condition [0x00007f1711abb000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   "http-nio-8080-exec-3" #18 daemon prio=5 os_prio=0 tid=0x00007f1740663000 nid=0x5a2 waiting on condition [0x00007f1728129000]
      java.lang.Thread.State: WAITING (parking)
   	at sun.misc.Unsafe.park(Native Method)
   	- parking to wait for  <0x00000000f744ce30> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
   	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
   	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:103)
   	at org.apache.tomcat.util.threads.TaskQueue.take(TaskQueue.java:31)
   	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
   	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
   	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
   	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
   	at java.lang.Thread.run(Thread.java:748)
   
   ....
   ```

3. 排查死锁案例

   `DeadLockDemo`

   ```java
   public class DeadLockDemo {
   
   
       public static void main(String[] args) {
           DeadLock lock1 = new DeadLock(true);
           DeadLock lock2 = new DeadLock(false);
           Thread thread1 = new Thread(lock1);
           Thread thread2 = new Thread(lock2);
           thread1.start();
           thread2.start();
       }
   }
   
   /**
    * <p>定义锁对象</p>
    *
    * @author luyanan
    * @since 2020/5/9
    */
   class MyLock {
   
       public static Object obj1 = new Object();
   
       public static Object obj2 = new Object();
   }
   
   
   /**
    * <p>死锁代码</p>
    *
    * @author luyanan
    * @since 2020/5/9
    */
   class DeadLock implements Runnable {
   
       private boolean flag;
   
   
       public DeadLock(boolean flag) {
           this.flag = flag;
       }
   
       @Override
       public void run() {
           if (flag) {
               while (true) {
                   synchronized (MyLock.obj1) {
                       System.out.println(Thread.currentThread().getName() + "----if获得obj1锁");
                       synchronized (MyLock.obj2) {
                           System.out.println(Thread.currentThread().getName() + "----if获得obj2锁");
                       }
                   }
   
               }
           } else {
               while (true) {
                   synchronized (MyLock.obj2) {
                       System.out.println(Thread.currentThread().getName() + "----否则获得obj2锁");
                       synchronized (MyLock.obj1) {
                           System.out.println(Thread.currentThread().getName() + "----否则获得obj1锁");
                       }
                   }
               }
           }
       }
   
   
   }
   
   ```

   运行结果:

   ```text
   Thread-0----if获得obj1锁
   Thread-1----否则获得obj2锁
   ```

4. `jstack`分析

   ```text
   PS C:\Program Files\Java\jdk1.8.0_151\bin> .\jps.exe
   12624 KotlinCompileDaemon
   5088 Launcher
   5200 RemoteMavenServer
   15080
   9480 Launcher
   17276 DeadLockDemo
   8124 Jps
   PS C:\Program Files\Java\jdk1.8.0_151\bin> .\jstack.exe 17276
   2020-05-09 09:12:28
   Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.151-b12 mixed mode):
   
   "DestroyJavaVM" #13 prio=5 os_prio=0 tid=0x0000000003784000 nid=0x7e8 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "Thread-1" #12 prio=5 os_prio=0 tid=0x000000001e58f000 nid=0x38c4 waiting for monitor entry [0x000000001efff000]
      java.lang.Thread.State: BLOCKED (on object monitor)
           at com.jvm.DeadLock.run(DeadLockDemo.java:67)
           - waiting to lock <0x000000076bc5b918> (a java.lang.Object)
           - locked <0x000000076bc5b928> (a java.lang.Object)
           at java.lang.Thread.run(Thread.java:748)
   
   "Thread-0" #11 prio=5 os_prio=0 tid=0x000000001e58e000 nid=0x3f90 waiting for monitor entry [0x000000001eeff000]
      java.lang.Thread.State: BLOCKED (on object monitor)
           at com.jvm.DeadLock.run(DeadLockDemo.java:57)
           - waiting to lock <0x000000076bc5b928> (a java.lang.Object)
           - locked <0x000000076bc5b918> (a java.lang.Object)
           at java.lang.Thread.run(Thread.java:748)
   
   "Service Thread" #10 daemon prio=9 os_prio=0 tid=0x000000001e4d6800 nid=0x387c runnable [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "C1 CompilerThread2" #9 daemon prio=9 os_prio=2 tid=0x000000001e42e000 nid=0x195c waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "C2 CompilerThread1" #8 daemon prio=9 os_prio=2 tid=0x000000001e404000 nid=0xd88 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "C2 CompilerThread0" #7 daemon prio=9 os_prio=2 tid=0x000000001e403800 nid=0x2384 waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "Monitor Ctrl-Break" #6 daemon prio=5 os_prio=0 tid=0x000000001e3e8800 nid=0xec0 runnable [0x000000001e8fe000]
      java.lang.Thread.State: RUNNABLE
           at java.net.SocketInputStream.socketRead0(Native Method)
           at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
           at java.net.SocketInputStream.read(SocketInputStream.java:171)
           at java.net.SocketInputStream.read(SocketInputStream.java:141)
           at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:284)
           at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:326)
           at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:178)
           - locked <0x000000076b8c9dc8> (a java.io.InputStreamReader)
           at java.io.InputStreamReader.read(InputStreamReader.java:184)
           at java.io.BufferedReader.fill(BufferedReader.java:161)
           at java.io.BufferedReader.readLine(BufferedReader.java:324)
           - locked <0x000000076b8c9dc8> (a java.io.InputStreamReader)
           at java.io.BufferedReader.readLine(BufferedReader.java:389)
           at com.intellij.rt.execution.application.AppMainV2$1.run(AppMainV2.java:64)
   
   "Attach Listener" #5 daemon prio=5 os_prio=2 tid=0x000000001ceac800 nid=0xafc waiting on condition [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "Signal Dispatcher" #4 daemon prio=9 os_prio=2 tid=0x000000001ce61800 nid=0xff0 runnable [0x0000000000000000]
      java.lang.Thread.State: RUNNABLE
   
   "Finalizer" #3 daemon prio=8 os_prio=1 tid=0x000000001ce39800 nid=0x4134 in Object.wait() [0x000000001e19e000]
      java.lang.Thread.State: WAITING (on object monitor)
           at java.lang.Object.wait(Native Method)
           - waiting on <0x000000076b608ec8> (a java.lang.ref.ReferenceQueue$Lock)
           at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:143)
           - locked <0x000000076b608ec8> (a java.lang.ref.ReferenceQueue$Lock)
           at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:164)
           at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:209)
   
   "Reference Handler" #2 daemon prio=10 os_prio=2 tid=0x0000000003874000 nid=0x3ad8 in Object.wait() [0x000000001e09f000]
      java.lang.Thread.State: WAITING (on object monitor)
           at java.lang.Object.wait(Native Method)
           - waiting on <0x000000076b606b68> (a java.lang.ref.Reference$Lock)
           at java.lang.Object.wait(Object.java:502)
           at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
           - locked <0x000000076b606b68> (a java.lang.ref.Reference$Lock)
           at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)
   
   "VM Thread" os_prio=2 tid=0x000000001ce16800 nid=0x71c runnable
   
   "GC task thread#0 (ParallelGC)" os_prio=0 tid=0x0000000003799800 nid=0x11f0 runnable
   
   "GC task thread#1 (ParallelGC)" os_prio=0 tid=0x000000000379b000 nid=0x1f28 runnable
   
   "GC task thread#2 (ParallelGC)" os_prio=0 tid=0x000000000379c800 nid=0x1eb8 runnable
   
   "GC task thread#3 (ParallelGC)" os_prio=0 tid=0x000000000379e800 nid=0x9bc runnable
   
   "VM Periodic Task Thread" os_prio=2 tid=0x000000001e52f800 nid=0x1b0 waiting on condition
   
   JNI global references: 33
   
   
   Found one Java-level deadlock:
   =============================
   "Thread-1":
     waiting to lock monitor 0x000000001e5932d8 (object 0x000000076bc5b918, a java.lang.Object),
     which is held by "Thread-0"
   "Thread-0":
     waiting to lock monitor 0x000000000387e8a8 (object 0x000000076bc5b928, a java.lang.Object),
     which is held by "Thread-1"
   
   Java stack information for the threads listed above:
   ===================================================
   "Thread-1":
           at com.jvm.DeadLock.run(DeadLockDemo.java:67)
           - waiting to lock <0x000000076bc5b918> (a java.lang.Object)
           - locked <0x000000076bc5b928> (a java.lang.Object)
           at java.lang.Thread.run(Thread.java:748)
   "Thread-0":
           at com.jvm.DeadLock.run(DeadLockDemo.java:57)
           - waiting to lock <0x000000076bc5b928> (a java.lang.Object)
           - locked <0x000000076bc5b918> (a java.lang.Object)
           at java.lang.Thread.run(Thread.java:748)
   
   Found 1 deadlock.
   ```







###  2.5 `jmap`

1. 生成堆转储快照

   > The jmap command prints shared object memory maps or heap memory details of a specified process, core file, or remote debug server

2. 打印出堆内存相关的信息

   ```bash
   -XX:+PrintFlagsFinal -Xms300M -Xmx300M
   jmap -heap PID
   ```

   ```text
   [root@localhost bin]# jps -l
   1500 sun.tools.jps.Jps
   1422 org.apache.catalina.startup.Bootstrap
   [root@localhost bin]# jmap -heap 1422
   Attaching to process ID 1422, please wait...
   Debugger attached successfully.
   Server compiler detected.
   JVM version is 25.231-b11
   
   using thread-local object allocation.
   Parallel GC with 2 thread(s)
   
   Heap Configuration:
      MinHeapFreeRatio         = 0
      MaxHeapFreeRatio         = 100
      MaxHeapSize              = 482344960 (460.0MB)
      NewSize                  = 10485760 (10.0MB)
      MaxNewSize               = 160432128 (153.0MB)
      OldSize                  = 20971520 (20.0MB)
      NewRatio                 = 2
      SurvivorRatio            = 8
      MetaspaceSize            = 21807104 (20.796875MB)
      CompressedClassSpaceSize = 1073741824 (1024.0MB)
      MaxMetaspaceSize         = 17592186044415 MB
      G1HeapRegionSize         = 0 (0.0MB)
   
   Heap Usage:
   PS Young Generation
   Eden Space:
      capacity = 33554432 (32.0MB)
      used     = 7793328 (7.4322967529296875MB)
      free     = 25761104 (24.567703247070312MB)
      23.225927352905273% used
   From Space:
      capacity = 3145728 (3.0MB)
      used     = 3073136 (2.9307708740234375MB)
      free     = 72592 (0.0692291259765625MB)
      97.69236246744792% used
   To Space:
      capacity = 3670016 (3.5MB)
      used     = 0 (0.0MB)
      free     = 3670016 (3.5MB)
      0.0% used
   PS Old Generation
      capacity = 20971520 (20.0MB)
      used     = 9777736 (9.324775695800781MB)
      free     = 11193784 (10.675224304199219MB)
      46.623878479003906% used
   
   13885 interned Strings occupying 1225968 bytes.
   
   ```

3. `dump`出堆内存相关的信息

   ```bash
   jmap -dump:format=b,file=heap.hprof PID
   ```

   ```bash
   [root@localhost bin]# jps -l
   1526 sun.tools.jps.Jps
   1422 org.apache.catalina.startup.Bootstrap
   [root@localhost bin]# jmap -dump:format=b,file=heap.hprof  1422
   Dumping heap to /root/apache-tomcat-8.5.54/bin/heap.hprof ...
   Heap dump file created
   ```

4. 要是在发生堆内存的时候, 能自动`dump` 出该文件就好了. 

    一般在开发中, `JVM`参数可以加上下面两句,这样内存溢出的时候, 会自动`dump` 该文件

   ````bash
   -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap.hprof
   ````

5. 关于`dump` 下来的文件

    一般`dump` 下来的文件,可以结合工具使用





## 3. 常用工具

###  3.1 `jconsole`

`jconsole` 工具是`JDK` 自带的可视化监控工具,查看`Java`应用程序的运行情况、监控堆信息、永久区的使用情况、类加载信息等. 

![image-20200509092425767](http://files.luyanan.com//img/20200509092439.png) 



### 3.2 ` jvisualvm`

#### 3.2.1 监控本地的java进程

可以监控本地的java进程的CPU、类、线程等. 



#### 3.2.2  监控远程的java进程

比如监控远端`tomcat, 演示部署在阿里云服务的	`tomcat`

1. 在` jvisualvm` 中选中远程,右击"添加"

2. 主机名上服务器的ip地址,然后点击确定

3. 右击该主机的ip地址,添加`JMX`[也就是通过`JMX`技术具体监控远程服务器哪个进程]

4. 要想让服务器的`tomcat`被连接,需要改一下`bin/catalina.sh`文件

    主要下面的`8998` 不要跟服务器上的其他端口冲突

   ```bash
   JAVA_OPTS="$JAVA_OPTS -Dcom.sun.management.jmxremote -
   Djava.rmi.server.hostname=31.100.39.63 -Dcom.sun.management.jmxremote.port=8998
   -Dcom.sun.management.jmxremote.ssl=false -
   Dcom.sun.management.jmxremote.authenticate=true -
   Dcom.sun.management.jmxremote.access.file=../conf/jmxremote.access -
   Dcom.sun.management.jmxremote.password.file=../conf/jmxremote.password"
   
   ```

5. 在`../conf` 目录下添加两个文件`jmxremote.access`和`jmxremote.password`

   `jmxremote.access` 文件

   ```properties
   guest readonly
   manager readwrite
   ```

   `jmxremote.password`

   ```text
   guest guest
   manager manager
   ```

   授予权限: `chmod 600 *jmxremot*`

6. 将连接服务器地址改为公网IP地址

   修改`hosts` 文件, 在最后面加上一行

   ```bash
   [内网ip地址]  [公网ip地址]
   ```

7. 设置上述端口对应的阿里云安全策略和防火墙策略

8. 启动`tomcat`, 来到`bin` 目录

   ```bash
   ./startup.sh
   ```

9. 查看`tomcat` 启动日志以及端口监听

   ```bash
   tail -f ../logs/catalina.out
   lsof -i tcp:8080
   ```

10. 插件8998监听情况

    ```bash
    lsof -i:8998 得到PID
    netstat -antup | grep PID
    ```

11. 在刚才的`JMX`中输入8998端口,并且输入用户名和密码则登录成功

    ```bash
    端口:8998
    用户名:manager
    密码:manager
    ```







### 3.3 `Arthas`

`github`: https://github.com/alibaba/arthas

> Arthas allows developers to troubleshoot production issues for Java applications without modifying code or restarting servers.

`Arthas` 是`ALibaba`开源的`java` 诊断工具,采用命令行交互模式，是排查`jvm`相关问题的利器

`Arthas` 是`Alibaba` 开源的java诊断工具,深受开发者喜爱. 

当你遇到以下类似问题而束手无策的时候,`Arthas` 可以帮你解决: 

1. 这个类是从哪个jar 包加载的,为什么会报各种类的相关`Exception`? 
2. 我改的代码为什么没有执行到? 难道是我没有`commit`? 分支搞错了? 
3. 遇到问题无法在线上`debug`,难道只能通过加日志重新发布吗? 
4. 线程遇到某个用户的数据处理有问题, 但线上同样无法`debug`,线下无法重现
5. 是否有一个全局视角来查看系统的运行情况? 
6. 有什么办法监控到`JVM`的实时运行状态? 
7. 怎么快速定位应用的热点,生成火焰图. 



`Arthas` 支持`JDK6+`, 支持`Linux/Windows/Mac`,采用命令行交互模式,同时提供丰富的`Tab`自动补全功能,进一步方便进行问题的定位和诊断. 

#### 3.3.1 下载安装

```bash
curl -O https://alibaba.github.io/arthas/arthas-boot.jar
java -jar arthas-boot.jar
# 然后可以选择一个Java进程
```

**Print usage**

```bash
java -jar arthas-boot.jar -h

```





#### 3.3.2 常用命令

> 具体每个命令怎么使用? 大家可以自行查询资源

```bash
version:查看arthas版本号
help:查看命名帮助信息
cls:清空屏幕
session:查看当前会话信息
quit:退出arthas客户端
---
dashboard:当前进程的实时数据面板
thread:当前JVM的线程堆栈信息
jvm:查看当前JVM的信息
sysprop:查看JVM的系统属性
---
sc:查看JVM已经加载的类信息
dump:dump已经加载类的byte code到特定目录
jad:反编译指定已加载类的源码
---
monitor:方法执行监控
watch:方法执行数据观测
trace:方法内部调用路径，并输出方法路径上的每个节点上耗时
stack:输出当前方法被调用的调用路径
......

```





### 3.4 `MAT`

java 堆分析器,用于查找内存漏洞

`Heap Dump`,称为堆转储文件,是java 进程在某个时间点的快照

`下载地址：` https://www.eclipse.org/mat/downloads.php



#### 3.4.1 `Dump`信息包含的内容

- All Objects

   Class, fields, primitive values and references

- All Classes

   Classloader, name, super class, static fields

- Garbage Collection Roots

  Objects defined to be reachable by the JVM

- Thread Stacks and Local Variables

  The call-stacks of threads at the moment of the snapshot, and per-frame information about local objects



#### 3.4.2  获取`Dump` 文件

- 手动

  ```bash
  jmap -dump:format=b,file=heap.hprof 44808
  
  ```

- 自动

  ```bash
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=heap.hprof
  
  ```





#### 3.4.3  使用

- `Histogram`

  `Histogram`  可以列出内存中的对象,对象的个数以及大小

  ```text
  Class Name:类名称，java类名
  Objects:类的对象的数量，这个对象被创建了多少个
  Shallow Heap:一个对象内存的消耗大小，不包含对其他对象的引用
  Retained Heap:是shallow Heap的总和，即该对象被GC之后所能回收到内存的总和
  ```

  >  右击类名--->List Objects--->with incoming references--->列出该类的实例

  > 右击Java对象名--->Merge Shortest Paths to GC Roots--->exclude all ...--->找到GC Root以及原因

- `Leak Suspects`

  查找并分析内存泄漏的可能原因

  > Reports--->Leak Suspects--->Details

  

- `Top Consumers`

   列出大对象



### 3.4  `GC` 日志分析工具

要想分析日志的信息,首先的拿到`GC` 日志文件才行, 所以要先配置一下

````bash
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps
-Xloggc:gc.log
````



- 在线的

  http://gceasy.io/

- `GCViewer`

  