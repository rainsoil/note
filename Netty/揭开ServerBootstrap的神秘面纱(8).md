# 揭开ServerBootstrap 的神秘面纱

##  1. 简介

我们首先来看一下 ServerBootstrap 服务端的启动代码

```java
 public void start(int port) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.channel(NioServerSocketChannel.class)
                    .group(bossGroup, workerGroup)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {

                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);
            // 绑定端口, 开始接受进来的请求
            ChannelFuture future = bootstrap.bind(port).sync();
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) {
        new NIOChatServer().start(8080);
    }
```

服务端基本写法和客户端的代码相比, 没有很大的差别, 基本上也是进行了如下几个部分的初始化：

1. EventLoopGroup: 不论是服务端代码还是客户端, 都必须指定 EventLoopGroup. 在上面的代码中, 指定了NioEventLoopGroup, 表示一个 NIO的EventLoopGroup, 不过服务端需要指定两个 `EventLoopGroup`, 一个是bossGroup, 用于处理客户端的连接请求， 另一个是 workGroup, 用于处理与各个客户端连接的IO操作
2. ChannalType: 指定channel的类型, 因为是服务器端, 因此用到了NioServerSocketChannel 
3. handler :  设置数据处理器

##  2. NioServerSocketChannel 的创建

我们在分析客户端channel 初始化过程已经提到, Channal 是对Java 底层Socket 连接的抽象, 并且知道了客户端Channel 的具体类型是 NioServerSocketChannel . 那么, 自然的服务端的Channel 类型就是NioServerSocketChannel . 那么接下来我们按照分析客户端的流程对服务端的代码也同样的分析一遍, 通过前面的分析, 我们已经知道了, Chanel的类型的指定是在初始化时 , 通过Bootstrap 的channel 方法设置的, 服务端也是同样的方式.

再看服务端代码, 我们调用了 ServerBootystrap 的channel(NioSocketChannel.class)  方法, 传的参数是 NioSocketChannel 对象 如此， 按照客户端代码同样的流程, 我们可以确定 NioSocketChannel的实例化也是通过 ReflectiveChannelFactory 来创建的., 而ReflectiveChannelFactory 中的clazz 字段被 赋值为  NioServerSocketChannel.class, 因此当调用 NioServerSocketChannel 的 newChannel  方法, 就能获取到一个 NioServerSocketChannel的 实例. newChannel() 方法的源码如下：

```java
  @Override
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }
```

最后, 我们来总结一下:

1.  ServerBootstrap 中 的channelFactory 的实现类是ReflectiveChannelFactory类
2. 创建的Channel 具体类型是NioServerSocketChannel

channel 实例化过程, 其实就是调用chanellFactory 的 newChannel()   方法， 而实例化的Channel 具体类型就是初始化 ServerBootstrap  时传给 chanel()  方法的实参, 因此, 上面代码案例中的服务端 ServerBootstrap , 创建的 Channel 实例就是 NioServerSocketChannel  的实例。

## 3. 服务端Channel 的初始化

接下来我们来分析一下  NioServerSocketChannel   的实例化过程, 先看一下 NioServerSocketChannel   的 类层次结构图:

