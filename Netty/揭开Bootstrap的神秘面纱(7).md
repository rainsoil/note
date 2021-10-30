#  揭开Bootstrap 的神秘面纱

##  1. 客户端Bootstrap

### 1. Channel简介

在Netty中,Channel是一个Socket的抽象, 它为用户提升了关于Socket状态(是否是连接还是断开) 以及对Socket的读写等操作, 每当Netty 建立一个连接后， 都创建一个对应的Channel 实例.

除了TCP协议外, Netty 还支持很多的其他的连接协议, 并且每种协议还有NIO(非阻塞IO) 和OLO( Old - io,即传统的IO ）版本的区别. 不同 协议不同的阻塞类型的连接都有不同的Channel 类型与之对应. 下面是一些常见的Channel 类型

| 类名                   | 解释                                                         |
| ---------------------- | ------------------------------------------------------------ |
| NioSocketChannel       | 异步非阻塞的客户端TCP Socket连接                             |
| NioServerSocketChannel | 异步非阻塞的服务端TCP socket连接                             |
| NioDatagramChannel     | 异步非阻塞的UDP 连接                                         |
| NioSctpChannel         | 异步的客户端Sctp(Stream Control Transmission Protocol,流控制传输协议) 连接 |
| NioSctpServerChannel   | 异步的服务端连接                                             |
| OioSocketChannel       | 同步阻塞的客户端 TCP Socket连接                              |
| OioServerSocketChannel | 同步阻塞的服务端 TCP Socket连接                              |
| OioDatagramChannel     | 同步阻塞的UDP连接                                            |
| OioSctpChannel         | 同步的Sctp连接                                               |
| OioSctpServerChannel   | 同步的客户端 TCP Socket 连接                                 |

下面我们来看一下Channel的总体类图：

![](http://files.luyanan.com//img/20190923212744.png)

### 2. NioSocketChannel的创建

Bootstrap是Netty 提供了一个便利的工厂类, 我们可以通过它来完成Netty的客户端或者服务端的Netty的初始化, 下面我先来看一个例子, 从客户端和服务端的角度来分别分析一下Netty的程序是如何启动的, 首先, 首先我们从客户端的代码片段开始

```java
    public void connect(String hostName, int port, String nickName) {

        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {

                        }
                    });
            // 发起同步连接操作
            ChannelFuture future = bootstrap.connect(hostName, port).sync();
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
```

上面的客户端代码虽然简单, 却展示了Netty 客户端初始化所需的内容.

1. EventLoopGroup : 不论是服务端还是客户端, 都必须指定EventLoopGroup. 在这个例子中, 指定了NioEventLoopGroup,  表示一个NIO的EventLoopGroup .
2. ChannelType: 指定Channel的类型， 因为是客户端, 因此使用了NioSocketChannel.
3. Handler: 设置处理数据的handler

下面我们继续深入代码, 看一下客户端通过 Bootstrap 启动后, 都做了哪些工作? 我们看一下NioSocketChannel的类层次结构如下:

![](http://files.luyanan.com//img/20190924185100.png)

回到我们在客户端连接代码的初始化Bootstrap 中调用channel() 方法, 传入的参数为 NioSocketChannel.class  , 在这个方法中 其实就是初始化了一个ReflectiveChannelFactory 的对象

```java
 public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
```

而ReflectiveChannelFactory  实现了ChannelFactory 接口, 它提供了唯一的方法, 即newChannel() 方法, 我们看到其实现代码如下:

```java
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }
```

根据上面代码提示, 我们可以得出

1. Bootstrap中的 ChannelFactory 的实现类是 ReflectiveChannelFactory  
2. 通过channel() 方法创建的Channel 具体类型是 `NioSocketChannel`

Channel 的实例化过程就是调用 ChannelFactory 的newChannel() 方法, 而实例化的Channel 具体类型又是和初始化Bootstrap 时传入的channel() 方法的参数相关.  因此对于客户端的Bootstrap而言, 创建的Channel 实例就是`NioSocketChannel`

###  3.客户端Channel 的初始化

前面我们已经知道了如何设置一个Channel的类型, 并且了解到Channel是通过 ChannelFactory 的newChannel()  方法来实例化的, 那么ChannelFactory 的newChannel()方法又是在哪里被调用的呢?  继续追踪, 我们发现其调用链如下:

![](http://files.luyanan.com//img/20190925155028.png)

在AbstractBootstrap 的 initAndRegister() 中调用了ChannelFactory 的newChannel()  来创建一个NioSocketChannel 的实例, 具体代码如下：

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
 ........
 }
```

 在newChannel()  方法中, 利用反射机制调用类对象的newInstance() 方法来创建一个新的Channel  实例, 相当于调用 NioSocketChannel的默认构造器, NioSocketChannel的默认构造器代码如下:

```java
    public NioSocketChannel() {
        this(DEFAULT_SELECTOR_PROVIDER);
    }

   public NioSocketChannel(SelectorProvider provider) {
        this(newSocket(provider));
    }
```

这里的代码比较关键, 我们看到,  这个构造器中会调用newSocket) 来打开一个新的Java NIO 的 SocketChannel ：

```ja
    private static SocketChannel newSocket(SelectorProvider provider) {
        try {
            /**
             *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
             *  {@link SelectorProvider#provider()} which is called by each SocketChannel.open() otherwise.
             *
             *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
             */
            return provider.openSocketChannel();
        } catch (IOException e) {
            throw new ChannelException("Failed to open a socket.", e);
        }
    }
```

接下来会调用父类, 即 AbstractNioByteChannel的构造器

```java
   public NioSocketChannel(SocketChannel socket) {
        this(null, socket);
    }
   public NioSocketChannel(Channel parent, SocketChannel socket) {
        super(parent, socket);
        config = new NioSocketChannelConfig(this, socket.socket());
    }

```

并传入参数 parent = null,ch 为刚才 newSocket() 常见的 Java NIO  的SocketChannel 对象, 因此新创建的NioSocketChannel 对象中, parent 暂时为 null

```java
    protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
        super(parent, ch, SelectionKey.OP_READ);
    }
```

继续调用 父类的 AbstractNioChannel的构造器, 并传入参数 readInterestOp =  SelectionKey.OP_READ, 

```java
  protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
        super(parent);
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false);
        } catch (IOException e) {
            try {
                ch.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially initialized socket.", e2);
                }
            }

            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
    }
