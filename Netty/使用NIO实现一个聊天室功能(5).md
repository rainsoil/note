# 使用NIO实现一个聊天室公共功能

## 主要实现功能:

 * 功能1:  客户端通过Java NIO 连接到服务端, 支持多客户端的连接
 * 功能2:  客户端初次连接的时候, 服务端提示输入昵称, 如果昵称已经有人使用, 则提示重新输入, 如果昵称唯一, 则登录成功,
 * 之后发送消息都需要按照规定格式带着昵称发送消息
 * 功能3: 客户端登录后, 发送已经设置好的欢迎信息和在线人数给客户端, 并且通知其他客户端该客户端已经上线
 * 功能4: 服务端收到已登录客户端输入内容, 转发至其他客户端

##  服务端代码

```java
package com.formula.netty.chat;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.nio.charset.Charset;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;

/**
 * @author luyanan
 * @since 2019/9/21
 * <p>网络多客户端聊天室
 * 功能1:  客户端通过Java NIO 连接到服务端, 支持多客户端的连接
 * 功能2:  客户端初次连接的时候, 服务端提示输入昵称, 如果昵称已经有人使用, 则提示重新输入, 如果昵称唯一, 则登录成功,
 * 之后发送消息都需要按照规定格式带着昵称发送消息
 * 功能3: 客户端登录后, 发送已经设置好的欢迎信息和在线人数给客户端, 并且通知其他客户端该客户端已经上线
 * 功能4: 服务端收到已登录客户端输入内容, 转发至其他客户端
 * </p>
 **/
public class NIOChatServer {


    private int port = 8080;

    //相当于自定义协议格式，与客户端协商好
    private static String USER_CONTENT_SPILIT = "#@#";

    private Charset charset = Charset.forName("utf-8");
    private Selector selector;


    private static String USER_EXIST = "系统提示：该昵称已经存在，请换一个昵称";


    /**
     * <p>用来记录在线人数, 以及昵称</p>
     *
     * @author luyanan
     * @since 2019/9/21
     */
    private static HashSet<String> users = new HashSet<>();

    public NIOChatServer(int port) throws IOException {
        this.port = port;

        ServerSocketChannel server = ServerSocketChannel.open();

        // 绑定端口号
        server.bind(new InetSocketAddress(this.port));
        // 设置为非阻塞
        server.configureBlocking(false);
        // 初始化轮询器
        this.selector = Selector.open();

        // 注册事件
        server.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("服务已经启动, 端口号为: " + this.port);
    }


    /**
     * <p>开始监听</p>
     *
     * @return {@link}
     * @author luyanan
     * @since 2019/9/21
     */
    public void listen() throws IOException {

        while (true) {
            int wait = selector.select();
            if (wait == 0) {
                continue;
            }

            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {

                SelectionKey selectionKey = iterator.next();
                iterator.remove();
                process(selectionKey);
            }

        }
    }

    private void process(SelectionKey selectionKey) throws IOException {

        if (selectionKey.isAcceptable()) {
            ServerSocketChannel server = (ServerSocketChannel) selectionKey.channel();
            SocketChannel client = server.accept();
            // 设置为非阻塞模式
            client.configureBlocking(false);
            //注册选择器, 并设置为读取模式, 收到一个连接请求, 然后起一个 socketChannel , 并注册到selector 上, 之后这个连接的数据, 就由这个 socketCHannel 处理
            client.register(selector, SelectionKey.OP_READ);
            // 将此对应的 channel 设置为准备接受其他客户端请求
            selectionKey.interestOps(SelectionKey.OP_ACCEPT);
            System.out.println("有客户端连接: IP地址为: " + client.getRemoteAddress());
            client.write(charset.encode("请输入您的昵称"));
        }
        // 处理来自客户端的数据读取请求
        if (selectionKey.isReadable()) {
            // 返回该 selectionKey 对应的channel, 其中有数据需要读取
            SocketChannel client = (SocketChannel) selectionKey.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            StringBuffer content = new StringBuffer();
            try {
                while (client.read(buffer) > 0) {
                    buffer.flip();
                    content.append(charset.decode(buffer));
                }
                // 将此对应的channel  设置为准备下一次接受数据
                selectionKey.interestOps(SelectionKey.OP_READ);
            } catch (IOException e) {
                selectionKey.cancel();
                if (selectionKey.channel() != null) {
                    selectionKey.channel().close();
                }
            }
            if (content.length() > 0) {
                String[] arrayContent = content.toString().split(USER_CONTENT_SPILIT);
                //  注册用户
                if (arrayContent != null && arrayContent.length == 1) {
                    String nickName = arrayContent[0];
                    if (users.contains(nickName)) {
                        client.write(charset.encode(USER_EXIST));
                    } else {
                        users.add(nickName);
                        int onlineCount = onlineCount();
                        String message = "欢迎: " + nickName + "进入聊天室！当前在线人数: " + onlineCount;
                        message = nickName + " 说" + message;
                        if (users.contains(nickName)) {
                            // 不会发送给此内容的客户端
                            broadCase(client, message);
                        }
                    }

                }
            }

        }
    }


    private void broadCase(SocketChannel client, String message) throws IOException {
        //  广播数据到所有的 SocketChannel
        for (SelectionKey key : selector.keys()) {
            Channel channel = key.channel();
            //  如果client 不为空, 不会发给此内容的客户端
            if (channel instanceof SocketChannel && channel != client) {
                SocketChannel socketChannel = (SocketChannel) channel;
                socketChannel.write(charset.encode(message));
            }
        }
    }


    public int onlineCount() {
        int res = 0;
        for (SelectionKey key : selector.keys()) {
            Channel channel = key.channel();
            // 如果client 不为空, 不会发给此内容的客户端
            if (channel instanceof SocketChannel) {
                res++;
            }
        }
        return res;
    }


    public static void main(String[] args) throws IOException {
        new NIOChatServer(8080).listen();
    }
}

```

