-------------------
#  4. ReentrantLock底层原理分析



## J.U.C 简介

java.util.concurrent是在并发编程中比较常用的工具类,里面包含很多用来在并发场景中使用的组件.比如线程池,阻塞队列,计时器,同步器,并发集合等.并发包的作者是大名鼎鼎的 Doug Lea.

## lock
### Lock简介
在Lock接口出现之前,java中的应用程序对于多线程的并发安全处理只能基于synchronized 关键字来解决,但是 synchronized 在有些场景下会存在一些短板,也就是它不适用于所有的并发场景. 但是在java5之后,Lock的出现可以解决synchronized 在某些场景中的短板,它比synchronized 更加的灵活.

### Lock的实现
Lock本质上是一个接口,它定义了释放和获取锁的抽象方法,定义成接口就意味着它定义了锁的一个标准规范,也同时意味着锁的不同实现.实现Lock接口类有很多,以下为几个常见的锁实现

**ReentrantLock**: 表示重入锁,它是唯一实现了Lock接口的锁. 重入锁指的是线程在获得锁之后,再次获得锁不需要阻塞,而是直接关联一次计数器增加重入次数.

**ReentrantWriteReadLock**: 重入读写锁,它实现了ReadWriteLock接口,在这个类中维护了两个类,一个是ReadLock,一个是WriteLock,他们都分别实现了Lock接口.读写锁是一种适合于读多写少的场景下解决线程安全问题的工具,基本原则是:读和读不互斥,读和写互斥,写和写互斥.也就是说影响数据变化的操作都会互斥.

