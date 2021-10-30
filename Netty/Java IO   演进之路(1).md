#  Java IO  演进之路

在此之前, 必须要了解几个概念

## 概念

###   阻塞(Block) 和非阻塞(Non-Block)

阻塞和非阻塞是进程在访问数据的时候,数据是否准备就绪的一种处理方式,当数据没有准备的时候

**阻塞:** 往往需要等待缓冲区中的数据准备好了之后才处理其他的事情， 否则一直卡在那里.

**非阻塞:**当我们的进程访问我们的数据缓冲区的时候,如果数据还没有准备好就直接返回, 不会等待. 如果数据已经准备好,也直接返回。

###   同步(Synchronization)和  异步(Asynchronous)

同步和异步都是基于应用程序和操作系统处于IO事件所采用的方式, 比如:

**同步:** 是应用程序要直接参与IO读写的操作.

**异步:**所有的IO读写交给操作系统去处理,应用程序只需要等待通知。

同步方式在处理IO事件的时候，必须阻塞在某个方法上面等到我们的IO事件完成(阻塞IO事件或者通过轮询IO事件的方式),对于异步来说,所有的IO读写都交给了操作系统.  这个时候,我们就可以做其他的事情,并不需要完成真正的IO操作,当操作完成IO后, 会给我们的应用程序一个通知,

同步: 阻塞到IO事件, 阻塞到read或者write,这个时候我们就完全不能做自己的事情,让读写方法加入到线程里面, 然后阻塞线程实现, 对线程的性能开销毕竟大.

##  BIO与NIO对比

下表总结了Java BIO(Block IO)和NIO(Non-Block IO) 之间的差异

| IO模型 | BIO            | NIO                     |
| ------ | -------------- | ----------------------- |
| 通信   | 面向流         | 面向缓冲区(多路复用)    |
| 处理   | 阻塞IO(多线程) | 非阻塞IO(反应堆Reactor) |
| 触发   | 无             | 选择器(轮询机制)        |

###  面向流与面向缓冲

java NIO和BIO之间第一个最大的区别是,BIO是面向流的,NIO的面向缓冲区的.Java BIO面向流意味着每次从流中读一个或者多个字节,直至读取所有字节, 他们没有被缓存在任何地方. 此外,他不能前后移动流中的数据. 

如果需要前后移动从流中读取数据,需要先将他们缓存到一个缓冲区. Java NIO的缓冲导向方法略有不同, 数据读取到一个它稍后处理的缓冲区, 需要时可以在缓冲区中前后移动, 这就增加了处理过程的灵活性, 但是, 还需要检查是否该缓冲区中包含所有需要处理的数据, 而且, 需确保当更多的数据读入到缓冲区的时候, 不会覆盖缓冲区里面尚未处理的数据.

###  阻塞与非阻塞

Java BIO的各种流是阻塞的. 这意味着,当一个线程调用 read()或者 write() 时,该线程被阻塞,直到有一些数据被读取或者数据完全写入, 该线程再次期间不能在干任何是事情了.  Java NIO的非阻塞模式,使一个线程从某通道发送请求读取数据, 但是它仅能够得到目前可用的数据, 如果目前没有数据可用时，就什么都不会获取. 而不是保持线程阻塞, 所以直至数据变的可以读取之前,该线程可以继续做其他的事情, 非阻塞写也是如此, 一个线程请求写入一些数据到某通道,但不需要等待它完全写入, 这个线程同时可以去做别的事情,线程通常将非阻塞IO的空闲时间用于在其他通道上执行IO操作, 所以一个单独的线程现在可以管理多个输入和输出通道(channel)

###  选择器的问世

Java NIO 的选择器(Selector) 允许一个单独的线程来监视多个输入通道, 你可以注册多个通道使用一个选择器,然后使用一个单独的线程来选择通道, 这些通道里已经可以处理的输入,或者选择已准备写入的通道,这种选择机制,使得一个单独的线程很容易的来管理多个通道.

###   NIO和BIO 如何影响应用程序的设计

