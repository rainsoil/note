

--------------------
# 6. ConcurrentHashMap的源码分析

## ConcurrentHashMap的初步使用以及场景

#### CHM的使用
ConcurrentHashMap是J.U.C包里面提供的一个线程安全并且高效的HashMap,所以ConcurrentHashMap在并发场景中使用的效率比较高
#### api 的使用
ConcurrentHashMap 是Map的派生类,所以api基本上和HashMap是类似的,只要就是put、get这些方法,接下来基于ConcurrentHashMap 的put和get方法作为切入点分析ConcurrentHashMap的源码实现
## ConcurrentHashMap的源码分析
先要做一个声明,本文中分析的ConcurrentHashMap是基于JDK1.8版本的
#### jdk1.7和jdk1.8版本的变化
ConcurrentHashMap和HashMap的实现源码差不多,但是因为ConcurrentHashMap需要支持并发操作,所以实现上要比hashMap稍微复杂一些.

在JDK1.7的实现上,ConcurrentHashMap 由一个个Segment 组成,简单来说,ConcurrerntHashMap 是一个Segment 数组,它通过继承ReentrantLock 来进行加锁,通过每次锁住一个segment 来保证每个segment 内的操作的线程安全性从而实现全局线程安全.

整个结构图如下：

![](http://files.luyanan.com//img/20190813095858.jpg)

当我们操作分布在不同的segment上的时候,默认情况下,理论上可以同时支持16个线程的并发写入.

**相比于1.7版本,他做了两个改进**

1. 取消了segment分段设计,直接使用Node数组来保存数据,并且采用Node数组元素作为锁来实现每一行数据进行加锁来进一步减少并发冲突的概率
2. 将原本数组+单向链表的数据结构变更为了数组+单向链表+红黑树的结构.为什么要引入红黑树呢?在正常情况下,key hash 之后如果能够很均匀的分散在数组中,那么table 数组中的每个队里偶尔的长度主要为0，或者1. 但是实际情况下,还是会存在一些队列长度多长的情况下. 如果还采用单向列表的方式,那么查询某个节点的时间复杂度就变为了O(N);因此对于队列长度超过8的列表,JDK1.8采用了红黑树的结构,那么查询的时间复杂度就会降低到O(logN),可以提升查找的性能.


![](http://files.luyanan.com//img/20190813105503.png)

这个结构和JDK1.8中的HashMap的数据结构基本一致,但是为了保证线程安全性,COncurrentHashMap的实现会稍微复杂一下.接下来我们从源码层面来了解一下他的原理.我们基于put和get方法来分析它的实现即可.

### put方法第一阶段

```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }


  final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 计算hash值
        int hash = spread(key.hashCode());
        // 用来记录链表的长度
        int binCount = 0;
        // 这里其实就是个自旋,当出现线程竞争时不断自旋
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 如果数组为空,则进行数组初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            // 通过hash值对应的数组下表得到第一个节点,以volatile 读的方式来读取table数组中的元素,保证每次拿到的数据都是最新的
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果该下表返回的节点为空,则直接通过CAS 将新的值封装成Node插入即可,如果CAS失败,说明存在竞争,进入下一次循环,
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

  假如在上面的这段代码中存在两个线程,在不加锁的情况下: 线程A 成功执行 casTabAt操作后,随后的线程B可以通过tabAt 方法立即看到 table[i] 的改变. 原因如下: 线程A的casTabAt 操作,具有 volatile 读写相同的内存语义,根据volatile 的happens-before 规则: 线程A 的casTabAt 操作一定对线程B的tabAt 操作可见.

#### initTable

数组初始化方法,这个方法比较简单,就是初始化一个合适大小的数组,sizeCtl 这个要单独说一下,如果没有搞懂这个属性的意义,可能会被搞晕,这个标志是在Node数组初始化或者扩容的时候的一个控制位标识,负数代表正在进行初始化或者扩容操作.

-1 代表正在初始化

-N 代表有N-1 个线程正在进行扩容操作, 这里不是简单的理解成n个线程,sizeCtl 就是-N,这块在后续讲扩容的时候会说明.

0 表示Node数组还没有被初始化,整数代表初始化或者下一次扩容的大小.

```java
  private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 被其他线程抢占了初始化操作,则直接让出自己的CPU 时间片
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 通过CAS操作,将sizeCrl 替换为-1, 表示当前线程抢占到了初始化资格
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // 默认的初始容量为16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                                // 初始化数组,长度为16,或者初始化在构造concurrentHashMap的时候传入的长度
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        // 将这个数组赋值给table
                        table = tab = nt;
                        // 计算下次扩容的大小,实际就是当前容量的0.075倍,这里使用了右移操作.
                        sc = n - (n >>> 2);
                    }
                } finally {
                    //  设置sizeCtl 为sc,如果默认是16的话,那么这个时候sc  = 16*0.75 = 12
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }


