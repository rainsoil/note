#  Dubbo 服务注册以及服务消费源码分析

## Invoker 是什么?

服务的发布分为三个阶段: 

1. 第一个阶段会创造一个 invoker
2. 第二个阶段会把经历过一系列处理的 invoker(各种包装), 在 在DubboProtocol 中保存到 `exporterMap` 中.
3.  第三个阶段把dubbo协议的url 注册到注册中心上去. 

我们来简单看看invoker 到底是一个啥东西? 

incoker 是Dubbo 领域模型中非常重要的一个概念, 和 `ExtensionLoader` 的重要性是一样的, 如果invoker 没有搞懂, 那么不算是看懂了Dubbo 的源码. 我们继续回到  `ServiceConfig`中`export` 的代码,以这个作为入口来分析前面 export出去的invoker 到底是什么东西? 

 ```java
       Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
 ```



### ProxyFacotory.getInvoker

这是一个代理工程, 用来生成invoker, 从它的定义来看, 它是一个自适用扩展点, 看到这样的扩展点, 我们几乎可以不假思索的想到它会存在这样一个动态适配类. 

```java
 private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
```



### ProxyFactory

这个方法的简单解读为: 它是一个SPI扩展点, 并且默认的扩展实现是`javassist`, 这个接口中有三个方法, 而且都是加了 `@Adaptive` 的自适应扩展点, 所以如果调用`getInvoker()` 方法, 应该会返回一个 `ProxyFactory$Adaptive`. 

```java
@SPI("javassist")
public interface ProxyFactory {

    /**
     * create proxy.
     *
     * @param invoker
     * @return proxy
     */
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    /**
     * create proxy.
     *
     * @param invoker
     * @return proxy
     */
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

    /**
     * create invoker.
     *
     * @param <T>
     * @param proxy
     * @param type
     * @param url
     * @return invoker
     */
    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

}
```



###  ProxyFactory$Adaptive

这个自适用扩展点, 做了两件事情. 

- 通过 `ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(extName)` 获取了一个指定名称的扩展点. 
- 在 `dubbo-rpc-api/resources/META-INF/com.alibaba.dubbo.rpc.ProxyFactory` 中定义了  `javassis=JavassisProxyFactory`,调用JavassisProxyFactory` 的getInvoker方法. 

```java
public class ProxyFactory$Adaptive implements org.apache.dubbo.rpc.ProxyFactory {
    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0);
    }

    public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0, boolean arg1) throws org.apache.dubbo.
            rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null)
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getProxy(arg0, arg1);
    }

    public org.apache.dubbo.rpc.Invoker getInvoker(java.lang.Object arg0, java.lang.Class arg1, org.apache.dubbo.common.URL arg2) throws org.apache.dubbo.rpc.RpcException {
        if (arg2 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg2;
        String extName = url.getParameter("proxy", "javassist");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
        org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
        return extension.getInvoker(arg0, arg1, arg2);
    }


}

```



###  JavassistProxyFactory.getInvoker

javassist 是一个动态类库, 用来实现动态代理的. 

proxy: 接口的实现 `com.dubbo.spring.server.UserApiImpl`

type: 接口全称: `com.dubbo.spring.userApi`

url: 协议地址: registry://...

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



### javassist生成的动态代理代码

通过断点的方式(Wrapper258行), 在 `Wrapper.getWrapper` 中的 `makeWrappe`, 会创建一个动态代理, 核心的方法: `invokeMethod` 代码如下: 

```java
public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws
java.lang.reflect.InvocationTargetException {
com.dubbo.spring.userApi w;
try {
w = ((com.dubbo.spring.userApi) $1);
} catch (Throwable e) {
throw new IllegalArgumentException(e);
}
try {
if ("sayHello".equals($2) && $3.length == 1) {
return ($w) w.info((java.lang.String) $4[0]);
}
} catch (Throwable e) {
throw new java.lang.reflect.InvocationTargetException(e);
}
throw new org.apache.dubbo.common.bytecode.NoSuchMethodException("Not
found method \"" + $2 + "\" in class
com.dubbo.spring.userApi.");
}
```



构建好了代理类后, 返回一个 `AbstractproxyInvoker`, 它返回了一个 `doInvoker` 方法, 这个方法似乎看到了dubbo 消费者调用过来的时候触发的影子, 因为 `wrapper.invokeMethod` 本质上就是触发上面动态代理类的方法 `invokeMethod`.

```java
return new AbstractProxyInvoker<T>(proxy, type, url) {
@Override
protected Object doInvoke(T proxy, String methodName,
Class<?>[] parameterTypes,
Object[] arguments) throws Throwable {
return wrapper.invokeMethod(proxy, methodName, parameterTypes,
arguments);
}
```

所以, 简单总结一下 invoke 本质上应该是一个代理, 经过层层包装后 最终进行了发布. 当消费者发起请求的时候, 会获取到这个invoke 进行调用. 

最终发布出去的invoke , 也不是一个单纯的代理, 也是经过层层包装的. 
> InvokerDelegate(DelegateProviderMetaDataInvoker(AbstractProxyInvoker()))

## 服务注册流程.

关于服务发布这一条线完成之后, 再来了解一下服务注册的过程, 希望大家还记得我们之所以走到这一步, 是因为我们在 `RegistryProtoco` 这个类中, 看到了服务发布的流程. 



```java

 @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 这里获取的是zookeeper 注册中心的url zookeeper://ip:port
        URL registryUrl = getRegistryUrl(originInvoker);
        // url to export locally
        //  这里是获得服务提供者的url   dubbo://ip:port
        URL providerUrl = getProviderUrl(originInvoker);

        // Subscribe the override data
        // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
        //  the same service. Because the subscribed is cached key with the name of the service, it causes the
        //  subscription information to cover.
        // 订阅 override数据, 在admin 控制台可以针对服务进行治理, 比如修改权重、修改路由机制等, 当有注册中心由此服务的覆盖配置注册进行的时候,
        // 推送消息给提供者, 重新暴露服务.
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
        //export invoker
        // 这里交给具体的协议去暴露服务
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // url to registry
        // 根据invoker 中的url 获取registry实例 , zookeeperRegistry
        final Registry registry = getRegistry(originInvoker);
        // 获取要注册到注册中心的url : dubbo://ip:port
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
        //to judge if we need to delay publish
        boolean register = registeredProviderUrl.getParameter("register", true);
        // 是否配置了注册中心, 如果是,则需要配置
        if (register) {
            // 注册到注册中心的URL
            register(registryUrl, registeredProviderUrl);
            providerInvokerWrapper.setReg(true);
        }

        // 注册中心的订阅
        // Deprecated! Subscribe to override rules in 2.6.x or before.
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);
        //Ensure that a new exporter instance is returned every time export
        // 保存每次export 都返回一个新的exporter 实例
        return new DestroyableExporter<>(exporter);
    }