无论是选择NIO还是BIO工具, 可能会响应你应用程序设计的以下几个方面:

1. 对NIO或者BIO类的AIP的调用
2. 数据处理逻辑
3. 用来处理数据的线程数

####  1. API调用

当然,使用NIO掉的调用时看起来与使用BIO时有所不同,但这并不意外，以为并不是仅从一个InputStream 逐字节读取,而是数据必须先读取缓冲区再处理、。

####  2. 数据处理

使用纯粹的NIO 设计相较BIO设计,数据处理也收到影响

在BIO设计中, 我们从 inputstream 或者Reader 逐字节读取数据, 假设你正在处理一基于行的文本数据流,例如有如下一段文本:

```
Name:Tom
Age:18 
Email: tom@qq.com 
Phone:13888888888
```

该文本行的流可以这样处理:

```java
      FileInputStream fis = new FileInputStream("D:/info.txt");
        BufferedReader reader = new BufferedReader(new InputStreamReader(fis));
        String nameLine = reader.readLine();
        String ageLine = reader.readLine();
        String emailLine = reader.readLine();
        String phoneLine = reader.readLine();
```

请注意处理状态 由程序执行多久决定. 换句话来说,一旦readLine() 方法返回,你就知道肯定文本行已经读完了, readline() 阻塞直到整行读完, 这就是原因. 你也知道此行包含名称; 同样,第二个readline() 调用返回的时候, 你知道这行包含年龄. 正如你可以看到.该处理程序仅在有新数据读取的时候运行, 并知道每步的数据是什么. 一旦正在运行的线程已处理过读取的某些数据, 该线程不会再回退数据(大多如此). 下图也说明了这条原则:

