# 使用Netty手写Tomcat容器

前面学了一下netty的使用，我们知道Tomcat的底层也是使用的Netty, 这里使用Netty手写一个麻雀虽小五脏俱全的Tomcat容器

## 请求类封装 	Request

```JAVA
package com.formula.netty.tomcat.http;

import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.http.HttpRequest;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p>请求封装</p>
 **/
public class Request {

    private ChannelHandlerContext chc;


    private HttpRequest httpRequest;

    public Request(ChannelHandlerContext chc, HttpRequest httpRequest) {
        this.chc = chc;
        this.httpRequest = httpRequest;
    }


    public String getUrl() {
        return this.httpRequest.uri();
    }

    public String getMethod() {
        return httpRequest.method().name();
    }


}

```

##  返回封装

```java
package com.formula.netty.tomcat.http;

import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.http.*;

import java.io.UnsupportedEncodingException;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p>返回封装</p>
 **/
public class Response {

    private ChannelHandlerContext chc;

    private HttpRequest httpRequest;

    public Response(ChannelHandlerContext chc, HttpRequest httpRequest) {
        this.chc = chc;
        this.httpRequest = httpRequest;
    }

    public void write(String out) throws UnsupportedEncodingException {

        if (out == null || out.length() == 0) {
            return;
        }

        try {
            // 设置http协议以及请求头协议
            FullHttpResponse response = new DefaultFullHttpResponse(    // 设置http版本为1.1
                    HttpVersion.HTTP_1_1,
                    // 设置响应状态码
                    HttpResponseStatus.OK,
                    // 将输出值写出 编码为UTF-8
                    Unpooled.wrappedBuffer(out.getBytes("UTF-8")));
            response.headers().set("Content-Type", "text/html;");
// 当前是否支持长连接
//            if (HttpUtil.isKeepAlive(r)) {
//                // 设置连接内容为长连接
//                response.headers().set(CONNECTION, HttpHeaderValues.KEEP_ALIVE);
//            }
            chc.write(response);
            chc.flush();
        } finally {
            chc.close();
        }
    }
}

```

## servlet 封装

```java
package com.formula.netty.tomcat.http;


/**
 * @author luyanan
 * @since 2019/9/19
 * <p></p>
 **/
public abstract class Servlet {


    public void service(Request request, Response response) throws Exception {
        if ("GET".equalsIgnoreCase(request.getMethod())) {
            doGet(request, response);
        } else {
            doPost(request, response);
        }
    }

    public abstract void doPost(Request request, Response response) throws Exception;

    public abstract void doGet(Request request, Response response) throws Exception;
}

```

##  这里我们创建两个servlet

FirstServlet

```java
package com.formula.netty.tomcat.servlet;

import com.formula.netty.tomcat.http.Request;
import com.formula.netty.tomcat.http.Response;
import com.formula.netty.tomcat.http.Servlet;

import java.io.UnsupportedEncodingException;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p></p>
 **/
public class FirstServlet extends Servlet {


    @Override
    public void doPost(Request request, Response response) throws Exception {
        response.write("FirstServlet");
    }

    @Override
    public void doGet(Request request, Response response) throws Exception {

        this.doPost(request, response);
    }
}

```

SecondServlet

```java
package com.formula.netty.tomcat.servlet;

import com.formula.netty.tomcat.http.Request;
import com.formula.netty.tomcat.http.Response;
import com.formula.netty.tomcat.http.Servlet;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p></p>
 **/
public class SecondServlet extends Servlet {


    @Override
    public void doPost(Request request, Response response) throws Exception{
        response.write("SecondServlet");
    }

    @Override
    public void doGet(Request request, Response response) throws Exception{

        this.doPost(request, response);
    }
}

```

##  Tomcat 主启动类