![](http://files.luyanan.com//img/20190927173532.png)

首先, 我们来追踪一下  NioServerSocketChannel    的默认构造, 和 NioSocketChannel  类似, 构造器都是调用 newSocket() 来打来一个Java 的NIO Socket. 不过需要注意的是, 客户端的 newSocket 方法调用的是openSocketChannel()，而 服务端的 newSocket()  调用的是 openServerSocketChannel(). 顾名思义, 一个是客户端是 java SocketChannel,  一个是服务端的 Java ServerSocketChannel,  来看代码

```java
 private static ServerSocketChannel newSocket(SelectorProvider provider) {
        try {
            /**
             *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
             *  {@link SelectorProvider#provider()} which is called by each ServerSocketChannel.open() otherwise.
             *
             *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
             */
            return provider.openServerSocketChannel();
        } catch (IOException e) {
            throw new ChannelException(
                    "Failed to open a server socket.", e);
        }
    }

    /**
     * Create a new instance
     */
    public NioServerSocketChannel() {
        this(newSocket(DEFAULT_SELECTOR_PROVIDER));
    }
```

接下来会调用重载的方法

```java
  public NioServerSocketChannel(ServerSocketChannel channel) {
        super(null, channel, SelectionKey.OP_ACCEPT);
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
```

这个构造方法中, 调用父类构造方法时传入的参数是 `SelectionKey.OP_ACCEPT`.  作为对比, 我们回复一下, 在 客户端的Channel 初始化的时候, 传入的参数为 SelectionKey.OP_READ。在服务启动后需要监听客户端的连接请求, 因此这里我们设置为 SelectionKey.OP_ACCEPT, 也就是通知 selector 对客户端的连接请求感兴趣.

接着和客户端对比分析一下, 会逐层的调用父类的构造器 NioServerSocketChannel -> AbstractNioMessageChannel -> AbstractNioChannel -> AbstractChannel。同样在 AbstractChannel 中实例化一个unsafe 和 pipeline.

```java
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

 不过, 这里需要注意的是, 客户端的unsafe 是 AbstractNioByteChannel#NioByteUnsafe 的实例, 而服务端的unsafe 的实例是  AbstractNioMessageChannel.AbstractNioUnsafe 的实例。 因为 AbstractNioMessageChannel 重写了 newUnsafe()   方法, 其源代码如下：

```java
    @Override
    protected AbstractNioUnsafe newUnsafe() {
        return new NioMessageUnsafe();
    }
```

最后再总结一下 NioServerSocketChannel 实例化过程的执行逻辑

1. 调用 NioServerSocketChannel.newSocket(DEFAULT_SELECTOR_PROVIDER) 方法打开一个新的 Java Nio ServerSocketChannel 

2. AbstractChannel 初始化被赋值的属性:

   - parent: 设置为 null
   - unsafe: 通过 newUnsafe()  实例化 一个 unsafe  对象, 类型是 AbstractNioMessageChannel#AbstractNioUnsafe
   - pipeline :  DefaultChannelPipeline 实例

3. AbstractNioChannel  中被赋值的属性:

   - ch: 赋值为 Java Nio  的ServerSocketChannel, 调用NioSocketChannel 的 newSocket()  获取
   - readInterstOp: 默认赋值为 `SelectionKey.OP_ACCEPT`
   - ch: 设置为非阻塞, 调用 ch.configureBlocking(false)方法

4. NioServerSocketChannel:

     config: new NioServerSocketChannelConfig(this, javaChannel().socket());

##  4. ChannelPipeline 初始化

服务端ChannelPipeline 的初始化跟 客户端的一样

## 5. 服务端Channel 注册到Selector

服务端Channel 的注册过程跟客户端的一样

##  6. bossGroup 和workGroup

在客户端的时候,我们初始化了一个 EventLoopGroup 对象, 而在服务端初始化时, 我们设置了两个EventLoopGroup. 一个是 bossGroup, 一个是  workGroup, 那么这两个 EventLoopGroup 都是干什么的呢? 接下来我们来详细探究一下. 其实 , bossGroup 只用于服务端的 accept,  也就是用于处理客户端新连接接入请求, 我们可以将Netty 比喻为一个餐馆, bossGroup 就像一个大堂经理, 当客户来到餐馆吃饭的时候, 大堂经理就会引导顾客就坐, 为顾客端茶送水等. 而workGroup 就是实际上干活的厨师. 他们负责客户端连接通道的IO操作, 当大堂尽力接待顾客后, 就可以稍作休息, 而此时厨师们(workGroup) 就开始忙碌的准备饭菜 ，	关于bossGroup和workerGroup 的关系, 我们可以用下图来表示, 

![](http://files.luyanan.com//img/20190927205854.png)

首先, 服务端的 bossGroup 不断监听是否有客户端的连接, 当发现有新的客户端连接到来时 , bossGroup 就会为此连接初始化各种资源, 然后从 workerGroup 中选出一个EventLoop 绑定到此客户端连接中, 那么接下来服务器和客户端的交互过程就全在此分配的 EventLoop 中完成, 

首先在ServerBootstrap  初始化时, 调用了 bootstrap.group(bossGroup,workGroup)  设置了两个EventLoop,	我们跟踪进去就会发现：

```java
    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        super.group(parentGroup);
        if (childGroup == null) {
            throw new NullPointerException("childGroup");
        }
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = childGroup;
        return this;
    }