```



### 服务注册核心代码

从export 方法中处理出来的部分代码, 就是服务注册的流程. 

```java
  // url to registry
        // 根据invoker 中的url 获取registry实例 , zookeeperRegistry
        final Registry registry = getRegistry(originInvoker);
        // 获取要注册到注册中心的url : dubbo://ip:port
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
        //to judge if we need to delay publish
        boolean register = registeredProviderUrl.getParameter("register", true);
        // 是否配置了注册中心, 如果是,则需要配置
        if (register) {
            // 注册到注册中心的URL
            register(registryUrl, registeredProviderUrl);
            providerInvokerWrapper.setReg(true);
        }
```



### getRegistry

1. 将url转换为对应配置的注册中心的具体协议. 
2. 根据具体协议, 从registryFactory 中获取指定的注册中心实现. 

那么这个 registryFactory 具体是怎么赋值的呢? 

```java
   private Registry getRegistry(final Invoker<?> originInvoker) {
        // 将url 转换为配置的具体协议, 比如 zookeeper://ip:port, 这样后续获取的注册中心就是基于zk的实现
        URL registryUrl = getRegistryUrl(originInvoker);
        return registryFactory.getRegistry(registryUrl);
    }
```



在 RegistryProtoco 中存在这样一段代码, 很明显这是通过依赖注入来实现的扩展点. 

```java
    public void setRegistryFactory(RegistryFactory registryFactory) {
        this.registryFactory = registryFactory;
    }
```

按照扩展点的加载规则, 我们可以先看看  `/META-INF/dubbo/internal` 路径下找到 RegistryFactory的配置文件, 这个歌factory 有多个扩展点的实现. 

```java
dubbo=org.apache.dubbo.registry.dubbo.DubboRegistryFactory
multicast=org.apache.dubbo.registry.multicast.MulticastRegistryFactory
zookeeper=org.apache.dubbo.registry.zookeeper.ZookeeperRegistryFactory
redis=org.apache.dubbo.registry.redis.RedisRegistryFactory
consul=org.apache.dubbo.registry.consul.ConsulRegistryFactory
etcd3=org.apache.dubbo.registry.etcd.EtcdRegistryFactory

```

接着, 找到RegistryFactory 的实现, 发现它里面有个自适应的方法, 根据url 中的protocol 传入的值进行适配. 

```java
@SPI("dubbo")
public interface RegistryFactory {