```java
package com.formula.netty.tomcat;

import com.formula.netty.tomcat.http.Servlet;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.sctp.nio.NioSctpServerChannel;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpRequestDecoder;
import io.netty.handler.codec.http.HttpResponseDecoder;
import io.netty.handler.codec.http.HttpResponseEncoder;

import javax.sound.midi.Track;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p></p>
 **/
public class Tomcat {

    /**
     * <p>端口号</p>
     *
     * @author luyanan
     * @since 2019/9/19
     */
    private int port = 8080;


    private Map<String, Servlet> servletMap = new HashMap<String, Servlet>();


    private Properties webxml = new Properties();

    /**
     * <p>初始化方法</p>
     *
     * @author luyanan
     * @since 2019/9/19
     */
    private void init() {

        // 加载配置文件并初始化servletMap

        try {
            String path = this.getClass().getResource("/").getPath();
            FileInputStream fis = new FileInputStream(path + "web.properties");
            webxml.load(fis);

            // 初始化 servletMap

            for (Object o : webxml.keySet()) {
                String key = o.toString();
                if (key.endsWith(".url")) {

                    String servletName = key.replaceAll("\\.url$", "");
                    String url = webxml.getProperty(key);
                    // className
                    String className = webxml.getProperty(servletName + ".className");
                    Servlet servlet = (Servlet) Class.forName(className).newInstance();
                    servletMap.put(url, servlet);
                }
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }


    public void start() {

        init();
        // netty 封装了NIO, Reactor模型. BOss Worker,
        // Boss 线程
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        // Worker 线程
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {


            ServerBootstrap bootstrap = new ServerBootstrap();
            //链路式编程
            bootstrap.group(bossGroup, workerGroup)
                    //主线程处理类， 底层使用反射
                    .channel(NioServerSocketChannel.class)
// 子线程处理类
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            // 无锁化串行编程
                            // Netty 对Http协议的封装,顺序有要求
// HttpResponseEncoder 编码器
                            socketChannel.pipeline().addLast(new HttpResponseEncoder());
                            //HttpRequestDecoder 解码器
                            socketChannel.pipeline().addLast(new HttpRequestDecoder());


                            //  业务处理类
                            socketChannel.pipeline().addLast(new TomcatHandler(servletMap));
                        }
                    })
                    // 针对主线程的配置,分配线程最大数量  128
                    .option(ChannelOption.SO_BACKLOG, 128)
                    //针对子线程的配置, 保持长连接
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            // 启动服务区

            ChannelFuture channelFuture = bootstrap.bind(port).sync();
            System.out.println("Tomcat 启动成功, 端口号为: " + this.port);
            channelFuture.channel().closeFuture().sync();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            //  关闭线程池
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();

        }
    }

    public static void main(String[] args) {
        new Tomcat().start();
    }
}

```

## TomcatHandler  处理类

```java
package com.formula.netty.tomcat;

import com.formula.netty.tomcat.http.Request;
import com.formula.netty.tomcat.http.Response;
import com.formula.netty.tomcat.http.Servlet;
import com.sun.org.apache.regexp.internal.RE;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.codec.http.HttpRequest;
import io.netty.util.concurrent.EventExecutorGroup;

import java.util.Map;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p></p>
 **/
public class TomcatHandler extends ChannelInboundHandlerAdapter {

    private Map<String, Servlet> servletMap = null;

    public TomcatHandler(Map<String, Servlet> servletMap) {
        this.servletMap = servletMap;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof HttpRequest) {
            HttpRequest httpRequest = (HttpRequest) msg;
            Request request = new Request(ctx, httpRequest);

            Response response = new Response(ctx, httpRequest);

            String url = request.getUrl();
            if (servletMap.containsKey(url)) {
                servletMap.get(url).service(request, response);
            } else {
                response.write("404 - Not Found");
            }

        }
    }
}

```

## 配置文件 web.properties

```
servlet.one.url=/firstServlet.do
servlet.one.className=com.formula.netty.tomcat.servlet.FirstServlet

servlet.two.url=/secondServlet.do
servlet.two.className=com.formula.netty.tomcat.servlet.SecondServlet

```

我们启动Tomcat类下的main 方法后, 可以看到

![](http://files.luyanan.com//img/20190920093509.png)

