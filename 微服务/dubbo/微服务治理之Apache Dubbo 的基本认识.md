# 微服务治理之Apache Dubbo 的基本认识

## 为什么要使用dubbo

###  远程通信背景

技术架构的发展从单体到分布式，是一种顺势而为的架构演进,也是一种被逼无奈的技术变革. 架构的复杂度能够体现公司的业务的复杂度,也能从侧面体现公司的产品的发展势头是向上的. 

和传统的单体架构相比,分布式多了一个远程服务之间的通信,不管是soa还是微服务,他们本质上都是对于业务服务的提炼和复用. 那么远程服务之间的调用才是实现分布式的关键因素. 

而在远程通信这个领域, 其实有很多的技术,比如JAVA 的RMI、webService、Hessian 、Dubbo、Thrift 等RPC 框架. 现在我们接触的比较多的应该就是RPC 框架Dubbo 以及应用协议Http. 其实每一个技术都是在某一个阶段产生它的价值,随着架构的变化以及需求的变化, 技术的解决方案也在变. 

我们知道RPC 的底层原理, 服务与服务之间的调用无非就是跨进程通信而已. 我们可以使用socket 来实现通信,也可以使用nio来实现高性能通信. 我们不用这些开源的RPC 框架, 也可以完成通信的过程. 但是为什么要用现成的框架呢? 

原因是: 如果我们自己去开发一个网络通信,需要考虑到:

1. 底层网络通信协议的处理
2. 序列化和反序列化的处理工作. 

但是这些工作本身应该是通用的, 应该是一个中间件服务. 为整个公司提供远程通信的服务. 而不应该由业务开发人员来自己实现, 所以才有了这样的rpc框架，使得我们调用远程方法的时候就想调用本地方法那么简单,不需要关心底层通信逻辑. 

###  大规模化服务对于服务治理的要求

我认为到目前为止, 还只是满足了通信的基本需求, 但是当企业开始大规模的服务化以后, 远程通信带来的弊端就越来越明显了,比如说:

1. 服务链路变长, 如何实现对服务链路的跟踪和监控. 
2. 服务的大规模集群使得服务之间需要依赖第三方注册中心来解决服务的发现和服务的感知问题. 
3. 服务通信之间的异常, 需要有一种保护机制防止一个节点故障引发大规模的系统故障, 所以要有容错机制. 
4. 服务大规模集群会使得客户端需要引负载均衡机制来实现转发. 

而这些对于服务治理的要求, 传统的PRC技术在这样的场景中显得有点力不从心, 因为很多的企业开始研发自己的PRC 框架, 比如阿里的HFS、Dubbo;京东的JSF框架、当当的dubbox 、新浪的motan、蚂蚁金服的sofa等. 有技术输出能力的公司,都会研发适合自己场景的rpc框架, 要么从0带1开发, 要么是基于现有的思路结合公司的业务进行改造 . 而没有技术输出能力的公司, 遇到服务治理的需求, 会优先选择哪些比较成熟的开源框架,而Dubbo 就是其中的一个. 

duboo 主要是一个分布式服务治理解决方案, 那么什么是服务治理呢? 服务治理主要是针对大规模服务化以后, 服务之间的路由、负载均衡、容错机制、服务降级这些问题的解决方案, 而Dubbo 实现的不仅仅是远程通信, 并且还解决了服务路由、负载、降级、容错等功能. 

## dubbo 的发展历史

Dubbo 是阿里内部使用的一个分布式服务治理框架,2012年开源, 因为Dubbo 在公司内部经过了很多的验证相对来说比较成熟,所以在很短的时间内就被很多互联网公司使用, 再加上阿里出来的很多技术大牛进入各个创业公司担仁技术架构后,都以Dubbo 作为主推的RPC框架使得dubbo 很快成为了很多互联网公司的首要选择. 并且很多公司在应用dubbo时, 会基于自身业务特性进行优化和改进, 所以也衍生了很多的把本. 比如京东的JSF、比如新浪的Motan、比如当当的dubbox. 

在2014年10月份, dubbo 停止了维护, 在后来的2017年的9月份, 阿里宣布重启dubbo, 并且对于dubbo 做好了长期投入的准备,并且在这段时间内dubbo 进行了非常多的更新,目前的版本已经达到了2.7 . 

2018年1月8号, Dubbo 创始人之一梁飞在Dubbo 交流群里透漏了Dubbo 3.0 正在动工的消息, Dubbo 3.0 内核于Dubbo 2.0 完全不同, 但兼容Dubbo 2.0 。Dubbo 3.0 将支持可选Service Mesh. 

2018年2月份, Dubbo 捐给了Apache. 另外, 阿里巴巴对于Spring Cloud Alibaba 生态的完善, 以及Spring Cloud 团队对于 alibaba 整个服务治理生态的支持, 所以Dubbo 未来仍然是国内绝大部分公司的首要选择. 