```

显然, 这个方法初始化了两个字段, 一个是 group = parentGroup, 它是在 super.group(parentGroup); 中完成初始化的, 另一个是 this.childGroup = childGroup;  接着从 应用程序的启动代码来看调用了  bootstrap.bind()  方法来监听一个本地端口,  bing()  方法会触发如下调用链

>  AbstractBootstrap.bind() -> AbstractBootstrap.doBind() -> AbstractBootstrap.initAndRegister()

源码看到这里为止，我们发现 AbstractBootstrap 的 initAndRegister()方法已经分析过了, 再来回顾一下这个方法

```java

    final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }
        return regFuture;
    }
```



这里 group()  方法返回的是上面我们提到的 bossGroup,  而这里的channel 其实就是 NioServerSocketChannel. 的实例， 因此我们可以猜测到 group().register(channel)  将 bossGroup 和NioSocketChannel 就关联起来了, 那么 workerGroup  具体是在哪里与 NioSocketChannel  关联起来的呢 ? 我们继续往下看 init(channel ) 方法

```java
 @Override
    void init(Channel channel) throws Exception {
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            channel.config().setOptions(options);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                // We add this handler via the EventLoop as the user may have used a ChannelInitializer as handler.
                // In this case the initChannel(...) method will only be called after this method returns. Because
                // of this we need to ensure we add our handler in a delayed fashion so all the users handler are
                // placed in front of the ServerBootstrapAcceptor.
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```



 实际上init()  方法在ServerBootstrap 被重写了, 从上面的代码来看, 它为pipeline  添加了一个 ChannelInitializer, 而这个 ChannelInitializer 中添加了一个非常关键的 ServerBootstrapAcceptor 的 handler,  在ServerBootstrapAcceptor 中 重写了channelRead()  方法 . 其主要代码如下:

```java
   @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler);

            for (Entry<ChannelOption<?>, Object> e: childOptions) {
                try {
                    if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + child, t);
                }
            }

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
```

ServerBootstrapAcceptor 中的 childGroup 是构造此对象 传入的currentChildGroup,   也就是 workerGroup, 而这里的 channel 是NioSocketChannel 的实例.  因此这里的 childGroup 的 regster()  方法就是将 workerGroup 中的某个EventLoop 和NioSocketChannel 关联上了 . 既然这样, 那么现在的问题是 ServerBootstrapAcceptor 的channelRead()  是在哪里被调用的呢?  其实, 当一个client 连接到server 的时候,  Java 底层Nio ServerSocketChannel 就会有一个 SelectionKey.OP_ACCEPT 事件就绪 , 接着会调用到 NioServerSocketChannel 的 doReadMessages()

方法:

```java
  @Override
    protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = javaChannel().accept();

        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
            logger.warn("Failed to create a new channel from an accepted socket.", t);

            try {
                ch.close();
            } catch (Throwable t2) {
                logger.warn("Failed to close a socket.", t2);
            }
        }

        return 0;
    }
