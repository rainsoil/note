#  Dubbo 服务发布源码分析

## Dubbo 对于Spring 的扩展

最早我们使用spring 的配置来实现dubbo 服务的发布, 方便大家的同时也意味着Dubbo 里面和Spring 肯定有那种说不清的关系. 

###  Spring 的标签扩展

在Spring中定义了两个接口, 

-  NamespaceHandler : 注册一堆 BeanDefinitionParser，利用他们来进行解析 
-  BeanDefinitionParser : 用于解析每个element 的内容

Spring 默认会加载jar 包下的`META-INF/spring.handlers` 文件寻找对应的 NamespaceHandler。  Dubbo-config 模块下的dubbo-config-spring

![](http://files.luyanan.com//img/20191127134642.png)

###  Dubbo的接入实现

Dubbo 中Spring 的扩展就是使用Spring的自定义类型, 所以同样有 NamespaceHandler、BeanDefinitionParser . 而 NamespaceHandler  是 DubboNamespaceHandler . 

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class, true));
        registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("metrics", new DubboBeanDefinitionParser(MetricsConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```

 BeanDefinitionParser  全部都使用了  DubboBeanDefinitionParser , 如果我们想看 `dubbo:service`的配置, 就直接看  `DubboBeanDefinitionParser(ServiceBean.class, true)`

这里面主要做了一件事情, 把不同的配置分别转换为Spring 容器中的Bean 对象. 

-  application  对应 ApplicationConfig 
-  registry 对应 RegistryConfig 
-  monitor 对应 MonitorConfig 
-  provider 对应 ProviderConfig 
-  consumer 对应 ConsumerConfig 

我们仔细看, 发现涉及到服务发布和服务调用的解析, 使用的是  ServiceBean 和 referenceBean . 并不是config 结尾的, 这两个类稍微特殊些, 当然它同时也继承了  ServiceConfig 和 ReferenceConfig . 

```java
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
```

### DubboBeanDefinitionParser

这里面是实现具体配置文件解析的入口, 它重写了parse方法, 对Spring 的配置进行解析. 我们关注一下ServiceBean的解析, 实际就是解析`dubbo:service` 这个标签中对应的属性. 

```java
if (ServiceBean.class.equals(beanClass)) {
            String className = element.getAttribute("class");
            if (className != null && className.length() > 0) {
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        }
```



### ServiceBean 的实现

ServiceBean 这个类, 分别实现了InitializingBean, DisposableBean,        ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,        ApplicationEventPublisherAware.

#### InitializingBean

接口为bean 提供了初始化方法的方式, 它只包括 ` afterPropertiesSet `方法, 凡是继承该接口的类, 在初始化bean 的时候会执行该方法, 被重写的方法为  ` afterPropertiesSet `

####  DisposableBean 

被重写的方法为 destroy ， bean 被销毁的时候, sring容器会自动执行 ` destroy ` 方法, 比如释放资源. 

####   ApplicationContextAware 

实现了这个接口的Bean，当Spring 的容器初始化的时候, 会自动将  ApplicationContext 注入进来. 

####   ApplicationListener 

 ApplicationEvent  事件监听, Spring 容器启动会会发一个事件通知,  被重写的方法为 ` onApplicationEvent `, ` onApplicationEvent ` 方法传入的对象是 ContextRefreshedEvent , 这个对象是Spring 的上下文被刷新或者加载完毕后触发的. 因此服务就是在Spring 的上下文刷新后进行导出操作的. 

####   BeanNameAware 

获取自身初始化时, 本身的bean 的id属性,被重写的方法为 ` setBeanName `

####  ApplicationEventPublisherAware

这是一个异步事件发送器, 被重写的方法为 ` setApplicationEventPublisher `, 简单来说, 在spring中提供了类似于消息队列的异步事件解耦功能. (典型的观察者模式. )



####  Spring 事件发送监听由3个部分组成. 

1.  .ApplicationEvent :表示事件本身, 自定义事件需要继承该类. 
2.  ApplicationEventPublisherAware :  事件发送器, 需要实现该接口。 
3.  ApplicationListener : 事件监听器接口. 

![](http://files.luyanan.com//img/20191127142152.png)

### ServiceBean 中服务暴露服务. 

在ServiceBean中,我们暂且只需要关注 两个方法, 分别是

- 在初始化bean的时候会执行该方法  afterPropertiesSet 
- spring 容器启动后悔发送一个事件通知  onApplicationEvent 

#### afterPropertiesSet 

我们发现这个方法里面, 就是把dubbo中配置的 application、registry、service、protocol  等信息加载到对应的config 实体类中, 便于后续的使用. 

#### onApplicationEvent 

spring 容器启动后, 会受到这样一个事件通知, 这里面做了两件事情. 

- 判断服务是否已经发布过. 
- 如果没有发布, 则调用export 接口进行服务发布的流程(这里就是入口)

```java
 @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        if (!isExported() && !isUnexported()) {
            if (logger.isInfoEnabled()) {
                logger.info("The service ready on spring started. service: " + getInterface());
            }
            // 入口
            export();
        }
    }
```



####   export

 serviceBean  中重写了export.方法. 实现了一个事件的发布, 并且调用了`super.export()` , 也就是调用父类的 `export`  方法. 

```java
 @Override
    public void export() {
        super.export();
        // Publish ServiceBeanExportedEvent
        publishExportEvent();
    }
```



###  ServiceConfig  配置类

先整体来看一下这个类的作用,从名字来看, 它应该和其他所有的config 类一样去实现对配置文件中的service 的配置信息的存储. 实际上, 这个类并不单纯, 所有的配置它都放在一个 AbstractServiceConfig  的抽象类,自己实现了对于服务发布之前要做的操作逻辑.

```java
  public synchronized void export() {
        // 检查并且更新配置信息
        checkAndUpdateSubConfigs();

        // 如果当前的服务是否需要发布, 通过配置实现 :@Service(export = false)
        if (!shouldExport()) {
            return;
        }

        // 检查是否需要延时发布, 通过配置 @Service(delay=1000)实现, 单位是毫秒
        if (shouldDelay()) {
            delayExportExecutor.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
            // 如果没有配置delay , 则直接调用doExport 进行发布
            doExport();
        }
    }
```

####   doExport 

这里仍然是在实现发布前的各种判断, 比如刷新

```java
 protected synchronized void doExport() {
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        // 服务是否已经发布过
        if (exported) {
            return;
        }
        // 设置发布的状态
        exported = true;

        // path 表示服务路径,默认使用interfaceName
        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
        doExportUrls();
    }
```

#### doExportUrls

1. 记载所有配置的注册中心地址
2. 遍历所有配置的协议. protocol
3. 针对每种协议发布一个对应协议的服务. 

```java
  private void doExportUrls() {
        // 加载所有配置的注册中心地址, 组装成一个url
        // //(registry://ip:port/org.apache.dubbo.registry.RegistryService 的东西)
        List<URL> registryURLs = loadRegistries(true);
        for (ProtocolConfig protocolConfig : protocols) {
            // group和version 组成一个pathkey(serviceName)
            String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
            //ApplicationModel 用来存储 ProviderModel, 发布的服务的元数据, 后续会用到
            ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);
            ApplicationModel.initProviderModel(pathKey, providerModel);
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```



#### doExportUrlsFor1Protocol

发布指定协议的服务, 我们以Dubbo 为例, 由于代码太多, 就不全部贴出来 . 

1. 前面的一大串if else 代码, 是为了把当前服务下配置的<dubbo:method> 参数进行解析, 保存到map 集合中. 
2. 获取当前服务需要暴露的ip和端口号
3. 把解析到的所有数据, 组装成一个URL,大概是  `dubbo://192.168.9.110:20880/com.example.ISayHelloService`

```java
 private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
     ......
           // 以上都是解析<dubbo:method>和<dubbo:service> 中配置参数的代码,
        // export service
        // 获取当前服务发布的目标id和port
        String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = this.findConfigedPorts(protocolConfig, name, map);
        // 组装URL
        URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

        // 这里是通过ConfiguratorFactory 去实现动态改变配置的功能,
        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        // 如果scope ！=null, 则发布服务, 默认 scope 为null, 如果scope 不为none, 判断是否为local或者remote, 从而发布
        // local 或者remote服务, 默认两个都会发布 .
        String scope = url.getParameter(SCOPE_KEY);
        // don't export when none is configured
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

            // export to local if the config is not remote (export to remote only when config is remote)
            // injvm 发布到本地
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // export to remote if the config is not local (export to local only when config is local)
            // 发布远程服务
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                if (!isOnlyInJvm() && logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                if (CollectionUtils.isNotEmpty(registryURLs)) {
                    for (URL registryURL : registryURLs) {
                        //if protocol is only injvm ,not register
                        if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                            continue;
                        }
                        url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                        }

                        // For providers, this is used to enable custom proxy to generate invoker
                        String proxy = url.getParameter(PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                        }

                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
                MetadataReportService metadataReportService = null;
                if ((metadataReportService = getMetadataReportService()) != null) {
                    metadataReportService.publishProvider(url);
                }
            }
        }
        this.urls.add(url);
    }
```

> ####  Local 
>
> 服务只是injvm 的服务, 提供一种消费者和提供者都在一个jvm 内的调用方式. 使用injvm 协议, 只是一个伪协议, 它不开启端口,不发起远程调用, 只是JVM内部直接关联(通过集合的方法保存了发布的服务信息),但执行Dubbo的Filter 链. 简单来说, 就是你本地的dubbo服务调用, 都依托于dubbo 的标准来进行, 这样可以享受到dubbo 的一些配置服务. 
>
> ####  Remote
>
> 表示根据配置的注册中心进行远程发布, 遍历多个注册中心, 进行协议的发布. 
>
> 1. incoker 是一个代理类, 它是dubbo 的核心模型, 其他模型都是向它靠拢, 或转换成它. 它代表一个可执行体, 可向它发起invoke 调用, 他有可能是一个本地的实现, 也有可能是一个远程的实现, 也有可能是一个集群实现. 
> 2. ` DelegateProviderMetaDataInvoker`, 因为2.7 引入了元数据, 所以这里对invoker 进行了委托, 把invoker 交给了 DelegateProviderMetaDataInvoker  来处理 
> 3. 调用 protocol.export(invoker) 来发布这个代理. 
> 4. 添加到  exporters  集合中. 



#### protocol.export

` protocol.export` 这个protocol 是什么呢? 找到定义出发现它是一个自适应扩展点, 打开Protocol 这个扩展点, 又可以看到他是一个在方法层面上的自适应扩展， 意味着它实现了对于export 这个方法的适配, 也就意味着Protocol 是一个动态代理类,  `Protocol$Adaptive`

```java
 private static final Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

这个动态代理类, 会根据url 中配置的 protocol name 来实现对应协议的适配. 

```java
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {
        throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy()of interface org.apache.dubbo.rpc.Protocol is not adaptive method !");
    }

    public int getDefaultPort() {
        throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort()of interface org.apache.dubbo.rpc.Protocol is not adaptive method !");
    }

    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.
            RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
                org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys ([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }

    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException

    {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = (url.getProtocol() == null ? "dubbo" : url.getProtocol());
        if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys ([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol) ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```



那么在当前的场景中, protocol 会调用谁呢?  目前发布的invoker(URL), 实际上是一个 registry://协议, 所以 Protocol$Adaptive，会通过 getExtension(extName)得到一个 RegistryProtocol 



####   RegistryProtocol.export

很明显, 这个RegistryProtocol 是用来实现服务注册的, 则里面会有很多处理逻辑. 

1. 实现对应协议的服务发布
2. 实现服务注册
3. 订阅服务重写

```java
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



####   doLocalExport

先通过doLocalExport 来暴露一个服务, 本质上应该是启动一个通信服务, 主要的步骤是将本地ip和20880 端口打开, 进行监听。 

 originInvoker : 应该是 registry://ip:port/com.alibaba.dubbo.registry.RegistryService  

key:  从 originInvoker 中获得发布协议的 url: dubbo://ip:port/... 

 bounds: 一个 prviderUrl 服务 export 之后，缓存到 bounds 中，所以一个 providerUrl 只会对应一个 exporter  

```java
   private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);

        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
            // 对原有的 invoker 委托给了invokerDelegate
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
            // 将invoke 转换成了exporter 并且启动netty 服务
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
        });
    }
```

 InvokerDelegete : 是  RegistryProtocol 的一个静态内部类, 该类是一个 originInvoker  的委托类, 该类存储了 originInvoker ,其父类  InvokerWrapper  还会存储 providerUrl，InvokerWrapper 会调用  originInvoker  的invoke方法, 也会销毁invoke. 可以管理invoke的生命周期. 



####   DubboProtocol.export

基于动态代理的适配, 很自然的就过渡到了 DubboProtocol 这个协议类, 但是实际上是 DubboProtocol 吗? 

这里并不是获取一个单纯的 DubboProtocol  扩展点, 而是会通过Wrapper 对Protocol 进行装饰, 装饰器分别为  QosProtocolWrapper/ProtocolListenerWrapper/ProtocolFilterWrapper/DubboProtocol . 

为什么是这样呢? 我们再来看看SPI的代码

#####  Wrapper 包装

在 ` ExtensionLoader.loadClass `   这个方法中, 有一段这样的判断, 如果当前这个类是一个Wrapper 包装类, 也及时这个wrapper 中有构造方法, 参数是当前被加载的扩展点的类型, 则把这个wrapper 类加入到  cacheWrapperClass  缓存中., 

```java
 } else if (isWrapperClass(clazz)) {
            cacheWrapperClass(clazz);
        } 

    private void cacheWrapperClass(Class<?> clazz) {
        if (cachedWrapperClasses == null) {
            cachedWrapperClasses = new ConcurrentHashSet<>();
        }
        cachedWrapperClasses.add(clazz);
    }

```

我们可以在dubbo 的配置文件中找到三个wrapper

1.  QosprotocolWrapper，  如果当前配置了注册中心, 则会启动一个QosServer, qos 是dubbo 的在线运维命令, dubbo2.5.8 版本重构了telnet模块, 提供了新的telnet 命令支持, 新版本的telnet 端口与dubbo 协议的端口是不同的端口, 默认的为22222
2.  ProtocolFilterWrapper  对invoker 进行filter 包装, 实现请求的过滤. 
3.  ProtocolListenerWrapper 用于服务export 时候插入监听机制. 

>  qos=org.apache.dubbo.qos.protocol.QosProtocolWrapper filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper 

接着, 在 ` getExtension->createExtension ` 方法中, 会对  cacheWrapperClass  集合进行判断, 如果集合不为空, 则进行包装

```java
 Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
```



####   ProtocolFilterWrapper 

这是一个过滤器的包装, 使用责任链模式, 对invoker 进行包装. 

```java
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (REGISTRY_PROTOCOL.equals(invoker.getUrl().getProtocol())) {
            return protocol.export(invoker);
        }
        return protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
    }


// 构建责任链, 基于激活扩展点
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
....
    }
```



我们看如下文件:

>  /dubbo-rpc-api/src/main/resources/META-INF/dubbo/internal/com.alibaba.dubbo.rpc.Filter  

默认提供了非常多的过滤器, 然后基于条件激活扩展点, 来对invoker 进行包装, 从而实现在远程调用的到时候, 会经过这些filter 进行过滤. 

####  DubboProtocol.export

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



####  openServer

去开启一个服务, 并且放入缓存中-> 在同一台机器上(单网卡上), 同一个端口上仅允许启动一个服务器实例

```java
  private void openServer(URL url) {
        // find server.
        // 获取 host:port,并将其作为服务器实例的key, 用于标识当前的服务器实例.
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        // client 也可以暴露一个只有server 可以调用的服务
        boolean isServer = url.getParameter(IS_SERVER_KEY, true);
        if (isServer) {
            // 是否在serverMap 中缓存了
            ExchangeServer server = serverMap.get(key);
            if (server == null) {
                synchronized (this) {
                    server = serverMap.get(key);
                    if (server == null) {
                        serverMap.put(key, createServer(url));
                    }
                }
            } else {
                // server supports reset, use together with override
                // 服务已经创建, 则根据url 中配置 重置服务器. 
                server.reset(url);
            }
        }
    }
```



####  createServer

创建服务, 开启心跳检测,默认使用netty, 组装url . 

```java
 private ExchangeServer createServer(URL url) {
        // 组装url, 在url 中添加心跳时间,编解码参数
        url = URLBuilder.from(url)
                // send readonly event when server closes, it's enabled by default
                // 当服务关闭后, 发送一个只读的事件, 默认是开启状态.
                .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
                // 启动心跳配置
                // enable heartbeat by default
                .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
                .addParameter(CODEC_KEY, DubboCodec.NAME)
                .build();
        String str = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);

        // 通过SPI检测是否存在server 参数所代表的Transporter 扩展, 不存在则抛出异常
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
        }

        // 创建ExchangeServer
        ExchangeServer server;
        try {
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }

        str = url.getParameter(CLIENT_KEY);
        if (str != null && str.length() > 0) {
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }

        return server;
    }
```



####  Exchangers.bind

```java
 public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        // 获取exchange, 默认为HeaderExchanger。
        // 调用 HeaderExchanger 的bind方法 创建ExchangeServer 实例. 
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).bind(url, handler);
    }
```

####  HeaderExchanger.bind

这里面包含多个逻辑:

1.   new DecodeHandler(new HeaderExchangeHandler(handler)) 
2.  Transporters.bind 
3.  new HeaderExchangeServer  

目前我们只需要关心  transporters.bind  方法即可: 

```java
 @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
```



####   Transporters.bind 

```java

    public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handlers == null || handlers.length == 0) {
            throw new IllegalArgumentException("handlers == null");
        }
        ChannelHandler handler;

        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            // 如果handler 的元素数量大于1, 则创建ChannelHandlerDispatcher 分发器
            handler = new ChannelHandlerDispatcher(handlers);
        }
        // 获取自适应Transporter 实例, 并调用实例方法
        return getTransporter().bind(url, handler);
    }
```

#### getTransporter

getTransporter 是一个自适应扩展点, 它针对bind方法添加了自适应注解, 意味着 bind方法的具体实现, 会基于` Transporter$Adaptive ` 方法进行适配, 那么在这里面默认的通讯协议是netty, 所以它会采用netty4 的实现, 也就是 ` org.apache.dubbo.remoting.transport.netty4.NettyTransporter `



```java
 public static Transporter getTransporter() {
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }

```

####   NettyTransporter.bind 

创建一个  nettyserver 

```java
   @Override
    public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }
```

####   nettyserver 

初始化一个 nettyserver , 并且从url中获取相应的ip/port, 然后调用 `doOpen()`

```java
  public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }

 public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, handler);
        localAddress = getUrl().toInetSocketAddress();

        // 获取ip和端口号
        String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
        int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
        if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
            bindIp = ANYHOST_VALUE;
        }
        bindAddress = new InetSocketAddress(bindIp, bindPort);
        this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);
        this.idleTimeout = url.getParameter(IDLE_TIMEOUT_KEY, DEFAULT_IDLE_TIMEOUT);
        try {
            // 调用模板方法 doOpen 启动服务器
            doOpen();
            if (logger.isInfoEnabled()) {
                logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
            }
        } catch (Throwable t) {
            throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
                    + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
        }
        //fixme replace this with better method
        DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
        executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
    }
```



####  doOpen()

开启Netty服务

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

然后大家要注意, 这里用到了一个handler 来处理客户端传过来的请求. 

 nettyServerHandler 

 NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this); 

这个handler 是一个链路, 它的正确组成应该是: 

```java
MultiMessageHandler(heartbeatHandler(AllChannelHandler(DecodeHandler(HeaderExchangeHeadler(dubboProtocol

```

后续接收到的请求, 会一层一层的处理, 比较繁琐. 

###  invoker  是什么?

从前面的分析来看, 服务的发布分为三个阶段.

1. 第一个阶段会创建一个invoker
2. 第二个阶段会把经过一系列处理的invoker(各种包装),在 DubboProtocol  中保存到 exporterMap  中.
3. 第三个节点把dubbo协议的url 地址注册到注册中心

invoker 是Dubbo 领域一个非常重要的概念, 和` ExtensionLoader ` 的重要性是一样的, 如果invoker 没有搞懂, 不算看懂了Dubbo的源码. 我们继续回到 ServiceConfig  的 export 的代码, 这段代码是还没有分析过的, 以这个作为入口来分析我们前面export出去的invoker 到底是什么东西? 

```java
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));

```

####   proxyFactory.getInvoker

这是一个代理工厂, 用来生成 invoker, 从它的定义来看, 它是一个自适应扩展点, 看到这样的扩展点,我们几乎可以不假思索的想到它会存在一个动态适配类. 

```java
    private static final ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

```



#### ProxyFactory

这个方法的简单解读为: 它是一个SPI 扩展点, 并且默认的扩展实现为 `javassit`, 这个接口中有三个方法, 并且都加了 ` @Adaptive ` 自适应扩展点, 所以如果调用 getInvoker 方法, 应该会返回一个 ` ProxyFactory$Adaptive  `

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



####  ProxyFactory$Adaptive 

这个自适应扩展点, 做了两件事情: 

1. 通过 ` ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(extName) `  获取一个指定名称的扩展点
2.  在 dubbo-rpc-api/resources/META-INF/com.alibaba.dubbo.rpc.ProxyFactory 中，定义了 javassis=JavassisProxyFactory 
3.  调用 JavassisProxyFactory 的 getInvoker 方法 

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



####   JavassistProxyFactory.getInvoker 

 javassist : 是一个动态类库, 用来实现动态代理. 

 proxy : 接口的实现,   com.example.SayHelloServiceImpl 

  type : 接口全称:  com.example.ISayHelloService 

url: 协议地址: registry://....

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



####   javassist 生成的动态代理代码 

通过断点的方式( Wrapper 258),在  ` Wrapper.getWrapper 中的 makeWrapper ` 会创建一个动态代码, 核心的方法 invokeMethod 代码如下: 

```java
  public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws java.lang.reflect.InvocationTargetException {
        com.ISayHelloService w;
        try {
            w = ((com.ISayHelloService) $1);
        } catch (Throwable e) {
            throw new IllegalArgumentException(e);
        }
        try {
            if ("sayHello".equals($2) && $3.length == 1) {
                return ($w) w.sayHello((java.lang.String) $4[0]);
            }
        } catch (Throwable e) {
            throw new java.lang.reflect.InvocationTargetException(e);
        }
        throw new org.apache.dubbo.common.bytecode.NoSuchMethodException("Not found method \"" + $2 + "\" in class com.ISayHelloService.");
    }
```

构建好代理类后, 返回一个 AbstractproxyInvoker ,并且实现了 doInvoke  方法, 这个方法似乎看到了dubbo 消费者调用过来的时候触发的影子,  因为 wrapper.invokeMethod 本质上就是触发上面动态代理类的方法 invokeMethod.  

```java
 return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
```

所以简单总结一下 invoke 本质上应该是一个代理, 经过层层包装最终进行了发布, 当消费者发起请求的时候, 会获得这个invoker 进行调用. 

最终发布触发的invoker, 也不是一个单纯的代理, 也是经过多层包装的. 

>  InvokerDelegate(DelegateProviderMetaDataInvoker(AbstractProxyInvoker())) 



##  服务注册流程

关于服务发布这一条线分析完成后,再来了解一下服务注册的过程, 希望大家还记得我们之所以走到这一步,是因为我们在 RegistryProtocol 这个类中, 看到了服务发布的流程. 



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

###  服务注册核心代码

从export 方法中抽离出来的部分代码, 就是服务注册的流程. 

```java
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

###   getRegistry 

1. 把url 转换为对应配置的注册中心的具体协议
2. 根据具体的协议,从 registryFactory  中获取指定的注册中心实现

那么这个 registryFactory  具体是怎么赋值的呢? 

```java
 private Registry getRegistry(final Invoker<?> originInvoker) {
        // 将url 转换为配置的具体协议, 比如 zookeeper://ip:port, 这样后续获取的注册中心就是基于zk的实现
        URL registryUrl = getRegistryUrl(originInvoker);
        return registryFactory.getRegistry(registryUrl);
    }
```

在 RegistryProtocol  中存在这样一段代码, 很明显这是通过依赖注入来实现扩展点的. 

```java
public void setRegistryFactory(RegistryFactory registryFactory) {
        this.registryFactory = registryFactory;
    }
```



根据扩展点的加载规则, 我们可以先看看  /META-INF/dubbo/internal 路径下找到 RegistryFactory 的配置文件, 这个factory 有多个扩展点的实现

```
dubbo=org.apache.dubbo.registry.dubbo.DubboRegistryFactory
multicast=org.apache.dubbo.registry.multicast.MulticastRegistryFactory
zookeeper=org.apache.dubbo.registry.zookeeper.ZookeeperRegistryFactory
redis=org.apache.dubbo.registry.redis.RedisRegistryFactory
consul=org.apache.dubbo.registry.consul.ConsulRegistryFactory
etcd3=org.apache.dubbo.registry.etcd.EtcdRegistryFactory
```



接着, 找到 RegistryFactory  的实现, 发现它里面有一个自适应的方法, 根据url 中protocol 传入的值进行适配. 

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

###   RegistryFactory$Adaptive  

由于前面的代码中, url 中的protocol 已经改成了zookeeper, 那么这个时候根据zookeeper 获取的spi 扩展点应该是` ZookeeperRegistryFactory `

```java
import org.apache.dubbo.common.extension.ExtensionLoader;
public class RegistryFactory$Adaptive implements org.apache.dubbo.registry.RegistryFactory {
 public org.apache.dubbo.registry.Registry getRegistry(org.apache.dubbo.common.URL arg0) {
 if (arg0 == null) throw new IllegalArgumentException("url == null");
 org.apache.dubbo.common.URL url = arg0;
 String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
 if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.registr
y.RegistryFactory) name from url (" + url.toString() + ") use keys([protocol])");
 org.apache.dubbo.registry.RegistryFactory extension = (org.apache.dubbo.registry.RegistryFactory)Exten
sionLoader.getExtensionLoader(org.apache.dubbo.registry.RegistryFactory.class).getExtension(extName);
 return extension.getRegistry(arg0);
 }
}

```



###  ZookeeperRegistryFactory 

这个方法中并没有 getRegistry  方法,而是在父类  ` AbstractRegistryFactory `

- 从缓存 REGISTRIES  中, 根据key 获取对应的Registry
- 如果不存在, 则创建 Registry

```jav
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



###   createRegistry

创建一个 zookeeperRegistry，把 url和zookeepertransporter  作为参数传入. 

zookeepertransporter   这个属性也是基于依赖注入来赋值的, 具体的流程就不在分析了, 这个的值应该是  CuratorZookeeperTransporter , 表示具体使用什么框架来和zk 产生连接. 

```java
  @Override
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
```





### ZookeeperRegistry

这个方法中使用了 CuratorZookeeperTransport  来实现zk 的连接. 

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





###  registry.register(registedProviderUrl);  

继续往下分析, 会调用 ` registry.register(registedProviderUrl);  ` 去将 dubbo:// 的协议注册到zookeeper 上. 

这个方法会调用  FailbackRegistry 类中的 register ,为什么呢? 因为  ZookeeperRegistry  这个类中并没有registry 这个方法, 但是它的父类 FailbackRegistry  中存在 registry方法, 而这个类又重写了  AbstractRegistry  类中的registry 方法, 所以我们可以直接定位到 FailbackRegistry  这个类中的registry  方法中. 



```java
  public void register(URL registryUrl, URL registeredProviderUrl) {
        Registry registry = registryFactory.getRegistry(registryUrl);
        registry.register(registeredProviderUrl);
    }
```



####  FailbackRegistry.register  

-  FailbackRegistry  从名字来看, 是一个失败重试机制. 
- 调用父类的 registry方法, 将当前url 添加到缓存集合中. 

调用 doRegister  这个方法, 这个方法很明显, 是一个抽象方法, 会由 ` ZookeeperRegistry ` 的子类实现. 

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



####   ZookeeperRegistry.doRegister  

最终调用curator 的客户端把服务地址注册到zk 上去. 

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