**StampedLock:** StampedLock 是JDK8引入的新的锁机制,可以简单认为是读写锁的一个改进版本,读写锁虽然可以通过分离读和写的功能使得读和读之间可以完全并发,但是读和写是有冲突的,如果大量的线程存在,可能会引起写线程的饥饿,StampedLock 是一种乐观的读策略,使得乐观锁完全不会阻塞写线程.
### Lock的类关系图
Lock有很多的锁的实现,但是直观的实现是ReentrantLock 重入锁
![](http://files.luyanan.com//img/20190729211742.png)
```
      ReentrantLock lock = new ReentrantLock();
        //  如果锁可用就直接获得锁,如果不可用就直接阻塞
        lock.lock();
        //  和lock方法相似,但是阻塞的线程可中断,抛出InterruptedException 异常
        lock.lockInterruptibly();
        // 非阻塞获取锁,尝试获取锁,r如果成功返回true
        lock.tryLock();
        // 带有超时时间的获取锁的方法
        lock.tryLock(11, TimeUnit.DAYS);
        //释放锁
        lock.unlock();
```
### ReentrantLock 重入锁
重入锁,表示支持重新进入的锁,也就是说,如果当前线程t1 通过调用lock方法获得了锁之后,再次调用lock,是不会再去阻塞去获取锁,直接增加重试次数就行了.synchronized 和ReentrantLock 都是可重入锁.很多人不理解为什么锁会存在重入的特性,那是因为对于同步锁的理解程度还不够.比如在下面这类的场景中,c存在多个加锁的方法的相互调用,其实就是一种重入特性的场景.

#### 重入锁的设计目的

比如调用demo方法获得了当前的对象锁,然后在这个方法中再去调用demo2,demo2中的存在同一个实例锁,这个时候当前线程会因为无法获得demo2的对象锁而阻塞, 就会产生死锁. 重入锁设计的目的就是为了避免线程的死锁.
```
public class ReentrantLockDemo {

    public static void main(String[] args) {
        ReentrantLockDemo reentrantLockDemo = new ReentrantLockDemo();
        new Thread(reentrantLockDemo::demo).start();
    }

    public synchronized void demo() {
        System.out.println("begin demo");
        demo2();
    }

    private synchronized void demo2() {
        System.out.println("begin demo");
    }

}
```

ReentrantLock 的使用案例
```
public class ReentrantLockDemo {

    private static int count;

    static ReentrantLock lock = new ReentrantLock();

    public static void inc() {


        try {
            lock.lock();
            TimeUnit.MILLISECONDS.sleep(2);
            count++;
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }


    }

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 100; i++) {
            new Thread(() -> ReentrantLockDemo.inc()).start();
        }
        TimeUnit.SECONDS.sleep(30);
        System.out.println(count);
    }
    }
```
### ReentrantWriteReadLock 
我们以前的理解的锁,基本上都是排他锁,也就是这些锁在同一时刻只允许一个线程进行访问,而读写锁在同一时刻可以允许多个线程访问,但是在写线程访问时,所有的读线程和写线程都会被阻塞. 读写锁维护了一堆锁,一个读锁,一个写锁;一般情况下,读写锁的性能都比排他锁强,因为大部分场景都是读多写少的. 在读多于写的情况下,读写锁能够提供比排他锁更好的并发量和吞吐量.

```
package com.notes.concurrent.synchronizeds;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * @author luyanan
 * @since 2019/7/29
 * <p>读写锁</p>
 **/
public class ReentrantWriteReadLockDemo {

    static Map<String, Object> cache = new HashMap<>();


    static ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    static ReentrantReadWriteLock.ReadLock readLock = lock.readLock();
    static ReentrantReadWriteLock.WriteLock writeLock = lock.writeLock();

    public static final Object get(String key) {
        System.out.println("开始读取数据");
        readLock.lock();
        try {
            return cache.get(key);
        } finally {
            readLock.unlock();
        }
    }

    public static final Object put(String key, Object value) {
        System.out.println("开始写数据");
        writeLock.lock();
        try {
            return cache.put(key, value);
        } finally {
            writeLock.unlock();
        }
    }

}

```
在这个案例中,通过hashmap 来模拟了一个内存缓存,然后使用读写锁来保证这个内存缓存d的线程安全性.当执行读操作的时候,需要获取读锁,在并发访问的时候,读锁不会阻塞,因为读锁不会影响执行结果.

在执行写操作的时候,线程必须获取写锁,当已有线程持有写锁的情况下,当前线程会被阻塞,只有当写锁释放后,其他读写操作才能继续执行. 使用读写锁提升了读操作的并发性,也保证了每次写操作对所有读写操作的可见性.

- 读锁和读锁共享
- 读锁和写锁不可以共享(排他)
- 写锁和写锁不可以共享(排他)


### ReentrantLock 的实现原理
我们知道锁的基本原理是基于将多线程并行任务通过某一种机制实现线程的串行执行,从而达到线程安全性的目的. 在synchronized 中 ,我们分析了偏向锁,轻量级锁,乐观锁以及自旋锁来优化了synchronized 的加锁开销,同时在重量级锁阶段,通过线程的阻塞以及唤醒来达到线程竞争和同步的目的. 那么在ReentrantLock中,也一定会存在这样的需求去解决问题,就是在多线程竞争重入锁的时候,竞争失败的线程是如何阻塞和被 唤醒的呢？
### AQS
####  AQS是什么？
在Lock中 用到了一个同步队列AQS,全称 AbstractQueuedSynchronizer,它是一个同步工具也是Lock用来实现线程同步的核心组件. 如果你搞懂了AQS，那么J.U.C 中的绝大部分的工具你都能掌握.

####  AQS的两种功能
从使用层面上来说,AQS的功能分为两种:独占和共享

独占锁,每次只能有一个线程持有锁,比如前面演示的ReentrantLock 就是以独占方式实现的互斥锁.

共享锁:允许多个线程同时获取锁,并发访问共享资源,比如ReenteantWriteReadLock

#### AQS的内部实现
AQS内部维护的是一个FIFO的双向链表,这种结构的特点是每个数据结构都有两个指针,分别指向直接的后继节点和直接前驱节点. 所以双向链表可以从任意一个节点开始很方便的访问前驱和后继. 每个Node其实是由线程封装,当线程抢占锁失败后就会被封装成Node节点加入到AQS队列中. 当获取锁的线程释放锁以后,会从队列中唤醒一个阻塞的节点(线程)

![](http://files.luyanan.com//img/20190731213443.jpg)

#### Node的组成
```
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

       
        volatile int waitStatus;

       // 前置节点
        volatile Node prev;

        // 后继节点 
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
         // 当前线程
        volatile Thread thread;

// 存储在condition队列中的后继节点
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
         // 是否为共享锁
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

     // 将线程构造成一个Node,添加到等待队列
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

      // 这个方法会在Condition 中用到
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

#### 释放锁以及添加线程对于队列的变化
当出现锁竞争以及释放锁的时候, AQS 同步队列中的节点会发生变化,首先看一下添加节点的场景
![](http://files.luyanan.com//img/20190731213400.jpg)
里面会涉及两个变化
1. 新的线程封装成Node节点追加到同步队列中,设置prev节点以及修改当前节点的前置节点d的next 指向自己。
2. 通过CAS将tail 重新指向新的尾部节点


head节点表示获得锁成功的节点,当头节点在释放同步状态的时候,会唤醒后继节点,如果后继节点获得锁成功,会将自己设置为头节点,节点的变化过程如下:
![](http://files.luyanan.com//img/20190731215144.jpg)
这个过程也设涉及两个变化
1. 修改head节点指向下一个获得锁的节点
2. 新的获得锁的节点,将prev的节点设置为null

设置head节点不需要用CAS,原因是设置head节点是由获得锁的线程来完成的,而同步锁只能由一个线程获取,所以不需要CAS保证,只需要把head节点设置为原首节点的后继节点,并且断开与原head节点的next引用即可.

### ReentrantLock 的源码分析
以ReentrantLock 作为切入点,来看看在这个场景下是如何使用AQS来实现线程同步的
#### ReentrantLock的时序图
调用ReentrantLock的lock()方法
![](http://files.luyanan.com//img/20190802215141.jpg)
**ReentrantLock.lock()**

这个是ReentrantLock获取锁的入口
```
   public void lock() {
        sync.lock();
    }
```

syn实际上是一个抽象的静态内部类,它继承了AQS来实现重入锁的逻辑.我们前面说过AQS是一个同步队列,他能够实现线程的阻塞以及唤醒,但是它并不具备业务功能,所以在不同的同步场景中,会继承AQS来实现对应场景的功能.

Sync有两个具体的实现:
- NonfairSync: 表示可以存在抢占锁的功能,也就是说不管当前队列上是否有其他线程等待,新线程都有机会抢占锁.
- FairSync:表示所有线程都严格按照FIFO来获取锁.

#### NonfairSync.lock()
以非公平锁为例,来看看lock中的实现
1. 非公平锁和公平锁的最大的区别在于,在非公平锁中我抢占锁的逻辑是 不管有没有线程排队,我先上来cas 去抢占一下.
2. CAS成功,就表示成功获得了锁.
3. CAS失败,就调用 acquire(1); 走锁竞争逻辑

```
    final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```
CAS的实现逻辑
```
   protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```
通过cas 乐观锁的方式来做比较并替换,这段代码的意思是,如果当前内存中的state的值于预期值expect 相等,则替换update.更新成功则返回true,否则返回false.

这个操作是原子的,不会出现线程安全的问题,这里面涉及到了Unsafe的操作,以及涉及到state 这个属性的意思.

state是AQS中的一个属性, 它在不同的实现中所表达的含义是不一样的,对于重入锁的实现来说,它表示一个同步状态.它有两个含义的表示:
1. 当state =0时,表示无锁状态.
2. 当state >0时,表示已有线程获得了锁,也就是state =1,但是因为ReentrantLock 允许重入,所以同一个线程多次获得同步锁的时候,state 就会递增,比如重入5次,那么state = 5; 而在释放锁的时候,同样需要释放5次知道state = 0 其他线程才有资格获得锁.

###### Unsafe类
Unsafe累是在sun.misc 包下,不属于java标准.但是在很多的java的基础类库中,包括以下被广泛使用的高性能的开发库都是基于Unfafe类开发的,比如 Netty,Hadoop,Kafka等.

Unsafe可以被认为是java中留下的后门,提供了一些低层次的操作,如直接内存访问,线程的挂起和恢复,CAS,线程的同步,内存屏障等.

而CAS 就是Unsafe类中提供的一个原子操作,第一个参数为需要改变的对象,第二个为偏移量(即之前求出来的headOffset的值),第三个参数为期待的值,第四个为更新后的值。

整个方法的作用是如果当前的值于预期的值var4相等,则更新为新的期望的值var5, 如果更新成功,则返回true,否则返回false.

######  stateOffset
一个java对象可以看做是一段内存,每个字段都得按照一定的顺序放在这段内存里,通过这个方法可以准确的告诉你某个字段相对于对象的起始内存地址的字节偏移.用于在后面的 compareAndSwapInt 中,去根据偏移量找到对象在内存中的具体位置.

所以stateOffset 表示state 这个字段在AQS 类的内存中相对于该类首地址的偏移量

###### compareAndSwapInt

在 unsafe.cpp中,可以找到 compareAndSwapInt 的实现
```
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj); // 将java对象解析成JVM的oop(普通对象指针)
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);// 根据对象p和地址偏移量找到地址
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e; // 基于cas比较并替换,x表示需要更新的值,addr表示state在内存中的值,e 表示预期值
UNSAFE_END
```

###### AQS.acquire
acquire 是AQS中的方法,如果CAS 操作未能成功,说明state 已经不为0了,此时继续acquire(1)操作
> 大家思考一下,acquire 方法中的1的参数是用来做什么的?
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

这个方法的主要逻辑是:
1. 通过tryAcquire 尝试获取独占锁,如果成功返回true,如果失败返回false
2. 如果tryAcquire 失败,则会通过 addWaiter  方法将当前线程封装成Node节点添加到AQS 队列尾部.
3. acquireQueued 将Node作为参数,通过自旋去尝试获得锁.

###### NonfairSync.tryAcquire
```
       protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```
这个方法的作用的尝试获取锁,如果成功则返回true,如果失败则返回false, 它是重写AQS类中的tryAcquire 方法,并且大家仔细看一下 AQS中的tryAcquire 方法的定义,并没有实现,而是抛出异常.按照一般的思维模式,既然是一个不实现的模板方法,那应该定位是abstract, 让子类来实现?
###### ReentrantLock.nonfairTryAcquire
```
    final boolean nonfairTryAcquire(int acquires) {
            // 获得当前执行的线程
            final Thread current = Thread.currentThread();
            // 获得state 值
            int c = getState();
            if (c == 0) { // 表示无锁状态
                if (compareAndSetState(0, acquires)) {// cas替换 state的值,如果cas成功表示获得锁成功
                    // 保存当前获得锁的线程,下次再来的时候就不再尝试竞争锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //  如果同一个线程来获得锁,直接增加重入次数
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

```
1. 获得当前线程,判断当前锁的状态
2. 如果state=1 表示当前是无锁状态,通过cas 更新state状态的值
3. 当前线程是属于重入,则增加重入次数



###### AQS.addWaiter
```
 private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // tail是AQS中表示同比队列队尾的属性,默认为null
        Node pred = tail;
        // tail 不为空的情况下,说明队列中存在节点
        if (pred != null) {
            // 把当前线程的Node的prev 指向tail
            node.prev = pred;
            //  通过cas把Node节点加入到AQS队列中,也就是设置为tail
            if (compareAndSetTail(pred, node)) {
                // 设置成功后,把原tail节点的next指向当前node
                 pred.next = node;
                return node;
            }
        }
        // tail = null,把node添加到同步队列
        enq(node);
        return node;
    }
```
当tryAcquire 方法获得锁失败之后,则会先调用 addWaiter 将当前线程封装成Node

入参mode表示当前节点的状态,传递的参数是Node.EXCLUSIVE,表示独占状态,意味着重入锁用到了AQS的独占锁功能.

1. 将当前线程封装成Node
2. 当前链表中的tail节点是否为空,如果不为空,则通过cas操作把当前线程的node节点添加到CAS 队列
3. 如果为空或者cas失败,调用enq 将节点添加到AQS 队列

###### enq
enq 就是通过自旋把当前节点加入到队列中
```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

#####  图解分析
假设3个线程来争抢锁,那么截至到enq方法运行结束之后,或者调用addWaiter()方法结束后,AQS中的链表结构
![](http://files.luyanan.com//img/20190806134628.jpg)
###### AQS.acquireQueued
```
 final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //  获得当前节点的prev节点
                final Node p = node.predecessor();
                // 如果是nead节点,说明有资格去争抢锁
                if (p == head && tryAcquire(arg)) {
                    // 获得锁成功,也就是ThreadA 已经释放了锁,然后设置head为ThreadB 获得执行权限
                    setHead(node);
                    // 把原head 节点从链表中移除
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // ThreadA 可能还没有释放锁,使得ThreadB 在执行 tryAcquire时就会返回false
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    // 并且返回当前线程在等待过程中有没有中断过
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
通过addWaiter 方法把线程添加到链表后,会接着把Node作为参数传递给acquireQueued 方法中去竞争锁.
1. 获得当前节点的prev节点
2. 如果prev 节点为head节点,那么它就有资格去争抢锁,调用tryAcquire 抢占锁.
3. 抢占锁成功之后,把获得锁的节点设置为head, 并且移除原来的初始化head 节点
4. 如果获得锁失败,根据waitStatus决定是否要挂起锁.
5. 最后,通过cancelAcquire 取消获得锁的操作.



###### NofairSync.tryAcquire
这个方法在前面分析过,就是通过state 的状态来判断是否处于无锁状态,然后再通过cas进行竞争锁操作. 成功表示获得锁,失败表示获得锁失效.

######  shouldParkAfterFailedAcquire
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 前置节点的waitStatus
        int ws = pred.waitStatus;
        // 如果前置节点为SIGNAL, 意味着只需要等待其他前置节点的线程被释放
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            // 返回true,意味着可以直接放心的挂起了.
            return true;
            // ws 大于0 意味着prev节点取消了 排队,直接移除这个节点就行了
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                // 相当于: pred = pred.prev;  node.prev = pred;
                node.prev = pred = pred.prev;
                //  这里采用循环,从双向列表中移除 CANCELLED的节点
            } while (pred.waitStatus > 0);
            pred.next = node;
            // 利用cas设置prev节点的状态为 SIGNAL
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
如果ThreadA的锁还没有释放的情况下,ThreadB 和ThreadC 来争抢锁 肯定是会失败,那么失败后还调用shouldParkAfterFailedAcquire 方法.

Node 有5种状态,分别是 CANCELLED（1），SIGNAL（-1）、CONDITION（-
2）、PROPAGATE(-3)、默认状态(0)

**CANCELLED:** 在同步队列中等待的线程等待超时或者中断,需要从同步队列中取消该Node的节点,其节点的waitStatus 为 CANCELLED, 即结束状态,进入该状态的节点将不会再变化.

**SIGNAL:** 只要前置节点释放锁,就会通知标识为SIGNAL状态的后续节点的线程
**CONDITION:** 和Condition 有关系.
**PROPAGATE:** 共享模式下,PROPAGATE状态的线程处于可运行状态,0: 初始状态.

这个方法的主要作用是通过Node的状态来判断ThreadA 竞争锁失败以后是否应该被挂起.

1. 如果ThreadA 的pred 节点状态为SIGNAL, 那就表示可以放心的挂起当前线程
2. 通过循环扫描链表把 CANCELLED 状态的节点移除
3. 修改pred 节点的状态为 SIGNAL,返回false
>  返回fasle时,也就是不需要挂起,返回true, 则需要调用 shouldParkAfterFailedAcquire  挂起当前线程.

###### parkAndCheckInterrupt
```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
 使用LockSupport.park 挂起当前线程为 WATING 状态.

 Thread.interrupted()  返回当前线程是否被其他线程触发过中断请求,也就是thread.interrupt(); 如果有触发过中断请求,那么这个方法会返回当前的中断标识true, 并且对中断标识进行复位标识已经响应过的中断请求.如果返回true,意味着在acquire方法中执行 selfInterrupt().


 ###### selfInterrupt
 ```
  static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
 ```
 标识如果当前线程在 acquireQueued 中被中断过,则需要产生一个中断请求,原因是线程在调用 acquireQueued 方法的时候是不会响应中断请求的
 ######  图解分析
 通过 acquireQueued 方法来竞争锁, 如果ThreadA  还在执行中没有释放锁的话,意味着ThreadB 和ThreadC 只能挂起了.

 ![](http://files.luyanan.com//img/20190806150744.png)


 ##### LockSupport
 LockSupport 类是java6 引入的一个类,提供了基本的线程同步原语.LockSupport 实际上是调用了 Unsafe类里面的函数,归结到Unsafe 里面,只有两个函数
 ```
   /**
     * Unblock the given thread blocked on <tt>park</tt>, or, if it is
     * not blocked, cause the subsequent call to <tt>park</tt> not to
     * block.  Note: this operation is "unsafe" solely because the
     * caller must somehow ensure that the thread has not been
     * destroyed. Nothing special is usually required to ensure this
     * when called from Java (in which there will ordinarily be a live
     * reference to the thread) but this is not nearly-automatically
     * so when calling from native code.
     * @param thread the thread to unpark.
     *
     */
    public native void unpark(Object thread);

    /**
     * Block current thread, returning when a balancing
     * <tt>unpark</tt> occurs, or a balancing <tt>unpark</tt> has
     * already occurred, or the thread is interrupted, or, if not
     * absolute and time is not zero, the given time nanoseconds have
     * elapsed, or if absolute, the given deadline in milliseconds
     * since Epoch has passed, or spuriously (i.e., returning for no
     * "reason"). Note: This operation is in the Unsafe class only
     * because <tt>unpark</tt> is, so it would be strange to place it
     * elsewhere.
     */
    public native void park(boolean isAbsolute, long time);
 ```

 unpark 函数为线程提供了"许可(permit)" ,线程调用park函数则等待"许可".这个有点像信号量,但是这个"许可" 是不能叠加的,"许可"是一次性的.

 permit 相当于0/1的开关, 默认是0, 调用一次unpark 就加1 变成了1,调用一个park 就会消费permit, 又会变成0. 如果再调用一次 park就会阻塞,因为permit 已经是0了,知道permit 变成1 ,这时调用unpark 会把 permit 设置为1.每个线程都已经一个相关的permit, permit最多只有一个,重复调用unpark 不会累计.

 ### 锁的释放流程
 如果这个时候 ThreadA 释放锁了,那么我们来看看锁被释放够会产生什么效果.
 ######  ReentrantLock.unlock
 ```
  public void unlock() {
        sync.release(1);
    }
 ```
 在unlock 中,会调用 release 方法来释放锁
 ```
    public final boolean release(int arg) {
        // 释放锁成功
        if (tryRelease(arg)) {
            // 得到aqs中的头部节点
            Node h = head;
            // 如果head节点不为空并且状态!=0 调用unparkSuccessor(h); 唤醒后续的节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
 ```
###### ReentrantLock.tryRelease
```
     protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```
这个方法可以认为是一个设置锁状态的操作,通过将state状态减掉传入的参数值(参数值为1),如果结果状态为0,就想排它锁的Owner设置为null, 使得其他线程有机会进行.

在排它锁中,加锁的时候的状态会增加1(当然可以自己修改这个值),在解锁的时候减掉1,同一个锁,在可以重入后,可能会被叠加为2、3、4这些值,只有unlock() 的次数和lock()的次数对应才会将Owner线程设置为空,而且也只有这种情况下,才会返回true.


###### unparkSuccessor
```
  private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        // 获得head 节点的状态
        int ws = node.waitStatus;
        if (ws < 0)
            // 设置head节点状态为0
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        // 得到head节点的下一个节点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
           // 如果下一个节点为null或者status>0  表示cancelled 状态,
            // 通过从尾部节点开始扫描,找到距离head最近的一个waitStatus<=0的节点
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // next节点不为空,直接唤醒这个线程即可.
        if (s != null)
            LockSupport.unpark(s.thread);
    }

```
**为什么在释放锁的时候是从tail 进行扫描的呢?**

我们再回到enq这个方法 来看一下一个新的节点是如何加入到链表中的
1. 将新的节点的prev 指向tail
2. 通过cas将tail设置为新的节点, 因为cas是原子操作 所以能够保证线程安全性.
3. t.next = node; 设置 原tail 的next节点指向新的节点

```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

![](http://files.luyanan.com//img/20190806171752.png)

在cas操作之后,t.next=node 操作之前.存在其他线程调用 unlock 方法从 head开始往后遍历,由于t.next=nodel 还没执行 ,意味着链表的关系还没有建立完整,就会导致遍历到t 节点的时候被中断.所有从后往前遍历,一定不会存在这个问题。

######  图解分析
通过锁的释放,原本的结构就发生了一些变化.head节点的waitStatus 变成了0,ThreadB被唤醒.

![](http://files.luyanan.com//img/20190806172259.jpg)

##### 原本挂起的线程执行
通过ReentrantLock.unlock ,原本挂起的线程是在acquireQueued 方法中,被唤醒之后从这个方法开始执行.

###### AQS.acquireQueued
```
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
这个方法前面已经分析过了,我们只关注一下ThreadB 被唤醒以后的执行流程.

由于ThreadB的 prev 节点指向的是head, 并且ThreadA 已经释放锁,所以这个时候调用tryAcquire 方法时 就可以顺利获取到锁。
1. 把ThreadB节点当成head
2. 把原head节点的next节点指向null



###### 图解分析
1. 设置新head 节点的prev 为null
2. 设置原head节点的next节点为null
![](http://files.luyanan.com//img/20190806214413.jpg)

###  公平锁和非公平锁的区别
锁的公平性是相对于获取锁的顺序而言的,如果是一个公平锁,那么获取锁的顺序j就应该符合请求的绝对时间排序,也就是FIFO。 在上面的例子来说,只要CAS设置成功,则表示当前线程获得了锁,而公平锁则不一样，差异点有两个
###### FairSync.tryAcquire 
非公平锁在获得锁的时候,会先通过CAS进行抢占,而公平锁则不会.
```
final void lock() {
            acquire(1);
        }
        
           protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
这个方法与nonfairTryAcquire(int acquires) 比较,不同的地方在于判断条件多了 hasQueuedPredecessors 方法,也就是加入了[同步队列中当前节点是否有前驱节点] 的判断,如果该方法返回true,则表示有线程比当前 线程更早的请求获取锁,因为需要等待前驱线程获取并释放锁之后才能继续获得锁.