```



####  tabAt

该方法获取对象中offset偏移地址对应的对象field的值.实际上这段代码的含义等价于tab[i], 但是为什么不直接使用tab[i] 来计算呢?

getObjectVolatile ,一旦看到 volatile关键字,就表示可见性. 因为对 volatile 写操作happens-before 于vloatile 读操作,因此其他线程对table 的修改对get 读取可见.

虽然table 数组本身是增加了volatile 属性,但是"volatile的数组只针对数组的引用具有volatile的语义,而不是它的元素".所以如果有其他的线程对这个数组的元素进行写操作,那么当前线程来读的时候不一定能读取到最新的值.

出于性能考虑,Doug Lea 直接通过Unsafe 类对table 进行操作

```java
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

```

######  图解分析

![](http://files.luyanan.com//img/20190813145909.png)

###  put方法第二阶段

在putVal方法执行完成以后,会通过addCount 来增加ConcurrentHashMap中的元素个数,并且还会可能触发扩容操作,这里会有两个非常经典的设计

1. 高并发下的扩容

2. 如何保证addCount 的数据安全性以及性能

   ```java
   
           //  将当前的concurrentHashMap 的元素数量加1,有可能触发 transfer操作(扩容)
           addCount(1L, binCount);
           return null;
       }
   ```

   

#### addCount

   在putVal 最后调用了addCount 的时候,传递了两个参数,分别是1和bigCount(链表长度),看看 addCount 方法里面做了什么操作呢?

   x表示这次需要在表中增加的元素个数,check参数表示是否需要进行扩容检查,大于等于0 都需要进行检查

   ```java
   private final void addCount(long x, int check) {
           CounterCell[] as; long b, s;
           // 判断 counterCells 是否为空,
           // 1. 如果为空,就通过CAS 操作尝试修改baseCount 变量,对这个变量进行原子累加操作(做这个操作的意义是: 如果在没有竞争的情况下
           // 仍然采用baseCount 来记录元素的个数)
           // 2. 如果CAS失败则说明存在竞争,这个时候不能采用baseCount 来累加,而是用过counterCells 来记录
           if ((as = counterCells) != null ||
               !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
               CounterCell a; long v; int m;
               // 是否冲突标识,默认没有冲突.
               // 这里有几个判断
               // 1. 计数表为空则直接调用 fullAddCount
               // 2. 从计数表汇总随机取出一个数组的位置为空,则直接调用 fullAddCount
               // 3. 通过CAS 修改CountCell随机位置的值,如果修改失败说明出现并发情况(这里又用到了一种巧妙的办法),调用 fullAddCount
               // Random 在线程并发的时候会有性能问题以及可能会产生相同的随机数,ThreadLocalRandom.getProbe,可以解决这个问题,并且性能要比Random 高
               boolean uncontended = true;
               if (as == null || (m = as.length - 1) < 0 ||
                   (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                   !(uncontended =
                     U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                   // 执行 fullAddCount 方法
                   fullAddCount(x, uncontended);
                   return;
               }
               // 链表长度小于等于1,不需要考虑扩容
               if (check <= 1)
                   return;
               // 统计concurrentHashMap 元素个数
               s = sumCount();
           }
           if (check >= 0) {
               Node<K,V>[] tab, nt; int n, sc;
               while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                      (n = tab.length) < MAXIMUM_CAPACITY) {
                   int rs = resizeStamp(n);
                   if (sc < 0) {
                       if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                           sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                           transferIndex <= 0)
                           break;
                       if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                           transfer(tab, nt);
                   }
                   else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                                (rs << RESIZE_STAMP_SHIFT) + 2))
                       transfer(tab, null);
                   s = sumCount();
               }
           }
       }
   ```



#### CounterCells 解释

ConcurrentHashMap 是采用CounterCells 数组来记录元素个数的,像一般的集合记录记录集合大小,直接定义一个size 成员变量就行了，当出现改变的时候只要更新这个变量就行了.为什么ConcurrentHashMap 要用这种形式来处理呢?

问题还是出在并发上,ConcurrentHashMap是并发集合,如果用一个成员变量来统计元素的个数的话,为了保证并发情况下共享变量的安全,势必会需要通过加锁或者自旋的方式来实现,如果竞争比较激烈的情况下,size 的设置上会出现比较大的冲突反而影响了性能,所以在ConcurrentHaashMap 采用了分片的方法来记录大小,具体什么意思,我们来分析下

```java
    // 标识当前cell数组是否在初始化或者扩容中的CAS 标识位
    private transient volatile int cellsBusy;

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    // counterCells 数组,总数值的分数分别存在每个cell中
    private transient volatile CounterCell[] counterCells;

   @sun.misc.Contended static final class CounterCell {
        volatile long value;
        CounterCell(long x) { value = x; }
    }

    // 看到这段代码就能明白了,CountCell 数组的每个元素都存储一个元素个数,而实际上我们调用size方法就是通过这个循环累加来得到的
    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