```

在 doReadMessages() 方法中, 通过调用 JavaChannel().accept() 方法获取到 客户端新连接的 SocketChannel 对象, 紧接着就实例化一个 NioSocketChannal,并且传入NioServerSocketChannel 对象 , 即this. 由此可知, 我们创建的这个NioSocketChannel 的父类 channel  就是 NioServerSocketChnanel 的实例. 接下来就经由 Netty的Pipelint 机制将读取事件逐个发送到各个handler 中, 于是就会触发我们前面提到的ServerBootstrapAcceptor 的 channelRead()
方法。

## 7. 服务端Selector 事件轮询

再回到服务端的 ServerBootstrap 的启动代码,是从bind()  方法开始的, ServerBootstrap 的bind()  方法实际上就是其父类的 AbstractBootstrap的  bind() 方法, 来看代码

```java
    public ChannelFuture bind() {
        validate();
        SocketAddress localAddress = this.localAddress;
        if (localAddress == null) {
            throw new IllegalStateException("localAddress not set");
        }
        return doBind(localAddress);
    }
 private ChannelFuture doBind(final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
  private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }

```

在doBind0() 方法中,调用的是 EventLoop的execute()  方法, 我们继续跟进去

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



在 execute()  中主要是创建线程, 将线程添加到 EventLoop 的无锁化串行任务队列. 我们重点关注一下 startThread() 方法, 继续看代码:

```java
private void startThread() {
        if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                doStartThread();
            }
        }
    }

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

我们发现 startThread()   最终调用的是  SingleThreadEventExecutor.this.run()  方法, 这个this 就是 NioEventLoop 对象:

```java
    @Override
    protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));

                        // 'wakenUp.compareAndSet(false, true)' is always evaluated
                        // before calling 'selector.wakeup()' to reduce the wake-up
                        // overhead. (Selector.wakeup() is an expensive operation.)
                        //
                        // However, there is a race condition in this approach.
                        // The race condition is triggered when 'wakenUp' is set to
                        // true too early.
                        //
                        // 'wakenUp' is set to true too early if:
                        // 1) Selector is waken up between 'wakenUp.set(false)' and
                        //    'selector.select(...)'. (BAD)
                        // 2) Selector is waken up between 'selector.select(...)' and
                        //    'if (wakenUp.get()) { ... }'. (OK)
                        //
                        // In the first case, 'wakenUp' is set to true and the
                        // following 'selector.select(...)' will wake up immediately.
                        // Until 'wakenUp' is set to false again in the next round,
                        // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                        // any attempt to wake up the Selector will fail, too, causing
                        // the following 'selector.select(...)' call to block
                        // unnecessarily.
                        //
                        // To fix this problem, we wake up the selector again if wakenUp
                        // is true immediately after selector.select(...).
                        // It is inefficient in that it wakes up the selector for both
                        // the first case (BAD - wake-up required) and the second case
                        // (OK - no wake-up required).

                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                    default:
                        // fallthrough
                }

                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```

终于看到似曾相似的代码, 上面的代码主要就是用一个死循环, 在不断的轮询 SelectionKey,select()  方法 主要用来解决JDK 的空轮训Bug, 而 processSelectedKeys()  就是针对不同的轮询事件 进行处理. 如果客户端有数据写入, 最终也会调用 AbstractNioMessageChannel 的 doReadMessages()方法, 总结一下:

1. Netty 的selector 事件轮询 是从EventLoop 的 execute()  方法开始的 
2. 在EventLoop 的 execute()  方法中, 会为每个任务创建一个线程, 并保存到无锁化串行任务队列
3. 线程任务队列的每个任务实际调用的是EventLoop 的 run()  方法
4. 在 run() 中 调用 processSelectedKeys() 处理轮询事件

## 8 Netty 解决JDK 空轮训Bug

各位应该早有耳闻 臭名昭著的 Java NIO epoll的bug, 它会导致 selector  空轮训, 最终导致CPU 100%. 官方声称在 JDK1.6 版本的 update18 修复了此问题, 但是知道JDK1.7 版本该问题仍然存在, 只不过该Bug 发生概率降低了一些而已, 它并没有被 根本解决。出现此bug 是因为当Selector 的轮询结果为空, 也没有 wakeup 或者新消息处理, 则发现空轮训, CPU使用率达到了100%. 我们来看下这个问题在issus的原始描述

