# Netty 大动脉Pileline

## 1. Pepeline 设计原理

###  1.1 Channel 与 ChannelPipeline

相信大家已经知道了, 在Netty中每个Channel 都有且仅有一个ChannelPipeline 与之对应, 他们的组成关系如下:

![](http://files.luyanan.com//img/20190929092313.png)

通过上图我们可以看到, 一个Cannel 包含了一个ChannelPipeline,而ChanelPipeline 中又维护了一个由ChannelHandlerContext  组成的双向链表, 这个链表的头是HeadContext, 链表的尾是 TailContext, 并且每个ChannelHandlerContext 中又关联着一个ChannelHandler.

上面的图示给了我们一个对 ChannelPipeline 的直观认识, 但是实际上Netty 实现的 Channel 是否真是这样的呢? 我们继续用源码来说话, 在前我们已经知道了一个Channel 的初始化的基本过程. 下面我们再回顾一下, 下面的代码 AbstractChannel构造器:

```java
  protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

AbstractChannel 有一个pipeline 字段,  在构造器中会初始化它为DefaultChannelPipeline的实例, 这里的代码就印证了一点: 每个Channel 都有一个 ChannelPipeline.  接着我们追踪一下 DefaultChannelPipeline的初始化过程. 首先进入到 DefaultChannelPipeline 构造器中

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

在 DefaultChannelPipeline构造器中, 首先将与之关联的 Channel 保存到 字段 chanel 中, 然后实例化两个 ChannelHandlerContext：一个是HeadContext 的实例head, 另一个 TailContext 实例tail.接着将head和tail 互相指向,构成一个双向链表.

特别注意点是: 我们在开始的示意图中head和tail 并没有包含ChannelHandler, 这里因为  tail 并没有包含 ChannelHandler，这是因为 HeadContext 和 TailContext
继承于 AbstractChannelHandlerContext 的同时也实现了 ChannelHandler 接口了，因此它们有 Context 和 Handler
的双重属性。

### 1.2 再谈ChannelPileline 的初始化

前面我们已经对ChannelPipeline 的初始化有了一个大致的了解, 不过当时重点没有关注ChannelPipeline,因为没有深入的分析它的初始化过程。 那么下面我们就来看一下具体的ChannelPipeline 的初始化都做了哪些工作吧. 先回顾一下, 在实例化一个Channel 时,会伴随着一个ChannelPipeline 的实例化, 并且次Channel 会与这个ChannelPipeline 相互关联, 这一点可以通过 NioSocketChannel 的父类 AbstractChannel的构造器予以佐证

```java
    protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

当实例化一个NioSocketChannel  时, 其pipeline 字段就是我们创建的 DefaultChannelPipeline 对象, 那么我们就来看一下 DefaultChannelPipeline的构造方法

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

可以看到, 在 DefaultChannelPipeline 的构造方法中，将传入的channel 赋值给字段 this.channel, 接着又实例化了两个特殊的字段, tail与head.这两个字段是一个双向链表的头和尾.  其实在DefaultChannelPipeline 中, 维护了一个以 AbstractChannelHandlerContext 为节点的双向链表, 这个链表是Netty 实现Pipeline 机制的关键, 再回顾一下 head和tail 的类层次结构:

![](http://files.luyanan.com//img/20190929101344.png)

![](http://files.luyanan.com//img/20190929101404.png)

从类层次结构图中可以很清楚的看到, head 实现了 ChannelInboundHandler，而tail实现了ChannelOutboundHandler 接口, 并且他们都实现了 ChannelHandlerContext接口, 因此可以说head和tail 即是一个 ChannelHandler，又是一个ChannelHandlerContext.  接着看 HeadContext 构造器的代码

```java
   HeadContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, HEAD_NAME, false, true);
            unsafe = pipeline.channel().unsafe();
            setAddComplete();
        }
```

它调用了 父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = false，outbound = true。 而TailContext 的构造器与HeadContext  正好相反 ， 它调用了父类 AbstractChannelHandlerContext的构造器, 并传入参数 inbound = true，outbound = false。 也就是说 header  是一个 OutBoundHandler，而tail是一个 InboundHandler， 关于这一点, 大家要特别注意， 因为在后面的分析中, 我们会反复用到 inbound 和 outbound 这两个属性。

###  1.3 ChannelInitializer的添加

前面我们已经分析过了Channel 的组成, 其中我们已经了解过了, 最开始的时候 ChannelPipeline 中含有两个 ChannelHandlerContext（同时也是ChannelHandler),但是这个Pipeline 并不能实现什么特殊的功能, 因为我们还没有给它添加自定义的Channel, 通常来说, 我们在初始化Bootstrap 的时候, 会添加自定义的 ChannelHandler,   就以我们具体的客户端启动代码来举例

```java
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
```

上面代码的初始化过程， 相信大家都不陌生, 在调用handler时， 传入了ChannelInitializer 对象，它提供了一个initChannel 方法给我们初始化 ChannelHandler,   那么这个初始化过程是怎样的呢? 下面我们来揭开它的神秘面纱, ChannelInitializer 实现了 ChannelHandler，那么它是在什么时候添加到ChannelPipeline 中的呢? 通过代码追踪,我们发现他是在Bootstrap  的init() 方法中添加到ChannelPipeline  中的, 其代码如下：

```java
   @Override
    @SuppressWarnings("unchecked")
    void init(Channel channel) throws Exception {
        ChannelPipeline p = channel.pipeline();
        p.addLast(config.handler());

        final Map<ChannelOption<?>, Object> options = options0();
        synchronized (options) {
            for (Entry<ChannelOption<?>, Object> e: options.entrySet()) {
                try {
                    if (!channel.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                        logger.warn("Unknown channel option: " + e);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to set a channel option: " + channel, t);
                }
            }
        }

        final Map<AttributeKey<?>, Object> attrs = attrs0();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                channel.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
        }
    }
```

从上面的代码来看, 将handler()  返回的 ChannelHandler 添加到 Pipeline 中, 而 handler()  返回的其实就是我们在初始化Bootstrap    时通过 handler()   方法设置的 ChannelInitializer 实例, 因为这里就是将 ChannelInitializer 插入到了 Pipeline 的末端, 此时Pipeline 的结构如下图所示：

![](http://files.luyanan.com//img/20190929105732.png)

这时候,有的小伙伴可能就有疑惑了, 我明明插入的是一个 ChannelInitializer 实例, 为什么在 ChannelPipeline 中的双向链表中的元素却是ChannelHandlerContext 呢? 我们继续去源码中寻找答案.

刚才, 我们提到, 在Bootstrap 的init() 方法中会调用p.addLast()  方法,将ChannelInitializer 插入到链表的末端.

```java
   @Override
    public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);

            newCtx = newContext(group, filterName(name, handler), handler);

            addLast0(newCtx);

            // If the registered is false it means that the channel was not registered on an eventloop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }


    private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
        return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
    }
```

addLast() 有很多重载的方法,我们只需要关心这个重要的方法就行. 上面的addLast() 方法中, 首先检查 Channelhandler  的名字是否重复, 如果不重复, 则调用 newContext() 方法为这个Handler 创建一个对应的 DefaultChannelHandlerContext 实例. 并与之关联起来(Contxt  中有一个handler 属性保存着对应的handler 实例). 为了添加一个 handler 到pipeline 中, 必须把此 handler 包装成 ChannelHandlerContext。因此在上面的代码中我们看到新实例化了一个 newCtx 对象,并将handler 作为参数传递到构造方法中,。 那么我们来看一下实例化 DefaultChannelHandlerContext  到底有什么玄机吧, 首先来看一下他的构造器.

```java
  DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
        if (handler == null) {
            throw new NullPointerException("handler");
        }
        this.handler = handler;
    }
```

在 DefaultChannelHandlerContext  的构造器中, 调用了两个很有意思的方法, isInbound()与 isOutbound()，这两个方法是做什么呢? 

```java

    private static boolean isInbound(ChannelHandler handler) {
        return handler instanceof ChannelInboundHandler;
    }

    private static boolean isOutbound(ChannelHandler handler) {
        return handler instanceof ChannelOutboundHandler;
    }
```

从源码中可以看到, 当一个hanler 实现了 ChannelInboundHandler 接口, 则 isInbound 返回true,类似的当一个 handler 实现了 ChannelOutboundHandler 接口, 则 isOutbound 就返回true. 而这两个 boolean 变量会传递到父类 AbstractChannelHandlerContext 中, 并初始化父类的这两个字段：inbound 与 outbound。

那么这里的  ChannelInitializer 所对应的 的DefaultChannelHandlerContext的inbound与inbound字段分别是什么呢? 那就看一下  ChannelInitializer  到底实现了哪个接口不就行了? 如下是 ChannelInitializer  的类结构图

![](http://files.luyanan.com//img/20190929114633.png)

从类图中可以清楚的看到, ChannelInitializer  仅仅实现了 ChannelInboundHandler 接口，因此这里实例化的 DefaultChannelHandlerContext 的 inbound = true，outbound = false。

兜了这么大一圈, 不就是 是 inbound 和 outbound 两个字段嘛， 为什么需要这么大费周折的去分析一番? 其实这两个字段关系到 pipeline 的事件的流向和分类, 因为是十分关键的. 至此, 我们 先记住一个结论. ChannelInitializer   所对应的 DefaultChannelHandlerContext  的  inbound = true，outbound = false。

当创建好 Context 之后, 就将这个Context 插入到 Pipeline的双向链表中, 

```java
  private void addLast0(AbstractChannelHandlerContext newCtx) {
        AbstractChannelHandlerContext prev = tail.prev;
        newCtx.prev = prev;
        newCtx.next = tail;
        prev.next = newCtx;
        tail.prev = newCtx;
    }

```

###  1.4 自定义ChannelHandler 的添加过程

前面我们已经分析过 ChannelInitializer 是如何添加到Pipeline 中的, 接下来就来探究 ChannelInitializer 在哪里被调用? ChannelInitializer 的作用以及我们自定义的ChannelInitializer 是如何插入到Pipeline 中的.

先简单来复习一下 Channel 的注册过程:

1. 首先在AbstractBootstrap 的 initAndRegister()  中, 通过group().register(channel) , 调用 MultithreadEventLoopGroup 的 register()  方法.
2. 在 MultithreadEventLoopGroup 的register()   中调用 next()   获取一个可用的 SingleThreadEventLoop，然后调用他的 register()  方法.
3. 在 SingleThreadEventLoop 的 register()   方法中, 通过 channel.unsafe().register(this.promise) 方法获取channel 的 unsafe 底层的IO操作对象, 然后调用它的 register()  方法.
4. 在  AbstractUnsafe 的register()  方法中, 调用 register0() 方法注册channel 对象
5. 在AbstractUnsafe的 register0() 方法中, 调用 AbstractNioChannel 的doRegister() 方法.
6. AbstractNioChannel 的 doRegister()方法调用 javaChannel().register(eventLoop().selector, 0, this)将 Channel
   对应的 Java NIO 的 SockerChannel 对象注册到一个 eventLoop 的 Selector 中，并且将当前 Channel 作为 attachment。

而我们自定义的ChannelHandler  的添加过程, 发生在 AbstractUnsafe的 register0() 方法中, 在这个方法中调用了 pipeline.fireChannelRegistered()方法, 其代码实现如下：

```java
    @Override
    public final ChannelPipeline fireChannelRegistered() {
        AbstractChannelHandlerContext.invokeChannelRegistered(head);
        return this;
    }

```

再看  AbstractChannelHandlerContext.invokeChannelRegistered(head); 方法:

```java
    static void invokeChannelRegistered(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelRegistered();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRegistered();
                }
            });
        }
    }
```

很显然, 这个代码会从head开始遍历Pipeline 的双向链表, 然后找到第一个属性 为 true 的
ChannelHandlerContext 实例. 想起来没, 我们在前面分析 ChannelInitializer 时， 花费了大量的篇幅来分析了inbound
和 outbound 属性 , 现在这里就用上了. 回想一下, ChannelInitializer  实现了 ChannelInboudHandler ,因此他对应的 ChannelHandlerContext 的inbound  属性就是true.  因此这里返回的就是  ChannelInitializer  实例所对应的 ChannelHandlerContext  对象. 如下图所示：

![](http://files.luyanan.com//img/20190929124703.png)

当获取到 inbound 的Context时，就调用它的 invokeChannelRegistered()

```java

    private void invokeChannelRegistered() {
        if (invokeHandler()) {
            try {
                ((ChannelInboundHandler) handler()).channelRegistered(this);
            } catch (Throwable t) {
                notifyHandlerException(t);
            }
        } else {
            fireChannelRegistered();
        }
    }
```

我们已经知道, 每个ChannelHandler 都和一个ChannelHandlerContext 相关联.我们可以通过ChannelHandlerContext 获取到对应的ChannelHandler.因此很明显, 这里的handler() 发那会的对象其实就是一开始我们就实例化的 ChannelInitializer 对象, 并接着调用了 ChannelInitializer 的 channelRegistered() 方法. 看到这里, 应该就会觉得眼熟了. ChannelInitializer 的 channelRegistered()这个方法我们一开始就已经接触到了, 但是我么么并没有深入的分析这个方法的调用过程, 下来我们俩看看这个方法中到底有什么玄机？ 继续看代码:

```java
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
```

 initChannel((C) ctx.channel())  这个方法我们也很熟悉, 他就是在我们初始化Bootstrap 的时候, 调用handler 方法传入的匿名内部类所实现的方法:

```java
 .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {

                            ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringDecoder());
                            pipeline.addLast(new StringEncoder());
                            pipeline.addLast(new ChatClientHandler(nickName));
                        }
```

当调用这个方法后,我们自定义的ChannelHandler 就插入到Pipeline中了, 此时Pipeline 的状态如下图所示：

![](http://files.luyanan.com//img/20190929134426.png)

当添加完自定义的ChannelHandler 后, 在 finally 代码块中就会删除自定义的 ChannelInitializer，也就是 remove(ctx); 最终调用 ctx.pipeline.remove(this); 因此最后的Pipeline 的状态如下:

![](http://files.luyanan.com//img/20190929134625.png)

至此, 自定义的ChannelHandler 的添加过程就分析的差不多了.

###  1.5. 给ChannelHandler  命令

不知道大家注意到没, pipeline.addXXX  都有一个重载的方法, 例如 addLast() 它有一个重载的版本是:

> ChannelPipeline addLast(String name, ChannelHandler handler);

第一个参数指定添加的 handler 的名字,(更加准确的是ChannelHandlerContext的名字),那么handler的名字有什么用呢? 如果我们不设置name, 那么handler默认的名字是怎样的呢? 带着这些疑问? 我们依旧去源代码去找答案, 还是以addLast() 方法为例.

```java
 @Override
    public final ChannelPipeline addFirst(String name, ChannelHandler handler) {
        return addFirst(null, name, handler);
    }

```

这个方法会调用重载的 addLast()  方法

```java
 @Override
    public final ChannelPipeline addFirst(EventExecutorGroup group, String name, ChannelHandler handler) {
        final AbstractChannelHandlerContext newCtx;
        synchronized (this) {
            checkMultiplicity(handler);
            name = filterName(name, handler);

            newCtx = newContext(group, name, handler);

            addFirst0(newCtx);

            // If the registered is false it means that the channel was not registered on an eventloop yet.
            // In this case we add the context to the pipeline and add a task that will call
            // ChannelHandler.handlerAdded(...) once the channel is registered.
            if (!registered) {
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }

            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
    }
```

第一个参数被设置为null, 我们不需要去关心它, 第二个参数就是这个handler的名字, 看代码可知, 在添加一个handler之前, 需要调用 checkMultiplicity(handler) 方法来确定新添加的handler 名字是否与已经添加的handler 的名字重复.

###  1.6 ChannelHandler 默认命名规则

如果我们调用的是如下的addLast()  方法

> ```
> ChannelPipeline addLast(ChannelHandler... handlers);
> ```

那么Netty 就会调用

```java
  private String filterName(String name, ChannelHandler handler) {
        if (name == null) {
            return generateName(handler);
        }
        checkDuplicateName(name);
        return name;
    }

 private String generateName(ChannelHandler handler) {
        Map<Class<?>, String> cache = nameCaches.get();
        Class<?> handlerType = handler.getClass();
        String name = cache.get(handlerType);
        if (name == null) {
            name = generateName0(handlerType);
            cache.put(handlerType, name);
        }

        // It's not very likely for a user to put more than one handler of the same type, but make sure to avoid
        // any name conflicts.  Note that we don't cache the names generated here.
        if (context0(name) != null) {
            String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
            for (int i = 1;; i ++) {
                String newName = baseName + i;
                if (context0(newName) == null) {
                    name = newName;
                    break;
                }
            }
        }
        return name;
    }
```

而  generateName() 方法会接着调用 generateName0()  方法来实际生成一个新的handler 名字:

```java
   private static String generateName0(Class<?> handlerType) {
        return StringUtil.simpleClassName(handlerType) + "#0";
    }
```

默认的命名规则很简单, 就是反射获取handler 的simpleClassName 加上 "#0", 因为我们的自定义ChatClientHandler 的名字就是 "ChatClientHandler#0"

## 2. Pipeline 的事件传播机制

我们已经知道 AbstractChannelHandlerContext 中有 inbound 和 outbound 两个 boolean 变量,分别用于标识 Context 所对应的handler 类型, 即:

1.  inbound 为true时表示其对应的 ChannelHandler 是 ChannelInboundHandler的子类.
2. outbound 为true时，表示对应的ChannelHandler 是 ChannelOutboundHandler 的子类.

这里大家肯定还有很多疑惑 , 不知道这两个字段到底有什么用? 这还要从ChannelPipeline  的事件传播机制说起， Netty 的事件传播机制可以分为两种: inbound 事件和outbound 事件, 如下是从Netty 对这两个事件的说明;‘

```java
 * <pre>
 *                                                 I/O Request
 *                                            via {@link Channel} or
 *                                        {@link ChannelHandlerContext}
 *                                                      |
 *  +---------------------------------------------------+---------------+
 *  |                           ChannelPipeline         |               |
 *  |                                                  \|/              |
 *  |    +---------------------+            +-----------+----------+    |
 *  |    | Inbound Handler  N  |            | Outbound Handler  1  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler N-1 |            | Outbound Handler  2  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  .               |
 *  |               .                                   .               |
 *  | ChannelHandlerContext.fireIN_EVT() ChannelHandlerContext.OUT_EVT()|
 *  |        [ method call]                       [method call]         |
 *  |               .                                   .               |
 *  |               .                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  2  |            | Outbound Handler M-1 |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  |               |                                  \|/              |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |    | Inbound Handler  1  |            | Outbound Handler  M  |    |
 *  |    +----------+----------+            +-----------+----------+    |
 *  |              /|\                                  |               |
 *  +---------------+-----------------------------------+---------------+
 *                  |                                  \|/
 *  +---------------+-----------------------------------+---------------+
 *  |               |                                   |               |
 *  |       [ Socket.read() ]                    [ Socket.write() ]     |
 *  |                                                                   |
 *  |  Netty Internal I/O Threads (Transport Implementation)            |
 *  +-------------------------------------------------------------------+
```



从上图可以看出, inbound  事件和outbound 事件 的流向是不一样的, inbound 事件的流行是从下往上的, 而 outbound 刚好相反, 是从上到下的. 并且 inbound 的传递方式是通过调用相应的ChannelHandlerContext.fireIN_EVT()方法，而 outbound 方法的传递方式是调用 ChannelHandlerContext.OUT_EVT()方法。例如：ChannelHandlerContext
的 fireChannelRegistered()调用会发送一个 ChannelRegistered 的 inbound 给下一个 ChannelHandlerContext，而
ChannelHandlerContext 的 bind()方法调用时会发送一个 bind 的 outbound 事件给下一个 ChannelHandlerContext

**inbound 事件传播的方法有：**

```java
public interface ChannelInboundHandler extends ChannelHandler {

    /**
     * The {@link Channel} of the {@link ChannelHandlerContext} was registered with its {@link EventLoop}
     */
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;

    /**
     * The {@link Channel} of the {@link ChannelHandlerContext} was unregistered from its {@link EventLoop}
     */
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;

    /**
     * The {@link Channel} of the {@link ChannelHandlerContext} is now active
     */
    void channelActive(ChannelHandlerContext ctx) throws Exception;

    /**
     * The {@link Channel} of the {@link ChannelHandlerContext} was registered is now inactive and reached its
     * end of lifetime.
     */
    void channelInactive(ChannelHandlerContext ctx) throws Exception;

    /**
     * Invoked when the current {@link Channel} has read a message from the peer.
     */
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;

    /**
     * Invoked when the last message read by the current read operation has been consumed by
     * {@link #channelRead(ChannelHandlerContext, Object)}.  If {@link ChannelOption#AUTO_READ} is off, no further
     * attempt to read an inbound data from the current {@link Channel} will be made until
     * {@link ChannelHandlerContext#read()} is called.
     */
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;

    /**
     * Gets called if an user event was triggered.
     */
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;

    /**
     * Gets called once the writable state of a {@link Channel} changed. You can check the state with
     * {@link Channel#isWritable()}.
     */
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;

    /**
     * Gets called if a {@link Throwable} was thrown.
     */
    @Override
    @SuppressWarnings("deprecation")
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}

```

**outbound 事件传播方法有：**

```java
public interface ChannelOutboundHandler extends ChannelHandler {
    /**
     * Called once a bind operation is made.
     *
     * @param ctx           the {@link ChannelHandlerContext} for which the bind operation is made
     * @param localAddress  the {@link SocketAddress} to which it should bound
     * @param promise       the {@link ChannelPromise} to notify once the operation completes
     * @throws Exception    thrown if an error accour
     */
    void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;

    /**
     * Called once a connect operation is made.
     *
     * @param ctx               the {@link ChannelHandlerContext} for which the connect operation is made
     * @param remoteAddress     the {@link SocketAddress} to which it should connect
     * @param localAddress      the {@link SocketAddress} which is used as source on connect
     * @param promise           the {@link ChannelPromise} to notify once the operation completes
     * @throws Exception        thrown if an error accour
     */
    void connect(
            ChannelHandlerContext ctx, SocketAddress remoteAddress,
            SocketAddress localAddress, ChannelPromise promise) throws Exception;

    /**
     * Called once a disconnect operation is made.
     *
     * @param ctx               the {@link ChannelHandlerContext} for which the disconnect operation is made
     * @param promise           the {@link ChannelPromise} to notify once the operation completes
     * @throws Exception        thrown if an error accour
     */
    void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;

    /**
     * Called once a close operation is made.
     *
     * @param ctx               the {@link ChannelHandlerContext} for which the close operation is made
     * @param promise           the {@link ChannelPromise} to notify once the operation completes
     * @throws Exception        thrown if an error accour
     */
    void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;

    /**
     * Called once a deregister operation is made from the current registered {@link EventLoop}.
     *
     * @param ctx               the {@link ChannelHandlerContext} for which the close operation is made
     * @param promise           the {@link ChannelPromise} to notify once the operation completes
     * @throws Exception        thrown if an error accour
     */
    void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;

    /**
     * Intercepts {@link ChannelHandlerContext#read()}.
     */
    void read(ChannelHandlerContext ctx) throws Exception;

    /**
    * Called once a write operation is made. The write operation will write the messages through the
     * {@link ChannelPipeline}. Those are then ready to be flushed to the actual {@link Channel} once
     * {@link Channel#flush()} is called
     *
     * @param ctx               the {@link ChannelHandlerContext} for which the write operation is made
     * @param msg               the message to write
     * @param promise           the {@link ChannelPromise} to notify once the operation completes
     * @throws Exception        thrown if an error accour
     */
    void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;

    /**
     * Called once a flush operation is made. The flush operation will try to flush out all previous written messages
     * that are pending.
     *
     * @param ctx               the {@link ChannelHandlerContext} for which the flush operation is made
     * @throws Exception        thrown if an error accour
     */
    void flush(ChannelHandlerContext ctx) throws Exception;
}

```

大家应该发现了规律, inbound 类似于的事件回调(响应请求的事件),而outbound 类似于主动触发(发起请求的事件) . 注意,如果我们捕获了一个事件, 并想让这个事件继续传递下去, 那么需要调用Context 对应的传播方法 fireXX, 

```java
public class MyInboundHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("连接成功");
        ctx.fireChannelActive();
    }
}

```

如上面的代码, MyInboundHandler 收到了一个 channelActive 事件, 他在处理完成后, 如果希望将事件继续传播下去, 那么需要接着调用 ctx.fireChannelActive();方法.

### 2.1 Outbound 事件传播方式

outbound  事件都是请求事件(request event), 即请求某些事件的发生, 然后通过Outbound 事件进行通知, 

outbound 事件的传播方向是:  tail ->customContext ->head.

我们接下来以 connect事件为例, 分析一下 Outbound 事件的传播机制。

首先, 当用户调用了 Bootstrap 的 connect() 方法时, 就会触发一个Connect 请求事件, 此调用会出发如下调用链:

![](http://files.luyanan.com//img/20190929143107.png)

继续追踪, 我们就会发现 AbstractChannel的connect()  其实调用了 DefaultChannelPipeline 的connect()  方法

```java
    @Override
    public ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
        return pipeline.connect(remoteAddress, localAddress, promise);
    }
```

而  pipeline.connect(remoteAddress, localAddress, promise） 方法的实现如下：

```java
    @Override
    public final ChannelFuture connect(
            SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
        return tail.connect(remoteAddress, localAddress, promise);
    }
```

可以看到, 当outbound(这里是 connect事件) 传递到Pipeline 后, 它其实是以tail 为起点开始传播的.

而 tail.connect 其实调用的是 AbstractChannelHandlerContext 的 connect()方法：

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

findContextOutbound() 方法顾名思义, 他的作用是以当前Context 为起点, 想Pipeline 中的Context 双向链表的前端寻找第一个 outbound  属性为true 的Context(即关联ChannelOutboundHandler的Context), 然后返回.

 findContextOutbound 的方法的代码如下:

```java

    private AbstractChannelHandlerContext findContextOutbound() {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.prev;
        } while (!ctx.outbound);
        return ctx;
    }
```

当我们找到一个 outbound 的Context后， 就调用它的 invokeConnect() 方法, 这个方法中会调用Context 其关联的ChannelHandler 的connect()  方法:

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

如果用户没有重写 ChannelHandler 的 connect()  方法, 那么就会调用 ChannelOutboundHandlerAdapter 的 connect()
实现：

```java
   @Override
    public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress,
            SocketAddress localAddress, ChannelPromise promise) throws Exception {
        ctx.connect(remoteAddress, localAddress, promise);
    }
```

我们看到， ChannelOutboundHandlerAdapter 的connect() 仅仅调用了 ctx.connect , 而这个又调回到了Context.connect -> Connect.findContextOutbound -> next.invokeConnect -> handler.connect -> Context.connect
这样的循环中，直到connect()  事件传递到了  DefaultChannelPipeline 的双向链表的头节点, 即head中,为什么会传递到head中呢? 回想一下, head实现了  ChannelOutboundHandler，因此它的outbound 的属性是true

因为head本身即是一个 ChannelHandlerContext，又实现了ChannelOutboundHandler 接口, 因此当 connect()  消息传递到head后, 会将消息转传递到对应的ChannelHandler 中处理, 而head的 handler() 方法返回的就是head本身.

```java
   @Override
        public ChannelHandler handler() {
            return this;
        }
```

因此 最终 connect()  事件是在head中被处理的, head的 connect() 事件处理逻辑如下:

```java
     @Override
        public void connect(
                ChannelHandlerContext ctx,
                SocketAddress remoteAddress, SocketAddress localAddress,
                ChannelPromise promise) throws Exception {
            unsafe.connect(remoteAddress, localAddress, promise);
        }
```

到这里, 整个connect()  请求事件就结束了, 下图中描述了整个 connect()  请求事件的处理过程.

![](http://files.luyanan.com//img/20190929145409.png)

我们仅仅以 connect()  请求事件为例, 分析了 outbound 事件的传播过程, 但是其实所有的 outbound 事件的传播都遵循一样的传播规律, 

### 2.2 inbound 事件传播方式

inbound 事件和 outbound 事件的处理过程是类似的, 只不过传播方向不同.

inbound 事件是一个通知事件,即某事件已经发生了, 然后通过inbound 事件进行通知， inbound 通常发生在Channel 的状态的改变或者IO事件就绪.

inboud 的特点是它的传播方向 是 head -> customContext -> tail。

上面我们分析了 connect()  这个outbound 事件, 那么接着分析  connect()  事件后会发生什么 inbound 事件, 并最终找到  inbound 和outbound 事件的联系. 当 connect() 这个outbound 传播到 unsafe 后, 其实是在 AbstractNioUnsafe 的connect() 方法中进行处理的.

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

在 AbstractNioUnsafe 的 connect() 方法中, 首先调用doConnect()   方法进行实际上的Socket连接, 当连接上后会调用 fulfillConnectPromise() 方法

```java
 private void fulfillConnectPromise(ChannelPromise promise, boolean wasActive) {
            if (promise == null) {
                // Closed via cancellation and the promise has been notified already.
                return;
            }

            // Get the state as trySuccess() may trigger an ChannelFutureListener that will close the Channel.
            // We still need to ensure we call fireChannelActive() in this case.
            boolean active = isActive();

            // trySuccess() will return false if a user cancelled the connection attempt.
            boolean promiseSet = promise.trySuccess();

            // Regardless if the connection attempt was cancelled, channelActive() event should be triggered,
            // because what happened is what happened.
            if (!wasActive && active) {
                pipeline().fireChannelActive();
            }

            // If a user cancelled the connection attempt, close the channel, which is followed by channelInactive().
            if (!promiseSet) {
                close(voidPromise());
            }
        }
```

我们看到,在fulfillConnectPromise() 中, 会通过调用 pipeline().fireChannelActive() 方法将通道激活的消息(即Socket 连接成功) 发送出去, 而这里, 当调用 pipeline.fireXXX 后, 就是 inbound 事件的起点, 因此当调用  pipeline().fireChannelActive(); 后， 就产生了一个 ChannelActive  inbound 事件, 我们就从这里看这个 inbound 事件是怎么传播的?

```java
    @Override
    public final ChannelPipeline fireChannelActive() {
        AbstractChannelHandlerContext.invokeChannelActive(head);
        return this;
    }
```

果然在 fireChannelActive 方法中, 调用的是 head.invokeChannelActive()  , 因此可以证明 inbound 事件在Pipeline 中的传输的起点是 head,那么在 head.invokeChannelActive()  中又做了什么呢?

```java
    static void invokeChannelActive(final AbstractChannelHandlerContext next) {
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeChannelActive();
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelActive();
                }
            });
        }
    }
```

上面的代码应该很熟悉了, 回想一下, 在outbound 事件(例如connect() 事假 )的传输过程 中, 我们也有类似的操作.

1. 首先调用 findContextInbound() , 从Pipeline 的双向链表中找到第一个属性inbound 为true 的Context, 然后将其返回.

2. 调用Context的invokeChannelActive() 方法, invokeChannelActive 方法的源码如下：

    ```java
     private void invokeChannelActive() {
           if (invokeHandler()) {
               try {
                   ((ChannelInboundHandler) handler()).channelActive(this);
               } catch (Throwable t) {
                   notifyHandlerException(t);
               }
           } else {
               fireChannelActive();
           }
       }
    ```

   这个方法和Outbound 的对应方法(如：invokeConnect()方法) 如出一辙, 与outbound 一样, 如果用户没有重写channelActive()  方法,那么就会调用 ChannelInboundHandlerAdapter 的 channelActive()方法：

   ```java
       @Override
       public void channelActive(ChannelHandlerContext ctx) throws Exception {
           ctx.fireChannelActive();
       }
   ```

   同样地, 在 ChannelInboundHandlerAdapter 的 channelActive()中，仅仅调用了 ctx.fireChannelActive()方法，因此就
   会进入 Context.fireChannelActive() -> Connect.findContextInbound() -> nextContext.invokeChannelActive() ->
   nextHandler.channelActive() -> nextContext.fireChannelActive()这样的循环中。同理，tail 本身既实现了
   ChannelInboundHandler 接口，又实现了 ChannelHandlerContext 接口，因此当 channelActive()消息传递到 tail 后，
   会将消息转递到对应的 ChannelHandler 中处理，而 tail 的 handler()返回的就是 tail 本身：

   ```java
      @Override
           public ChannelHandler handler() {
               return this;
           }
   ```

   因此， channelActive 的  inbound 事件最终是在tail 中处理的， 我们看一下他的处理方法

   ```java
      @Override
       public void channelActive(ChannelHandlerContext ctx) throws Exception {
       
       }
   ```

   

TailContext 的 channelActive()  方法是空的, 如果大家自行查看 TailContext 的inbound 处理方法的时候就会发现，他们的实现都是空的, 可见, 如果是inbound, 当用户没有实现自定义的处理器时, 那么默认是不处理的. 下图描述了inbound 事件的传输过程.

![](http://files.luyanan.com//img/20190929153135.png)

### 2.3 Pipeline 事件传播小结

####  outbound 事件总结:

1. OutBound 事件是请求事件(由 connect() 发起一个请求, 并最终由 unsafe 处理这个请求)
2. outbound   事件的发起者是 Channel
3. Outbound 事件的处理者是 unsafe
4. outbound  事件在Pipeline 中的传播方向是 tail -> head.
5. 在ChannelHandler 中处理事件时,如果这个handler 不是最后一个handler, 则需要调用 ctx 的方法(如ctx.connect()) 将此事件继续传播下去. 如果不这样做, 那么此事件的传播就会提前终止.
6. Outbound 事件流：Context.OUT_EVT() -> Connect.findContextOutbound() -> nextContext.invokeOUT_EVT() -> nextHandler.OUT_EVT() -> nextContext.OUT_EVT()

####  inbound 事件总结

1. inbound 事件是通知事件, 当某件事情已经就绪后, 通知上层.
2. inbound 事件发起者是 unsafe
3. inbound 事件处理者是Channel, 如果没有实现自定义的处理方法, 那么inbound 事件默认的处理者是TailContext, 并且其处理方法是空实现.
4. inbound 事件在 pipeline 中传输方向是head ->tail
5. 在 ChannelHandler 中处理事件时，如果这个 Handler 不是最后一个 Handler，则需要调用 ctx.fireIN_EVT()事
   件（如：ctx.fireChannelActive()方法）将此事件继续传播下去。如果不这样做，那么此事件的传播会提前终止。
6. Outbound 事件流：Context.fireIN_EVT() -> Connect.findContextInbound() -> nextContext.invokeIN_EVT() ->
   nextHandler.IN_EVT() -> nextContext.fireIN_EVT().



outbound  和inbound 事件设计上十分相似, 并且Context 与Handler 直接的调用关系也容易混淆,因此我们在阅读这里的源码时, 需要特别的注意.

##  3. Handler 的各种姿势

###  1. ChannelHandlerContext

每个ChannelHandler 被添加到ChannelPipeline 后,都会创建一个 ChannelHandlerContext  并与之创建的ChannelHandler  关联绑定, ChannelHandlerContext 允许 ChannelHandler 与其他的 ChannelHander  实现进行交互, ChannelHandlerContext 不会改变添加到其中的 ChannelHandler, 因此它是安全的. 下图描述了 ChannelHandlerContext、ChannelHandler、ChannelPipeline 的关系.

![](http://files.luyanan.com//img/20190929155228.png)

### 2. Channel 的生命周期

Netty 有一个简单但很强大的状态模型, 并完美映射到 ChannelInboundHandler 的各个方法上, 下面是Channel 的生命周期中的四个不同的状态.



| 状态                  | 描述                                            |
| --------------------- | ----------------------------------------------- |
| channelUnregistered() | Channel 已创建, 还未注册到一个EventLoop         |
| channelRegistered()   | Channel  已经注册到一个EventLoop                |
| channelActive()       | Channel 是活跃状态(连接到某个远端) 可以收发数据 |
| channelInactive()     | Channel 未连接到远端                            |

一个Channel 正常的生命周期如下图所示, 随着状态发生变化相应的事件产生, 这些事件被转发到ChannePipeline 中的ChannelHandler  来触发相应的操作.

![](http://files.luyanan.com//img/20190929160132.png)

### 3. ChannelHandler 常用的API

先看一个Netty 中整个Handler 体系的类关系图

![](http://files.luyanan.com//img/20190929163513.png)

Netty 定义了良好的类型层次结构来标识不同的处理程序类型, 所有的类型的父类是ChannelHandler. ChannelHandler 提供了在其生命周期中添加或从ChannelPipeline 中删除的方法.

| 状态              | 描述                                              |
| ----------------- | ------------------------------------------------- |
| handlerAdded()    | ChannelHandler 添加到实际上下文汇中准备处理事件   |
| handlerRemoved()  | 将ChannelHandler 从实际上下文中删除, 不在处理事件 |
| exceptionCaught() | 处理抛出的异常                                    |

Netty 还提供了一个实现了ChannelHandler 的抽象类ChannelHandlerAdapter。ChannelHandlerAdapter 实现了父类的所有方法, 基本上就是传递事件到ChannelPipeline 中的下一个ChannelHandler  直到结束. 我们也可以直接继承于ChannelHandlerAdapter ,然后重写里面的方法.

###  4 ChannelInboundHandler

ChannelInboundHandler  提供了一些方法再接受或者Channel 状态改变时被调用, 下面是 ChannelInboundHandler 的一些方法:

| 状态                      | 描述                                                 |
| ------------------------- | ---------------------------------------------------- |
| channelRegistered()       | ChannelHandlerContext的Channel被注册到EventLoop      |
| channelUnregistered()     | ChannelHandlerContext的Channel从EventLoop中注销      |
| channelActive()           | ChannelHandlerContext的Channel已激活                 |
| channelInactive           | ChannelHanderContxt的Channel结束生命周期             |
| channelRead               | 从当前Channel的对端读取消息                          |
| channelReadComplete       | 消息读取完成后执行                                   |
| userEventTriggered        | 一个用户事件被触发                                   |
| channelWritabilityChanged | 改变通道的可写状态，可以使用Channel.isWritable()检查 |
| exceptionCaught           | 重写父类ChannelHandler的方法，处理异常               |

Netty 提供了一个实现了 ChannelInboundHandler 接 口 并 继 承 ChannelHandlerAdapter 的 类 ：
ChannelInboundHandlerAdapter。ChannelInboundHandlerAdapter 实现了 ChannelInboundHandler 的所有方法，
作用就是处理消息并将消息转发到 ChannelPipeline 中的下一个 ChannelHandler。ChannelInboundHandlerAdapter
的 channelRead() 方 法 处 理 完 消 息 后 不 会 自 动 释 放 消 息 ， 若 想 自 动 释 放 收 到 的 消 息 ， 可 以 使 用
SimpleChannelInboundHandler，看下面的代码：

```java
public class UnreleaseHandler extends ChannelInboundHandlerAdapter {
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
       //手动释放消息
       ReferenceCountUtil.release(msg);
    }
}
```

SimpleChannelInboundHandler 会自动释放消息：

```java
public class ReleaseHandler extends SimpleChannelInboundHandler<Object> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
     //不需要手动释放
    }
}
```

ChannelInitializer 用来初始化 ChannelHandler，将自定义的各种 ChannelHandler 添加到 ChannelPipeline 中。

