#  tomcat基础认知

## 1. What is Tomcat

### 1.1 `Tomcat` 官网

`官网`: https://tomcat.apache.org

> The Apache Tomcat® software is an open source implementation of the Java Servlet, JavaServer Pages, Java Expression Language and Java WebSocket technologies.

### 1.2 Understand

为什么`tomcat` 是`Servlet`技术的实现呢? 
 在我们的理解中,`Tomcat`  可以被称为`web`容器或者`Servlet`容器
不妨手写一个`tomcat` 来推导一下

 ####  1.2.1 创建`Tomcat`类

java 是一门面对对象的语言

```java
//这就是我们写的Tomcat 
class MyTomcat{
  ....   
}
```



#### 1.2.2  web容器

`思考`: 怎么让`TOmcat` 具备`web` 服务的功能? 

![image-20200512085302976](http://files.luyanan.com//img/20200512085323.png)

![image-20200512085329919](http://files.luyanan.com//img/20200512085331.png)

在服务端用`Http` 来监听, 协议不好写，不妨用`java`封装好的`socket` 作为监听

```java
class MyTomcat{
    ServerSocket server=new ServerSocket(8080);
    // 等待客户端的连接请求
    Socket socket=server.accept();
}
```



####  1.2.3 `Servlet` 容器

`思考`: 怎么样让`Tomcat` 具有`Servlet` 容器的功能呢? 说白了就是能够装一个个的`Servlet`



#####  `servlet` 是啥? 

```java
public interface Servlet {
    void init(ServletConfig config) throws ServletException;
    ServletConfig getServletConfig();
    void service(ServletRequest req, ServletResponse res）throws ServletException,IOException;
    String getServletInfo();
    void destroy();
}

```

```java
class LoginServlet extends HttpServlet{
    doGet(request,response){}
    doPost(request,response){}
}
```

```xml
<servlet>
     <servlet-name>LoginServlet</servlet-name>
     <servlet-class>com.web.servlet.LoginServlet</servlet-class>
</servlet>
<servlet-mapping>
     <servlet-name>LoginServlet</servlet-name>
     <url-pattern>/login</url-pattern>
</servlet-mapping>
```



#### 1.2.4 优化`Tomcat`

```java
class MyTomcat{
    List list=new ArrayList();
    ServerSocket server=new ServerSocket(8080);
    Socket socket=server.accept();
    // 把请求和响应都封装在业务代码中的servlet
    // 只要把业务代码中一个个servlets添加到tomcat中即可
    list.add(servlets);
}

```



#### 1.2.5 画个图

![image-20200512090010840](http://files.luyanan.com//img/20200512090011.png)



####  1.2.6  获得的信息

1. `tomcat`需要支持`servlet` 规范

    `tomcat/lib/servlet-api.jar`

2. web容器

   希望`tomcat` 源码中也有`new ServerSocket(8080)`  的代码

3. `servlet` 容器

    希望`tomcat`源码中也会有`list.add(servlets)`的代码



## 2. 产品和源码

###  2.1 `Version Choose`

这里选择的的`Tomcat8.0.11`

`各个版本下载地址`： `https://archive.apache.org/dist/tomcat`

![image-20200512090443086](http://files.luyanan.com//img/20200512090444.png)





### 2.2 产品目录文件含义

- `bin`: 主要用来存放命令,`.bat`的`window`下的,`.sh`是`linux`下的
- `conf`:主要用来存放`tomcat`的配置文件
- `lib`: 存放`tomcat`依赖的一些`jar包`
- `logs`:存放`tomcat` 在运行的时候产生的日志文件
- `temp`: 存放运行时产生的临时文件
- `webapps`: 存放应用程序
- `work`: 存放`tomcat`  运行时编译后的文件, 比如`jsp` 编译后的文件



### 2.3 源码导入与调试

先从官网下载源码包`https://archive.apache.org/dist/tomcat/tomcat-8/v8.0.11/src/apache-tomcat-8.0.11-src.zip`

解压后,用idea 打开tomcat 源码

在源码目录下新建`pom.xml` 文件

![image-20200512093710241](http://files.luyanan.com//img/20200512093711.png)



`pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>Tomcat8.0</artifactId>
    <name>Tomcat8.0</name>
    <version>8.0</version>
    <build>
        <finalName>Tomcat8.0</finalName>

        <sourceDirectory>java</sourceDirectory>
        <testSourceDirectory>test</testSourceDirectory>
        <resources>
            <resource>
                <directory>java</directory>
            </resource>
        </resources>
        <testResources>
            <testResource>
                <directory>test</directory>
            </testResource>
        </testResources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3</version>
                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.easymock</groupId>
            <artifactId>easymock</artifactId>
            <version>3.4</version>
        </dependency>
        <dependency>
            <groupId>ant</groupId>
            <artifactId>ant</artifactId>
            <version>1.7.0</version>
        </dependency>
        <dependency>
            <groupId>wsdl4j</groupId>
            <artifactId>wsdl4j</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>javax.xml</groupId>
            <artifactId>jaxrpc</artifactId>
            <version>1.1</version>
        </dependency>

        <dependency>
            <groupId>org.eclipse.jdt.core.compiler</groupId>
            <artifactId>ecj</artifactId>
            <version>4.5.1</version>
        </dependency>
    </dependencies>
</project>

```

然后右键`AS  a Maven Project` 即可. 



## 3. `Tomcat` 架构设计

![image-20200512094150674](http://files.luyanan.com//img/20200512094153.png)



###  3.1 各个组件的定义

`官网`：https://tomcat.apache.org/tomcat-8.0-doc/architecture/overview.html

#### `server`

> In the Tomcat world, a Server represents the whole container. Tomcat provides a default implementation of the Server interface which is rarely customized by users.



####  `Service`

> A Service is an intermediate component which lives inside a Server and ties one or more Connectors to exactly one Engine. The Service element is rarely customized by users, as the default implementation is simple and sufficient: Service 
> interface.Engine

#### `Connector`

> A Connector handles communications with the client. There are multiple connectors available with Tomcat. These include the HTTP connector which is used for most HTTP traffic, especially when running Tomcat as a standalone server, and the AJP connector which implements the AJP protocol used when connecting Tomcat to a web server such as Apache HTTPD server. Creating a customized connector is a significant effort.



####  `Engine`

> An Engine represents request processing pipeline for a specific Service. As a Service may have multiple Connectors, the Engine receives and processes all requests from these connectors, handing the response back to the appropriate connector for transmission to the client. The Engine interface may be implemented to supply custom Engines, though this is uncommon. 
> Note that the Engine may be used for Tomcat server clustering via the jvmRoute parameter. Read the Clustering documentation for more information.



#### `Host`

> A Host is an association of a network name, e.g. www.yourcompany.com, to the Tomcat server. An Engine may contain multiple hosts, and the Host element also supports network aliases such as yourcompany.com and abc.yourcompany.com. Users rarely create custom Hosts because the StandardHost implementation provides significant additional functionality.



#### `Context`

> A Context represents a web application. A Host may contain multiple contexts, each with a unique path. The Context interface may be implemented to create custom Contexts, but this is rarely the case because the StandardContext provides significant additional functionality.



###  3.2  两个核心组件

`Connector`: 主要负责处理`Socket`连接,以及`Request`与`Response`的转化

`Contailer`: 包括`Engine`、`Host`、`Context` 与`Wrapper`,主要负责内部的处理以及`Servlet` 的管理

####   3.2.1 `Connector`

思想: 高内聚、低耦合

`EndPoint`:提供字节给`Processor`

`Processor`: 提供`Tomcat Request`对象给`Adapter`

`Adapter` : 提供`ServletRequest`给容器

![image-20200512101826771](http://files.luyanan.com//img/20200512101827.png)



##### 3.1.1.1 `EndPoint`

监听通讯端口, 是对传输层的抽象,用来实现`TCP/IP`协议的. 

对应的的抽象类为`AbstractEndPoint`, 有很多的实现类, 比如`NioEndPoint`，`JIoEndPoint`等。在其中有两个组件，一个是`Acceptor`，另外一个是`SocketProcessor`。

`Acceptor` 用于监听`Socket`连接请求,`SocketProcessor`用于处理接收到的`Socket` 请求. 



##### 3.2.1.2 `Processor`

`Processor` 是用于实现http协议的,也就是说`Processor` 是针对应用层协议的抽象

`Processor` 接受来自`EndPoint`的`Socket`,然后解析成`Tomcat Request`和`Tomcat Response`对象,最后通过`Adapter` 提交给容器

对应的抽象类为`AbstractProcessor`,有很多的实现类, 比如`AjpProcessor`、`Http11Processor`等。



#####  3.2.1.3  `Adapter`

`ProcessHandler`接口负责解析请求并生成`Tomcat Request`

需要把这个`Request`对象转换为`ServletRequest`. 

`Tomcat`优化`CoyoteAdapter`, 这是适配器模式的经典运用,连接器调用`CoyoteAdapter` 的`service` 方法,传入的是`Tomcat Request` 对象,`CoyoteAdapter` 负责将`Tomcat Request` 转化为`ServletRequest`, 再调用容器的`service` 方法,

##### 3.2.1.4 优化图解

`Endpoint`接收`Socket`连接，生成一个`SocketProcessor`任务提交到线程池去处理`SocketProcessor`的`run`方法会调用`Processor`组件去解析应用层协议，`Processor`通过解析生成Request对象后，会调用`Adapter`的`service`方法。

![image-20200512103740714](http://files.luyanan.com//img/20200512103741.png)



#### 3.2.2 `Contailer`

通过`Connector`后, 我们已经能够获取到对应的`Servlet`



### 4.3 `Request Process Flow`

`官网`：https://tomcat.apache.org/tomcat-8.0-doc/architecture/requestProcess/request-process.png





## 5. 思维扩展

### 5.1 自定义类加载器

`WebAppClassLoader`,打破了双亲委派模式,先自己尝试去加载这个类,找不到再委托给父类加载器,通过复写`findClass`和`loadClass` 实现. 



### 5.2 `Session`管理

`Context` 中的`Manage` 对象, 查看`Manage` 中的方法,可以发现有跟`Session`的方法. 

`Session` 的核心原理是通过`Filter`拦截`Servlet`请求, 将标准的`ServletRequest` 包装一下,换成`Spring`的`Request`对象,这样当我们调用`Request`对象的`getSession` 方法的时候,`Spring`在背后为我们创建和管理`Session`