> This is an issue with poll (and epoll) on Linux. If a file descriptor for a connected socket is polled with a request
> event mask of 0, and if the connection is abruptly terminated (RST) then the poll wakes up with the POLLHUP (and
> maybe POLLERR) bit set in the returned event set. The implication of this behaviour is that Selector will wakeup and
> as the interest set for the SocketChannel is 0 it means there aren't any selected events and the select method
> returns 0.

具体解释为: 在部分Linux的2.6的 kernel 中, poll 和epoll 对于突然中断的连接 socket 会对返回的 eventSet事件集合置为 POLLHUP，也可能是 POLLERR。 eventSet 事件集合发生了变化, 这就可能导致Selector 会被唤醒.

这是与操作系统机制有关系的, JDK 虽然仅仅是一个兼容各个操作系统平台的软件。但很遗憾在JDK5和JDK6 最初的版本中(严格意义上来讲, JDK版本都是) , 这个问题并没有解决，而将这个帽子抛给了操作系统方, 这也就是这个bug 最终一直到2013 年才最终修复的原因.

在Netty 中 最终的解决办法是: 创建一个新的Selector, 将可用的事件重新注册到新的Selector 中来终止空轮训. 回顾事件轮询的关键代码:

 ```java
protected void run() {
for (;;) {
switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
case SelectStrategy.CONTINUE:
continue;
case SelectStrategy.SELECT:
select(wakenUp.getAndSet(false));
//省略 select 的唤醒逻辑
default:
}
//事件轮询处理逻辑
}
}
 ```

前面我们有提到了 select() 方法解决JDK 空轮训的bug ,它到底是如何解决的呢? 下来我们来一探究竟, 进入select()  方法的源码

```java
  private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
            for (;;) {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if (timeoutMillis <= 0) {
                    if (selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
                // Selector#wakeup. So we need to check task queue again before executing select operation.
                // If we don't, the task might be pended until select operation was timed out.
                // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                int selectedKeys = selector.select(timeoutMillis);
                selectCnt ++;

                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                    // - Selected something,
                    // - waken up by user, or
                    // - the task queue has a pending task.
                    // - a scheduled task is ready for processing
                    break;
                }
                if (Thread.interrupted()) {
                    // Thread was interrupted so reset selected keys and break so we not run into a busy loop.
                    // As this is most likely a bug in the handler of the user or it's client library we will
                    // also log it.
                    //
                    // See https://github.com/netty/netty/issues/2426
                    if (logger.isDebugEnabled()) {
                        logger.debug("Selector.select() returned prematurely because " +
                                "Thread.currentThread().interrupt() was called. Use " +
                                "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                    }
                    selectCnt = 1;
                    break;
                }

                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    // The selector returned prematurely many times in a row.
                    // Rebuild the selector to work around the problem.
                    logger.warn(
                            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                            selectCnt, selector);

                    rebuildSelector();
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;
            }

            if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                            selectCnt - 1, selector);
                }
            }
        } catch (CancelledKeyException e) {
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
            // Harmless exception - log anyway
        }
    }
```

从上面的代码中可以看到, Selector 每一次轮询都计数 selectCnt ++, 开始轮询会计时并赋值给 timeoutMillis, 轮询完成后悔计时赋值给 time. 这两个时间差会有一个时间差, 而这个时间差就是每次轮询所消耗的时间, 从上面的逻辑看出， 如果每次轮询消耗的时间为0 ，且重复次数超过512次， 则调用rebuildSelector();, 则重构Selector, 我们跟进到源码中就会发现

