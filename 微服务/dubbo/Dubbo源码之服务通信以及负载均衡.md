# Dubbo源码之服务通信和负载均衡



##  客户端生成的proxy

消费者初始化完成后, 会生成一个proxy, 而这个proxy 本质上是一个动态代理类. 

###  JavassistProxyFactory.getProxy

```jav
 public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```

首先我们来分解一下, 这个invoker 实际上是: `MockClusterWrapper(FailoverCluster(directory))`, 然后通过  `InvokerInvocationHandler`  做了一层包装变成了 `InvokerInvocationHandler(MockClusterWrapper(FailoverCluster(directory)))`.

###  proxy.getProxy

这个方法里面, 会生成一个动态代理的方法, 我们通过 debug 可以看到动态字节码的拼接过程, 它代理了当前这个接口的方法 `info`, 并且方法里面是使用 `handler.invoke` 进行调用的. 

```java
public java.lang.String info(java.lang.String arg0){
Object[] args = new Object[1];
args[0] = ($w)$1;
Object ret = handler.invoke(this, methods[0], args);
return (java.lang.String)ret;
}
```



## 消费端调用的过程

handler 的调用链路为: `InvokerInvocationHandler(MockClusterWrapper(FailoverCluster(directory)))`

###  图解调用链

![](http://files.luyanan.com//img/20191203095407.png)

### InvokerInvocationHandler.invoke

这个方法主要判断当前调用的远程方法, 如果是 `toString`、`hashcode`、`equals`, 就直接返回. 否则, 调用 `invoke.invoke` 进入到 `MockClusterWrapper.invoke`  方法. 

```java
  @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }

        //
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
```



###  MockClusterInvoker.invoke

Mock, 这里面有两个逻辑:

1. 是否客户端强制配置了 mock调用, 那么在这种场景中主要用来解决服务端还没开发好的时候直接使用本地数据进行测试. 
2. 是否出现了异常,如果出现异常则使用配置好的Mock类来实现服务的降级. 

```java
 @Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

        // 从url 中获取 MOCK_KEY 对应的key
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (value.length() == 0 || value.equalsIgnoreCase("false")) {
            //no mock
            // 如果没有配置mock, 则直接传递给下个invoke 调用.
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            //  如果强制为本地调用, 则执行 mockinvoke
            if (logger.isWarnEnabled()) {
                logger.warn("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
            }
            //force:direct mock
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
            try {
                result = this.invoker.invoke(invocation);
            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                }

                if (logger.isWarnEnabled()) {
                    logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                }
                result = doMockInvoke(invocation, e);
            }
        }
        return result;
    }
```





###  AbstractClusterInvoker.invoke

下一个invoke, 应该是进入`FailoverClusterInvoke`,但是这里它又用到了模板方法, 所以直接进去到父类的invoke方法. 

1. 绑定 `attachments`, Dubbo 中, 可以通过 `RpcContext` 上的   `setAttachment`  和  `getAttachment` 在服务消费方和提供方之间进行参数的隐形传递, 所以这段代码中会去绑定 `attachments`. 

   > RpcContext.getContext().setAttachment("index", "1")

2. 通过list 获取 invoke 列表, 这个列表基本可以猜测到从 `directory` 里面获取到, 但是这里面还实现了服务路由的逻辑, 简单来说,就是先拿到invoke 列表,然后通过 route 进行服务路由,筛选出符合路由规则的服务提供者, 

3. ` initLoadBalance` 初始化负载均衡机制

4. 执行  doInvoke

```java
@Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();

        // binding attachments into invocation.
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addAttachments(contextAttachments);
        }

        List<Invoker<T>> invokers = list(invocation);
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }
```



###  initLoadBalance

不用看这个代码, 基本才能猜测到, 会从url 中获取到当前的负载均衡算法, 然后使用SPI机制来获取负载均衡的扩展点, 然后返回一个具体的实现. 

```java
   protected LoadBalance initLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
        if (CollectionUtils.isNotEmpty(invokers)) {
            return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(RpcUtils.getMethodName(invocation), LOADBALANCE_KEY, DEFAULT_LOADBALANCE));
        } else {
            return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(DEFAULT_LOADBALANCE);
        }
    }
```



###  FailoverClusterInvoker.doInvoke

这段代码里面是实现容错的逻辑. 
- 获取重试的次数,并且进行循环
- 获取目标服务, 并记录当前已经调用过的目标服务,防止下次继续将请求发送过去.
- 如果执行成功, 则返回结果. 
- 如果出现异常,判断是否为业务异常,如果是, 则抛出. 否则, 进行下一次重试. 

![](http://files.luyanan.com//img/20191203103553.png)



- 这里的 `invoke` 是`provider` 的一个可调用 `service`的抽象, `invoke` 封装了 `provider` 地址以及 `service` 接口信息. 
- `Directory` 代表多个 `Invoker` , 可以看到 `List<Invoker>`,但是与 `List` 不同的是, 它的值可能是动态变化的, 比如 注册中心推送变更. 
- `Cluster` 将 `Directory` 中的多个 `Invoker` 伪装成一个 `Invoker` , 对上层透明, 伪装过程包含了容错逻辑, 调用失败后, 重试另一个. 
- `LoadBalance` 负责从多个 `Invoker` 中选出一个用于本地调用, 选的过程包含了负载均衡算法, 失败调用后, 需要重选. 



```java
  @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyInvokers = invokers;
        checkInvokers(copyInvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
        int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //Reselect before retry to avoid a change of candidate `invokers`.
            //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            if (i > 0) {
                checkWhetherDestroyed();
                copyInvokers = list(invocation);
                // check again
                checkInvokers(copyInvokers, invocation);
            }
            // 通过 负载均衡获取目标 invoke
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            // 记录已经调用过的服务, 下次调用会进行过滤.
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 服务调用成功, 直接返回结果. 
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + methodName
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers
                            + " (" + providers.size() + "/" + copyInvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                // 如果是业务异常, 直接抛出不进行重试. 
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                // 记录异常信息, 进行下一次循环
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(le.getCode(), "Failed to invoke the method "
                + methodName + " in the service " + getInterface().getName()
                + ". Tried " + len + " times of the providers " + providers
                + " (" + providers.size() + "/" + copyInvokers.size()
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                + Version.getVersion() + ". Last error is: "
                + le.getMessage(), le.getCause() != null ? le.getCause() : le);
    }
```



## 负载均衡

###  select

在调用 `invoke.invoke` 之前, 会需要通过select 选择一个合适的服务进行调用, 而这个选择的过程其实就是负载均衡的实现. 

所有负载均衡实现类都继承自 `AbstractLoadBalance`, 该类实现了`LoadBalance` 接口, 并封装了一些公共的逻辑, 所以在分析负载均衡之前, 先来看看 `AbstractLoadBalance` 的逻辑, 首先来看 负载均衡的入口方法 `select`,如下：

```java
   @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        //  如果 invokers 列表中仅有一个invoker, 直接返回即可. 无需进行负载均衡.
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
        // 调用 doSelect 方法进行负载均衡, 该方法为抽象方法, 由子类实现
        return doSelect(invokers, url, invocation);
    }
```

负载均衡的子类实现由四个, 默认情况下是 `RandomLoadBalance`

###  RandomLoadBalance

```java
 @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Number of invokers
        int length = invokers.size();
        // Every invoker has the same weight?
        boolean sameWeight = true;
        // the weight of every invokers
        int[] weights = new int[length];
        // the first invoker's weight
        int firstWeight = getWeight(invokers.get(0), invocation);
        weights[0] = firstWeight;
        // The sum of weights
        int totalWeight = firstWeight;
        // 下面这个循环有两个作用, 第一是计算总权重 totalWeight
        // 第二是检测每个服务提供者的权重是否相同.
        for (int i = 1; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // save for later use
            weights[i] = weight;
            // Sum
            // 累计权重
            totalWeight += weight;
            // 检测当前服务提供者的权重与上一个服务提供者的权重是否相同.
            // 不相同的话, 则将sameWeight 设置为false
            if (sameWeight && weight != firstWeight) {
                sameWeight = false;
            }
        }
        // 下面的if分支主要用于获取随机数, 并计算随机数落在哪个区间.
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            // 随机获取一个 [0,totalWeight] 区间内的数字
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            //  循环让 offset 数减去服务者权重值, 当offset 小于0时, 返回相应的invoke
            // 举例说明一下，我们有 servers = [A, B, C]，weights = [5, 3, 2]，offset= 7。
            // 第一次循环，offset - 5 = 2 > 0，即 offset > 5，
            // 表明其不会落在服务器 A 对应的区间上。
            // 第二次循环，offset - 3 = -1 < 0，即 5 < offset < 8，
            // 表明其会落在服务器 B 对应的区间上
            for (int i = 0; i < length; i++) {
                // 让随机值 offset 减去权重值
                offset -= weights[i];
                if (offset < 0) {
                    // 返回相应的invoker
                    return invokers.get(i);
                }
            }
        }
        // If all invokers have the same weight value or totalWeight=0, return evenly.
        // 如果所有服务提供者权重值相同, 此时直接随机返回一个即可. 
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }
```

通过 `RegistryDirectory` 中获取的invoke 是什么呢?这个很重要, 因为它决定了接下来的调用过程, 这个时候我们需要去了解这个invoke 是在哪里被初始化的. 

## 可调用的Invoker初始化过程

###  RegistryDirectory

在 `RegistryDirectory` 中有一个成员属性, 保存了服务地址对应的invoke 信息. 

> private volatile Map> urlInvokerMap;

### toInvokers

这个invoke 是动态的, 基于注册中心的变化而变化的, 它的初始化过程的链路是 `RegistryDirectory.notify->refreshInvoker->toInvokers` 下面你的这段代码中： 

```java
     if (invoker == null) { // Not in the cache, refer again
                try {
                    boolean enabled = true;
                    if (url.hasParameter(DISABLED_KEY)) {
                        enabled = !url.getParameter(DISABLED_KEY, false);
                    } else {
                        enabled = url.getParameter(ENABLED_KEY, true);
                    }
                    if (enabled) {
                        invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
                }
                if (invoker != null) { // Put new invoker in cache
                    newUrlInvokerMap.put(key, invoker);
                }
            } else {
                newUrlInvokerMap.put(key, invoker);
            }
```



是基于 `protocol.refer` 来构建 的invoke, 并且使用 `InvokerDelegate` 进行了委托, 在 dubboprotoco 中, 是这样构建 invoke的. 返回的是一个 `DubboInvoker` 对象. 

```java
@Override
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws
RpcException {
optimizeSerialization(url);
// create rpc invoker.
DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url,
getClients(url), invokers);
invokers.add(invoker);
return invoker;
}
```



所以这个invoker 应该是 `InvokerDelegate(ProtocolFilterWrapper(ListenerInvokerWrapper(DubboInvoker())`

ProtocolFilterWrapper->  这是一个 invoker 的过滤链路

ListenerInvokerWrapper-> 这里面暂时没做任何的实现

所以我们可以直接看到DubboInvoker 这个类里面

## DubboInvoker

###  AbstractInvoker.invoke

这里面也是对 Invocation的attachments进行处理，把attachment加入到Invocation中

这里的attachment , 实际上是目标服务的接口信息以及版本信息. 

###  DubboInvoker.doInvoker

这里面看到一个很熟悉的东西, 就是 ExchangeClient, 这个是客户端和服务端之间的连接. 

然后如果当前方法有返回值的话, 也就是 `isOneway=false`, 则执行else 逻辑, 然后通过异步的形式进行通信. 

```java
@Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        // 将目标方法以及版本号作为参数放入到invocation 中.
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);

        // 获取客户端连接.
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            // 判断方法是否有返回值
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            // 获取超时时间,默认为1s
            int timeout = getUrl().getMethodParameter(methodName, TIMEOUT_KEY, DEFAULT_TIMEOUT);
            // 如果没有返回值
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
                AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
                CompletableFuture<Object> responseFuture = currentClient.request(inv, timeout);
                responseFuture.whenComplete((obj, t) -> {
                    if (t != null) {
                        asyncRpcResult.completeExceptionally(t);
                    } else {
                        asyncRpcResult.complete((AppResponse) obj);
                    }
                });
                RpcContext.getContext().setFuture(new FutureAdapter(asyncRpcResult));
                return asyncRpcResult;
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```



###   currentClient.request

currentClient 实际是一个 `ReferenceCountExchangeClient(HeaderExchangeClient())`. 所以它的调用链路是 `ReferenceCountExchangeClient->HeaderExchangeClient->HeaderExchangeChannel->(request方 法)`,  最终将构建好的RpcInvocation， 组装到一个request 对象中进行传递. 

```java
 @Override
    public CompletableFuture<Object> request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        // 创建请求对象
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setData(request);
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
        try {
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```

channel.send的调用链路

> AbstractPeer.send ->AbstractClient.send->NettyChannel.send

通过 `NioSocketChannel` 把消息发送出去. 

```java
ChannelFuture future = channel.writeAndFlush(message);
```



##  服务端接收消息的处理流程. 

客户端把消息发送出去之后, 服务端会收到消息,然后把执行的结果返回到客户端

###  客户端接收到消息

服务端这边接受消息的处理链路, 也比较复杂, 我们回到NettyServer 中创建io 的过程. 

```java
protected void doOpen() throws Throwable {
        bootstrap = new ServerBootstrap();

        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        // FIXME: should we use getTimeout()?
                        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // bind
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();

    }
```



handler 配置的是  `nettyServerHandler`

server-idle-handler  表示心跳处理机制. 

> final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);

Handler 与Servlet中的filter 很像, 通过Handler 可以完成通讯报文的解码编码, 拦截指定的报文, 统一对日志错误进行处理, 统一对请求进行计数、控制Handler 执行与否. 

###  handler.channelRead()

服务端收到读的请求时, 会进入到这个方法. 

接着通过 `handler.received` 来处理msg, 这个handler 的链路很长, 比较复杂，我们需要逐步剖析. 

```java
 @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        try {
            handler.received(channel, msg);
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.channel());
        }
    }
```

> handler->MultiMessageHandler->HeartbeatHandler->AllChannelHandler->DecodeHandler- >HeaderExchangeHandler->

最后进入这个方法->  `DubboProtocol$requestHandler(receive)`

`MultiMessageHandler`: 复合消息处理

`HeartbeatHandler`: 心跳消息处理， 接受心跳并发送心跳响应 

`AllChannelHandler`: 业务线程转换处理器, 把接受到的消息封装成 `ChannelEventRunnable` 可执行任务给线程池处理. 

`DecodeHandler`:l 业务解码处理器



![](http://files.luyanan.com//img/20191203134829.png)

###  HeaderExchangeHandler.received

交互层请求响应处理, 有三种处理方式: 

1. `handlerRequest`: 双向请求
2. `handler.received`  单向请求
3. `handleResponse` 响应消息

```java
  @Override
    public void received(Channel channel, Object message) throws RemotingException {
        channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
        final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        try {
            if (message instanceof Request) {
                // handle request.
                Request request = (Request) message;
                if (request.isEvent()) {
                    handlerEvent(channel, request);
                } else {
                    if (request.isTwoWay()) {
                        handleRequest(exchangeChannel, request);
                    } else {
                        handler.received(exchangeChannel, request.getData());
                    }
                }
            } else if (message instanceof Response) {
                handleResponse(channel, (Response) message);
            } else if (message instanceof String) {
                if (isClientSide(channel)) {
                    Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                    logger.error(e.getMessage(), e);
                } else {
                    String echo = handler.telnet(channel, (String) message);
                    if (echo != null && echo.length() > 0) {
                        channel.send(echo);
                    }
                }
            } else {
                handler.received(exchangeChannel, message);
            }
        } finally {
            HeaderExchangeChannel.removeChannelIfDisconnected(channel);
        }
    }

```



###  ExchangeHandler.reply

接着进入到 `ExchangeHandler.reply` 这个方法中:

- 把 `message` 转换为 `invocation`
- 调用 `getInvoker` 获取一个invoker 对象
- 然后通过 `Result result = invoker.invoke(inv)`  进行调用



我们发现 `ExchangeHandler.reply` 其实是一个接口方法, 其实实现是在DubboProtocol 里面, 

```java
   private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

        @Override
        public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

            if (!(message instanceof Invocation)) {
                throw new RemotingException(channel, "Unsupported request: "
                        + (message == null ? null : (message.getClass().getName() + ": " + message))
                        + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
            }

            Invocation inv = (Invocation) message;
            Invoker<?> invoker = getInvoker(channel, inv);
            // need to consider backward-compatibility if it's a callback
            if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || !methodsStr.contains(",")) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                if (!hasMethod) {
                    logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                            + " not found in callback service interface ,invoke will be ignored."
                            + " please update the api interface. url is:"
                            + invoker.getUrl()) + " ,invocation is :" + inv);
                    return null;
                }
            }
            RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
            Result result = invoker.invoke(inv);
            return result.completionFuture().thenApply(Function.identity());
        }

        @Override
        public void received(Channel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                reply((ExchangeChannel) channel, message);

            } else {
                super.received(channel, message);
            }
        }

        @Override
        public void connected(Channel channel) throws RemotingException {
            invoke(channel, ON_CONNECT_KEY);
        }

        @Override
        public void disconnected(Channel channel) throws RemotingException {
            if (logger.isDebugEnabled()) {
                logger.debug("disconnected from " + channel.getRemoteAddress() + ",url:" + channel.getUrl());
            }
            invoke(channel, ON_DISCONNECT_KEY);
        }

        private void invoke(Channel channel, String methodKey) {
            Invocation invocation = createInvocation(channel, channel.getUrl(), methodKey);
            if (invocation != null) {
                try {
                    received(channel, invocation);
                } catch (Throwable t) {
                    logger.warn("Failed to invoke event method " + invocation.getMethodName() + "(), cause: " + t.getMessage(), t);
                }
            }
        }

        private Invocation createInvocation(Channel channel, URL url, String methodKey) {
            String method = url.getParameter(methodKey);
            if (method == null || method.length() == 0) {
                return null;
            }

            RpcInvocation invocation = new RpcInvocation(method, new Class<?>[0], new Object[0]);
            invocation.setAttachment(PATH_KEY, url.getPath());
            invocation.setAttachment(GROUP_KEY, url.getParameter(GROUP_KEY));
            invocation.setAttachment(INTERFACE_KEY, url.getParameter(INTERFACE_KEY));
            invocation.setAttachment(VERSION_KEY, url.getParameter(VERSION_KEY));
            if (url.getParameter(STUB_EVENT_KEY, false)) {
                invocation.setAttachment(STUB_EVENT_KEY, Boolean.TRUE.toString());
            }

            return invocation;
        }
    };
```



###  getInvoker

这里面获取到一个invoker 的实现

> DubboExporter exporter = (DubboExporter) exporterMap.get(serviceKey);

这段代码非常熟悉, exporterMap 是在服务发布的过程中, 保存的 invoker吗? 而key, 就是对应的 `interface:port`

```java
Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        boolean isCallBackServiceInvoke = false;
        boolean isStubServiceInvoke = false;
        int port = channel.getLocalAddress().getPort();
        String path = inv.getAttachments().get(PATH_KEY);

        // if it's callback service on client side
        isStubServiceInvoke = Boolean.TRUE.toString().equals(inv.getAttachments().get(STUB_EVENT_KEY));
        if (isStubServiceInvoke) {
            port = channel.getRemoteAddress().getPort();
        }

        //callback
        isCallBackServiceInvoke = isClientSide(channel) && !isStubServiceInvoke;
        if (isCallBackServiceInvoke) {
            path += "." + inv.getAttachments().get(CALLBACK_SERVICE_KEY);
            inv.getAttachments().put(IS_CALLBACK_SERVICE_INVOKE, Boolean.TRUE.toString());
        }

        String serviceKey = serviceKey(port, path, inv.getAttachments().get(VERSION_KEY), inv.getAttachments().get(GROUP_KEY));
        DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

        if (exporter == null) {
            throw new RemotingException(channel, "Not found exported service: " + serviceKey + " in " + exporterMap.keySet() + ", may be version or group mismatch " +
                    ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() + ", message:" + inv);
        }

        return exporter.getInvoker();
    }
```



###  exporterMap

```java
Map<String, Exporter<?>> exporterMap = new ConcurrentHashMap<String, Exporter<?
>>();
```



在服务发布时, 实际上就是把invoker 包装成了DubboExpoter, 然后放入到了 exporterMap 中了. 

```java
  @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        // 获取服务标识, 理解成服务坐标也行, 由服务组名, 服务名,服务版本号以及端口组成,比如:
        // //${group}/com.example.ISayHelloService:${version}:20880
        URL url = invoker.getUrl();

        // export service.
        String key = serviceKey(url);
        // 创建DubboExporter
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        // 将 key,exporter    键值对放入缓存中
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }

            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        // 启动服务
        openServer(url);
        optimizeSerialization(url);

        return exporter;
    }
```



###  invoker.invoke(inv);

接着调用 `invoker.invoke(inv);`

那么在回忆一下, 此时的invoker 是一个什么呢? 

> invoker=ProtocolFilterWrapper(InvokerDelegate(DelegateProviderMetaDataInvoker(AbstractProxy Invoker)))

最后一定会进入到这个代码中. 

###  AbstractProxyInvoker

在 `AbstractProxyInvoker` 里面, `doInvoker` 本质上调用的是 `wrapper.invokeMethod()`. 

```java
    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```



而 Wrapper 是一个动态代理类, 它的定义是这样的, 最后通过调用 `w.info()` 方法进行处理

```java
public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws
java.lang.reflect.InvocationTargetException {
com.dubbo.spring.UserApi w;
try {
w = ((com.dubbo.spring.UserApi) $1);
} catch (Throwable e) {
throw new IllegalArgumentException(e);
}
try {
if ("info".equals($2) && $3.length == 1) {
return ($w) w.info((java.lang.String) $4[0]);
}
} catch (Throwable e) {
throw new java.lang.reflect.InvocationTargetException(e);
}
throw new org.apache.dubbo.common.bytecode.NoSuchMethodException("Not
found method \"" + $2 + "\" in class
ccom.dubbo.spring.UserApi.");
}
```

到此为止, 服务端的处理过程就分析完了. 

