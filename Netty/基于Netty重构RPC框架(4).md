#   基于Netty 重构RPC框架

## 1. RPC 概述

下面的这张图, 大概很多人都见到过, 这是Dubbo 官网中的一张图描述了项目架构的演进过程:

![](http://files.luyanan.com//img/20190920102437.png)

他描述了每一种架构需要的具体配置和组织形态. 当网站流量很小时， 只需一个应用, 将所有的功能都部署到一起, 以减少部署节点和成本，我们通常会采用单一应用架构,之后出现了ORM框架，主要用于简化增删改查工作流的,数据访问框架ORM是关键.

随着用户量增加, 当访问量逐渐增加, 单一应用增加机器, 带来的加速度越来越小 , 我们需要将应用超分成互相不干扰的几个应用, 以提升效率 。 于是出现了垂直应用架构. MVC架构就是一种非常经典的用于加速前端页面开发的架构.

当垂直应用越来越多, 应用之间交互不可避免, 将核心业务抽取出来, 作为独立的服务逐渐形成稳定的服务中心, 使前端应用能够更快速的响应, 多变的市场需求, 就出现了分布式服务架构, 分布式架构下服务数量逐渐增加, 为了提高管理效率, PRC 框架应运而生, PRC 用于提高业务复用以及整合的, 分布式服务框架下RPC 是关键,

下一代框架, 将会是流动计算架构占据主流, 当服务越来越多,容量的评估, 小服务的资源浪费等问题，逐渐明显. 此时, 需要增加一个调度中心, 基于访问压力实时管理集群容量, 提高集群利用率, SOA 架构就是用于提高及其利用率的, 资源调度和治理中心SOA是关键.

Netty 基本上是作为架构的技术底层而存在的, 主要完成高性能的网络通信.

## 2. 环境预设

###  1. 先将项目环境搭建起来, 创建项目, 在pom.xml 中添加

```xml
	<dependency>
		  <groupId>io.netty</groupId>
		  <artifactId>netty-all</artifactId>
		  <version>4.1.6.Final</version>
		</dependency>

		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
			<version>1.16.10</version>
		</dependency>
```

### 2. 创建项目结构

在没有RPC框架之前,我们的服务调用是这样的, 如下图: 

![](http://files.luyanan.com//img/20190921143134.png)

从上图可以看出接口的调用完成没有规律可寻, 想怎么调就怎么调, 这导致业务发展到一定阶段后, 对接口的维护变得非常困难, 于是有人提出了服务治理的概念, 所有服务间不允许直接调用, 而是先到注册中心进行登记, 再由注册中心统一协调和管理所有的服务状态并对外发布, 调用者只需要记住服务名称， 去找注册中心获取服务即可.

这样极大的规范了服务的管理, 可以题号了所有服务端可控性, 整个设计思想其实在我们生活中也能找到活生生的例子. 例如 : 我们平时工作交流, 大多都是用IM工具, 而不是面对面吼. 大家只需要相互记住运行商(也就是注册中心) 提供的号码(如: 腾讯QQ) 即可, 再比如: 我们打电话, 所有电话号码由运营商分配, 我们需要和某一个人通话 , 只需要拨通对方的号码,运营商(注册中心,如中国联通、移动、电信) 就会帮我们将信号转接过去.

目前流行的RPC 服务治理框架主要有Dubbo 和Spring Cloud ,  下面我们以比较经典的Dubbo 为例,. Dubbo 核心模块主要有四个:

- Registry 注册中心
- Priovider 服务端
- Consumer 消费端
- Monitor 监控中心

为了方便, 我们将所有的模块全部都放到同一个项目中,主要模块包括

- api: 主要用来定义对外开放的功能与服务接口
- protocol: 主要定义自定义传输协议的内容
- registry: 主要负责保存所有可用的服务名和服务地址
- provider: 实现对外提供的所有服务的具体功能
- constomer: 客户端调用
- monitor： 完成调用链监控

下面, 我们先将项目结构搭建好, 具体的项目结构截图如下: 

![](http://files.luyanan.com//img/20190920165539.png)

## 3. 代码实战

### 1. 创建API 模块

首先创建API模块, prodiver 和consumer 都遵循API模块的规范, 为了简化,创建两个Service 接口, 分别是IHelloService   接口, 实现一个hello() 方法, 主要目的是用来确认服务是否可用, 具体代码如下:

```java
package com.formula.netty.rpc.api;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public interface IHelloService {

    String hello(String name);

}

```

创建 ICalculationService 接口， 完成模拟业务 加、减、乘、除 运算， 具体代码如下:

```java
package com.formula.netty.rpc.api;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p>计算的service</p>
 **/
public interface ICalculationService {


    /**
     * <p>加</p>
     *
     * @param a
     * @param b
     * @return {@link int}
     * @author luyanan
     * @since 2019/9/20
     */
    int add(int a, int b);

    /**
     * <p>减</p>
     *
     * @param a
     * @param b
     * @return {@link int}
     * @author luyanan
     * @since 2019/9/20
     */
    int sub(int a, int b);

    /**
     * <p>乘</p>
     *
     * @param a
     * @param b
     * @return {@link int}
     * @author luyanan
     * @since 2019/9/20
     */
    int mult(int a, int b);

    /**
     * <p>除</p>
     *
     * @param a
     * @param b
     * @return {@link int}
     * @author luyanan
     * @since 2019/9/20
     */
    int div(int a, int b);
}

```

至此, API模块就定义完成了,非常简单, 接下来我们要确定传输规则, 也就是传输协议. 协议内容当然要自己定义才能显出Netty的优势.

### 2. 创建自定义协议

在Netty中要完成一个自定义协议, 其实非常简单, 只需要定义一个普通的Java类即可. 我们现在手写PRC 主要完成对Java代码的远程调用(类似于RMI)，远程调用Java代码哪些内容是必须由网络来传输的呢? 比如: 服务名称？需要调用该服务的哪个方法？方法的实参是什么? 这些信息都需要通过客户端传送到服务端去.

下面我们来看一下具体的代码实现, 定义 `InvokerProtocol`

```java
package com.formula.netty.rpc.protocol;

import lombok.Data;

import java.io.Serializable;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p>自定义传输协议</p>
 **/
@Data
public class InvokerProtocol implements Serializable {

    /**
     * <p>类名</p>
     *
     * @author luyanan
     * @since 2019/9/20
     */
    private String className;

    /**
     * <p>方法名</p>
     *
     * @author luyanan
     * @since 2019/9/20
     */
    private String method;

    /**
     * <p>形参列表</p>
     *
     * @author luyanan
     * @since 2019/9/20
     */
    private Class<?>[] paramsType;


    /**
     * <p>参数列表</p>
     *
     * @author luyanan
     * @since 2019/9/20
     */
    private Object[] params;

}

```

从上面的代码来看, 协议中主要包含类名、函数名、形参列表和实参列表， 通过这些信息可以定位到一个具体的业务逻辑实现.

### 3. 实现Provider 服务端业务逻辑

我们将API中定义的所有功能在provider 中实现, 分别创建两个实现类:

HelloServiceImpl

```java
package com.formula.netty.rpc.provider;

import com.formula.netty.rpc.api.IHelloService;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class HelloServiceImpl implements IHelloService {


    @Override
    public String hello(String name) {
        return "Hello:" + name;
    }
}

```

CalculationServiceImpl

```java
package com.formula.netty.rpc.provider;

import com.formula.netty.rpc.api.ICalculationService;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class CalculationServiceImpl implements ICalculationService {


    @Override
    public int add(int a, int b) {
        return a + b;
    }

    @Override
    public int sub(int a, int b) {
        return a - b;
    }

    @Override
    public int mult(int a, int b) {
        return a * b;
    }

    @Override
    public int div(int a, int b) {
        return a / b;
    }
}

```

###  4 . 完成Registry 服务注册

Registry 注册中心主要功能就是将所有Provider 的服务名称和服务引用地址都注册到一个容器中, 并对外发布。Registry 应该要启动一个对外的服务, 很明显应该作为服务端, 并提供一个对外可以访问的接口, 先启动一个Netty, 创建Registry 类, 具体代码如下:

```java
package com.formula.netty.rpc.registry;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.codec.LengthFieldPrepender;
import io.netty.handler.codec.serialization.ClassResolver;
import io.netty.handler.codec.serialization.ClassResolvers;
import io.netty.handler.codec.serialization.ObjectDecoder;
import io.netty.handler.codec.serialization.ObjectEncoder;


/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class Registry {


    private int port;

    public Registry(int port) {
        this.port = port;
    }

    public void start() {

        EventLoopGroup boosGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {


            ServerBootstrap strap = new ServerBootstrap();

            strap.group(boosGroup, workerGroup)

                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {

                            ChannelPipeline pipeline = ch.pipeline();
                            //自定义协议解码器
                            /** 入参有5个，分别解释如下
                             maxFrameLength：框架的最大长度。如果帧的长度大于此值，则将抛出TooLongFrameException。
                             lengthFieldOffset：长度字段的偏移量：即对应的长度字段在整个消息数据中得位置
                             lengthFieldLength：长度字段的长度。如：长度字段是int型表示，那么这个值就是4（long型就是8）
                             lengthAdjustment：要添加到长度字段值的补偿值
                             initialBytesToStrip：从解码帧中去除的第一个字节数
                             */
                            pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
                            //自定义协议编码器
                            pipeline.addLast(new LengthFieldPrepender(4));
                            //对象参数类型编码器
                            pipeline.addLast("encoder",new ObjectEncoder());
                            //对象参数类型解码器
                            pipeline.addLast("decoder",new ObjectDecoder(Integer.MAX_VALUE,ClassResolvers.cacheDisabled(null)));
                            pipeline.addLast(new RegistryHandler());

                        }

                    })
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            ChannelFuture future = strap.bind(port).sync();
            System.out.println("RPC Registry listen : " + this.port);
            future.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            boosGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }


    public static void main(String[] args) {
        new Registry(8080).start();
    }
}

```

在RegistryHandler 中实现注册的具体逻辑，上面的代码主要实现服务注册和服务调用的功能, 因为所有模块都创建在同一个项目中, 为了简化，服务端没有实现远程调用, 而是直接扫描本地的class,然后利用反射调用. 代码实现如下：

```java
package com.formula.netty.rpc.registry;

import com.formula.netty.rpc.protocol.InvokerProtocol;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;

import java.io.File;
import java.lang.reflect.Method;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class RegistryHandler extends ChannelInboundHandlerAdapter {

    /**
     * <p>用来保存所有可用的服务</p>
     *
     * @author luyanan
     * @since 2019/9/20
     */
    public static Map<String, Object> registryMap = new ConcurrentHashMap<>();


    /**
     * <p>保存所有相关服务类</p>
     *
     * @author luyanan
     * @since 2019/9/20
     */
    private static List<String> classNames = new ArrayList<>();


    public RegistryHandler() {
        // 递归扫描
        scannerClass("com.formula.netty.rpc.provider");
        doRegistry();
    }

    private void doRegistry() {
        if (classNames.size() == 0) {
            return;
        }
        try {
            for (String className : classNames) {
                Class<?> clazz = Class.forName(className);
                Class<?> i = clazz.getInterfaces()[0];
                registryMap.put(i.getName(), clazz.newInstance());
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }

    private void scannerClass(String pkg) {

        URL url = this.getClass().getClassLoader().getResource(pkg.replaceAll("\\.", "/"));
        File dir = new File(url.getFile());
        for (File file : dir.listFiles()) {
            if (file.isDirectory()) {
                scannerClass(pkg + "." + file.getName());
            } else {
                classNames.add(pkg + "." + file.getName().replaceAll(".class", "").trim());
            }
        }
    }


    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        Object result = new Object();
        InvokerProtocol protocol = (InvokerProtocol) msg;
        // 当客户端建立连接时, 需要从自定义协议中获取信息， 拿到具体的服务和实参
        // 使用反射调用
        if (registryMap.containsKey(protocol.getClassName())) {
            Object clazz = registryMap.get(protocol.getClassName());
            Method method = clazz.getClass().getMethod(protocol.getMethod(), protocol.getParamsType());
            result = method.invoke(clazz, protocol.getParams());
        }
        ctx.write(result);
        ctx.flush();
        ctx.close();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

至此, 注册中心的功能就基于完成, 接下来来看一下客户端的代码实现.

### 5. 实现Consumer 远程调用

梳理一下基本的实现思路, 主要完成这样一个功能, API模块中的接口功能主要在服务端实现(没有在在客户端实现), 因此, 客户端调用API接口中定义的某一个方法的时候, 实际上是要发起一次网络情况去调用服务端的某一个服务, 而这个网络服务首先被注册中心接收, 由注册中心先确定要调用的服务的位置, 再将请求转发至真正的服务实现, 最终调用服务端代码. 将返回值通过网络传输给客户端. 整个过程对于客户端而言是完成无感知的, 就像调用本地方法一样,

下面我们来看具体的代码实现.

RpcProxy

```java
package com.formula.netty.rpc.consumer.proxy;

import com.formula.netty.rpc.protocol.InvokerProtocol;
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.sctp.nio.NioSctpChannel;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.codec.LengthFieldPrepender;
import io.netty.handler.codec.serialization.ClassResolvers;
import io.netty.handler.codec.serialization.ObjectDecoder;
import io.netty.handler.codec.serialization.ObjectEncoder;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class RpcProxy {

    /**
     * <p>使用反射创建</p>
     *
     * @param clazz
     * @return {@link T}
     * @author luyanan
     * @since 2019/9/20
     */
    public static <T> T create(Class<T> clazz) {

        Class[] interfaces = clazz.isInterface() ? new Class[]{clazz} : clazz.getInterfaces();
        return (T) Proxy.newProxyInstance(clazz.getClassLoader(), interfaces, new ProxyHandler(clazz));
    }


    static class ProxyHandler implements InvocationHandler {

        private Class clazz;

        public ProxyHandler(Class clazz) {
            this.clazz = clazz;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (Object.class.equals(method.getDeclaringClass())) {
                try {
                    return method.invoke(this, args);
                } catch (Throwable t) {
                    t.printStackTrace();
                }
            } else {
                return rpcinvoke(proxy, method, args);
            }
            return null;
        }

        private Object rpcinvoke(Object proxy, Method method, Object[] args) {

            // 传输协议封装
            InvokerProtocol protocol = new InvokerProtocol();
            protocol.setClassName(this.clazz.getName());
            protocol.setMethod(method.getName());
            protocol.setParams(args);
            protocol.setParamsType(method.getParameterTypes());


             final  RpcProxyHandler handler = new RpcProxyHandler();
            EventLoopGroup group = new NioEventLoopGroup();
            try {
                Bootstrap bootstrap = new Bootstrap();
                bootstrap.group(group)
                        .channel(NioSocketChannel.class)
                        .option(ChannelOption.TCP_NODELAY, true)
                        .handler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(SocketChannel ch) throws Exception {

                                ChannelPipeline pipeline = ch.pipeline();
                                //自定义协议解码器
                                /** 入参有5个，分别解释如下
                                 maxFrameLength：框架的最大长度。如果帧的长度大于此值，则将抛出TooLongFrameException。
                                 lengthFieldOffset：长度字段的偏移量：即对应的长度字段在整个消息数据中得位置
                                 lengthFieldLength：长度字段的长度：如：长度字段是int型表示，那么这个值就是4（long型就是8）
                                 lengthAdjustment：要添加到长度字段值的补偿值
                                 initialBytesToStrip：从解码帧中去除的第一个字节数
                                 */
                                pipeline.addLast("frameDecoder", new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
                                //自定义协议编码器
                                pipeline.addLast("frameEncoder", new LengthFieldPrepender(4));
                                //对象参数类型编码器
                                pipeline.addLast("encoder", new ObjectEncoder());
                                //对象参数类型解码器
                                pipeline.addLast("decoder", new ObjectDecoder(Integer.MAX_VALUE, ClassResolvers.cacheDisabled(null)));
                                pipeline.addLast("handler", handler);
                            }
                        });

                ChannelFuture future = bootstrap.connect("localhost", 8080).sync();
                future.channel().writeAndFlush(protocol).sync();
                future.channel().closeFuture().sync();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                group.shutdownGracefully();
            }


            return handler.getResponse();
        }
    }

}

```

接收网络调用的返回值

```java
package com.formula.netty.rpc.consumer.proxy;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import org.omg.CORBA.OBJ_ADAPTER;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class RpcProxyHandler extends ChannelInboundHandlerAdapter {


    private Object response;

    public Object getResponse() {
        return response;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        response = msg;
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("client exception is general");
    }
}

```

完成客户端调用的代码

```java
package com.formula.netty.rpc.consumer;

import com.formula.netty.rpc.api.ICalculationService;
import com.formula.netty.rpc.api.IHelloService;
import com.formula.netty.rpc.consumer.proxy.RpcProxy;

/**
 * @author luyanan
 * @since 2019/9/20
 * <p></p>
 **/
public class RpcCustomer {

    public static void main(String[] args) {
        IHelloService helloService = RpcProxy.create(IHelloService.class);
        String hello = helloService.hello("张三");
        System.out.println(hello);
        ICalculationService calculationService = RpcProxy.create(ICalculationService.class);
        System.out.println(calculationService.add(8, 2));
        System.out.println(calculationService.sub(8, 2));
        System.out.println(calculationService.mult(8, 2));
        System.out.println(calculationService.div(8, 2));
    }

}

```