```java
 public void rebuildSelector() {
        if (!inEventLoop()) {
            execute(new Runnable() {
                @Override
                public void run() {
                    rebuildSelector();
                }
            });
            return;
        }

        final Selector oldSelector = selector;
        final Selector newSelector;

        if (oldSelector == null) {
            return;
        }

        try {
            newSelector = openSelector();
        } catch (Exception e) {
            logger.warn("Failed to create a new Selector.", e);
            return;
        }

        // Register all channels to the new Selector.
        int nChannels = 0;
        for (;;) {
            try {
                for (SelectionKey key: oldSelector.keys()) {
                    Object a = key.attachment();
                    try {
                        if (!key.isValid() || key.channel().keyFor(newSelector) != null) {
                            continue;
                        }

                        int interestOps = key.interestOps();
                        key.cancel();
                        SelectionKey newKey = key.channel().register(newSelector, interestOps, a);
                        if (a instanceof AbstractNioChannel) {
                            // Update SelectionKey
                            ((AbstractNioChannel) a).selectionKey = newKey;
                        }
                        nChannels ++;
                    } catch (Exception e) {
                        logger.warn("Failed to re-register a Channel to the new Selector.", e);
                        if (a instanceof AbstractNioChannel) {
                            AbstractNioChannel ch = (AbstractNioChannel) a;
                            ch.unsafe().close(ch.unsafe().voidPromise());
                        } else {
                            @SuppressWarnings("unchecked")
                            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                            invokeChannelUnregistered(task, key, e);
                        }
                    }
                }
            } catch (ConcurrentModificationException e) {
                // Probably due to concurrent modification of the key set.
                continue;
            }

            break;
        }

        selector = newSelector;

        try {
            // time to close the old selector as everything else is registered to the new one
            oldSelector.close();
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to close the old Selector.", t);
            }
        }

        logger.info("Migrated " + nChannels + " channel(s) to the new Selector.");
    }
```

在rebuildSelector() 方法中, 主要做了三件事情:

1. 创建一个新的Selector
2. 将原来的Selector中的注册事件全部取消
3. 将可用的事件重新注册到Selector 中, 并激活

## 9. Netty 对Selector 中 KeySet 的优化

分析完Netty 对JDK 空轮训bug 的解决方案, 接下来我们再来看看一个很有意思的细节, Netty 对Selector 中存储SelectionKey 的HashSet 也做了优化。 在前面的Netty分析中, Netty 对Selector 有重构, 创建一个新的Selector 其实就是调用 openSelector()   方法, 来看代码:

```java
 public void rebuildSelector() {
        if (!inEventLoop()) {
            execute(new Runnable() {
                @Override
                public void run() {
                    rebuildSelector();
                }
            });
            return;
        }

        final Selector oldSelector = selector;
        final Selector newSelector;

        if (oldSelector == null) {
            return;
        }

        try {
            newSelector = openSelector();
         // 省略其他代码   
        }
```

下面我们来进入 openSelector()

```java
  private Selector openSelector() {
        final Selector selector;
        try {
            selector = provider.openSelector();
        } catch (IOException e) {
            throw new ChannelException("failed to open a new selector", e);
        }

        if (DISABLE_KEYSET_OPTIMIZATION) {
            return selector;
        }

        final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

        Object maybeSelectorImplClass = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    return Class.forName(
                            "sun.nio.ch.SelectorImpl",
                            false,
                            PlatformDependent.getSystemClassLoader());
                } catch (ClassNotFoundException e) {
                    return e;
                } catch (SecurityException e) {
                    return e;
                }
            }
        });

        if (!(maybeSelectorImplClass instanceof Class) ||
                // ensure the current selector implementation is what we can instrument.
                !((Class<?>) maybeSelectorImplClass).isAssignableFrom(selector.getClass())) {
            if (maybeSelectorImplClass instanceof Exception) {
                Exception e = (Exception) maybeSelectorImplClass;
                logger.trace("failed to instrument a special java.util.Set into: {}", selector, e);
            }
            return selector;
        }

        final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;

        Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    selectedKeysField.setAccessible(true);
                    publicSelectedKeysField.setAccessible(true);

                    selectedKeysField.set(selector, selectedKeySet);
                    publicSelectedKeysField.set(selector, selectedKeySet);
                    return null;
                } catch (NoSuchFieldException e) {
                    return e;
                } catch (IllegalAccessException e) {
                    return e;
                } catch (RuntimeException e) {
                    // JDK 9 can throw an inaccessible object exception here; since Netty compiles
                    // against JDK 7 and this exception was only added in JDK 9, we have to weakly
                    // check the type
                    if ("java.lang.reflect.InaccessibleObjectException".equals(e.getClass().getName())) {
                        return e;
                    } else {
                        throw e;
                    }
                }
            }
        });

        if (maybeException instanceof Exception) {
            selectedKeys = null;
            Exception e = (Exception) maybeException;
            logger.trace("failed to instrument a special java.util.Set into: {}", selector, e);
        } else {
            selectedKeys = selectedKeySet;
            logger.trace("instrumented a special java.util.Set into: {}", selector);
        }

        return selector;
    }
```