```

 然后继续调用父类 AbstractChannel 的构造器

```java
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

至此, NioSocketChannel 就初始化结束了, 我们可以稍微总结一下 NioSocketChannel 初始化所做的工作内容：

1. 调用NioSocketChannel  的 newSocket(DEFAULT_SELECTOR_PROVIDER) 打开一个新的Java NioSocketChannel,

2. AbstractChannel (Channel parent)  中需要初始化的属性:

   - id: 每一个Channel 都拥有的一个唯一的id
   - parent: 属性值为 null
   - unsafe: 通过 Unsafe() 实例化一个 unsafe 对象, 他的类型是AbstractNioByteChannel.NioByteUnsafe 内部类.
   - pipeline: 是通过DefaultChannelPipeline 新创建的实力

3. AbstractNioChannel  的属性 

   - ch: 赋值为 Java SocketChannel, 即NioSocketChannel  的newSocket() 方法返回的Java Nio SocketChannel, 
   - readInterestOp： 赋值为SelectionKey.OP_READ
   - ch:被配置为非阻塞, 即调用ch.configureBlocking(false)。

### 4. Unsafe 字段的变化

我们简单提到了, 在实例化NioSocketChannel 的过程中, 会在父类 AbstractChannel 的构造方法中调用newUnsafe() 来获取一个 unsafe 实力,那么unsafe 是怎么初始化的, 他的作用是什么?

其实 unsafe 特别关键, 它封装了对Java 底层的Socket 的操作, 因此实际上是通过 Netty的上层 和Java 底层的重要的桥梁. 那么下面我们来看一下它提供的方法

