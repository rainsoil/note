#  JVM之调优

## 1. 重新认识JVM

我们画一张图来展示JVM的大体物理结构

![image-20200511085019210](http://files.luyanan.com//img/20200511085020.png)



## 2. GC优化

内存被使用了以后, 难免会有不够用或者达到设定值的时候,就需要对内存空间进行垃圾回收. 

### 2.1 垃圾收集发生的时机

GC是由JVM 自动完成的,根据JVM系统环境而定,所以时机是不确定的. 当然, 我们也可以手动进行垃圾回收,比如调用`system.gc()` 方法通知JVM进行一次垃圾回收,但是具体什么时刻运行也无法控制. 也就是说,`System.gc()` 只是通知要回收,什么时候回收是由JVM控制的,但是不建议手动调用此方法,因为消耗的资源比较大. 

**一般以下几种情况会发生垃圾回收:**

1. 当`Eden`区或者`S`区不够用了
2. 老年代空间不够用了. 
3. 方法区空间不够用了
4. `System.gc()`



###  2.2  实验环境准备

我们这里准备一个`springboot` 项目,然后配置相应的参数

#### 2.3  GC日志文件

![image-20200511090135498](http://files.luyanan.com//img/20200511090136.png)

要想分析日志文件,得先拿到日志文件才行, 所以要先配置一下

```bash
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -Xloggc:gc.log
```

然后启动项目,然后看到默认使用的是 `ParallelGC`

####  2.3.1 `ParallelGC` 日志

> 吞吐量优先

```text
Java HotSpot(TM) 64-Bit Server VM (25.151-b12) for windows-amd64 JRE (1.8.0_151-b12), built on Sep  5 2017 19:33:46 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16636048k(7426340k free), swap 40753296k(25478464k free)
CommandLine flags: -XX:-BytecodeVerificationLocal -XX:-BytecodeVerificationRemote -XX:InitialHeapSize=266176768 -XX:+ManagementServer -XX:MaxHeapSize=4258828288 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:TieredStopAtLevel=1 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
2020-05-11T09:04:45.331+0800: 3.376: [GC (Allocation Failure) [PSYoungGen: 65024K[Young区回收前]->8314K[Young区回收后](75776K[Young区总大小])] 65024K[整合堆回收前]->8330K[整个堆回收后](249344K[堆总大小]), 0.0110320 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 


```

`注意:` 如果回收的差值中间有出入,则说明这部分空间是`Old` 空间释放出来的. 

#### 2.3.2 `CMS` 日志

> 停顿时间优先

```bash
参数设置：-XX:+UseConcMarkSweepGC -Xloggc:cms-gc.log
```





####  2.3.3  `G1`日志

> 停顿时间优先

参数设置 

```bash
-XX:+UseG1GC -Xloggc:g1-gc.log
```



```text
Java HotSpot(TM) 64-Bit Server VM (25.151-b12) for windows-amd64 JRE (1.8.0_151-b12), built on Sep  5 2017 19:33:46 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16636048k(7679220k free), swap 40753296k(25282952k free)
CommandLine flags: -XX:-BytecodeVerificationLocal -XX:-BytecodeVerificationRemote -XX:InitialHeapSize=266176768 -XX:+ManagementServer -XX:MaxHeapSize=4258828288 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:TieredStopAtLevel=1 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC #使用了G1垃圾收集器
-XX:-UseLargePagesIndividualAllocation 
# 什么时候发生的GC,相对的时间刻,GC发生的区域young,总共花费的是时间0.0035105s
2020-05-11T09:21:41.242+0800: 1.976: [GC pause (G1 Evacuation Pause) (young), 0.0035105 secs]
# 多少个垃圾回收线程,并行的时间
   [Parallel Time: 2.8 ms, GC Workers: 4]
   # GC线程开始对于上面的 1.976 的时间刻
      [GC Worker Start (ms): Min: 1975.7, Avg: 1975.7, Max: 1975.7, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.3, Avg: 0.7, Max: 1.5, Diff: 1.2, Sum: 2.7]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.3]
      [Object Copy (ms): Min: 1.2, Avg: 1.9, Max: 2.3, Diff: 1.1, Sum: 7.6]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 1.3, Max: 2, Diff: 1, Sum: 5]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.2]
      [GC Worker Total (ms): Min: 2.7, Avg: 2.7, Max: 2.7, Diff: 0.0, Sum: 10.8]
      [GC Worker End (ms): Min: 1978.4, Avg: 1978.4, Max: 1978.4, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 0.6 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.5 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 14.0M(14.0M)->0.0B(17.0M) Survivors: 0.0B->2048.0K Heap: 14.0M(254.0M)->2817.5K(254.0M)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-05-11T09:21:41.489+0800: 2.222: [GC pause (G1 Evacuation Pause) (young), 0.0047721 secs]
   [Parallel Time: 3.5 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 2222.1, Avg: 2222.1, Max: 2222.1, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.4, Max: 0.8, Diff: 0.5, Sum: 1.5]
      [Update RS (ms): Min: 0.5, Avg: 0.9, Max: 1.8, Diff: 1.3, Sum: 3.5]
         [Processed Buffers: Min: 1, Avg: 1.3, Max: 2, Diff: 1, Sum: 5]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.1, Max: 0.3, Diff: 0.3, Sum: 0.3]
      [Object Copy (ms): Min: 1.4, Avg: 2.1, Max: 2.5, Diff: 1.1, Sum: 8.5]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 7.3, Max: 13, Diff: 12, Sum: 29]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 3.5, Avg: 3.5, Max: 3.5, Diff: 0.0, Sum: 13.9]
      [GC Worker End (ms): Min: 2225.6, Avg: 2225.6, Max: 2225.6, Diff: 0.0]
   [Code Root Fixup: 0.1 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 1.1 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 1.0 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 17.0M(17.0M)->0.0B(149.0M) Survivors: 2048.0K->3072.0K Heap: 19.8M(254.0M)->6462.5K(254.0M)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-05-11T09:21:42.162+0800: 2.895: [GC pause (Metadata GC Threshold) (young) (initial-mark), 0.0097849 secs]
   [Parallel Time: 4.7 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 2895.2, Avg: 2895.2, Max: 2895.2, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.7, Avg: 1.0, Max: 1.4, Diff: 0.7, Sum: 4.0]
      [Update RS (ms): Min: 0.0, Avg: 0.3, Max: 0.4, Diff: 0.4, Sum: 1.0]
         [Processed Buffers: Min: 0, Avg: 2.8, Max: 6, Diff: 6, Sum: 11]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.2]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 0.4, Diff: 0.4, Sum: 0.6]
      [Object Copy (ms): Min: 2.8, Avg: 3.2, Max: 3.5, Diff: 0.7, Sum: 12.7]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 2.5, Max: 4, Diff: 3, Sum: 10]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 4.6, Avg: 4.6, Max: 4.7, Diff: 0.0, Sum: 18.6]
      [GC Worker End (ms): Min: 2899.9, Avg: 2899.9, Max: 2899.9, Diff: 0.0]
   [Code Root Fixup: 0.1 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 5.0 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 4.7 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.1 ms]
   [Eden: 89.0M(149.0M)->0.0B(138.0M) Survivors: 3072.0K->14.0M Heap: 94.8M(254.0M)->16.8M(254.0M)]
 [Times: user=0.00 sys=0.00, real=0.01 secs] 
2020-05-11T09:21:42.172+0800: 2.905: [GC concurrent-root-region-scan-start]
2020-05-11T09:21:42.178+0800: 2.912: [GC concurrent-root-region-scan-end, 0.0068436 secs]
2020-05-11T09:21:42.178+0800: 2.912: [GC concurrent-mark-start]
2020-05-11T09:21:42.183+0800: 2.916: [GC concurrent-mark-end, 0.0043306 secs]
2020-05-11T09:21:42.183+0800: 2.917: [GC remark 2020-05-11T09:21:42.183+0800: 2.917: [Finalize Marking, 0.0001185 secs] 2020-05-11T09:21:42.183+0800: 2.917: [GC ref-proc, 0.0002940 secs] 2020-05-11T09:21:42.184+0800: 2.917: [Unloading, 0.0019141 secs], 0.0024715 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-05-11T09:21:42.186+0800: 2.919: [GC cleanup 17M->17M(254M), 0.0006070 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 

```



### 2.4 GC日志分析工具

#### 2.4.1 `gceasy`

> 官网: https://gceasy.io

可以比较不同的垃圾回收器的吞吐量和停顿时间

比如打开前面生成的 `cms-gc.log`和`g1-gc.log`

![image-20200511101225972](http://files.luyanan.com//img/20200511101227.png)

![image-20200511101254302](http://files.luyanan.com//img/20200511101255.png)





#### 2.4.2 `GCViewer`

![image-20200511101536127](http://files.luyanan.com//img/20200511101538.png)





### 2.5 G1 调优指南

### 2.5.1 调优

是否选用G1垃圾收集器的判断依据

> https://docs.oracle.com/javase/8/docs/technotes/guides/vm/G1.html#use_cases

1. 50% 以上的堆被存活对象占用
2. 对象分配和晋升的速度变化比较大
3. 垃圾回收时间比较长

`思考:` https://blogs.oracle.com/poonam/increased-heap-usage-with-g1-gc

1. 使用`G1GC`垃圾收集器: `-XX:+UseG1GC`

   修改配置参数,获取到gc日志,使用`GCViewer`分析吞吐量和响应时间

   ```text
   Throughput       Min Pause       Max Pause      Avg Pause       GC count
     99.16%         0.00016s         0.0137s        0.00559s          12 
   ```

2. 调整内存大小再获取gc日志分析

   ```bash
   -XX:MetaspaceSize=100M 
   -Xms300M
   -Xmx300M
   ```

   比如设置堆内存的大小,获取到gc日志,使用`GCViewer` 分析吞吐量和响应时间

   ```text
   Throughput       Min Pause       Max Pause      Avg Pause       GC count
     98.89%          0.00021s        0.01531s       0.00538s           12 
   ```

3. 调整最大的停顿时间

   ```bash
   -XX:MaxGCPauseMillis=20    设置最大GC停顿时间指标
   ```

   比如设置最大的停顿时间, 获取到gc日志,然后分析吞吐量和停顿时间

   ```text
   Throughput       Min Pause       Max Pause      Avg Pause       GC count
     98.96%          0.00015s        0.01737s       0.00574s          12 
   ```

4. 启动并发GC时堆内存占分比

   ```text
   -XX:InitiatingHeapOccupancyPercent=45 G1用它来触发并发GC周期,基于整个堆的使用率,而不只是某一代内存的 使用比例。值为 0 则表示“一直执行GC循环)'. 默认值为 45 (例如, 全部的 45% 或者使用了45%).
   ```

   比如设置该百分比参数,获取到gc日志,分析吞吐量和响应时间

   ```text
   Throughput       Min Pause       Max Pause      Avg Pause       GC count
     98.11%          0.00406s        0.00532s       0.00469s          12 
   ```

   





####  2.5.2  最佳指南

`官网建议`: https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#recommendations

1.  不要手动设置新生代和老年代的大小,只要设置整个堆的大小

   G1收集器在运行过程中,会自己调整新生代和老年代的大小

   其实是通过`adapt` 代的大小来调整对象晋升的速度和年龄,从而达到为收集器设置的暂停时间目标. 

   如果手动设置了大小就意味着放弃了G1的自动调优

2. 不断调优暂停时间目标

   一般情况下这个值设置到100ms或者200ms 都是可以的(不同情况下会不一样), 但是如果设置成50ms 就不太合理, 暂停时间设置的太短的话，就会导致出现G1跟不上垃圾产生的速度,最终退化为`Full GC`, 所以对这个参数的调优是一个持续的过程, 逐步调整到最佳状态,暂停时间只是一个目标,总不能总是得到满足. 

3. 使用`-XX:ConcGCThreads=n` 来增加标记线程的数量

    `IHOP`如果阈值设置过高, 可能就会遇到转移失败的风险,比如对象进行转移时空间不足. 如果阈值设置过低,就会时标记周期运行过于频繁,并且有可能混合收集期回收不到空间. 

4. `MixedGC` 调优

   ```bash
   -XX:InitiatingHeapOccupancyPercent 
   -XX:G1MixedGCLiveThresholdPercent
   -XX:G1MixedGCCountTarger
   -XX:G1OldCSetRegionThresholdPercent
   ```

5. 适当增加堆内存大小





## 3. 高并发场景分析

> 以每秒3000笔订单为例

![image-20200511104857175](http://files.luyanan.com//img/20200511104859.png)



## 4. JVM 性能优化指南

> 1.  发现问题
>    - GC频繁
>    - 死锁
>    - OOM
>    - 线程池不够用
>    - CPU负载过高
> 2. 排查问题
>    - 打印出gc日志, 查看`minor gc/major gc`,结合工具`gcviewer`
>    - `jstack` 查看线程堆栈信息
>    - `dump`  出堆文件,使用`MAT` 或其他工具分析
>    - 合理使用`jdk`自带的`jconsole`，`jvisualvm`、`阿里的arthas` 等实时查看JVM状态
>    - 灵活应用`jps` 、`jinfo`、`jstat`、`jmap` 等常用命令
> 3. 解决方案
>    - 适当增加内存大小/选择合适的垃圾收集器
>    - 使用ZK/redis 实现分布式锁
>    - 设置本地、`nginx` 等缓存减少对后端服务器的访问
>    - 后端代码优化及时释放资源/合理设置线程池中的参数
>    - 集群部署从而减少单节点的压力
>    - 利用一些消息中间件比如`MQ` 、`Kafka` 等实现异步消息
>    - .....





## 5.  常见问题思考



###  5.1 内存泄漏和内存溢出的区别

**内存泄漏**: 对象无法得到及时的回收持续占用内存的对象,从而造成内存空间的浪费

**内存溢出**: 内存泄漏到一定的程序就会造成内存溢出, 但是内存溢出也可能是大对象导致的.

### 5.2 `young gc` 会有`stw` 吗? 

不管什么GC, 都会有`stop the world`,只是发生时间的长短



###  5.3 `major gc` 和`full gc`的区别

`major gc` 是指老年代的gc,而`full gc` 是指新生到+老年代+方法区的gc



### 5.4 G1和CMS的区别

CMS 用于老年代的回收,而G1 用于新生代和老年代的回收

G1是用了`Region` 方法对堆内存进行了划分,且基于标记整理算法实现, 整体减少了垃圾碎片的产生



### 5.5 什么是直接内存

直接内存是在java堆外，直接向系统申请的内存空间,通常访问直接内存的速度会优于java堆,因此处于性能的考虑, 读写频繁的场合可能会考虑到使用直接内存

### 5.6 不可达的对象一定要被回收吗? 

即使在可达性分析法中不可达的对象,也并非是"非死不可"的, 这时候他们暂时处于"缓刑阶段", 要真正宣告一个对象死亡,至少要经历两次标记过程,可达性分析法中不可达的对象被第一次标记并且进行一次筛选,筛选的条件是此对象是否有必要执行`ﬁnalize ` 方法,当对象没有覆盖`ﬁnalize ` 方法, 或者`ﬁnalize ` 方法已经被虚拟机调用过时, 虚拟机将这两种情况视为没有必要执行. 

被判定为需要执行的对象将会被放在一个队列中进行第二次标记,除非这个对象与引用链上的任何一个对象建立关联, 否则就会被真的回收. 



### 5.7 方法区中的无用类回收

方法区主要回收的是无用的类,那么如何判断一个类是无用的类呢？

判断一个常量是否是"废弃常量" 比较简单,而要判定一个类是否是"无用的类" 的条件则相对要苛刻的多,类需要同时满足下面3个条件才能算"无用的类"

- 该类所有的实例都已经被回收了,也就是java堆中不存在该类的任何实例
- 加载该类的`ClassLoader` 已经被回收
- 该类对应的`java.lang.Class` 对象没有在任何地方被引用,无法在任何地方通过反射该类的方法. 

虚拟机可以对满足上述3个条件的无用类进行回收,这里说的仅仅是"可以", 而并不是跟对象一样不使用了就必然会被回收. 



### 5.7 不同的引用

JDK12以后, Java对引用进行了扩充: `强引用`、`软引用`、`弱引用`和`虚引用`





