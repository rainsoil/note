# 大名鼎鼎的EventLoop

##  1. EventLoopGroup 和Reactor

一个Netty 在启动的时候, 至少要指定一个EventLoopGroup(如果使用的是Nio, 一般指定的是NioEventLoopGroup),那么, 这回NioEventLoopGroup 在Netty中 到底扮演着什么样的角色呢? 我们知道, Netty 是Reactor 模型的一个实现, 我们就从Reactor 的线程模型开始

### 1. 浅谈Reactor 线程模型

Reactor 线程模型有三种: 单线程模型、多线程模型、主从线程模型. 首先来看一下单线程模型, 如下图所示:
![](http://files.luyanan.com//img/20190928171643.png)

所以单线程, 即Acceptor 处理的和 handler处理都在同一个线程中处理. 这个模型的坏处显而易见, 当其中的某个handler 阻塞时, 会导致其他所有的Client 的handler 都得不到执行, 并且更严重的是Handler的阻塞会导致整个服务不能接受新的client 请求,(因为Accepor 也被阻塞了). 因为有这么多的缺陷，所以单线程Reactor 模型应用场景比较少.

那么, 什么是多线程模型呢? Reactor 的多线程模型与单线程模型的区别就是Acceptor 是一个单独的线程处理, 并且有一组特定的NIO线程来负责各个客户端连接的IO操作,Reactor 的多线程模型如下图所示:

![](http://files.luyanan.com//img/20190928172413.png)

Reactor 多线程模型有如下特点:

1. 有一个专门的线程, 即Acceptor 线程用于监听客户端的TCP 连接请求.
2. 客户端连接的IO操作都由一个特定的NIO线程负责, 每个客户端连接都要与一个特定的NIO线程绑定, 因此在这个客户端连接中的所有IO操作都是在同一个线程中完成的.
3. 客户端连接有很多, 但是NIO线程数是比较少的, 因此一个Nio线程可以同时绑定到多个客户端连接中.

接下来我们再来看看 Reactor 主从多线程模型. 一般情况下,Reactor 的多线程模型已经可以很好的工作了, 但是我们想象这样一个场景, 如果我们的服务器需要同时处理大量的客户端连接请求, 或者我们需要在客户端连接的时候进行一些权限的校验, 那么单线程的Acceptor 很有可能就处理不过来了, 造成了大量的客户端不能连接到服务端,

Reactor 的主从多线程模型就是在这样的情况下提出的, 他的特点是: 服务端接受客户端的连接请求不再是一个线程, 而是由一个独立的线程池组成, 其线程模型如下图所示：

![](http://files.luyanan.com//img/20190928173213.png)

可以看到, Reactor 的主从多线程模型和Reactor 多线程模型很类似, 只不过Reactor 的主从多线程模型的Acceptor 使用了线程池来处理大量的客户端请求.

### 2. EventLoopGroup 与Reactor关联

我们介绍了三种Reactor 的线程模型, 那么他们和NioEventLoopGroup 又有什么关系呢?其实, 不同的设置NioEventLoopGroup 的方式就对应了不同的Reactor 的线程模型.

1. 单线程模型,来看下面的应用代码:

   ```
      EventLoopGroup group = new NioEventLoopGroup(1);
      ServerBootstrap bootstrap = new ServerBootstrap();
      bootstrap.group(group);
   ```

   注意, 我们实例化了一个 NioEventLoopGroup ,然后接着我们调用  bootstrap.group(group)设置服务端的EventLoopGroup .有人可能会有疑惑, 我记得在启动服务端的Netty程序的时候, 需要设置  bossGroup 和 workerGroup, 为何这里只设置一个 bossGroup ? 其实原因很简单, ServerBootstrap 重写了 group  方法.

   ```java
    @Override
       public ServerBootstrap group(EventLoopGroup group) {
           return group(group, group);
       }
   ```

   因此当传入一个group的时候, 那么bossGroup和workerGroup  就是同一个 NioEventLoopGroup了. 这是, 因为bossGroup 和workerGroup 就是同一个NioEventLoopGroup,并且这个NioEventLoopGroup 线程池数量只设置了1个线程, 也就是 Netty中的Acceptor 与后续的所有客户端连接的IO 操作都是在同一个线程中处理的. 那么对应的Reactor 的线程模型中, 我们这样设置 NioEventLoopGroup时, 就相当于Reactor 的单线程模型.

2. 多线程模型, 再来看下面的应用代码

   ```java
    EventLoopGroup bossGroup = new NioEventLoopGroup(128);
         
               ServerBootstrap bootstrap = new ServerBootstrap();
               bootstrap.group(bossGroup)
   ```

   从上面的代码来看, 我们只需要将 bossGroup 的参数设置为大于1的数,其实就是Reactor 多线程模型.

3. 主从线程模型, 实现主从线程模型的代码如下：

   ```java
    EventLoopGroup bossGroup = new NioEventLoopGroup();
           EventLoopGroup workerGroup = new NioEventLoopGroup();
               ServerBootstrap bootstrap = new ServerBootstrap();
               bootstrap.group(bossGroup, workerGroup)
   ```

   

 bossGroup 为主线程, 而 workerGroup 中的线程是CPU 核心数乘以2, 因此对应的到Reactor 线程模型中, 我们知道, 这样设置的NioEventLoopGroup 其实就是Reactor 主从线程模型.

### 3. EventLoopGroup 实例化

首先, 我们先纵览一下 EventLoopGroup 的l类结构图， 如下所示:

![](http://files.luyanan.com//img/20190928205510.png)

我们先来看一下EventLoopGroup 初始化的时序图

![](http://files.luyanan.com//img/20190928205752.png)

基本步骤如下：

1. EventLoopGroup(其实是 MultithreadEventExecutorGroup) 内部维护了一个类为 EventExecutor children 数组, 其大小为nThreads，这样就初始化了一个线程池。

2. 如果我们在实例化NioEventLoopGroup 时, 如果指定线程池大小 , 则 nThreads 为指定值, 否则为 CPU核心数*2

3. 在MultithreadEventExecutorGroup会调用 newChild() 抽象方法来初始化 children数组 

4. 抽象方法 newChild()  其实是在 NioEventLoopGroup 中实现的，由它返回一个NioEventLoop 实例.

5. 初始化NioEventLoop 主要属性:

   provider: 在 NioEventLoopGroup 构造器中通过 SelectorProvider 的 provider()方法获取 SelectorProvider。

   selector:在 NioEventLoop 构造器中调用 selector = provider.openSelector()方法获取 Selector 对象。



##  2. 任务执行者EventLoop

NioEventLoop 继承自SingleThreadEventLoop, 而SingleThreadEventLoop 又继承自 SingleThreadEventExecutor。而SingleThreadEventExecutor 是Netty 对本地线程的封装, 它内部有一个 Thread thread 属性, 存储了一个Java 本地线程. 因此我们可以简单的认为, 一个 NioEventLoop  就是和一个特定的线程绑定, 并且在其生命周期内 ，绑定的线程都不会发生改变.

![](http://files.luyanan.com//img/20190928210925.png)

NioEventLoop 的类层次结构图还是有些复杂的，不过我们只需要关注几个重要点即可。首先来看 NioEventLoop 的继
承链：NioEventLoop->SingleThreadEventLoop->SingleThreadEventExecutor->AbstractScheduledEventExecutor。
在 AbstractScheduledEventExecutor 中, Netty 实现了 NioEventLoop 的 schedule 功能，即我们可以通过调用一个
NioEventLoop 实例的 schedule 方法来运行一些定时任务。而在 SingleThreadEventLoop 中，又实现了任务队列的功
能，通过它，我们可以调用一个 NioEventLoop 实例的 execute()方法来向任务队列中添加一个 task,并由 NioEventLoop
进行调度执行。

通常来说, NioEventLoop 负责执行两个任务, 第一个任务是作为IO 线程, 执行与Channel 相关的IO 操作， 包括调用Selector 等待就绪的Io事件等、读写数据与数据的处理等. 而第二个任务是作为任务队列, 执行 taskQueue 中的任务, 例如用户调用 eventLoop.schedule 提交的定时任务也是这个线程执行的。

### 2.1 NioEventLoop 的初始化

先看一下 NioEventLoop 实例化的运行时序图

![](http://files.luyanan.com//img/20190928211242.png)

从上图看到, SingleThreadEventExecutor 有一个名为 thread  的Thread 类型字段, 这个字段就是与SingleThreadEventExecutor 关联的本地线程 , 我们看看 thread 是在哪里被赋值的? 

```java
   private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    for (;;) {
                        int oldState = STATE_UPDATER.get(SingleThreadEventExecutor.this);
                        if (oldState >= ST_SHUTTING_DOWN || STATE_UPDATER.compareAndSet(
                                SingleThreadEventExecutor.this, oldState, ST_SHUTTING_DOWN)) {
                            break;
                        }
                    }

                    // Check if confirmShutdown() was called at the end of the loop.
                    if (success && gracefulShutdownStartTime == 0) {
                        logger.error("Buggy " + EventExecutor.class.getSimpleName() + " implementation; " +
                                SingleThreadEventExecutor.class.getSimpleName() + ".confirmShutdown() must be called " +
                                "before run() implementation terminates.");
                    }

                    try {
                        // Run all remaining tasks and shutdown hooks.
                        for (;;) {
                            if (confirmShutdown()) {
                                break;
                            }
                        }
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            threadLock.release();
                            if (!taskQueue.isEmpty()) {
                                logger.warn(
                                        "An event executor terminated with " +
                                                "non-empty task queue (" + taskQueue.size() + ')');
                            }

                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
    }
```

SingleThreadEventExecutor  启动时会调用 doStartThread() 方法, 然后调用 executor.execute() 方法，将当前线程赋值给thread, 在这个线程中所作的主要的事情就是调用 SingleThreadEventExecutor .this.run() 方法, 而因为 NioEventLoop 实现了这个方法, 因此根据多态性, 调用的是 NioEventLoop  的 run() 方法.

### 2.2 EventLoop 与Channel 的关联

在Netty中, 每个Channel 都有且仅有一个EventLoop与之关联, 他们的关联过程如下：

![](http://files.luyanan.com//img/20190928211919.png)

从上图中我们可以看到, 当调用AbstractChannel$AbstractUnsafe.register()方法后, 就完成了 Channel 与EventLoop的关联, register 方法的具体实现如下:

```java
    @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```

在io.netty.channel.AbstractChannel.AbstractUnsafe.register()方法中, 就将一个EventLoop 赋值给AbstractChannel内部的eventLoop 字段, 这段代码就是完成Channel 与EventLoop的关联.

###  2.3 EventLoop的启动

在前面已经知道, NioEventLoop 其实就是一个SingleThreadEventExecutor，因此NioEventLoop  的启动就是 NioEventLoop  所绑定的本地线程的启动.

按照这个思路,我们只需要找到在哪里调用了SingleThreadEventExecutor的thread字段的start()  方法就知道是在哪里启动这个线程了, 从前面中的分析来看, 我们已经知道了 thread.start()  被封装到了 SingleThreadEventExecutor的startThread()方法中, 来看代码:

```java
  private void startThread() {
        if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                doStartThread();
            }
        }
    }
```

STATE_UPDATER 是 SingleThreadEventExecutor 内部维护的一个属性, 它的作用是标识当前的thread 状态, 在初始的时候, ，STATE_UPDATER == ST_NOT_STARTED，因此第一次调用 startThread()  方法时, 就会进入到if 语句, 进而调用到 thread.start()  方法, 而这个关键的 startThread()  方法又是在哪里被调用的呢? 用方法调用关系反向查找功能, 我们发现 statrThread 是在SingleThreadEventExecutor的 execute 方法中调用

```java
    @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

既然如此, 那现在我们的工作就变为了寻找在哪里第一次调用了SingleThreadEventExecutor的 execute() 方法, 在
AbstractChannel$AbstractUnsafe 的 register()中调用 eventLoop.execute()方法，在 EventLoop 中进行 Channel 注册
代码的执行，AbstractChannel$AbstractUnsafe 的 register()部分代码如下

```java
 @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```

很显然, 一路从Bootstrap 的bind() 方法跟踪到AbstractChannel$AbstractUnsafe 的 register()方法，整个代码都是在主线程中运行的. 因为上面的 eventLoop.inEventLoop() 返回为false. 于是进入到 else 分支， 在这个分支中调用了  eventLoop.execute() 方法 , 而NioEventLoop 没有实现 execute() 方法, 因此调用的是 SingleThreadEventExecutor的execute()

```java
  @Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

我们已经分析过了，inEventLoop == false，因此执行到 else 分支，在这里就调用 startThread()方法来启动
SingleThreadEventExecutor 内部关联的 Java 本地线程了。
总结一句话：当 EventLoop 的 execute()第一次被调用时，就会触发 startThread()方法的调用，进而导致 EventLoop
所对应的 Java 本地线程启动。

我们总结一下EventLoop启动过程完整的时序图

![](http://files.luyanan.com//img/20190928214302.png)