## Dubbo 的基本使用.

我们基于dubbo 的最新版本2.7 来开始讲解, 首先还是基于一个demo 使用dubbo 完成基本的远程通信

创建三个项目, 一个server, 一个client,最后一个对外提供的api 项目

###  接口项目

我们在接口项目里面编写一个api类, 作为对外提供的接口

```java
package com.dubbo.simple;

/**
 * @author luyanan
 * @since 2019/11/20
 * <p>用户api</p>
 **/
public interface UserApi {


    String info(String id);

}

```



###  服务端项目

添加 dubbo、logback 和 上面我们编写的api 项目的jar 依赖

####   pom  jar 依赖

```xml
  <dependency>
            <groupId>com.dubbo.simple</groupId>
            <artifactId>dubbo-simple-api</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.2</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.29</version>
        </dependency>

        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.2.3</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>
```



接下来我们实现UserApi,并且配置dubbo 相关的东西

####  接口实现

```java
package com.dubbo.simple;

/**
 * @author luyanan
 * @since 2019/11/20
 * <p>接口实现</p>
 **/
public class UserApiImpl implements UserApi {


    @Override
    public String info(String id) {
        return "张三:" + id;
    }
}

```



#### 创建配置文件发布服务

在resulrces/META-INF/spring 下创建 application.xml  文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"

       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://code.alibabatech.com/schema/dubbo
http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!--  提供方应用信息,用于计算依赖关系-->
    <dubbo:application name="dubbo-simple-server"/>

    <!-- 使用multicast 广播注册中心暴露服务地址 -->
    <dubbo:registry address="N/A"/>

    <!-- 用dubbo 协议在20880 端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="20880"/>

    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.dubbo.simple.UserApi" ref="userApi"/>

    <!-- 和本地bean 一样实现服务 -->
    <bean id="userApi" class="com.dubbo.simple.UserApiImpl"/>
</beans>

```



####  添加日志支持 logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="STDOUT"
              class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%date{ISO8601} %-5level
                [%thread] %logger{32} - %message%n</pattern>
        </layout>
    </appender>
    <root>
        <level value="DEBUG"/>
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```



#### 启动服务

> ```
> Main.main(args);
> ```
>
> 

### 客户端服务

####  添加配置文件

在resources 下 创建 application.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"

       xsi:schemaLocation="http://www.springframework.org/schema/beans
http://www.springframework.org/schema/beans/spring-beans.xsd
http://code.alibabatech.com/schema/dubbo
http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!--  提供方应用信息,用于计算依赖关系-->
    <dubbo:application name="dubbo-simple-server"/>

    <!-- 使用multicast 广播注册中心暴露服务地址 -->
    <dubbo:registry address="N/A"/>


    <dubbo:reference interface="com.dubbo.simple.UserApi" id="userApi" url="dubbo://192.168.86.1:20880/com.dubbo.simple.UserApi"/>
</beans>

```

其中 , dubbo:reference 中的url 地址就是 userApi 接口暴露出来的地址，我们可以在server 端启动的时候从日志中找出. 

####  启动client

```java
public class App {
    public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext(new String[]{"application.xml"});
        UserApi userApi = applicationContext.getBean(UserApi.class);
        System.out.println(userApi.info("111"));
    }
}

```

运行之后, 我们就可以看到

> 张三:111

## 关于DubboMain 启动的真相

我们刚刚使用  Main.main(args);  来启动dubbo 服务, 到底是如何实现的呢? 

正常情况下. 我们会认为服务的发布, 需要tomcat或者jetty 这类的容器支持, 但是只用dubbo 后, 我们并不需要这样重的容器去支持, 同时也会增加复杂性和浪费资源,Dubbo 提供了几种容器让我们去启动和发布服务. 

### 容器类型：

1.  Spring Container : 自动加载 META-INF/spring   目录下的所有spring 配置. 
2.  logback Container: 自动装配 logback 日志
3.  Log4j Container: 自动配置log4j 的配置

Dubbo 提供了一个Main.main 快速启动相应的容器, 默认情况下, 只会启动spring 容器. 

### 原理分析

默认情况下, spring 容器本质上就是加载spring ioc 容器, 然后启动一个netty 服务实现服务的发布, 下面是spring 容器启动的代码:

```java
  public void start() {
        String configPath = ConfigUtils.getProperty("dubbo.spring.config");
        if (StringUtils.isEmpty(configPath)) {
            configPath = "classpath*:META-INF/spring/*.xml";
        }

        context = new ClassPathXmlApplicationContext(configPath.split("[,\\s]+"), false);
        context.refresh();
        context.start();
    }