## 客户端代码

```java
package com.formula.netty.chat;

import java.io.IOException;
import java.io.Reader;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.Charset;
import java.util.Iterator;
import java.util.Scanner;
import java.util.Set;

/**
 * @author luyanan
 * @since 2019/9/21
 * <p></p>
 **/
public class NIOChatClient {


    private final InetSocketAddress socketAddress = new InetSocketAddress("localhost", 8080);

    private Selector selector;

    private SocketChannel client;

    private String nickName;
    private Charset charset = Charset.forName("UTF-8");

    private static String USER_EXIST = "系统提示：该昵称已经存在，请换一个昵称";
    private static String USER_CONTENT_SPILIT = "#@#";


    public NIOChatClient() throws IOException {
        selector = Selector.open();
        // 连接远程主机的IP和端口
        client = SocketChannel.open(socketAddress);
        client.configureBlocking(false);
        client.register(selector, SelectionKey.OP_READ);
    }


    public void session() {
        // 开辟一个新的线程从服务器端读数据
        new Reader().start();
        //  开辟一个新的线程往服务端写数据
        new Write().start();
    }

    private class Reader extends Thread {

        @Override
        public void run() {
            try {
                while (true) {
                    int select = selector.select();
                    if (select == 0) {
                        continue;
                    }
                    Set<SelectionKey> selectionKeys = selector.selectedKeys();
                    Iterator<SelectionKey> iterator = selectionKeys.iterator();
                    while (iterator.hasNext()) {
                        SelectionKey selectionKey = iterator.next();
                        iterator.remove();
                        process(selectionKey);
                    }
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        private void process(SelectionKey selectionKey) {
            if (selectionKey.isReadable()) {
                // 使用NIO Server 读取Channel的数据, 这个和全局变量client是一样的, 只是因为注册了一个 socketChannel
                SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
                ByteBuffer buffer = ByteBuffer.allocate(1024);

                String content = "";
                try {
                    while (socketChannel.read(buffer) > 0) {
                        buffer.flip();
                        content = content + charset.decode(buffer);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
                //  若系统发送通知名字已经存在， 则需要换个昵称
                if (USER_EXIST.endsWith(content)) {

                    nickName = "";
                }
                System.out.println(content);
                selectionKey.interestOps(SelectionKey.OP_READ);
            }
        }
    }


    private class Write extends Thread {

        @Override
        public void run() {
            //  在主线程中, 从键盘读取数据输入到服务器端
            Scanner scanner = new Scanner(System.in);
            try {
                while (scanner.hasNextLine()) {
                    String nextLine = scanner.nextLine();
                    if ("".equals(nextLine)) {
                        continue;
                    }
                    if ("".equals(nickName)) {
                        nickName = nextLine;
                        nextLine = nickName + USER_CONTENT_SPILIT + nextLine;
                    }
                    client.write(charset.encode(nextLine));


                }
                scanner.close();
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
    }

    public static void main(String[] args) throws IOException {
        new NIOChatClient().session();
    }
}

```