```java
 interface Unsafe {

        /**
         * Return the assigned {@link RecvByteBufAllocator.Handle} which will be used to allocate {@link ByteBuf}'s when
         * receiving data.
         */
        RecvByteBufAllocator.Handle recvBufAllocHandle();

        /**
         * Return the {@link SocketAddress} to which is bound local or
         * {@code null} if none.
         */
        SocketAddress localAddress();

        /**
         * Return the {@link SocketAddress} to which is bound remote or
         * {@code null} if none is bound yet.
         */
        SocketAddress remoteAddress();

        /**
         * Register the {@link Channel} of the {@link ChannelPromise} and notify
         * the {@link ChannelFuture} once the registration was complete.
         */
        void register(EventLoop eventLoop, ChannelPromise promise);

        /**
         * Bind the {@link SocketAddress} to the {@link Channel} of the {@link ChannelPromise} and notify
         * it once its done.
         */
        void bind(SocketAddress localAddress, ChannelPromise promise);

        /**
         * Connect the {@link Channel} of the given {@link ChannelFuture} with the given remote {@link SocketAddress}.
         * If a specific local {@link SocketAddress} should be used it need to be given as argument. Otherwise just
         * pass {@code null} to it.
         *
         * The {@link ChannelPromise} will get notified once the connect operation was complete.
         */
        void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);

        /**
         * Disconnect the {@link Channel} of the {@link ChannelFuture} and notify the {@link ChannelPromise} once the
         * operation was complete.
         */
        void disconnect(ChannelPromise promise);

        /**
         * Close the {@link Channel} of the {@link ChannelPromise} and notify the {@link ChannelPromise} once the
         * operation was complete.
         */
        void close(ChannelPromise promise);

        /**
         * Closes the {@link Channel} immediately without firing any events.  Probably only useful
         * when registration attempt failed.
         */
        void closeForcibly();

        /**
         * Deregister the {@link Channel} of the {@link ChannelPromise} from {@link EventLoop} and notify the
         * {@link ChannelPromise} once the operation was complete.
         */
        void deregister(ChannelPromise promise);

        /**
         * Schedules a read operation that fills the inbound buffer of the first {@link ChannelInboundHandler} in the
         * {@link ChannelPipeline}.  If there's already a pending read operation, this method does nothing.
         */
        void beginRead();

        /**
         * Schedules a write operation.
         */
        void write(Object msg, ChannelPromise promise);

        /**
         * Flush out all write operations scheduled via {@link #write(Object, ChannelPromise)}.
         */
        void flush();

        /**
         * Return a special ChannelPromise which can be reused and passed to the operations in {@link Unsafe}.
         * It will never be notified of a success or error and so is only a placeholder for operations
         * that take a {@link ChannelPromise} as argument but for which you not want to get notified.
         */
        ChannelPromise voidPromise();

        /**
         * Returns the {@link ChannelOutboundBuffer} of the {@link Channel} where the pending write requests are stored.
         */
        ChannelOutboundBuffer outboundBuffer();
    }
```

从源码中可以看出, 这些方法其实都是对应到相关的Java 底层的Socket 的操作

继续回到 AbstractChannel 的构造方法, 在这里调用了 newUnsafe() 获取了一个新的 Unsafe 对象, 而newUnsafe 方法在NioSocketChannel 中被重写了, 来看代码

```java
    @Override
    protected AbstractNioUnsafe newUnsafe() {
        return new NioSocketChannelUnsafe();
    }
```

NioSocketChannel 的 newUnsafe 方法会返回一个 NioSocketChannelUnsafe 实例, 从这里我们可以确定了, 在实例化中的NioSocketChannel 中的unsafe 字段, 其实是一个 NioSocketChannelUnsafe 的实例.

###  5. Pipeline 的实例化

上面我们分析了 NioSocketChannel  的大体初始化过程, 但是我们漏掉了一个关键的过程. 即ChannelPipeline 的初始化. 在Pipeline 的 注释说明中写到 "Each channel has its own pipeline and it is created automatically when a new channel is created." , 我们知道在实例化一个Channel 时， 需要都要实例化一个ChannelPipeline, 而我们确定在 AbstractChannel 的构造器中看到了 pipeline 字段被初始化为 DefaultChannelPipeline 的实例, 接下来我们就来看一下

```java
    protected DefaultChannelPipeline(Channel channel) {
        this.channel = ObjectUtil.checkNotNull(channel, "channel");
        succeededFuture = new SucceededChannelFuture(channel, null);
        voidPromise =  new VoidChannelPromise(channel, true);

        tail = new TailContext(this);
        head = new HeadContext(this);

        head.next = tail;
        tail.prev = head;
    }
```