上面的代码的主要功能就是利用反射机制, 获取到JDK底层的 Selector 的 class 对象, 用反射方法从 class 对象中获取两个字段selectedKeys 和publicSelectedKeys, 这两个字段就是用来存储已注册事件的, 然后,将这两个对象 重新复制为Netty 创建的 SelectedSelectionKeySet, 是不是有种偷梁换柱的感觉呢?

我们先来看一下 selectedKeys 和publicSelectedKeys 到底是什么类型? 打开SelectorImpl 的源码, 看其构造方法:

```java
    protected Set<SelectionKey> selectedKeys = new HashSet();
    protected HashSet<SelectionKey> keys = new HashSet();
    private Set<SelectionKey> publicKeys;
    private Set<SelectionKey> publicSelectedKeys;

    protected SelectorImpl(SelectorProvider var1) {
        super(var1);
        if (Util.atBugLevel("1.4")) {
            this.publicKeys = this.keys;
            this.publicSelectedKeys = this.selectedKeys;
        } else {
            this.publicKeys = Collections.unmodifiableSet(this.keys);
            this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
        }

    }
```

我们发现  selectedKeys 和publicSelectedKeys 就是HashSet, 下面我们再来看一下 Netty 创建 的 SelectedSelectionKeySet 对象的源代码:

```java

final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    private SelectionKey[] keysA;
    private int keysASize;
    private SelectionKey[] keysB;
    private int keysBSize;
    private boolean isA = true;

    SelectedSelectionKeySet() {
        keysA = new SelectionKey[1024];
        keysB = keysA.clone();
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        if (isA) {
            int size = keysASize;
            keysA[size ++] = o;
            keysASize = size;
            if (size == keysA.length) {
                doubleCapacityA();
            }
        } else {
            int size = keysBSize;
            keysB[size ++] = o;
            keysBSize = size;
            if (size == keysB.length) {
                doubleCapacityB();
            }
        }

        return true;
    }

    private void doubleCapacityA() {
        SelectionKey[] newKeysA = new SelectionKey[keysA.length << 1];
        System.arraycopy(keysA, 0, newKeysA, 0, keysASize);
        keysA = newKeysA;
    }

    private void doubleCapacityB() {
        SelectionKey[] newKeysB = new SelectionKey[keysB.length << 1];
        System.arraycopy(keysB, 0, newKeysB, 0, keysBSize);
        keysB = newKeysB;
    }

    SelectionKey[] flip() {
        if (isA) {
            isA = false;
            keysA[keysASize] = null;
            keysBSize = 0;
            return keysA;
        } else {
            isA = true;
            keysB[keysBSize] = null;
            keysASize = 0;
            return keysB;
        }
    }

    @Override
    public int size() {
        if (isA) {
            return keysASize;
        } else {
            return keysBSize;
        }
    }

    @Override
    public boolean remove(Object o) {
        return false;
    }

    @Override
    public boolean contains(Object o) {
        return false;
    }

    @Override
    public Iterator<SelectionKey> iterator() {
        throw new UnsupportedOperationException();
    }
}

```

源码篇幅不长, 但很精辟, SelectedSelectionKeySet  同样继承了 AbstractSet, 因此复制给  selectedKeys 和publicSelectedKeys  不存在类型转换异常的问题.  我们看到在 SelectedSelectionKeySet 中禁用了 remove(),contails()方法和iterator() 方法, 只保留了 add()  方法, 而且底层存储结构用的是数组 SelectionKey[] keys.  那么Netty 为什么这样设计呢? 主要目的还是简化我们在轮询事件时的操作, 不需要每次轮询时都要移除key.