#### fullAddCount 源码分析

fullAddCount  主要是用来初始化 CountCell来记录元素个数的,里面包含了扩容,初始化等操作

```java
  // See LongAdder version for explanation
    private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        // 获得当前线程的probe的值,如果值为0,则初始化当前线程的probe的值,probe就是随机数
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
            // 由于重新生成了probe, 未冲突标志位设置为true
            wasUncontended = true;
        }
        boolean collide = false;
        // True if last slot nonempty
        // 自旋
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
            //  说明countCells 已经被初始化过了
            if ((as = counterCells) != null && (n = as.length) > 0) {
                // 通过该值于当前线程probe 求与,获得cells的下标元素,和hash表获得索引是一样的
                if ((a = as[(n - 1) & h]) == null) {
                    // cellsBusy  = 0 表示countCells 不在初始化或者扩容状态下
                    if (cellsBusy == 0) {            // Try to attach new Cell
                        // 构造一个CounterCell 的值,传入元素个数
                        CounterCell r = new CounterCell(x); // Optimistic create
                        // 通过cas 设置cellsBusy 标识,防止其他线程来对countCells 并发处理
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;

                                // 将初始化的r对象的元素个数放在对应下标的位置
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                // 恢复标识位
                                cellsBusy = 0;
                            }
                            // 如果创建成功,退出循环
                            if (created)
                                break;
                            // 说明指定cells 下标位置的数据不为空,则进行下一次循环
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                // 说明在 addCount 方法中cas失败了,并且获取probe 的值不为空
                else if (!wasUncontended)       // CAS already known to fail

                    // 设置为未冲突标识,进入下一次自旋
                    // 由于指定下标位置的cell值不为空,则直接通过cas进行原子累加,如果成功,则直接退出
                    wasUncontended = true;      // Continue after rehash
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
                // 如果已经有其他线程建立了新的countCells或者counterCells 大于CPU核心数(很巧妙,线程的并发数不会超过CPU核心数)
                else if (counterCells != as || n >= NCPU)
                    // 设置当前线程的循环失败不进行扩容
                    collide = false;            // At max size or stale
                // 恢复 collide  状态,标识下次循环会进行扩容
                else if (!collide)
                    // 进入这个步骤,说米用countCells 数组容量不够,线程竞争比较大,所以先设置一个表示为正在扩容
                    collide = true;

                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    try {
                        if (counterCells == as) {// Expand table unless stale
                            // 扩容一倍 2变成4, 这个扩容比较简单
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        //  恢复标识
                        cellsBusy = 0;
                    }
                    collide = false;
                    // 继续下一次自旋
                    continue;                   // Retry with expanded table
                }
                // 更新随机数的值
                h = ThreadLocalRandom.advanceProbe(h);
            }
            // 初始化 CountCells 数组
            // cellsBusy = 0 表示没有在做初始化,通过CAS 更新cellsBusy 的值标注当前线程正在做初始化操作
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                        // 初始化容量为2
                        CounterCell[] rs = new CounterCell[2];
                       // 将x  也就是元素的个数放在指定的数组下标位置
                        rs[h & 1] = new CounterCell(x);
                        // 赋值给 counterCells
                        counterCells = rs;
                        // 设置初始化完成标识
                        init = true;
                    }
                } finally {
                    // 恢复标识
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            // 竞争激烈,其他线程占据cell数组,直接累加在base变量
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base
        }
    }
```

#### CounterCells 初始化图解

初始化长度为2的数组,然后随机得到一个指定的数组下标,将需要新增的值加入到对应的下标位置处