DefaultChannelPipeline 的构造器需要传入一个channel, 而这个channel 就是我们实例化的NioSocketChannel . DefaultChannelPipeline 会将这个 NioSocketChannel  对象保存在 channel 字段中, DefaultChannelPipeline 中还有两个特殊的字段, 即 head 和tail .这两个字段是双向链表的头和尾. 其实在DefaultChannelPipeline 中, 维护了一个以 AbstractChannelHandlerContext 为节点元素的双向链表, 这个链表是Netty 实现Pipeline 的关键. 关于DefaultChannelPipeline  中的双向链表以及它所起的作用, 我们暂时先不讲解, 先看看 HeadContext 的类继承层次结构如下所示：

![](http://files.luyanan.com//img/20190925170544.png)

TailContext的类继承层次如下所示：

![](http://files.luyanan.com//img/20190925170641.png)

我们可以看到, 链表中head 是一个 AbstractChannelHandlerContext 的构造器， 并传入参数 inbound = false, outbound = true. 而 TailContext 的构造器 与 HeadContext 的相反, 他调用了父类的AbstractChannelHandlerContext  的构造器, 并传入参数 inbound = true, outbound = false. 即 head 是一个 OutBoundHandler，而tail 是一个InBoundHandler.

### 6. EventLoop的初始化

回到最开始的 ChatClient 代码, 我们在一开始就实例化了一个NioEventLoopGroup 对象, 因为我们就从构造器中追踪一下 EventLoop 的初始化过程, 首先来看一下 NioEventLoopGroup 的类继承层次.

![](http://files.luyanan.com//img/20190925171327.png)

NioEventLoop 有几个重载的构造器, 不过内容都没有太大的区别, 最终都是调用 父类的MultithreadEventLoopGroup的构造器

```java
 /**
     * @see {@link MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, Executor, Object...)}
     */
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }
```

其实有个意思的地方是, 如果我们传入的线程数 nThreads 是0, 那么Netty 会为我们设置默认的线程数

DEFAULT_EVENT_LOOP_THREADS, 这个默认的线程数是怎么确定的呢?

其实很简单, 在静态代码中, 会首先确定 DEFAULT_EVENT_LOOP_THREADS的值

```java
    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }
```

Netty 首先会从系统属性中读取 `io.netty.eventLoopThreads` 的值, 如果我们没有设置的话, 那么就返回默认的值, 即处理器核心数*2, 回到 MultithreadEventLoopGroup 构造器中继续调用 父类MultithreadEventLoopGroup 的构造器

```java
  /**
     * Create a new instance.
     *
     * @param nThreads          the number of threads that will be used by this instance.
     * @param executor          the Executor to use, or {@code null} if the default should be used.
     * @param chooserFactory    the {@link EventExecutorChooserFactory} to use.
     * @param args              arguments which will passed to each {@link #newChild(Executor, Object...)} call
     */
    protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```



我们可以继续追踪到newChooser 方法里面看看其实现逻辑, 具体代码如下:

```java
public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {

    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

    private DefaultEventExecutorChooserFactory() { }

    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTowEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

    private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }

    private static final class PowerOfTowEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTowEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }

    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
}

```

上面的代码逻辑 主要表达的意思是: 即如果nThreads 是2的幂, 即使用 PowerOfTowEventExecutorChooser,否则使用

GenericEventExecutorChooser, 这两个 chooser 方法都重写了next() 方法, next() 方法的主要功能就是将数组索引循环位移, 如下图所示：

![](http://files.luyanan.com//img/20190925173905.png)

当索引移动最后一个位置的时候， 再调用 next() 方法就会将索引位置重新指向0

![](http://files.luyanan.com//img/20190925173943.png)

这个运算逻辑其实很简单, 就是每次让索引自增后和数据长度取模 , idx.getAndIncrement() % executors.length . 但是就连一个非常简单的数组索引运算, Netty 都帮我们做了优化, 因为在计算机底层, &与 比% 运算效率更高。

好了, 分析到这里我们就应该非常清楚MultithreadEventExecutorGroup 中的处理逻辑了., 简单做一个总结: 

1. 创建一个 大小为 nThreads 的SingleThreadEventExecutor 数组
2. 根据nThreads  的大小, 创建不同的 Chooser,即 如果 nThreads  是2的幂, 则使用 PowerOfTowEventExecutorChooser, 反之则使用GenericEventExecutorChooser, 不论使用哪个Chooser,他们的功能都是一样的, 即从 children 数组中 选出一个合适的EventExecutor  实例.
3. 调用 newChild方法初始化 children 数组

根据上面的代码. 我们也能知道， MultithreadEventLoopGroup 内部维护了一个 EventExecutor 数组, 而Netty EventLoopGroup的实现机制就是建立在 MultithreadEventLoopGroup之上的, 每当一个Netty 需要一个EventLoop 时， 会调用next() 方法获取一个可用的EventLoop .

上面代码的最后一部分是 newChild()  方法, 这是一个抽象方法, 它的任务是 实例化EventLoop 对象, 我们跟踪一下它的代码, 可以发现, 这个方法在 NioEventLoopGroup类中有实现, 其实内容很简单.

```java
    @Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
    }
```



其实逻辑很简单 就是实例化一个 NioEventLoop 对象, 然后返回NioEventLoop 对象

最后总结一下 整个NioEventLoop 的初始化过程.

1. EventLoopGroup(其实是MultithreadEventLoopGroup) 内部维护了一个类型为 EventExecutor children 数组, 其大小是nThreads , 这样就构成了一个线程池.
<<<<<<< HEAD
2. 如果我们在实例化 NioEventLoopGroup时, 如果指定了线程池大小, 则nThreads 就是指定的值, 反之是处理器核心数 * 2 oba
=======

2. 如果我们在实例化 NioEventLoopGroup时, 如果指定了线程池大小, 则nThreads 就是指定的值, 反之是处理器核心数 * 2 

3. MultithreadEventLoopGroup 中会调用 newChild 抽象方法来初始化children 数组

4. 抽象方法newChild 是在NioEventLoopGroup 中实现的, 它返回一个NioEventLoop 实例

5. NioEventLoop 属性赋值

   - provider：在NioEventLoopGroup  构造器中通过SelectorProvider.provider()获取一个 SelectorProvider。
   - selector: 在 NioEventLoop  构造器中通过调用provider.openSelector()方法 获取一个selector 对象

   


### 7. Channel 注册到Selector

   在前面的分析中, 我们提到Channel 会在Bootstrap 的initAndRegister() 中进行初始化, 但是这个方法还会将初始化好的channel 注册到NioEventLoop 的selector中, 接下来我们分析一下Channel的注册过程.

在回顾一下AbstractBootstrap的 initAndRegister 方法

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

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```

当channel 初始化后, 紧接着会调用config().group().register() 方法来向 selector 中注册Channel, 我们继续追踪的话, 会发现其调用链如下:

![](http://files.luyanan.com//img/20190925201913.png)

通过追踪调用链 , 最终我们发现是调用到了 unsafe 的 register 方法, 那么接下来我们仔细看一下

AbstractChannel$AbstractUnsafe.register() 方法中到底做了什么?

```java
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

首先, 将 eventLoop 赋值给Channel的 eventLoop 属性, 而我们直到其实这个 eventLoop 对象其实是MultithreadEventLoopGroup 的next() 方法生成的. 根据我们前面的分析, 我们可以确定next() 方法返回的eventLoop 对象是 NioEventLoop 实例, register 方法接着调用了register0 方法

```java
   private void register0(ChannelPromise promise) {
            try {
                // check if the channel is still open as it could be closed in the mean time when the register
                // call was outside of the eventLoop
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                doRegister();
                neverRegistered = false;
                registered = true;

                // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
                // user may already fire events through the pipeline in the ChannelFutureListener.
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise);
                pipeline.fireChannelRegistered();
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {
                    if (firstRegistration) {
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        // This channel was registered before and autoRead() is set. This means we need to begin read
                        // again so that we process inbound data.
                        //
                        // See https://github.com/netty/netty/issues/4805
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
```



register0() 方法又调用了AbstractNioChannel.doRegister() 方法,

```java
    @Override
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```

看到javaChannel 这个方法我们前面已经已经知道了, 它返回的是一个Java Nio 的 SocketChannel 对象, 这里我们将这个SocketChannel 注册到与EventLoop 关联的 selector 上.

我们总结一下Channel的注册过程

1.  首先在AbstractChannel 的 initAndRegister() 方法中, 通过group().register(channel) 调用MultithreadEventLoopGroup的 register() 方法.
2. 在MultithreadEventLoopGroup 的register() 方法中, 调用next() 方法 获取一个可用的 SingleThreadEventLoop,然后调用它的 register()  方法.
3. 在 SingleThreadEventLoop 的 register()  方法中, 调用channel.unsafe().register(this, promise) 方法来获取 channel 的unsafe() 底层操作对象, 然后调用 unsafe的 register 方法.
4. 在AbstractUnsafe 的register() 中, 调用 register0() 方法注册Channel 对象.
5. 在AbstractUnsafe的  register0() 方法中, 调用 AbstractNioChannel 的 doRegister() 方法
6. AbstractNioChannel的 doRegister() 方法通过javaChannel().register(eventLoop().selector, 0, this); 将channel 对应的Java  NIO 的Socket Channel 注册到 eventLoop 的 selector 中 ,并且将当前channel 作为attachment 与SocketChannel 关联.

总的来说, Channel 注册过程所作的工作就是将Chanel 与对应的EventLoop 关联, 这因此体现出了, 在Netty中, 每个Channel 都会关联一个特定的EventLoop ,  并且这个Channel 中的所有操作都是在这个EventLoop 中执行的. 当关联好Channel 和EventLoop 后, 会继续调用底层Java Nio 的SocketChannel 对象的register()  方法, 将底层Java NIO 的SocketChannel 注册到指定的 selector 中, 通过这两步就完成了Netty 对Channel 的注册.

### 8. Handler 的添加过程

Netty 有一个强大和灵活的地方就是基于 Pipeline 的自定义header 机制. 基于此, 我们可以像添加插件一样自由组合各种各样的handler来完成业务逻辑。例如我们需要处理http 数据, 那么就可以在 pipeline 之前添加一个针对HTTP 编解码的Handler, 接着添加我们的业务逻辑handler, 这样网络上的数据流就像通过一个管道一样, 从不同的headler 中流过并进行编解码, 最终再到达我们自己的handler中。 

我们先从体验一下自定义handler是如何以及何时添加到ChannelPipeline 中的, 首先我们看一下我们的用户代码

```java
 public void connect(String hostName, int port, String nickName) {

        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.SO_KEEPALIVE, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {

                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringDecoder());
                            pipeline.addLast(new StringEncoder());
                            pipeline.addLast(new ChatClientHandler(nickName));
                        }
                    });
            // 发起同步连接操作
            ChannelFuture future = bootstrap.connect(hostName, port).sync();
            future.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }
    }
```

这个代码片段就是实现了handler 的添加功能, 我们看到 , Bootstrap 的 handler 方法接收一个ChannelInitializer,而我们传的参数是一个派生于抽象类 ChannelInitializer的 匿名类, 它当然也实现了ChannelHandler 接口, 我们来看一下, ChannelInitializer类内到底有什么玄机.

```java
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ChannelInitializer.class);
    // We use a ConcurrentMap as a ChannelInitializer is usually shared between all Channels in a Bootstrap /
    // ServerBootstrap. This way we can reduce the memory usage compared to use Attributes.
    private final ConcurrentMap<ChannelHandlerContext, Boolean> initMap = PlatformDependent.newConcurrentHashMap();

    /**
     * This method will be called once the {@link Channel} was registered. After the method returns this instance
     * will be removed from the {@link ChannelPipeline} of the {@link Channel}.
     *
     * @param ch            the {@link Channel} which was registered.
     * @throws Exception    is thrown if an error occurs. In that case it will be handled by
     *                      {@link #exceptionCaught(ChannelHandlerContext, Throwable)} which will by default close
     *                      the {@link Channel}.
     */
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        // Normally this method will never be called as handlerAdded(...) should call initChannel(...) and remove
        // the handler.
        if (initChannel(ctx)) {
            // we called initChannel(...) so we need to call now pipeline.fireChannelRegistered() to ensure we not
            // miss an event.
            ctx.pipeline().fireChannelRegistered();
        } else {
            // Called initChannel(...) before which is the expected behavior, so just forward the event.
            ctx.fireChannelRegistered();
        }
    }

    /**
     * Handle the {@link Throwable} by logging and closing the {@link Channel}. Sub-classes may override this.
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        logger.warn("Failed to initialize a channel. Closing: " + ctx.channel(), cause);
        ctx.close();
    }

    /**
     * {@inheritDoc} If override this method ensure you call super!
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        if (ctx.channel().isRegistered()) {
            // This should always be true with our current DefaultChannelPipeline implementation.
            // The good thing about calling initChannel(...) in handlerAdded(...) is that there will be no ordering
            // suprises if a ChannelInitializer will add another ChannelInitializer. This is as all handlers
            // will be added in the expected order.
            initChannel(ctx);
        }
    }

    @SuppressWarnings("unchecked")
    private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
            try {
                initChannel((C) ctx.channel());
            } catch (Throwable cause) {
                // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
                // We do so to prevent multiple calls to initChannel(...).
                exceptionCaught(ctx, cause);
            } finally {
                remove(ctx);
            }
            return true;
        }
        return false;
    }

    private void remove(ChannelHandlerContext ctx) {
        try {
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                pipeline.remove(this);
            }
        } finally {
            initMap.remove(ctx);
        }
    }
}
```

ChannelInitializer 是一个抽象类, 它有一个抽象方法initChannel(), 我们看到的匿名类正是实现了这个方法, 并在这个方法中添加自定义的 handler的. 那么 initChannel() 是在哪里被调用的呢？ 其实是在 ChannelInitializer 的channelRegistered()  方法中.

接下来关注一下 channelRegistered 方法, 从上面的源码中, 我们可以看到, 在channelRegistered() 方法中, 会调用 initChannel()  方法, 将自定义的 handler 添加到 ChannelPipeline 中, 然后调用  ctx.pipeline().remove(this) 将自己从 ChannelPipeline 中删除, 上面的分析过程, 如下图所示：

![](http://files.luyanan.com//img/20190926200618.png)

接着 initChannle() 方法调用了后, 添加了自定义的 handler

![](http://files.luyanan.com//img/20190926200710.png)

最后将ChannelInitializer 删除

![](http://files.luyanan.com//img/20190926200754.png)

分析到这里, 我们已经简单了解自定义的 handler 是如何添加到 ChannelPipeline 中的了, 

### 9.  客户端发起连接请求.

经过上面的分析后, 我们大致了解了Netty 客户端初始化时, 所做的工作, 那么接下来直奔主题分析一下客户端是如何发起TCP 连接的.

首先, 客户端通过调用Bootstrap.connect() 方法进行连接, 在 connect() 方法中, 会进行一些参数检查后, 最终调用的是doConnect()  方法. 其代码实现如下:

```java
 private static void doConnect(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise connectPromise) {

        // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
        // the pipeline in its channelRegistered() implementation.
        final Channel channel = connectPromise.channel();
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (localAddress == null) {
                    channel.connect(remoteAddress, connectPromise);
                } else {
                    channel.connect(remoteAddress, localAddress, connectPromise);
                }
                connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            }
        });
    }

```

在doConnect() 中,会有 eventLoop 线程中调用 Channel的 connect() 方法, 而这个Channel的具体类型实际上就是NioSocketChannel, 前面已经分析过了., 继续追踪到到 channel.connect() 方法中, 我们发现它调用的是DefaultChannelPipeline.connect()  方法,  pipeline的  connect()  方法如下：

```java
    @Override
    public final ChannelFuture connect(
            SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
        return tail.connect(remoteAddress, localAddress, promise);
    }
```

tail 我们已经分析过了, 是一个TailContext 的实例 , 而TailContext 又是 AbstractChannelHandlerContext 的子类, 并且没有实现 connect()  方法, 因此这里调用的其实是 AbstractChannelHandlerContext的connect()  方法, 我们看一下这个方法的实现

```java
 @Override
    public ChannelFuture connect(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

        if (remoteAddress == null) {
            throw new NullPointerException("remoteAddress");
        }
        if (!validatePromise(promise, false)) {
            // cancelled
            return promise;
        }

        final AbstractChannelHandlerContext next = findContextOutbound();
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeConnect(remoteAddress, localAddress, promise);
        } else {
            safeExecute(executor, new Runnable() {
                @Override
                public void run() {
                    next.invokeConnect(remoteAddress, localAddress, promise);
                }
            }, promise, null);
        }
        return promise;
    }
```

上面的代码中有一个关键的地方, 即`final AbstractChannelHandlerContext next = findContextOutbound();`  这里调用 findContextOutbound() 方法, 从DefaultChannelPipeline 内的双向链表的tail  开始， 不断的向前找到第一个outbound为 true的AbstractChannelHandlerContext,   然后调用它的invokeConnect()  方法, 其代码如下：

```java
    private void invokeConnect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
        if (invokeHandler()) {
            try {
                ((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);
            } catch (Throwable t) {
                notifyOutboundHandlerException(t, promise);
            }
        } else {
            connect(remoteAddress, localAddress, promise);
        }
    }
```

前面我们有提到, 在DefaultChannelPipeline 的构造器中 , 实例化了 两个对象, head 和 tail . 并形成了双向链表的头和尾. head  是HeadContext是实例, 它实现了ChannelOutboundHandler接口, 并且它的outbound设置为 true,  因此在findContextOutbound()  方法中, 找到的AbstractChannelHandlerContext 对象其实就是head, 进而在invokeConnect()    方法中,我们向上转换为 ChannelOutboundHandler 就是没问题的了. 而又因为 HeadContext 重写了 connect() 方法, 因此实际调用的是HeadContext的connect()  方法, 我们接着追踪到HeadContext的 connect()  方法, 其代码如下：

```java

        @Override
        public void connect(
                ChannelHandlerContext ctx,
                SocketAddress remoteAddress, SocketAddress localAddress,
                ChannelPromise promise) throws Exception {
            unsafe.connect(remoteAddress, localAddress, promise);
        }
```

这个 connect() 方法非常简单, 只是调用了 unsafe的connect()  方法, 回顾一下 HeadContext 的构造器. 我们发现这个 unsafe  其实就是  pipeline.channel().unsafe()  返回的Channel 的 unsafe 字段, 到此为止， 我们应该已经知道, 其实是AbstractNioByteChannel.NioByteUnsafe 内部类. 最后, 我们找到创建 Socket 连接的关键代码继续追踪， 其实就是调用AbstractNioUnsafe的 connect()  方法, 

```java
   @Override
        public final void connect(
                final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            try {
                if (connectPromise != null) {
                    // Already a connect in process.
                    throw new ConnectionPendingException();
                }

                boolean wasActive = isActive();
                if (doConnect(remoteAddress, localAddress)) {
                    fulfillConnectPromise(promise, wasActive);
                } else {
                    connectPromise = promise;
                    requestedRemoteAddress = remoteAddress;

                    // Schedule connect timeout.
                    int connectTimeoutMillis = config().getConnectTimeoutMillis();
                    if (connectTimeoutMillis > 0) {
                        connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                            @Override
                            public void run() {
                                ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                                ConnectTimeoutException cause =
                                        new ConnectTimeoutException("connection timed out: " + remoteAddress);
                                if (connectPromise != null && connectPromise.tryFailure(cause)) {
                                    close(voidPromise());
                                }
                            }
                        }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
                    }

                    promise.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            if (future.isCancelled()) {
                                if (connectTimeoutFuture != null) {
                                    connectTimeoutFuture.cancel(false);
                                }
                                connectPromise = null;
                                close(voidPromise());
                            }
                        }
                    });
                }
            } catch (Throwable t) {
                promise.tryFailure(annotateConnectException(t, remoteAddress));
                closeIfClosed();
            }
        }
```

在这个 conect()  中, 又调用了 doConnect()  方法, 注意，这个方法并不是AbstractNioUnsafe 的方法, 而是AbstractNioChannel的抽象方法, doConnect()    方法是在 NioSocketChannel 中实现的, 因为进入NioSocketChannel的 doConnect()    方法中

```java
  @Override
    protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
        if (localAddress != null) {
            doBind0(localAddress);
        }

        boolean success = false;
        try {
            boolean connected = javaChannel().connect(remoteAddress);
            if (!connected) {
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;
        } finally {
            if (!success) {
                doClose();
            }
        }
    }
```

我们终于看到的最关键的代码, 首先是获取Java NIO 的SocketChannel , 获取 NioSocketChannel的 newSocket 返回的SocketChannel  对象,  最后调用 SocketChannel 的connect()   方法完成java NIO 底层的Socket 连接, 最后总结一下, 客户端Bootstrap 发起连接请求的的流程:

![](http://files.luyanan.com//img/20190926204647.png) 
>>>>>>> 14e9bd67a584e759133295239250bed1a61e7204