```



## 基于注册中心的Dubbo 服务

作为主流的服务治理组件, Dubbo 提供了很多丰富的功能, 那么最根本的就是要解决大规模集群之后的服务注册和发现的问题, 而dubbo 中对于注册中心这块是使用zookeeper 来支持的, 当前在目前最新的版本中, dubbo 能支持的注册中心有: consul、etcd、nacos、sofa、zookeeper、redis、multicast.

###  使用zookeeper 作为注册中心

#### pom修改

在服务端和客户端代码里面添加zookeeper 的jar

```xml
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.0.0</version>
        </dependency>
```

#### 服务端修改

##### 修改application.xml 配置文件

> ```
> <dubbo:registry address="zookeeper://192.168.86.128:2181"/><!--如果是集群, 则配置的方式是 address=”zookeeper://ip:host?backup=ip:host,ip:host”  -->
> ```

####  客户端修改

#####  修改application.xml 配置文件

```xml
    <dubbo:registry address="zookeeper://192.168.86.128:2181"/>
    <dubbo:reference interface="com.dubbo.simple.UserApi" id="userApi"/>
```

#### dubbo 集成 zookeeper 的实现原理

![](http://files.luyanan.com//img/20191120114700.png)

#### dubbo 缓存配置中心

在消费端的配置文件中指定如下路径

> ```
> <dubbo:registry address="zookeeper://192.168.86.128:2181" file="d:/dubbo-server"/>
> ```



### 多注册中心支持

Dubbo 中可以支持多注册中心, 有的时候， 客户端需要用调用的远程服务不在同一个注册中心, 那么客户端就需要配置多个注册中心来访问, 演示一个案例. 

####  配置多个注册中心

```java
    <dubbo:registry id="reg1" address="zookeeper://192.168.86.128:2181"/>
    <dubbo:registry id="reg2" address="zookeeper://192.168.86.129:2181"/>
```

#### 将服务注册到不同的注册中心

通过registry 设置注册中心的id

> ```xml
> <dubbo:service interface="com.dubbo.simple.UserApi" ref="userApi" register="reg1"/>
> ```

消费端配置多个注册中心, 实现代码和服务端一样. 

#### 注册中心的其他支持

1. 当设置 `<dubbo:registry  check="false"/> ` 时, 记录失败注册和订阅请求, 后台定时重试. 
2. 可通过 `<dubbo:registry  username="admin" password="1234"/>` 设置zookeeper 登录信息
3. 可通过 `<dubbo:registry group="dubbo"/>`  设置zookeeper 的根节点, 默认使用dubbo 作为dubbo 服务注册的namespace

## Dubbo 仅仅是一个RPC框架

到目前为止, 我么了解到Dubbo 的核心功能, 提供服务注册和服务发现, 以及基于Dubbo协议的远程通信, 我想, 大家以后不仅仅只认为Dubbo 是一个RPC 框架吧。 Dubbo 从另一个方面来看也可以认为是一个服务治理生态, 从目前已经讲过的内容上可以看到.

1. Dubbo 可以支持市面上主流的注册中心
2. Dubbo 提供了 Container 的支持, 默认提供了3种  Container ,我们还可以自行扩展. 
3. Dubbo 对于RPC 通信协议的支持, 不仅仅是原生的Dubbo协议, 它还围绕着rmi、hessian、http、webservice、thrift、rest.

有了多协议的支持,使得其他rpc 框架的应用程序可以快速的切入到dubbo 生态中。同时, 对于多协议的支持, 是的不同应用场景的服务, 可以选择合适的协议来发布服务, 并不一定使用dubbo 提供的长连接方式. 

###  集成 webservice 协议

webservice 是一个短连接 并且是基于http协议的方式来实现的rpc 框架

#### jar 依赖

```xml
<dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-frontend-simple</artifactId>
            <version>3.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.cxf</groupId>
            <artifactId>cxf-rt-transports-http</artifactId>
            <version>3.3.2</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jetty</groupId>
            <artifactId>jetty-server</artifactId>
            <version>9.4.19.v20190610</version>
        </dependency>
        <dependency>
            <groupId>org.eclipse.jetty</groupId>
            <artifactId>jetty-servlet</artifactId>
            <version>9.4.19.v20190610</version>
        </dependency>
```



#### 修改application.xml

```xml
    <dubbo:protocol name="webservice" port="8080" server="jetty"/>
    <!-- 声明需要暴露的服务接口 -->
    <dubbo:service interface="com.dubbo.simple.UserApi" ref="userApi"  protocol="dubbo,webservice"/>
```

添加多协议支持, 一个服务可以发布多种协议的支持,也可以实现不同服务发布不同的协议 

启动服务之后, 可以使用 ` http://localhost:8080/com.dubbo.simple.UserApi?wsdl ` 来获取到 webservice 的wsdl 描述文档