![](http://files.luyanan.com//img/20190813170838.jpg)

#### transfer  扩容阶段

判断是否需要扩容,也就是当更新后的键值对总数baseCount >= 阈值 sizeCtl时,进行rehash, 这里面会有两个逻辑

1. 如果当前正在处于扩容阶段,则当前线程会加入并且协助扩容
2. 如果当前没有在扩容,则直接出发扩容操作

```java
   // 如果binCount >=0,标识需要检查扩容
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            // s 标识集合大小,如果集合的长度大于或者等于扩容阈值(默认是0.75) 并且table不为空并且table的长度小于最大容量
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                // 这里是生成一个唯一的扩容戳
                int rs = resizeStamp(n);
                // sc< 0 , 也就是 sizeCtl < 0 说明已经有别的线程正在进行扩容了
                if (sc < 0) {
                    // 这5个条件只有有一个条件为true, 说明当前线程不能帮助进行此次的扩容了,直接跳出循环
                    // (sc >>> RESIZE_STAMP_SHIFT) !=  表示比较高   RESIZE_STAMP_SHIFT 位生成戳和rs 是否相等,相同
                    // sc == rs + 1  表示扩容结束
                    // sc == rs + MAX_RESIZERS  表示帮助线程线程已经达到最大值了
                    // nt = nextTable  表示扩容已经结束了
                    //  transferIndex <= 0  表示所有的 transfer 任务已经被领取完了,没有剩余的hash桶来给自己的这个线程做 transfer
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    // 当前线程尝试帮助此次扩容,如果成功,则调用transfer
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                // 如果当前没有在扩容，那么rs肯定是一个正数,通过(rs << RESIZE_STAMP_SHIFT)  将rs 设置为一个负数,+2 表示有一个线程在执行扩容
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                // 重新计数,判断是否需要开启下一轮扩容
                s = sumCount();
            }
        }
    }
```

#### resizeStamp

这块的逻辑理解起来,也有一点复杂

resizeStamp 用来生成一个和扩容有关的扩容戳,具体有什么用呢? 我们基于他的实现来做一个分析

```java
static final int resizeStamp(int n) {    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));}
```

Integer.numberOfLeadingZeros(n) 这个方法是返回无符号整数n 最高位非0位前面的0的个数

比如10的二进制是  0000 0000 0000 0000 0000 0000 0000 1010 

那么这个方法返回的值就是28

根据resizeStamp 的运算逻辑,我们来推演一下,加入n=16, 那么resizeStamp(16) = 32796,转换为二进制是 [0000 0000 0000 0000 1000 0000 0001 1100]

接下来来看,当第一个线程尝试进行扩容的时候,会执行下面这段代码

```java
U.compareAndSwapInt(this, SIZECTL, sc,                             (rs << RESIZE_STAMP_SHIFT) + 2)
```

rs 左移16位,相当于原本的二进制低位变成了高位  1000 0000 0001 1100 0000 0000 0000 0000 

然后再+2 = 1000 0000 0001 1100 0000 0000 0000 0000 +10 = 1000 0000 0001 1100 0000 0000 0000 0010

高16位代表扩容的标记,低16位代表并行扩容的线程数

| 高RESIZE_STAMP_BITS位 | 低 RESIZE_STAMP_SHIFT 位 |
| :-------------------: | ------------------------ |
|       扩容标记        | 并行扩容线程数           |

- 这样来存储有什么好处？
  1. 首先在CHM中是支持并发扩容的,也就是说如果当前的数组是需要进行扩容操作,可以由多个线程来共同负责,
  2. 可以保证每次扩容都生成唯一的时间戳,每次新的扩容,都有一个不同的n,这个生成的戳就是根据n来计算出来的一个数字,n不同,这个数字也不同.

- 第一次线程尝试扩容的时候,为什么是+2

  因为1 表示初始化,2 表示一个线程正在扩容,而且对sizeCtl 的操作都是基于位运算的,所以不会关心它本身的数值是多,只关心它在二进制上的数值,而sc +1 会在低16位上+1 .
  
  
  
#### transfer

 扩容是ConcurrentHashMap 的精华之一,扩容操作的核心在于数组的转移,在单线程环境下数据的转移很简单,无非就是把旧数组中的数据迁移到新的数组. 但是这种多线程环境下,在扩容的时候其他线程也可能正在添加元素,这时又触发了扩容怎么办? 可能大家想到的第一个解决方案就是加互斥锁,把转移过程锁住,虽然是可行的方案,但是会带来较大的性能开销. 因为互斥锁会导致所有访问临界区的线程陷入到阻塞状态,持有锁的线程耗时越长,其他竞争线程就会一直被阻塞,导致吞吐量较低.而且还可能导致死锁.

而ConcurentHashMap 并没有直接加锁,而是采用CAS实现无锁的并发同步策略,最精华的部分就是它可以利用多线程来进行协同扩容.

简单来说, 它把Node数组当作多个线程之间共享的任务队列,然后通过维护一个指针来划分每个线程锁负责的区间,每个线程通过区间逆向便利来实现扩容,一个已经迁移完的blucket 会被替换为一个ForwardingNode节点,标记当前bucket 已经被其他线程迁移完了,接下来分析一下它的源码:

1. fwd:这个类是个标识类,用于指向新表用的,其他线程遇到这个类会主动跳过这个类,因为这个类要么就是扩容迁移正在进行,要么就是已经完成扩容迁移,也就是这个类要保证线程安全,再进行操作.
2. advance: 这个变量是用于提示代码是否进行推进处理的,也就是当前桶处理完,处理下一个桶的标识.
3. finishing: 这个变量用于提示扩容是否结束用的.

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 将(n>>>3 相当于 n/8) 然后除以CPU核心数,如果得到的结果小于16,那么就使用16
        // 这里的目的是让每个CPU处理的桶一样多,避免处理转移任务不均匀的现象,如果桶较少的话,默认一个CPU(一个线程)处理16个桶
        // 这就是长度为16的时候,扩容的时候只会有一个线程来扩容.
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // nextTab 未初始化,nextTab是用来扩容的node数组
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                // 新建一个 n<<1 原始table 大小的nextTab ,也就是32
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                // 赋值给nextTab
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                // 扩容失败,sizeCtl就是用int的最大值
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            // 更新成员变量
            nextTable = nextTab;
            // 更新转移下标,表示转移时的下标
            transferIndex = n;
        }
        // 新的tab的长度
        int nextn = nextTab.length;
        // 创建一个fwd 节点,表示一个正在被迁移的Node,并且它的hash值为-1(MOVED), 也就是前面我们在讲putval方法的时候,会有一个判断 MOVED的逻辑.它的作用是
        // 用来占位的,表示原数组中位置i处的节点完成迁移以后,就会在i位置设置一个fwd 来告诉其他线程这个位置已经处理过了
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // 首次推进为true, 如果等于true,说明需要再次推进一个下标(i--),反之,如果是false, 那么就不能推进下标,需要将当前的下标处理完毕后才能继续推进
        boolean advance = true;
        // 判断是否已经扩容完成,完成就return, 退出循环
        boolean finishing = false; // to ensure sweep before committing nextTab
        // 通过for 自循环处理每个槽位中的链表元素,默认为advace为真,通过CAS设置
        // transferIndex 属性值,并初始化i和 bound的值,i指当前处理的槽位序号,
       //  bound 指需要处理的槽位边界,先处理槽位15的节点
        for (int i = 0, bound = 0;;) {
            // 这个循环使用CAS 不断的尝试为当前线程分配任务
            // 直到分配成功或者任务队列已经被全部分配完毕
            // 如果当前线程已经被分配过bucket 区域
            // 那么会通过i-- 指向下一个待处理的bucket 然后退出循环
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                // --i 表示下一个待处理的 bucket , 如果他 >=bound, 表示当前线程已经分配过 bucket 区域
                if (--i >= bound || finishing)
                    advance = false;
                // 表示所有的bucket 已经被分配完毕了
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                // 通过CAS 修改TRANSFERINDEX 为当前线程分配任务,处理的节点区间为 (nextBound,nextIndex)->(0,15)
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound; //0
                    i = nextIndex - 1;// 15
                    advance = false;
                }
            }
            //  i<0  说明已经完成遍历完旧的数组,也就是当前线程已经处理完所有的负债的  bucket
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 如果完成了扩容
                if (finishing) {
                    // 删除成员变量
                    nextTable = null;
                    // 更新table 数组
                    table = nextTab;
                    // 更新阈值(32*0.75 = 24)
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // sizeCtl  在迁移前会设置为( (rs << RESIZE_STAMP_SHIFT) + 2)
                // 然后,每增加一个线程参会迁移都会对sizeCtl 加1
                // 这里使用 CAS 操作对sizeCtl 的低16位进行减1,代表做完了属于自己的任务
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 第一个扩容的线程,执行 transfer 方法之前,会设置 sizeCtl  = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                    // 后续帮其扩容的线程执行 transfer 方法之前,会设置 sizeCtl = sizeCtl+1
                    // 每一个退出transfer 的方法的线程,退出之前会设置 sizeCtl = sizeCtl-1,那么最后一个线程退出时:必然有
                    //sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2)== resizeStamp(n) << RESIZE_STAMP_SHIFT
                    // 如果sc -2 不等于标识符 左移16位.如果他们相等了,说明没有线程在帮助他们扩容了,也就是说,扩容结束了,
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 如果相等,扩容结束了,更新finishing的值
                    finishing = advance = true;
                    // 再次循环检查一下整张表
                    i = n; // recheck before commit
                }
            }
            // 如果位置i处是空,没有任何节点,那么放入刚刚初始化的 ForwardingNode"空节点"
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 表示该位置已经完成了迁移,也就是如果线程A 已经处理过了这个节点,那么线程B处理这个节点时,hash值一定为MOVED
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