    /**
     * Connect to the registry
     * <p>
     * Connecting the registry needs to support the contract: <br>
     * 1. When the check=false is set, the connection is not checked, otherwise the exception is thrown when disconnection <br>
     * 2. Support username:password authority authentication on URL.<br>
     * 3. Support the backup=10.20.153.10 candidate registry cluster address.<br>
     * 4. Support file=registry.cache local disk file cache.<br>
     * 5. Support the timeout=1000 request timeout setting.<br>
     * 6. Support session=60000 session timeout or expiration settings.<br>
     *
     * @param url Registry address, is not allowed to be empty
     * @return Registry reference, never return empty value
     */
    @Adaptive({"protocol"})
    Registry getRegistry(URL url);

}
```

###  RegistryFactory$Adaptive

由于在前面的代码中, url 中的protocol 已经改成了 zookeeper, 那么这个时候根据zookeeper 获取的spi 扩展点应该是 `ZookeeperRegistryFactory`

```java
import org.apache.dubbo.common.extension.ExtensionLoader;
public class RegistryFactory$Adaptive implements
org.apache.dubbo.registry.RegistryFactory {
public org.apache.dubbo.registry.Registry
getRegistry(org.apache.dubbo.common.URL arg0) {
if (arg0 == null) throw new IllegalArgumentException("url == null");
org.apache.dubbo.common.URL url = arg0;
String extName = ( url.getProtocol() == null ? "dubbo" :
url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get
extension (org.apache.dubbo.registry.RegistryFactory) name from url (" +
url.toString() + ") use keys([protocol])");
org.apache.dubbo.registry.RegistryFactory extension =
(org.apache.dubbo.registry.RegistryFactory)ExtensionLoader.getExtensionLoader(or
g.apache.dubbo.registry.RegistryFactory.class).getExtension(extName);
return extension.getRegistry(arg0);
}
}

```



###  ZookeeperRegistryFactory

而这个类中并没有 `getRegistry` 方法, 而是在他的父类 `AbstractRegistryFactory`. 

- 从缓存REGISTRIES
- 如果不存在, 则创建 Registry. 

```java
 @Override
    public Registry getRegistry(URL url) {
        url = URLBuilder.from(url)
                .setPath(RegistryService.class.getName())
                .addParameter(INTERFACE_KEY, RegistryService.class.getName())
                .removeParameters(EXPORT_KEY, REFER_KEY)
                .build();
        String key = url.toServiceStringWithoutResolving();
        // Lock the registry access process to ensure a single instance of the registry
        LOCK.lock();
        try {
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            //create registry by spi/ioc
            //  创建注册中心
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // Release the lock
            LOCK.unlock();
        }
    }
```



### createRegistry(url)

创建一个 ZookeeperRegistry, 把url 和zookeepertransporter 作为参数传入. 

zookeepeTransporter 这个属性也是基于依赖注入来赋值的, 具体流程就不再进行分析,这个的值应该是 `CuratorZookeeperTransporter`,表示具体是使用说明框架来和zk 产生连接. 

```java
    @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
```



### ZookeeperRegistry

这个方法中使用 `CuratorZookeeperTransport` 来实现zk 的连接. 

```java
 public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        super(url);
        if (url.isAnyHost()) {
            throw new IllegalStateException("registry address == null");
        }
        // 获取group 的名称
        String group = url.getParameter(GROUP_KEY, DEFAULT_ROOT);
        if (!group.startsWith(PATH_SEPARATOR)) {
            group = PATH_SEPARATOR + group;
        }
        this.root = group;
        // 产生一个zookeeper 连接
        zkClient = zookeeperTransporter.connect(url);
        // 添加zookeeper 状态变化事件
        zkClient.addStateListener(state -> {
            if (state == StateListener.RECONNECTED) {
                try {
                    recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }
        });
    }
```

### RegistryProtocolregistry.register(registedProviderUrl);

继续往下分析 会调用  `egistry.register(registedProviderUrl)` 去将dubbo:// 的协议地址去注册到zookeeper 上. 

这个方法会调用  `FailbackRegistry` 类中的 registry, 为什么呢? 因为ZookeeperRegisry 这个类中并没有registry这个方法, 但是它的父类 FailbackRegistry 中存在这个方法, 而这个类又重写了 `AbstractRegistry` 类中的 registry 方法, 所以我们可以直接定位到 FailbackRegistry  这个类中的registry方法中. 

```java
   public void register(URL registryUrl, URL registeredProviderUrl) {
        Registry registry = registryFactory.getRegistry(registryUrl);
        registry.register(registeredProviderUrl);
    }
```



### FailbackRegistry.register

- FailbackRegistry 从名字来看, 是一个失败重试机制. 
- 调用父类的registry方法, 将当前url 添加到缓存集合中. 

调用doRegister 方法, 这个方法是一个抽象方法, 会由 ZookeeperRegistry 子类实现. 

```java
 @Override
    public void register(URL url) {
        super.register(url);
        removeFailedRegistered(url);
        removeFailedUnregistered(url);
        try {
            // Sending a registration request to the server side
            // 调用子类实现真正的服务注册, 把url 注册到zk上.
            doRegister(url);
        } catch (Exception e) {
            Throwable t = e;

            // If the startup detection is opened, the Exception is thrown directly.
            // 如果开启了启动时检测, 则直接抛出异常
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }

            // Record a failed registration request to a failed list, retry regularly
            // 将失败了的注册请求记录到失败列表, 定时重试
            addFailedRegistered(url);
        }
    }