![](http://files.luyanan.com//img/20190909103147.png)

(java BIO:  从一个阻塞的流中读取数据),而NIO的实现会有所不同,下面是一个简单的例子

```java
ByteBuffer buffer = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buffer);
```

注意 第二行,从通道读取字节到ByteBuffer . 当这个方法调用返回时,你不知道你所需的所有数据是否在缓冲区. 你所知道的是, 该缓冲区包含一些字节, 这使得处理有点困难.

假设第一次 read(buffer) 调用后, 读取缓冲区的数据只有半行,例如 "Name:An", 你能处理吗? 显然不能, 需要等到. 直到整行数据读取缓存, 在此之前, 对数据的任何处理毫无意义.

所以, 你怎么知道是否该缓冲区中包含了足够的数据可以处理了呢? 好了, 你不知道, 发现的方法只能查看缓冲区中的数据. 其结果是,在你知道所有数据都在缓冲区里之前,你必须检查几次缓冲区的数据, 这不仅效率低下,而且可能使得线程设计方案杂乱不堪。 例如:

```java
ByteBuffer buffer = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buffer); 
while(!bufferFull(bytesRead)) {
    bytesRead = inChannel.read(buffer);
}
```

bufferFull() 方法必须跟踪有多少数据读入缓冲区, 并返回真或者假, 这取决于缓冲区是否已满。 换句话来说, 如果缓冲区准备好被处理了,那么表示缓冲区满了.

bufferFull(） 方法扫描缓冲区, 但必须保持在bufferFull() 方法被调用之前状态相同，如果没有, 下一个读取缓冲区的数据可能无法读取到正确的位置, 这是不可能的  这确实需要注意的又一问题.

如果缓冲区已满, 他可以被处理,如果它不满, 并且在你的实际案例中有意义. 你或者能处理其中的部分数据, 但是许多情况下,并非如此. 下图展示了"缓冲区数据循环就绪“

![](http://files.luyanan.com//img/20190909105442.png)

####  3. 设置处理线程数

NIO 可让你只使用一个(或几个)单线程管理多个通道(网络连接或文件), 但付出的代价是解析数据可能会比从一个阻塞流中读取数据更复杂。

如果需要管理同时打开的成千上万的连接, 这些连接每次只是发送少量的数据, 例如聊天服务器, 实现NIO的服务器可能是一个优势. 同样, 如果你需要维持许多打开的连接到其他计算机上, 如P2P网络中, 使用一个单独的线程来管理你所有出站连接, 可能是一个优势, 一个线程多个连接的设计方案如:

![](http://files.luyanan.com//img/20190909110834.png)

Java NIO: 单线程管理多个连接。

如果你有少量的连接使用非常高的带宽, 一次发送大量的数据, 必须典型的IO 服务器实现非常契合. 下图说明了一个典型的IO服务器设计:

 ![](http://files.luyanan.com//img/20190909111038.png)

Java BIO: 一个典型的IO服务器设计- 一个连接通过一个线程处理.，



###  NIO 服务端代码示例

```java
package com.notes.io.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

/**
 * @author luyanan
 * @since 2019/9/9
 * <p>NIO 服务端</p>
 **/
public class NIOServerDemo {

    private int port;


    /**
     * <p>轮询器</p>
     *
     * @author luyanan
     * @since 2019/9/9
     */
    private Selector selector;

    /**
     * <p>缓冲区</p>
     *
     * @author luyanan
     * @since 2019/9/9
     */
    private ByteBuffer buffer = ByteBuffer.allocate(1024);

    public NIOServerDemo(int port) {

        this.port = port;
        // 绑定ip/端口号
        try {
            ServerSocketChannel server = ServerSocketChannel.open();
            server.bind(new InetSocketAddress(port));
            // BIO  升级版本NIO。 为了兼容BIO,NIO模型默认是采用阻塞
            server.configureBlocking(false);
            selector = Selector.open();
            server.register(selector, SelectionKey.OP_ACCEPT);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void listen() throws IOException {
        System.out.println("listen on :" + this.port);


        //  轮询主线程
        while (true) {
            selector.select();
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            //  不断的迭代,就叫轮询
            // 同步体现在这里, 因为每次只能拿到一个Key, 每次只能处理一种状态
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                process(key);

            }
        }


    }

    private void process(SelectionKey key) throws IOException {

        // 针对每一种状态给一个反应
        if (key.isAcceptable()) {
            ServerSocketChannel server = (ServerSocketChannel) key.channel();
            //  这个方法体现非阻塞, 不管你数据有没有准备好, 你都要给我一个状态和反馈
            SocketChannel channel = server.accept();
            channel.configureBlocking(false);
            // 当数据准备就绪的时候, 将状态设置为可读
            key = channel.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {

            SocketChannel socketChannel = (SocketChannel) key.channel();
            int len = socketChannel.read(buffer);
            while (len > 0) {
                buffer.flip();
                String content = new String(buffer.array(), 0, len);
                key = socketChannel.register(selector, SelectionKey.OP_WRITE);
                key.attach(content);
                System.out.println("读取内容:" + content);
            }
        } else if (key.isWritable()) {
            SocketChannel channel = (SocketChannel) key.channel();
            String content = (String) key.attachment();
            channel.write(ByteBuffer.wrap(("输出:" + content).getBytes()));
            channel.close();
        }

    }

    public static void main(String[] args) {
        try {
            new NIOServerDemo(8080).listen();
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
}

```



## Java AIO 详解

JDK1.7(NIO2) 才是实现真正的异步AIO, 把IO读写操作完全交给了操作系统, 

### 1. AIO(Asynchronout IO) 基本原理

服务端:AsynchronousServerSocketChannel

客户端: AsynchronousSocketChannel

用户处理器: CompletionHandler 接口, 这个接口实现应用程序向操作系统发起IO请求, 当完成后处理具体逻辑,否则做自己该做的事情.

“真正”的异步IO 需要操作系统更强的支持,在IO多路复用模型中 事件循环将文件句柄的状态事件通知给用户线程, 由用户线程自行读取数据、处理数据 . 而在异步IO模型中， 当用户线程收到通知时, 数据已经被内核读取完毕了, 并放在了用户线程指定的缓冲区内, 内核在IO完成后通知用户线程直接使用即可. 异步IO模型使用了Proactor 设计模式实现了这一机制, 如下图所示：

![](http://files.luyanan.com//img/20190909112008.png)

###  2. AOP 初体验

#### 服务端代码

```java
package com.notes.io.aio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousChannelGroup;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * @author luyanan
 * @since 2019/9/9
 * <p>AIO 服务端</p>
 **/
public class AIOServer {


    private int port;

    public AIOServer(int port) {
        this.port = port;
    }

    public static void main(String[] args) {
        new AIOServer(8080).listen();
    }


    public void listen() {
        try {
            ExecutorService executorService = Executors.newCachedThreadPool();
            AsynchronousChannelGroup channelGroup = AsynchronousChannelGroup.withCachedThreadPool(executorService, 1);
            //  工作线程, 用于监听回调的, 事件响应的时候需要回调.
            AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open(channelGroup);
            // 设置端口号
            serverSocketChannel.bind(new InetSocketAddress(this.port));
            System.out.println("服务已经启动, 端口号为:" + this.port);
            serverSocketChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
                //   实现 completed 方法来回调
                // 由操作系统出发, 回调有两个状态, 成功/ 失败
                ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

                @Override
                public void completed(AsynchronousSocketChannel result, Object attachment) {
                    System.out.println("IO操作开始, 开始获取数据");

                    try {
                        buffer.clear();
                        result.read(buffer).get();
                        buffer.flip();
                        result.write(buffer);
                        buffer.flip();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (ExecutionException e) {
                        e.printStackTrace();
                    } finally {
                        try {
                            result.close();
                            serverSocketChannel.accept(null, this);
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }

                    System.out.println("操作完成");
                }

                @Override
                public void failed(Throwable exc, Object attachment) {
                    System.err.println("操作失败:" + exc);
                }
            });


            Thread.sleep(Integer.MAX_VALUE);
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}

```

#### 客户端代码

```java
package com.notes.io.aio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.ExecutionException;

/**
 * @author luyanan
 * @since 2019/9/9
 * <p>AIO 客户端</p>
 **/
public class AIOClient {

    private AsynchronousSocketChannel asynchronousSocketChannel;


    public AIOClient() throws IOException {
        asynchronousSocketChannel = AsynchronousSocketChannel.open();
    }


    public void connect(String host, int port) throws InterruptedException {
        asynchronousSocketChannel.connect(new InetSocketAddress(host, port), null, new CompletionHandler<Void, Object>() {
            @Override
            public void completed(Void result, Object attachment) {
                try {
                    asynchronousSocketChannel.write(ByteBuffer.wrap("这是一条测试数据".getBytes())).get();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        });


        ByteBuffer bb = ByteBuffer.allocate(1024);
        asynchronousSocketChannel.read(bb, null, new CompletionHandler<Integer, Object>() {
            @Override
            public void completed(Integer result, Object attachment) {
                System.out.println("IO 操作完成");
                System.out.println("获取反馈结果:" + new String(bb.array()));
            }

            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        });
        Thread.sleep(Integer.MAX_VALUE);
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        new AIOClient().connect("localhost", 8080);
    }
}

```

结果

服务端

> 服务已经启动, 端口号为:8080
> IO操作开始, 开始获取数据
> 操作完成

客户端

> IO 操作完成
> 获取反馈结果:这是一条测试数据 

##   各IO 模型对比和总结



| 属性              | 同步阻塞IO(BIO) | 伪异步IO  | 非阻塞IO(NIO)  | 异步IO(AIO) |
| ----------------- | --------------- | --------- | -------------- | ----------- |
| 客户端数:IO线程数 | 1:1             | M:N(M>=1) | M:1            | M:0         |
| 阻塞类型          | 阻塞            | 阻塞      | 非阻塞         | 非阻塞      |
| 同步              | 同步            | 同步      | 同步(多路复用) | 异步        |
| API使用难度       | 检点            | 简单      | 复杂           | 一般        |
| 调试难度          | 简单            | 简单      | 复杂           | 复杂        |
| 可靠性            | 非常差          | 差        | 高             | 高          |
| 吞吐量            | 低              | 中        | 高             | 高          |