####  扩容过程图解

ConcurrentHashMap 支持并发扩容,实现方式,把Node数组进行拆分,让每个线程处理自己的区域,假设table的数组总长度为64,默认情况下,那么每个线程可以分到16个bucket. 然后每个线程处理的范围,按照倒序来做迁移,

通过for 自循环处理每个槽位中的链表元素,默认advace 为真,通过CAS 设置transferIndex 属性值,并初始化i和bound的值,i指当前处理的槽位序号,bound指需要处理的槽位边界,先处理槽位31的节点 (bound,i) = (16,31),从31的位置往前推动,

![](http://files.luyanan.com//img/20190815095535.jpg)

假设这个时候ThreadA在进行transfer ,那么逻辑图表示如下:

![](http://files.luyanan.com//img/20190815095920.png)

在当前假设情况下,槽位15中没有节点,则通过CAS 插入到第二步的初始化的 ForwardingNode节点,用于告诉其他线程该槽位已经处理过了.

![](http://files.luyanan.com//img/20190815101414.png)

#### sizeCtl 扩容退出机制

在扩容操作 transfer 的第2414行, 代码如下:

```java
 if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 第一个扩容的线程,执行 transfer 方法之前,会设置 sizeCtl  = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                    // 后续帮其扩容的线程执行 transfer 方法之前,会设置 sizeCtl = sizeCtl+1
                    // 每一个退出transfer 的方法的线程,退出之前会设置 sizeCtl = sizeCtl-1,那么最后一个线程退出时:必然有
                    //sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2)== resizeStamp(n) << RESIZE_STAMP_SHIFT
                    // 如果sc -2 不等于标识符 左移16位.如果他们相等了,说明没有线程在帮助他们扩容了,也就是说,扩容结束了,
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 如果相等,扩容结束了,更新finishing的值
                    finishing = advance = true;
                    // 再次循环检查一下整张表
                    i = n; // recheck before commit
                }
```

每存在一个线程执行完扩容操作,就通过CAS 执行sc -1

接着判断(sc-2) != resizeStamp(n)<< RESIZE_STAMP_SHIFT;  如果相等, 表示当前为整个扩容操作的最后一个线程,那么意味着整个扩容操作就结束了; 如果不相等,说明还得继续. 这么做的目的,一方面是防止不同扩容之间出现相同的sizeCtl, 另一方面,还可以避免sizeCtl的ABA问题 导致的扩容重叠问题.

#### 数据迁移阶段的实现分析

通过分配好迁移的区间之后,开始对数据进行迁移

```java
  // 对数组该节点位置加锁,开始处理数组该位置的迁移工作
                synchronized (f) {
                    // 在做一次校验
                    if (tabAt(tab, i) == f) {
                        // ln 表示低位,hn表示高位,接下来这段代码的作用是将链表拆分为两部分,0在低位,1在高位.

                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            // 遍历当前 bucket 的链表,目的是尽量重用Node链表尾部的一部分
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            // 如果最后更新的runBit是0,设置低位节点
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                // 否则,设置高位节点
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // 将低位的链表放在i位置上也就是不动
                            setTabAt(nextTab, i, ln);
                            // 将高位链表放在i+n位置点,表示此hash 桶已经被处理
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
```

####  高低位原理分析

CocurrentHashMap 在做链表迁移时,会用高低位来实现,这里有两个问题要分析一下,

1. 如何实现高低位链表的区分

   假设我们有这样一个队列

   ![](http://files.luyanan.com//img/20190815110930.png)

   第14个槽位插入新的节点之后,链表元素个数已经达到了8,且数组的长度为16,优先通过扩容来缓解链表过长的问题,库哦哦让这块的图解稍后在分析,先分析高低位扩容的原理.

   加入当前线程正在处理槽位为14的节点,它是一个链表结构,在代码中,首先定义两个变量节点 ln和hn,实际上就是lowNode和highNode, 分别保存hash值的第x位为0和不等于0的节点

   通过fn&n 就可以把这个链表的元素分为两类,A累是hash值的第X位为0,B类是hash值的第X位不等于0,并且通过lastRun记录最后要处理的节点.最终要达到的目的是,A类的链表保持位置不对,B类的链表为14+16(扩容增加的长度) = 30

   我们把14槽位的链表单独拎出来,我们用蓝色来标识fn&n=0的接待你,假设链表的分类是这样的

   ![](http://files.luyanan.com//img/20190815115924.jpg)

   ```java
    for (Node<K,V> p = f.next; p != null; p = p.next) {
                                   int b = p.hash & n;
                                   if (b != runBit) {
                                       runBit = b;
                                       lastRun = p;
                                   }
                               }
                               
   ```

   通过上面的代码遍历,会记录 runBit以及lastRun,按照上面的这个结构,那么runBit 应该是蓝色节点,lastRun 应该是第6个节点

   接着,再通过这段代码进行遍历,生成ln链以及hn链

   ```java
    for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                   int ph = p.hash; K pk = p.key; V pv = p.val;
                                   if ((ph & n) == 0)
                                       ln = new Node<K,V>(ph, pk, pv, ln);
                                   else
                                       hn = new Node<K,V>(ph, pk, pv, hn);
                               }
   ```

   

![](http://files.luyanan.com//img/20190815131117.png)

接着,通过CAS操作,把hn链放在i+n也就是 14+16的位置上,ln链保持原来的位置不动,并且设置当前节点为fwd, 表示已经被当前线程迁移完了.

```java
        // 将低位的链表放在i位置上也就是不动
                            setTabAt(nextTab, i, ln);
                            // 将高位链表放在i+n位置点,表示此hash 桶已经被处理
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            
```

迁移完成以后的数据分布如下：

![](http://files.luyanan.com//img/20190815132947.png)

#### 为什么要做高低位的划分

要想了解这么设计的目的,我们需要从ConcurrentHashMap 的根据下标获取对象的算法来看,在putVal 方法中1018行

```java
 else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
```

通过 (n-1)&hash 来获得在 table 中的数组下标来获取节点数据(&运算是二进制运算符,1&1=1,其它都为0)

> 假设我们的table长度为16,二进制是[0001 1000],减一以后的二进制是[0000 1111].
>
> 假如某个key的hash值=9对应的二进制是[0000 1001],那么按照(n-1)&hash的算法 0000 1111 & 0000 1001 = 0000 1001 ，运算结果是9
>
> 当我们扩容后,16 变成了32，那么(n-1)的二进制是[ 0001 1111 ] 
>
> 仍然以hash 值 = 9 的二进制计算为例
>
> 0001 1111 & 0000 1001 = 0000 1001,运算结果仍然是9
>
> 我们换一个数字,假如某个key的hash值为20,对应的二进制是[0001 0100],仍然按照(n-1)&hash 算法,分别在16长度和32长度下的计算结果
>
> 16位:0000 1111 & 0001 0100 = 0000 0100
>
> 32位: 0001 1111 & 0001 0100 = 0001 0100 
>
> 从结果上看,同样的一个hash值,在扩容前和扩容之后,得到的下标位置是不一样的,这样情况当然是不允许出现的,所以在扩容的时候就需要考虑的.
>
> 大家可以看到,16位的结果到32位的结果,正好增加了16
>
> 比如 20&15= 4 , 20&31=20 ; 4-20 = 16
>
> 比如 60&15 = 12, 60 & 31 = 28, 12-28 = 16
>
> 所以对于高位,直接增加了扩容的长度,当下次hash 获得数组位置的时候,可以直接定位到对应的位置,这个地方又是一个很巧妙的设计,直接通过高低位分类以后,就使得不需要在每次扩容的时候来重新计算hash.极大的提升了效率.

####  扩容结束以后的退出机制

如果线程扩容结束,那么需要退出,就会执行transfer 方法的如下代码

```java
   //  i<0  说明已经完成遍历完旧的数组,也就是当前线程已经处理完所有的负债的  bucket
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 如果完成了扩容
                if (finishing) {
                    // 删除成员变量
                    nextTable = null;
                    // 更新table 数组
                    table = nextTab;
                    // 更新阈值(32*0.75 = 24)
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // sizeCtl  在迁移前会设置为( (rs << RESIZE_STAMP_SHIFT) + 2)
                // 然后,每增加一个线程参会迁移都会对sizeCtl 加1
                // 这里使用 CAS 操作对sizeCtl 的低16位进行减1,代表做完了属于自己的任务
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    // 第一个扩容的线程,执行 transfer 方法之前,会设置 sizeCtl  = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                    // 后续帮其扩容的线程执行 transfer 方法之前,会设置 sizeCtl = sizeCtl+1
                    // 每一个退出transfer 的方法的线程,退出之前会设置 sizeCtl = sizeCtl-1,那么最后一个线程退出时:必然有
                    //sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2)== resizeStamp(n) << RESIZE_STAMP_SHIFT
                    // 如果sc -2 不等于标识符 左移16位.如果他们相等了,说明没有线程在帮助他们扩容了,也就是说,扩容结束了,
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 如果相等,扩容结束了,更新finishing的值
                    finishing = advance = true;
                    // 再次循环检查一下整张表
                    i = n; // recheck before commit
                }
            }
            
```

### put方法的第三阶段

如果对应的节点存在,判断这个节点的hash是不是等于MOVED(-1), 说明当前节点是ForwardingNode 节点,意味着有其他线程正在进行扩容,那么当前直接帮助它进行扩容,因为调用 helpTransfer 方法

```java
 else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
                
```

#### helpTransfer 

从名字上看,代表当前是去协助扩容

```java
  final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // 判断此时是否仍然在执行扩容,nextTab = null的时候说明扩容已经结束了
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            // 生成扩容戳
            int rs = resizeStamp(tab.length);
            // 说明扩容还未完成的情况下不断的循环来尝试将当前线程加入到扩容操作中
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                // 下面部分的整个代码表示扩容结束,直接退出循环
                // transferIndex <= 0  表示所有的Node 都分配了线程
                //  sc == rs + MAX_RESIZERS  表示扩容线程数达到最大扩容线程数
                // (sc >>> RESIZE_STAMP_SHIFT) != rs   如果在同一轮扩容中,那么sc 无符号右移比较高位和rs的值,那么应该是相等的,如果不相等,说明扩容结束了
                // sc == rs + 1  表示扩容结束

                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    // 跳出循环
                    break;
                // 在低16位上增加扩容线程数
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    // 帮助扩容
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        // 返回新的数组
        return table;
    }
```

### put 方法第四阶段

这个方法的主要作用是: 如果被添加的节点的位置已经存在了节点的时候,需要以链表的方式加入到节点中

如果当前节点已经是一棵红黑树,那么就会按照红黑树的规则将当前节点加入到红黑树中

```java
  V oldVal = null;
                // 给对应的头节点加锁
                synchronized (f) {
                    // 再次判断对应下标位置是否为f节点
                    if (tabAt(tab, i) == f) {
                        // 头节点的hash值大于0,说明是链表
                        if (fh >= 0) {
                            // 用来记录链表的长度
                            binCount = 1;
                            // 遍历链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果发现相同的key,则判断是否需要进行值的覆盖
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    // 默认情况下,直接覆盖旧的值
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                // 一直遍历到链表的最末端,直接把新的值加入到链表的最后面
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        // 如果当前的节点是一颗红黑树
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            // 则调用红黑树的插入方法插入新的值
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                // 同样,如果值已经存在,则直接替换
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
                
```

### put 方法的第五个阶段

判断链表的长度是否已经达到了临界值8, 如果达到了临界值,这个时候会根据当前数组的长度来决定是扩容还是将链表转换为红黑树.也就是说当前数组的长度小于64,就会先扩容,否则,会把当前链表转换为红黑树.

```java
  // 如果上面在做链表操作
                if (binCount != 0) {
                    // 如果链表的长度已经达到了临界值8就需要把链表转换为树结构
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    // 如果val是被替换的,则返回替换之前的值
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
```



#### treeifyBin

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            // tab的长度是不是小于64, 如果是,则执行扩容
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            // 否则将当前链表转换为红黑树结构存储
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                // 将链表转换为红黑树
                synchronized (b) {
                    if (tabAt(tab, index) == b) {
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }
```

#### tryPresize

```java
 private final void tryPresize(int size) {
        // 对size 进行修复,主要目的是防止传入的值不是一个2次幂的整数,然后通过tableSizeFor 来将入参转换为离该整数的最近的2次幂
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            // 这段代码和initTable是一样的,如果table 没有初始化,则开始初始化
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            // 0.75
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;
                    }
                }
            }
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            // 这段代码和addCount 后续部分代码是一样的,做辅助扩容操作
            else if (tab == table) {
                int rs = resizeStamp(n);
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```