## 10 .Handler 的添加过程

服务端handler 的添加过程和客户端的有点区别, 跟 EventLoopGroup 一样服务端的handler 也有两个, 一个是通过 handler() 方法设置的handler, 一个是通过 childHandler()  方法设置的 childHandler() ,通过前面的 bossGroup 和workerGroup 的分析, 其实我们可以大胆的猜测, handler 与 accept 过程有关, 即handler  负责处理客户端新连接接入的请求, 而 childHandler  就是负责和客户端连接的IO交互. 那么实际上是不是这样的呢？ 我们继续用代码来证明? 在前面我们已经了解到ServerBootstrap 中 重写了init()  方法, 在这个方法中也添加了 hanlder

```java
   @Override
    void init(Channel channel) throws Exception {
        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            channel.config().setOptions(options);
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }

                // We add this handler via the EventLoop as the user may have used a ChannelInitializer as handler.
                // In this case the initChannel(...) method will only be called after this method returns. Because
                // of this we need to ensure we add our handler in a delayed fashion so all the users handler are
                // placed in front of the ServerBootstrapAcceptor.
                ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
    }
```

在上面的initChannel()  方法中,  首先通过 handler()  方法获取一个 handler, 如果获取到的hanlder 不为空, 则添加到 pipeline中, 然后接着, 添加了一个ServerBootstrapAcceptor 的实力。 那么这里的handler() 方法 返回的是哪个对象呢？其实它返回的 handler 字段, 而这个对象就是我们在服务端的启动代码中设置的

> ```
> .group(bossGroup, workerGroup)
> ```

那么这个时候, pipeline 中的hanlder 情况如下:

![](http://files.luyanan.com//img/20190928164210.png)

根据我们原来客户端代码的分析来, 我们指定 chanel  绑定到 eventLoop(在这里是指 NioServerSocketChanel 绑定到bossGroup) 后， 会在pipeline 中触发 fireChannelRegistered 事件 ,接着就会触发 对ChannelInitializer的 initChannel()
方法的调用. 因为在绑定完成后， 此时的 pipeline 的内容如下:

![](http://files.luyanan.com//img/20190928164435.png)

在前面我们分析 bossGroup 和workerGroup 时, 已经知道了 ServerBootstrapAcceptor 的channelRead()  方法会为新建的Channel 设置handler 并注册到一个 eventLoop 中., 即

```java
 @Override
        @SuppressWarnings("unchecked")
        public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg;

            child.pipeline().addLast(childHandler);

            for (Entry<ChannelOption<?>, Object> e: childOptions) {
                try {
                    if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + child, t);
                }
            }

            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }

            try {
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
```

而这里的handler 就是我们在服务端启动代码中设置的handler

```java
  .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {

                        }
                    })
```

后续的步骤我们基本上清楚了, 当客户端连接Channel 注册后, 就会触发 ChannelInitializer 的 initChannel()方法的调用, 最后我们来总结一下 服务端handler 和childHandler 的区别和联系

1. 在服务端 NioSocketChannel 的 pipeline 中添加的是 hanlder 和 ServerBootstrapAcceptor。
2. 当有新的客户端连接请求时, 调用 用 ServerBootstrapAcceptor 的 channelRead()方法创建此连接的NioSocketChannel 并 添加 childHandler 到NioSocketChannel 对应的pileline 中, 并为此channel 绑定到  workerGroup 中的某个 eventLoop 中.
3. handler 是在 accept 阶段起作用的, 它处理客户端的连接请求.
4. childHandler 是在客户端连接建立后起作用, 他负责客户端连接的IO交互.

最后来看一张图, 加深理解。 下图描述了服务端从启动初始化到有新连接接入的变化过程:

![](http://files.luyanan.com//img/20190928165400.png)