```



###  ZookeeperRegistry.doRegister

最终调用curator 的客户端将服务地址注册到zk中. 

```java
  @Override
    public void doRegister(URL url) {
        try {
            zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

```



## 服务消费

###  思考服务消费应该要具备的逻辑

如果要实现服务的消费, 需要实现以下需求:

1. 生成远程服务的代理. 
2. 获取目标服务的url 地址
3. 实现远程网络通信. 
4. 实现负载均衡
5. 实现集群容错. 

![](http://files.luyanan.com//img/20191202141521.png)

### 服务的消费

消费端的代码是从下面这段代码开始的

> <dubbo:reference id="xxxService" interface="xxx.xxx.Service"/>

注解的方式的初始化入口是:

> ReferenceAnnotationBeanPostProcessor->ReferenceBeanInvocationHandler.init- >ReferenceConfig.get() 获得一个远程代理类



###  ReferenceConfig.get

```java
  public synchronized T get() {
        // 修改和检查配置
        checkAndUpdateSubConfigs();

        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        // 如果当前接口的远程代理引用为空, 则进行初始化.
        
        if (ref == null) {
            init();
        }
        return ref;
    }
```



###  init

初始化的过程, 和服务发布的过程类似, 会有很多的判断以及参数的组装, 我们只需要关注 createProxy, 创建代理类的方法. 

```java
 private void init() {
        if (initialized) {
            return;
        }
     ...
       ref = createProxy(map);

        String serviceKey = URL.buildKey(interfaceName, group, version);
        ApplicationModel.initConsumerModel(serviceKey, buildConsumerModel(serviceKey, attributes));
        initialized = true;
    }
```

###  createProxy(map)

代码比较长, 但是逻辑相对比较清晰. 

1. 判断是否为本地调用, 如果是, 则使用injvm 协议进行调用. 
2. 判断是否为点对点调用, 如果是则将url 保存到url 集合中, 如果url 为1, 进入步骤4, 如果urls>1, 则执行5
3. 如果是配置了注册中心, 遍历注册中心, 把url 添加到urls 集合中, url 为1, 进入步骤4, 如果urls >1, 执行步骤5. 
4. 直联构建 invoker
5. 构建 invokers集合, 通过cluster合并多个invoker
6. 最后调用 proxyFactory  生成代理类. 

```java
  @SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        // 判断是否在用一个jvm 进程中调用.

        if (shouldJvmRefer(map)) {
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            invoker = REF_PROTOCOL.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            urls.clear(); // reference retry init will add url to urls, lead to OOM
            // 如果url 不为空, 说明是点对点通信.
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                        // 检测url 协议是否为registry, 若是, 表明用户想使用指定的注册中心.
                        if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            // 将map 转换为查询字符串, 并作为refer 参数的值添加到url 中.
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            // 合并url, 移除服务提供者的一些配置(这些配置来源于用户配置的url属性. )
                            // 比如线程池相关配置,并保留服务提供者的部分配置, 比如版本、group、时间戳等.
                            // 最后将合并后的配置设置为url, 查询字符串.
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // assemble URL from register center's configuration
                // if protocols not injvm checkRegistry
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                    // 校验注册中心的配置以及是否有必要从配置中心组装url, 这里的代码实现和服务端类似.
                    // 也是根据注册中心进行解析得到URL, 这里的URL 肯定也是: registry://ip:port/org.apache.dubbo.service.RegsitryService
                    checkRegistry();
                    List<URL> us = loadRegistries(false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            URL monitorUrl = loadMonitor(u);
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    // 如果没有配置注册中心, 则报错.
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }

            // 如果值只配置了一个注册中心或者一个服务提供者,直接使用REF_PROTOCOL.refer
            if (urls.size() == 1) {
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                // 遍历urls 生成多个invoker
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
                // 如果registryURL 不为空, 构建静态directory
                if (registryURL != null) { // registry url is available
                    // use RegistryAwareCluster only when register's CLUSTER is available
                    // 使用RegistryAwareCluster
                    URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
                    // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                    // 通过 Cluster将多个invoker合并
                    //RegistryAwareClusterInvoker(StaticDirectory) ->
                    //FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                    invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { // not a registry url, must be direct invoke.
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }
// 检查invoker 的有效性. 
        if (shouldCheck() && !invoker.isAvailable()) {
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        /**
         * @since 2.7.0
         * ServiceData Store
         */
        MetadataReportService metadataReportService = null;
        if ((metadataReportService = getMetadataReportService()) != null) {
            URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
            metadataReportService.publishConsumer(consumerURL);
        }
        // create service proxy
        return (T) PROXY_FACTORY.getProxy(invoker);
    }
```



### REF_PROTOCOL.refer(interfaceClass, url)

这里通过指定的协议来调用refer 生成一个invoker 对象, invoker 前面讲过, 它是一个代理对象, 那么在当前的消费端而言, invoker 主要用于执行远程调用. 

这个protocol , 又是一个自适应扩展点, 它得到的是一个 `Protocol$Adaptive`

```java
Protocol refprotocol =
ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()

```

这段代码中, 根据当前的协议url, 得到一个指定的扩展点,传递进来的参数中, 协议地址为 `registry://`, 所以我们可以直接定位到 REF_PROTOCOL.refer 代码. 

###  Protocol$Adaptive中的refer方法

根据当前的协议扩展名registry,  获取一个被包装过 的RegistryProtocol

```java
public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0,
org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
if (arg1 == null) throw new IllegalArgumentException("url == null");
org.apache.dubbo.common.URL url = arg1;
String extName = ( url.getProtocol() == null ? "dubbo" :
url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get
extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ")
use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension =
(org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dub
bo.rpc.Protocol.class).getExtension(extName);
return extension.refer(arg0, arg1);
}

```



###  RegistryProtocol.refer

这里面的代码逻辑比较简单. 

- 组装注册中心协议的url
- 判断是否配置  legroup, 如果有，则 `cluster=getMergeableCluster()`, 构建invoker. 
- doRefer 构建invoker

```java
 @Override
    @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        // 根据配置的协议, 生成注册中心的url : zookeeper://
        url = URLBuilder.from(url)
                .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
                .removeParameter(REGISTRY_KEY)
                .build();
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // group="a,b" or group="*"
        // 判断group参数, 根据group 决定 cluster 的类型
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }
```



### doRefer

doRefer 里面就稍微复杂一些, 涉及到比较多的东西, 我们先关注主线. 

- 构建一个RegistryDirectory
- 构建一个 consumer://协议的地址注册到注册中心
- 订阅 zookeeper 中节点的变化. 
- 调用  cluster.join方法. 

```java
 private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        // RegistryDirectory 初始化
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        // 注册consumer://协议的url
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
            directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
            registry.register(directory.getRegisteredConsumerUrl());
        }
        directory.buildRouterChain(subscribeUrl);
        // 订阅事件监听
        directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));

        // 构建 invoker
        Invoker invoker = cluster.join(directory);
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```





###  Cluster是什么

我们只关注一下  invoker 这个代理类的创建过程, 其他的就暂不关心.

> Invoker invoker = cluster.join(directory);

cluster 其实是在  RegistryProtocol 中通过set方法完成依赖注入的, 并且, 它还是一个被包装的. 

```java
  public void setCluster(Cluster cluster) {
        this.cluster = cluster;
    }
```

cluster   扩展点的定义, 由于它是一个自适应扩展点, 那么就会动态生成一个  `Cluster$Adaptive` 的动态代理类. 

```java
@SPI(FailoverCluster.NAME)
public interface Cluster {

    /**
     * Merge the directory invokers to a virtual invoker.
     *
     * @param <T>
     * @param directory
     * @return cluster invoker
     * @throws RpcException
     */
    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;

}
```



###  Cluster$Adaptive

在动态适配的类中会基于extName, 选择一个合适的扩展点进行适配, 由于默认情况下 `cluster:failover`,所以`getExtension("failover")`  理论情况下会返回 `FailOverCluster`. 但实际上, 这里做了包装 `MockClusterWrapper（FailOverCluster）`. 

```java
public class Cluster$Adaptive implements org.apache.dubbo.rpc.cluster.Cluster {
public org.apache.dubbo.rpc.Invoker
join(org.apache.dubbo.rpc.cluster.Directory arg0) throws
org.apache.dubbo.rpc.RpcException {
if (arg0 == null) throw new
IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument ==
null");
if (arg0.getUrl() == null) throw new
IllegalArgumentException("org.apache.dubbo.rpc.cluster.Directory argument
getUrl() == null");
org.apache.dubbo.common.URL url = arg0.getUrl();
String extName = url.getParameter("cluster", "failover");
if(extName == null) throw new IllegalStateException("Failed to get
extension (org.apache.dubbo.rpc.cluster.Cluster) name from url (" +
url.toString() + ") use keys([cluster])");
org.apache.dubbo.rpc.cluster.Cluster extension =
(org.apache.dubbo.rpc.cluster.Cluster)ExtensionLoader.getExtensionLoader(org.apa
che.dubbo.rpc.cluster.Cluster.class).getExtension(extName);
return extension.join(arg0);
}
}
```



###  cluster.join

所以再回到 doRefer 方法, 下面这段代码, 实际上调用 `MockClusterWrapper(FailOverCluster.join)`. 

> Invoker invoker = cluster.join(directory);

所以这里返回的invoker, 应该是 `MockClusterWrapper(FailOverCluster（directory）)`. 

接着回到`ReferenceConfig.createProxy` 方法中的最后一行. 

###  proxyFactory.getProxy

拿到invoker 之后, 会调用获取一个动态代理类. 

> return (T) proxyFactory.getProxy(invoker); 

这里的proxyFactory 又是一个自适应扩展点, 所以会进入下面的方法. 



###  JavassistProxyFactory.getProxy

通过这个方法 生成了一个动态代理类, 并且对invoker 做了一层处理, `InvokerInvocationHandler` 意味着后续发起服务调用的时候, 会由 InvokerInvocationHandler 来进行处理. 

```java
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```

###  proxy.getProxy

在 proxy.getProxy 这个方法中会生成一个动态代理类, 通过debug 的形式可以看到动态代理类的原貌, 在getProxy这个方法位置增加一个断点. 

> proxy = (Proxy) pc.newInstance();

然后在debug 窗口, 找到ccp 这个变量 -> mMethods。

```java
public java.lang.String info(java.lang.String arg0){
Object[] args = new Object[1];
args[0] = ($w)$1;
Object ret = handler.invoke(this, methods[0], args);
return (java.lang.String)ret;
}

```



从这个info方法可以看到, 我们通过. 

@Reference 注入的一个对象实例本质上就是一个动态代理类, 通过调用这个类中的方法, 会触发  `handler.invoke()`, 而这个handler 就是InvokerInvocationHandler

##  网络连接的建立

前面分析的逻辑中, 只讲到了动态代理类的生成, 那么目标服务地址信息以及网络通信的建立在哪里实现呢? 我们继续回到 `RegistryProtocol.refer` 这个方法中. 

这里我们暂且关注 `directory.subscribe` 这个方法, 它是实现服务目标订阅的. 

```java
 private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        // RegistryDirectory 初始化
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        // 注册consumer://协议的url
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
            directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
            registry.register(directory.getRegisteredConsumerUrl());
        }
        directory.buildRouterChain(subscribeUrl);
        // 订阅事件监听
        directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));

        // 构建 invoker
        Invoker invoker = cluster.join(directory);
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```



###  RegistryDirectory.subscribe

订阅注册中心指定节点的变化, 如果发生变化, 则通知RegistryDirectory. Directory其实和服务的注册以及服务的发现有非常大的关联. 


```java
  public void subscribe(URL url) {
        //  设置 consumerUrl
        setConsumerUrl(url);
        // 把当前RegistryDirectory作为listener，去监听zk上节点的变化
        CONSUMER_CONFIGURATION_LISTENER.addNotifyListener(this);
        serviceConfigurationListener = new ReferenceConfigurationListener(this, url);
        // 订阅, 这里的registry 是zookeeperRegistry
        registry.subscribe(url, this);
    }
```

这里的registry 是ZookeeperRegistry, 会监听并获取路径下的节点, 监听的路径是: 

/dubbo/org.apache.dubbo.demo.DemoService/providers 、/dubbo/org.apache.dubbo.demo.DemoService/configurators、/dubbo/org.apache.dubbo.de mo.DemoService/routers 节点下面的子节点变动

###  FailbackRegistry.subscribe

listener为RegistryDirectory , 后续要用到, 移除失效的listener, 调用doSubscribe 进行订阅. 

```java
@Override
    public void subscribe(URL url, NotifyListener listener) {
        super.subscribe(url, listener);
        removeFailedSubscribed(url, listener);
        try {
            // Sending a subscription request to the server side
            doSubscribe(url, listener);
        } catch (Exception e) {
            Throwable t = e;

            List<URL> urls = getCacheUrls(url);
            if (CollectionUtils.isNotEmpty(urls)) {
                notify(url, listener, urls);
                logger.error("Failed to subscribe " + url + ", Using cached list: " + urls + " from cache file: " + getUrl().getParameter(FILE_KEY, System.getProperty("user.home") + "/dubbo-registry-" + url.getHost() + ".cache") + ", cause: " + t.getMessage(), t);
            } else {
                // If the startup detection is opened, the Exception is thrown directly.
                boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                        && url.getParameter(Constants.CHECK_KEY, true);
                boolean skipFailback = t instanceof SkipFailbackWrapperException;
                if (check || skipFailback) {
                    if (skipFailback) {
                        t = t.getCause();
                    }
                    throw new IllegalStateException("Failed to subscribe " + url + ", cause: " + t.getMessage(), t);
                } else {
                    logger.error("Failed to subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
                }
            }

            // Record a failed registration request to a failed list, retry regularly
            addFailedSubscribed(url, listener);
        }
    }
```



###  ZookeeperRegistry.doSubscribe

这个方法是订阅, 逻辑实现比较多, 可以分为两段来看, 这里的实现把所有的service层发起的订阅以及指定的service层发起的订阅分开处理, 所有service 层类似于监控中心发起的订阅. 指定的service层 发起的订阅可以看做是服务消费者的订阅. 我们只需要关心指定的service层发起的订阅即可. 

```java
  @Override
    public void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            if (ANY_VALUE.equals(url.getServiceInterface())) {
                String root = toRootPath();
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                if (listeners == null) {
                    zkListeners.putIfAbsent(url, new ConcurrentHashMap<>());
                    listeners = zkListeners.get(url);
                }
                ChildListener zkListener = listeners.get(listener);
                if (zkListener == null) {
                    listeners.putIfAbsent(listener, (parentPath, currentChilds) -> {
                        for (String child : currentChilds) {
                            child = URL.decode(child);
                            if (!anyServices.contains(child)) {
                                anyServices.add(child);
                                subscribe(url.setPath(child).addParameters(INTERFACE_KEY, child,
                                        Constants.CHECK_KEY, String.valueOf(false)), listener);
                            }
                        }
                    });
                    zkListener = listeners.get(listener);
                }
                zkClient.create(root, false);
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (CollectionUtils.isNotEmpty(services)) {
                    for (String service : services) {
                        service = URL.decode(service);
                        anyServices.add(service);
                        subscribe(url.setPath(service).addParameters(INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                List<URL> urls = new ArrayList<>();
                for (String path : toCategoriesPath(url)) {
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    // 如果该路径没有添加到 listener, 则创建一个map 来放置listener
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
                        // 如果没有添加过对于子节点的listener, 则创建, 通知服务变化, 回调NotifyListener
                        listeners.putIfAbsent(listener, (parentPath, currentChilds) -> ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds)));
                        zkListener = listeners.get(listener);
                    }
                    zkClient.create(path, false);
                    // 添加path 节点的当前节点以及子节点监听, 并且获取子节点信息
                    //  也就是 dubbo://ip:port/
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                //  调用notify 进行通知, 对已经可用的列表进行通知. 
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

```



###  FailbackRegistry.notify

调用 FailbackRegistry.notify, 对参数进行判断, 然后调用 `AbstractRegistry.notify` 方法。 

```java
   @Override
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        if (url == null) {
            throw new IllegalArgumentException("notify url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("notify listener == null");
        }
        try {
            doNotify(url, listener, urls);
        } catch (Exception t) {
            // Record a failed registration request to a failed list, retry regularly
            addFailedNotified(url, listener, urls);
            logger.error("Failed to notify for subscribe " + url + ", waiting for retry, cause: " + t.getMessage(), t);
        }
    }

```



###  AbstractRegistry.notify

这里面会针对每一个 category, 调用 `listener.notify` 进行通知, 然后更新本地的缓存文件. 

```java
protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        if (url == null) {
            throw new IllegalArgumentException("notify url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("notify listener == null");
        }
        if ((CollectionUtils.isEmpty(urls))
                && !ANY_VALUE.equals(url.getServiceInterface())) {
            logger.warn("Ignore empty notify urls for subscribe url " + url);
            return;
        }
        if (logger.isInfoEnabled()) {
            logger.info("Notify urls for subscribe url " + url + ", urls: " + urls);
        }
        // keep every provider's category.
        Map<String, List<URL>> result = new HashMap<>();
        for (URL u : urls) {
            if (UrlUtils.isMatch(url, u)) {
                String category = u.getParameter(CATEGORY_KEY, DEFAULT_CATEGORY);
                List<URL> categoryList = result.computeIfAbsent(category, k -> new ArrayList<>());
                categoryList.add(u);
            }
        }
        if (result.size() == 0) {
            return;
        }
        Map<String, List<URL>> categoryNotified = notified.computeIfAbsent(url, u -> new ConcurrentHashMap<>());
        for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
            String category = entry.getKey();
            List<URL> categoryList = entry.getValue();
            categoryNotified.put(category, categoryList);
            listener.notify(categoryList);
            // We will update our cache file after each notification.
            // When our Registry has a subscribe failure due to network jitter, we can return at least the existing cache URL.
            saveProperties(url);
        }
    }
```



消费端的listener 是最开始传递过来的RegistryDirectory,  所以这里会触发 的RegistryDirectory.notify().



###  RegistryDirectory.notify

invoke 的网络连接以及后续的配置变更, 都会调用 notify 方法. 

urls:  zk的path 数据, 这里表示的是  dubbo://

```java
 @Override
    public synchronized void notify(List<URL> urls) {
        // 对url 列表进行校验、过滤、然后分成 config、router、provider 3个分组map
        Map<String, List<URL>> categoryUrls = urls.stream()
                .filter(Objects::nonNull)
                .filter(this::isValidCategory)
                .filter(this::isNotCompatibleFor26x)
                .collect(Collectors.groupingBy(url -> {
                    if (UrlUtils.isConfigurator(url)) {
                        return CONFIGURATORS_CATEGORY;
                    } else if (UrlUtils.isRoute(url)) {
                        return ROUTERS_CATEGORY;
                    } else if (UrlUtils.isProvider(url)) {
                        return PROVIDERS_CATEGORY;
                    }
                    return "";
                }));

        List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
        this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

        // 如果route 路由节点变化, 则重新将route 下的数据生成route
        List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
        toRouters(routerURLs).ifPresent(this::addRouters);

        // providers
        // 获取 provider URL, 然后调用refreshOverrideAndInvoker 进行刷新
        List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
        refreshOverrideAndInvoker(providerURLs);
    }
```



###  refreshOverrideAndInvoker

- 逐个调用注册中心里面的配置, 覆盖原来的url, 组成最新的url 放入到overrideDirectoryUrl 存储. 
- 根据 provider  urls , 重新刷新  invoker

```java
   private void refreshOverrideAndInvoker(List<URL> urls) {
        // mock zookeeper://xxx?mock=return null
        overrideDirectoryUrl();
        refreshInvoker(urls);
    }

```



###  refreshInvoker

```java
 private void refreshInvoker(List<URL> invokerUrls) {
        Assert.notNull(invokerUrls, "invokerUrls should not be null");

        if (invokerUrls.size() == 1
                && invokerUrls.get(0) != null
                && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            // 如果是空协议, 则直接返回 不允许访问.
            this.forbidden = true; // Forbid to access
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
            destroyAllInvokers(); // Close all invokers
        } else {
            this.forbidden = false; // Allow to access
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            if (invokerUrls == Collections.<URL>emptyList()) {
                invokerUrls = new ArrayList<>();
            }
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<>();
                this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
            }
            // 如果url为空, 则直接返回.
            if (invokerUrls.isEmpty()) {
                return;
            }
            // 根据 invokerUrls 生成新的 invoker
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

            /**
             * If the calculation is wrong, it is not processed.
             *
             * 1. The protocol configured by the client is inconsistent with the protocol of the server.
             *    eg: consumer protocol = dubbo, provider only has other protocol services(rest).
             * 2. The registration center is not robust and pushes illegal specification data.
             *
             */
            if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                        .toString()));
                return;
            }

            // 转换为list
            List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
            // pre-route and build cache, notice that route cache should build on original Invoker list.
            // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
            routerChain.setInvokers(newInvokers);
            //  如果服务配置了分组,则把分组下的provider 包装成StaticDirectory, 组成一个invoker.
            // 实际上就是 按照group 进行合并.
            this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
            this.urlInvokerMap = newUrlInvokerMap;

            try {
                //  旧的url是否在新map 里面存在,不存在, 就销毁url 对应的invoker
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
```





###  toInvokers

这个方法中有比较长的判断和处理逻辑, 我们只需要关心invoker 是什么时候初始化的就行, 这里用到了 `protocol.refer` 来构建一个 invoker  . 

```java
invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url,
providerUrl);
```

构建完成之后, 会保存在`Map> urlInvokerMap`  这个集合中. 

```java
 private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();
        if (urls == null || urls.isEmpty()) {
            return newUrlInvokerMap;
        }
        Set<String> keys = new HashSet<>();
        String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
        for (URL providerUrl : urls) {
            // If protocol is configured at the reference side, only the matching protocol is selected
            if (queryProtocols != null && queryProtocols.length() > 0) {
                boolean accept = false;
                String[] acceptProtocols = queryProtocols.split(",");
                for (String acceptProtocol : acceptProtocols) {
                    if (providerUrl.getProtocol().equals(acceptProtocol)) {
                        accept = true;
                        break;
                    }
                }
                if (!accept) {
                    continue;
                }
            }
            if (EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
                continue;
            }
            if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
                logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() +
                        " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() +
                        " to consumer " + NetUtils.getLocalHost() + ", supported protocol: " +
                        ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
                continue;
            }
            URL url = mergeUrl(providerUrl);

            String key = url.toFullString(); // The parameter urls are sorted
            if (keys.contains(key)) { // Repeated url
                continue;
            }
            keys.add(key);
            // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
            Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
            Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
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
        }
        keys.clear();
        return newUrlInvokerMap;
    }

```



### protocol.refer

调用指定的协议来进行远程引用, protocol 是一个 `Protocol$Adaptive`,而真正的实现是: `ProtocolListenerWrapper(ProtocolFilterWrapper(QosProtocolWrapper(DubboProtocol.refer)`. 我们直接进入`DubboProtocol.refer`  方法. 

###  DubboProtocol.refer

- 优化序列化
- 构建DubboInvoker

在构建 DubboInvoker 时, 会构建一个ExchangeClient, 通过 `getClients(url)`  方法, 这里基本可以猜到是的服务的通信建立. 

```java
  @Override
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);

        // create rpc invoker.
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);

        return invoker;
    }
```



###  getClients

这里面是获取客户端连接的方法. 

- 判断是否为共享连接, 默认是共享一个连接进行通信. 
- 是否配置了多个连接通道  conections, 默认只有一个. 

```java
 private ExchangeClient[] getClients(URL url) {
        // whether to share connection

        boolean useShareConnect = false;

        int connections = url.getParameter(CONNECTIONS_KEY, 0);
        List<ReferenceCountExchangeClient> shareClients = null;
        // if not configured, connection is shared, otherwise, one connection for one service
        //   如果配置连接数, 则默认为共享连接. 
        if (connections == 0) {
            useShareConnect = true;

            /**
             * The xml configuration should have a higher priority than properties.
             */
            String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
            connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                    DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
            shareClients = getSharedClient(url, connections);
        }

        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (useShareConnect) {
                clients[i] = shareClients.get(i);

            } else {
                clients[i] = initClient(url);
            }
        }

        return clients;
    }
```



### getSharedClient

获取一个共享连接. 

```java
  private List<ReferenceCountExchangeClient> getSharedClient(URL url, int connectNum) {
        String key = url.getAddress();
        List<ReferenceCountExchangeClient> clients = referenceClientMap.get(key);

        //  检查当前的key ,检查连接是否已经创建过并且可用, 如果是, 则直接返回并且增加连接的个数的统计.
        if (checkClientCanUse(clients)) {
            batchClientRefIncr(clients);
            return clients;
        }

        // 如果连接已经关闭或者连接没有被创建.
        locks.putIfAbsent(key, new Object());
        synchronized (locks.get(key)) {
            clients = referenceClientMap.get(key);
            // dubbo check
            // 在创建l连接之前, 在做一次检查, 防止连接并发创建.
            if (checkClientCanUse(clients)) {
                batchClientRefIncr(clients);
                return clients;
            }

            //  连接数必须大于等于1
            // connectNum must be greater than or equal to 1
            connectNum = Math.max(connectNum, 1);

            // If the clients is empty, then the first initialization is
            // 如果当前消费者还没有和服务端产生连接, 则初始化.
            if (CollectionUtils.isEmpty(clients)) {
                clients = buildReferenceCountExchangeClientList(url, connectNum);
                // 创建clients 之后, 保存到map中.
                referenceClientMap.put(key, clients);

            } else {
                // 如果clients 不为空, 则从clients 数组中进行遍历.
                for (int i = 0; i < clients.size(); i++) {
                    ReferenceCountExchangeClient referenceCountExchangeClient = clients.get(i);
                    // If there is a client in the list that is no longer available, create a new one to replace him.
                    // 如果集合中存在一个连接但是这个连接处于closed 状态, 则重新构建一个进行替换
                    if (referenceCountExchangeClient == null || referenceCountExchangeClient.isClosed()) {
                        clients.set(i, buildReferenceCountExchangeClient(url));
                        continue;
                    }

                    // 增加个数 
                    referenceCountExchangeClient.incrementAndGetCount();
                }
            }

            /**
             * I understand that the purpose of the remove operation here is to avoid the expired url key
             * always occupying this memory space.
             */
            locks.remove(key);

            return clients;
        }
    }

```



### buildReferenceCountExchangeClientList

根据连接配置, 来构建指定个数的连接, 默认为1. 

```java
    private List<ReferenceCountExchangeClient> buildReferenceCountExchangeClientList(URL url, int connectNum) {
        List<ReferenceCountExchangeClient> clients = new ArrayList<>();

        for (int i = 0; i < connectNum; i++) {
            clients.add(buildReferenceCountExchangeClient(url));
        }

        return clients;
    }

    /**
     * Build a single client
     *
     * @param url
     * @return
     */
    private ReferenceCountExchangeClient buildReferenceCountExchangeClient(URL url) {
        ExchangeClient exchangeClient = initClient(url);

        return new ReferenceCountExchangeClient(exchangeClient);
    }
```



### initClient

终于进入到初始化客户端连接的方法, 猜测应该是根据url中配置的参数进行远程通信的构建. 

```java
  private ExchangeClient initClient(URL url) {

        // client type setting.
        //  获得连接类型
        String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
        // 添加默认序列化方式
        url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
        // enable heartbeat by default
       // 设置心跳时间
        url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));

        // BIO is not allowed since it has severe performance issue.
        // 判断str是否存在于扩展点中, 如果不存在则直接报错.
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }

        ExchangeClient client;
        try {
            // connection should be lazy
            // 是否需要延迟创建连接, 注意, 这里的requestHandler 是一个适配器. 
            if (url.getParameter(LAZY_CONNECT_KEY, false)) {
                client = new LazyConnectExchangeClient(url, requestHandler);

            } else {
                client = Exchangers.connect(url, requestHandler);
            }

        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }

        return client;
    }
```



###  Exchangers.connect

创建客户端连接. 

```java
  public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).connect(url, handler);
    }
```





### HeaderExchange.connect

主要关注 transporters.connect

```java
  public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
```



### NettyTransport.connect

使用netty 构建一个客户端连接. 

```java
   @Override
    public Client connect(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyClient(url, listener);
    }
```

##  总结

我们讲到了 `RegistryProtocol.refer`  过程中有一个关键步骤, 即在监听到服务提供者url时触发`RegistryDirectory.notify()` 方法 

`RegistryDirectory.notify()`  方法调用 `refreshInvoker()`  方法将服务提供者urls 转换为对应的远程  invoker, 最终会调用到 `DubboProtocol.refer()`  方法对应的DubboInvoker. 

DubboInvoker的构造方法中有一项入参 `ExchangeClient[] clients` , 即对应本文中要讲的网络客户端client, DubboInvoker 就是通过调用 `client.request()`  方法完成网络通信的请求发送和响应接收功能. 

client 的具体生成过程就是通过DubboProtocol 的 `initClient(URL url) ` 方法 创建了一个HeaderExchangeClient. 