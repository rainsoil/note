# Dubbo之API机制

## 简介

本文的源码是基于Dubbo 2.7.2 版本进行分析的.这里, 我们首先将Dubbo 里面用的比较多的SPI 机制做一个详细的分析

## Dubbo 中的SPI机制

###  关于Java SPI

了解Dubbo 里面的SPI机制之前, 我们先了解一下Java 提供了SPI(service provider interface)机制, SPI是JDK 内置的一种服务提供发现机制, 目前市面上有很多的框架都是用它来做服务的扩展发.简单来说, 它是一种动态替换发现的机制, 举个简单的例子, 我们想在运行时动态的给它添加实现, 你只需要添加一个实现, 然后把新的实现描述给JDK知道就行了,大家耳熟能详的如JDBC、日志框架都有用到. 

![](http://files.luyanan.com//img/20191123171634.png)

####  实现SPI需要遵循的标准

我们如何去实现一个标准的SPI发现机制呢? 其实很简单,只需要满足以下提交就行了. 

1.  需要在classpath 下创建一个目录, 该目录命名必须是:  `META-INF/service`
2. 在该目录下创阿基哪一个properties文件, 该文件需要满足以下几个条件
   1. 文件名必须是扩展接口的全路径名称
   2. 文件内部描述的是该扩展接口的所有实现类
   3. 文件的编码格式是UTF-8
3. 通过 java.util.ServiceLoader  的加载机制来发现. 

![](http://files.luyanan.com//img/20191123172353.png)

#### SPI的实际应用

SPI 在很多地方有应用, 可能大家都没有关注,最常用的就是JDBC 驱动,我们来看看是怎么应用的. 

JDK本身提供了数据访问的API，在java.sql 这个包里面, 

我们在链接数据库的时候, 一定需要用到java.sql.Driver 这个接口对吧, 然后我好奇的去看了下java.sql.Driver, 发现Driver 并没有实现, 而是提供了一套标准的api 接口.

因为我们在实际应用中用的比较多的是mysql, 所以在mysql的包里面看到了如下的目录结构. 

![](http://files.luyanan.com//img/20191123174548.png)

这个文件里面写的就是mysql的驱动实现, 是通过SPI 机制把java.sql.Driver 的启动做了集成, 这样就达到了各个数据库厂商自己去实现数据库连接, jdk 本身不关心怎么实现. 

#### SPI的缺点

1. JDK标准的SPI会一次性加载实例化扩展点的所有的实现, 什么意思呢? 就是如果你在META-INF/service 下的文件夹了N个实现类, 那么JDK启动的时候就会一次性的全部加载。如果有的扩展点实现初始化很耗时或者如果有的实现类并没有用到, 那么会很浪费资源. 
2. 如果扩展点加载失败, 会导致调用方报错, 而且这个错误很难定位到是这个原因. 

### Dubbo优化后的SPI机制

####  基于Dubbo SPI的实现自己的扩展

Dubbo 的SPI扩展机制, 有两个规则: 

1. 需要在resource目录下配置 META-INF/dubbo 或者 META-INF/dubbo/internal 或者 META-INF/services ,并基于SPI 接口去创建一个文件. 
2. 文件名称和接口文件必须保持一致, 文件内容和SPI有差异, 内容是Key对应Value。

Dubbo 针对的扩展点非常多,可以针对协议、拦截、集群、路由、负载均衡、序列化、容器..几乎里面用到的所有功能, 都可以实现自己的扩展.

![](http://files.luyanan.com//img/20191125093826.png)

比如, 我们可以针对协议做一个扩展

#####  扩展协议扩展点

1. 创建如下结构, 添加 `META-INF/dubbo`  目录, 添加文件, 文件名和Dubbo 提供的协议扩展点接口保持一致. 
      ![](http://files.luyanan.com//img/20191125094045.png)
      
2. 创建MyProtocol 协议类

       可以实现自己的协议, 我们为了模拟协议产生的作用, 修改了一个端口

      ```java
      package com.dubbo.spring.server;
      
      import org.apache.dubbo.common.URL;
      import org.apache.dubbo.rpc.Exporter;
      import org.apache.dubbo.rpc.Invoker;
      import org.apache.dubbo.rpc.Protocol;
      import org.apache.dubbo.rpc.RpcException;
      
      /**
       * @author luyanan
       * @since 2019/11/25
       * <p>自定义协议类</p>
       **/
      public class MyProtocol implements Protocol {
      
      
          @Override
          public int getDefaultPort() {
              return 8888;
          }
      
          @Override
          public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
              return null;
          }
      
          @Override
          public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
              return null;
          }
      
          @Override
          public void destroy() {
      
          }
      }
      
      ```
   
3. 在调用处执行如下代码

    ```java
   Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("myProtocol");
           System.out.println(protocol.getDefaultPort());
   ```

4. 输出结果, 可以看到运行结果, 是执行的自定义的协议扩张点. 

总的来说, 思路和SPI是差不多的, 都是基于约定的路径下制定配置文件, 目的是通过配置的方式轻松的实现功能的扩展. 我们的猜想是: 一定有一个地方通过读取制定路径下的所有文件进行load, 然后将对应的结果保存在一个map中, key 对应为名称, value 对应的为实现类. 那么这个实现, 一定就在ExtensionLoader 中, 接下来我们就可以基于这个猜想去看看代码的实现. 

### Dubbo 的扩展点实现原理

在查看它的代码之前,大家先思考两个问题, 所谓的扩展点, 就是通过制定目录下配置一个对应接口的实现类, 然后程序会进行查找和解析, 找到对应的扩展点. 那么这里就涉及到两个问题. 

1. 怎么解析
2. 被加载的类如何存储和使用

####  ExtensionLoader.getExtensionLoader(Protocol.class).getExtension("name")

我们从这段代码着手, 去看看到底做了什么事情, 能够通过这样一段代码实现扩展协议的查找和加载. 

```java
 @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }

        //  初始化ExtensionLoader
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

####  实例化ExtensionLoader

如果当前的type= ExtensionFactory.type ,那么   objectFactory  == null, 否则会创建一个自适应扩展点给到 objectFactory

```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

####  getExtension

```java
   @SuppressWarnings("unchecked")
    public T getExtension(String name) {
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        // 如果name 为true, 表示返回一个默认的扩展点
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        Holder<Object> holder = getOrCreateHolder(name);
        // 缓存一下, 如果实例中已经加载过, 则直接从缓存中获取
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    // 根据名称创建实例
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```



#### createExtension

根据名称创建扩展,   :getExtensionClasses()   加载指定路径下的所有文件

```java
   @SuppressWarnings("unchecked")
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            // 如果没有找到, 抛出异常
            throw findException(name);
        }
        try {
            // 这里用一个chm 来保存实例,做缓存使用.
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            // 实例注入, 对这里实例中的成员属性实现依赖注入的功能.
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```



####  getExtensionClasses()

这个方法, 会查找指定目录, ` /META-INF/dubbo || /META-INF/services` 下对应的type-> 也就是本次演示案例中Protocol 的properties 文件, 然后扫描这个文件下的配置信息, 然后保存到一个HashMap中(classes),key=name(对应protocol 文件中配置的myProtocol),value=对应配置的类的实例

```java

    private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    // 这里的代码就是假造的过程. 
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```



####  injectExtension

```java
 private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                // 获取实例对应的方法,判断方法是否是一个set方法
                for (Method method : instance.getClass().getMethods()) {
                    if (isSetter(method)) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        // 可以选择禁用依赖注入
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        // 获取方法的参数, 这个参数必须是一个对象类型并且是一个扩展点
                        Class<?> pt = method.getParameterTypes()[0];
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        try {
                            // 获取set方法中的属性名称, 并且根据属性名字进行加载
                            String property = getSetterProperty(method);
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                // 调用set方法进行赋值
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("Failed to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```



分析到这里, 我们发现所谓的扩展点, 套路有一样, 不管是 springfactorieyLoader，还是 Dubbo 的SPI. 实际上, dubbo 的功能还更强大, 比如自适应扩展点, 比如依赖注入. 



###  Adaptive 自适应扩展点 

什么叫自适应扩展点呢? 我们先来演示一个例子, 在下面这个例子中, 我们传入一个 Compiler  接口, 它就会传入一个  AdaptiveCompiler。这个就叫自适应。 

```java
        Compiler adaptiveExtension = ExtensionLoader.getExtensionLoader(Compiler.class).getAdaptiveExtension();
        System.out.println(adaptiveExtension.getClass());
```

它是怎么实现的呢? 我们呢根据返回的AdaptiveCompiler 上, 看到类上有一个注解@Adaptive. 这个就是一个自适应扩展点的标识.它可以修饰在类上, 也可以修饰在方法上, 这两者有什么区别呢? 

简单来说, 放在类上, 说明当前类是一个确定的自适应扩展点的类. 如果放在方法级别, 那么需要生成一个动态字节码, 来进行转发. 

比如拿Protocol 这个接口来说, 它里面定义了 export 和 refer 两个抽象方法, 这两个方法分别带有@Adaptive 注解, 标识是一个自适应方法. 

我们知道 Protocol 是一个通信协议的接口, 具体有多种实现, 那么这个时候选择那一种呢? 取决于我们在使用dubbo 的时候配置的协议名, 而这里的方法层面的@Adaptive 就决定了当前这个方法会采用何种协议来发布服务. 

####  getAdaptiveExtension()

这个方法主要就是根据传入的接口返回一个自适应的实现类

```java
   public T getAdaptiveExtension() {
        // cachedAdaptiveInstance 是一个缓存, 在dubbo 中大量用到了这种内存缓存
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            // 很明显,这里是创建一个自适应扩展点的实现
                            instance = createAdaptiveExtension();
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```

####  createAdaptiveExtension()

这个方法中做两件事情: 

1. 获取一个自适应扩展点的实例
2. 实现依赖注入

```java
  private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

getAdaptiveExtensionClass() 这个方法在前面讲过了, 会加载当前传入类型的所有扩展点, 保存在一个HashMap中, 这里有一个判断逻辑, 如果  cachedApdaptiveClass！= null, 直接返回这个  cachedApdaptiveClass .

>   cachedApdaptiveClass , 还记得前面讲过 Adaptive  可以放在两个位置, 一个是类级别, 一个是方法级别. 那么这个cachedApdaptiveClass  很明显, 就是放在类上的Adaptive.
>
>  cachedAdaptiveClass  应该是加载解析 /META-INF/dubbo 下的扩展点的时候加载进来的, 在加载完之后如果这个类有 @Adaptive 标识, 则会赋值. 
>
>   

如果cachedAdaptiveClass   不存在, dubbo 会动态生成一个代理类 `Protocol$Adaptive`,  前面的名字是protocol 是根据前面  ExtensionLoader  所加载的扩展点来定义的. 

```java
   private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```



#### createAdaptiveExtensionClass

动态生成字节码, 然后进行动态加载。 那么这个时候返回的class 如果加载的是Protocol.class, 就应该是 Protocol$Adaptive , 这个 cachedDefaultName  实际上就是扩展点接口的@SPI 注解对应的名字, 如果此时加载的是Protocol.class, 那么 cachedDefaultName=dubbo  

```java
   private Class<?> createAdaptiveExtensionClass() {
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        ClassLoader classLoader = findClassLoader();
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```



####  Protocol$Adaptive

动态生成的代码类, 以下 是通过debug 拿到的代码类. 

```java
package org.apache.dubbo.rpc;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
public void destroy()  {
throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public int getDefaultPort()  {
throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
}
public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
org.apache.dubbo.common.URL url = arg0.getUrl();
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.export(arg0);
}
public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
if (arg1 == null) throw new IllegalArgumentException("url == null");
org.apache.dubbo.common.URL url = arg1;
String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
return extension.refer(arg0, arg1);
}
}
```

####  图形理解

简单来说, 上面的基于方法层面的@Adaptive,基于实现原理的图形大概是这样的

![](http://files.luyanan.com//img/20191125120700.png)

####   injectExtension 

对于扩展点进行依赖注入,简单来说, 如果当前加载的扩展点中存在一个成员属性(对象), 并且提供了set方法,那么这个方法就会执行依赖注入的. 

```java
 private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                // 获取实例对应的方法,判断方法是否是一个set方法
                for (Method method : instance.getClass().getMethods()) {
                    if (isSetter(method)) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        // 可以选择禁用依赖注入
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        // 获取方法的参数, 这个参数必须是一个对象类型并且是一个扩展点
                        Class<?> pt = method.getParameterTypes()[0];
                        // 如果不是对象类型, 就跳过
                        if (ReflectUtils.isPrimitives(pt)) {
                            continue;
                        }
                        try {
                            // 获取set方法中的属性名称, 并且根据属性名字进行加载
                            // 根据class 以及name, 使用自适用扩展点进行加载并且赋值到当前的set方法
                            String property = getSetterProperty(method);
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                // 调用set方法进行赋值
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("Failed to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```



####   objectFactory

在 injectExtension  这个方法中, 我们发现入口出的代码首先判断了objectFactory 这个对象是否为空, 这个是在哪里初始化的呢? 实际上我们在获取 ExtensionLoader  的时候, 就对 objectFactory  进行了初始化. 

```java
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```



然后通过 `ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension())` 去获取一个自适应的扩展点, 进入到ExtensionFactory 这个接口中, 可以看到他是一个扩展点, 并且有一个自己实现的自适应扩展点 AdaptiveExtensionFactory . 

> @ Adaptive 加载到类上表示这是一个自适应的适配器类, 表明我们在调用  getAdaptiveExtension 方法的时候, 不需要走上面那么复杂的流程, 会直接加载到 AdaptiveExtensionFactory。然后在  getAdaptiveExtensionClass()方法处有判断 

![](http://files.luyanan.com//img/20191125135900.png)我们看到除了自定义的自适用适配器以外, 还有别的实现, 一个是SPI， 一个是 AdaptiveExtensionFactory ,  AdaptiveExtensionFactory  轮询这两个, 从一个中获取到就返回. 

```java
    @Override
    public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
```

###  Activate  自动激活扩展点

自动激活扩展点, 有点类似于springboot的  conditional，根据条件进行自动激活. 但是这里涉及的初衷是对于一个类会加载多个扩展点的实现, 这个时候可以通过自动激活扩展点进行动态加载, 从而简化我们的配置工作.

举个例子, 我们可以看看 org.apache.dubbo.Filter 这个类, 它有非常多的实现, 比如说  CacheFilter , 这个缓存过滤器, 配置信息如下:

```java
@Activate(group = {CONSUMER, PROVIDER}, value = CACHE_KEY)
public class CacheFilter implements Filter {
    
```

通过下面这段代码, 演示关于Filter 的自动激活扩展点的效果, 当不添加注释的那段代码的时候, 结果为10, 添加之后结果为11, 会自动把 cacheFilter  加载进来. 

```java
        URL url = new URL("", "", 0);
//        url.addParameter("cache", "cache");
        ExtensionLoader<Filter> loader = ExtensionLoader.getExtensionLoader(Filter.class);
        List<Filter> filters = loader.getActivateExtension(url, "cache");
        System.out.println(filters.size());
```