#### 客户端使用webservice 请求服务

1. 客户端的配置jar依赖
2. 配置多个协议支持, 实现方式和服务端一样. 

### Dubbo 对于REST 协议的支持

Dubbo 中的REST(表述性资源转移)支持, 是基于 于 JAX-RS2.0(Java API for RESTful Web Services)  来实现的. 

REST 是一种架构风格, 简单来说就是对于api 接口的约束, 基于URL 定位资源, 使用http 动词(GET/POST/PUT/DELETE) 来描述操作. 

#### JAX-RS协议说明

REST 很早就提出来了, 在早期的开发人员为了实现REST, 会使用各种工具来实现, 比如Servlets 就经常用来开发RESTful 的程序,随着REST 被越来越多的开发人员采用, 所以 JCP(Java community process)  提出了 JAX-RS 规范, 并且提供了一种新的基于注解的方式来开发RESTful 服务. 有了这样的一个规范, 使得开发人员不需要通讯层的东西, 只需要关注资源以及数据对象. 

 JAX-RS 规范的实现有：Apache CXF、Jersey(由 Sun 公司提供的 JAX-RS 的 参考实现)、RESTEasy( jboss 实现)等。  

而Dubbo 里面实现的REST 就是基于JBoss 提供的  RESTEasy  框架来实现的, Spring MVC 中的RESTful 实现我们用的比较多, 它也是 JAX-RS 规范的一种实现. 

#### jar 依赖

```xml
<dependency>
 <groupId>org.jboss.resteasy</groupId>
 <artifactId>resteasy-jaxrs</artifactId>
 <version>3.8.0.Final</version>
</dependency>
<dependency>
 <groupId>org.jboss.resteasy</groupId>
 <artifactId>resteasy-client</artifactId>
 <version>3.8.0.Final</version>
</dependency>

```

#### 添加新的协议支持

> <dubbo:protocol name="rest" port="8080" server="jetty"/>

####   提供新的服务

@Path("user"):  指定访问UserApi 的url 的相对路径是 /user,即http://localhost:8080/user

@Path("info/{id}") 指定访问info 方法的url 相对路径为/info,再结合上一个@Path 为 UserApi 指定的路径, 则调用UserApi.info() 的完整路径为 http://localhost:8080/user/info 

@GET: 指定访问info 方法用HTTP  GET 方法

```java
@Path("user")
public interface UserApi {


    @GET
    @Path("info/{id}")
    String info(@PathParam("id") String id);

}

```

#### 客户端配置

> ```
> <dubbo:reference interface="com.dubbo.simple.UserApi" id="userApi" protocol="rest"/>
> ```

#### 在服务接口获取上下文的方式

既然是http 协议的REST 接口, 那么我们想要获取请求的上下文, 怎么做呢? 

1. 第一种, 

>  HttpServletRequest request=(HttpServletRequest)RpcContext.getContext().getRequest(); 

2. 通过注解的方式

   ```java
   @Path("user")
   public interface UserApi {
   
   
       @GET
       @Path("info/{id}")
       String info(@PathParam("id") String id, @Context HttpServletRequest servletRequest);
   
   }
   
   ```

   



## Dubbo  监控平台的安装

Dubbo 的监控平台也做了更新, 不过目前的功能还没有完善, 在这个网站上 下载 Dubbo-Admin 的包. 

https://github.com/apache/dubbo-admin

1. 修改  dubbo-admin-server/src/main/resources/application.properties   中的配置信息
2.  mvn clean package 进行构建 
3.  mvn –projects dubbo-admin-server spring-boot:run 
4.  访问 localhost:808 

## Dubbo 的终端操作方法. 

Dubbo 中提供了一种基于终端操作的方法来实现服务治理. 使用 telnet localhost 20880   连接到服务对应的端口. 

### 常见命令

#### ls

-  ls: 显示服务列表 
- ls -l: 显示服务详细信息列表 
- ls XxxService: 显示服务的方法列表 
- ls -l XxxService: 显示服务的方法详细信息列表 

####  ps

- ps: 显示服务端口列表 
- ps -l: 显示服务地址列表 
- ps 20880: 显示端口上的连接信息 
- ps -l 20880: 显示端口上的连接详细信息 

####  cd 

-  cd XxxService: 改变缺省服务，当设置了缺省服务，凡是需要输入服务名作 为参数的命令，都可以省略服务参数 
- cd /: 取消缺省服务 

####  pwd

-  pwd: 显示当前缺省服务 

####  count

-  count XxxService: 统计 1 次服务任意方法的调用情况 
- count XxxService 10: 统计 10 次服务任意方法的调用情况 
- count XxxService xxxMethod: 统计 1 次服务方法的调用情况 
- count XxxService xxxMethod 10: 统计 10 次服务方法的调用情况 