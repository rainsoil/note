# Netty和NIO之前世今生

## 1. Java NIO 三件套

在NIO中有几个核心对象需要掌握:缓冲区(Buffer)、选择器(Selector)、通道(Channel)

### 1. 缓冲区Buffer

#### 1. buffer 操作基本API

缓冲区实际上是一个容器对象, 更直接的说, 其实就是一个数组, 在NIO库中, 所有的数据都是用缓冲区处理的. 在读取数据的时候,它是直接读取到缓冲区的; 在写入数据的时候, 它也是写入到缓冲区的; 任何时候访问NIO中的数据,都是将他放到缓冲区. 而在面向流I/O 系统中,所有数据都是直接写入或者直接将数据读取到Stream 对象中.

在NIO中, 所有的缓冲区类型都继承于抽象类Buffer, 最常用的就是ByteBuffer.  对于java中的基本类型,基本都有一个具体的Buffer类型与之相对应.他们之间的继承关系如下图所示:

![](http://files.luyanan.com//img/20190917202103.png)

下面是一个简单的使用IntBuffer的例子

```java
package com.formula.netty.base.buffer;

import java.nio.Buffer;
import java.nio.IntBuffer;

/**
 * @author luyanan
 * @since 2019/9/17
 * <p></p>
 **/
public class IntBufferDemo {

    public static void main(String[] args) {
        // 分配新的int 缓冲区,参数为缓冲区容量
        // 新缓冲区的当前位置为0,其界限(限制位置)将为其容量,它将具有一个底层实现数组,其数组偏移量将为零.
        IntBuffer buffer = IntBuffer.allocate(8);
        for (int i = 0; i < buffer.capacity(); i++) {
            int j = 2 * (i + 1);
            // 将给定整数写入到此缓冲区当前的位置,当前位置递增
            buffer.put(j);
        }
        //  重设此缓冲区,将限制设置为当前位置,然后将当前位置设置为0
        buffer.flip();
        // 查看在当前位置和限制位置是否有元素
        while (buffer.hasRemaining()) {
            //  读取此缓冲区当前位置的元素,然后当前位置递减
            int j = buffer.get();
            System.out.println(j + " ");

        }

    }

}

```



结果

```
2 
4 
6 
8 
10 
12 
14 
16 
```

#### 2. Buffer的基本原理

在谈到缓冲区的时候,我们说缓冲区对象本质上是一个数组,但它其实是一个特殊的数组，缓冲区对象内置了一些机制,能够追踪和记录缓冲区的状态变化情况,如果我们使用get() 方法从缓冲区获取或者使用put() 方法把数据写入缓冲区,都会引起缓冲区状态的变化.

在缓冲区中,最重要的属性有下面三个,他们一起合作完成对缓冲区内部状态的变化追踪

##### position

指定下一个将要被写入或者读取的元素索引,它的值由get()/put() 方法自动更新,在新创建一个Buffer对象的时候,position 被初始化为0;

#####  limit

指定还有多少数据需要取出(从缓冲区写入通道时)，或者还有多少空间可以放入数据(在从通道读入到缓冲区时).

##### capacity

指定了可以存储在缓冲区中的最大数据容量,实际上,他指定了底层数组的大小，或者是至少指定了准许我们使用的底层数组的容量.

以上三个属性值之间有一些相对大小的关系:  0 <= position <= limit <= capacity;  如果我们创建一个新的容量大小为10 的 ByteBuffer 对象,在初始化的时候,position 设置为0， limit 和 capacity 被设置为10, 在以后使用 ByteBuffer 对象过程中， capacity的值不会发生变化,  position和limit的值将会随着使用而变化.

 下面我们用代码来演示一遍, 准备一个text文档, 存放在E盘, 输入以下内容:

> TOM.

下面我们用一段代码来验证 position、limit、capacity这几个值的变化过程, 代码如下:

```java
package com.formula.netty.base.buffer;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.nio.Buffer;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p></p>
 **/
public class BufferDemo {

    public static void main(String[] args) throws IOException {
        // 这里用的是文件IO处理
        FileInputStream fis = new FileInputStream("D://test.txt");
        // 创建文件的操作管道
        FileChannel channel = fis.getChannel();
        //分配一个10个大小的缓冲区,说白了就是分配一个10个大小的byte数组
        ByteBuffer buffer = ByteBuffer.allocate(10);
        output("初始化", buffer);
        // 先读一下
        channel.read(buffer);
        output("调用read", buffer);
        // 准备操作之前, 先锁定操作范围.
        buffer.flip();
        output("调用flip()", buffer);
        // 判断有没有可读数据
        while (buffer.remaining() > 0) {
            byte b = buffer.get();
        }
        output("调用get()", buffer);

        // 可以理解为解锁
        buffer.clear();
        output("调用clear()", buffer);
        //最后把通道关闭

    }

    /**
     * <p>将这个缓冲里面的实时状态给打印出来</p>
     *
     * @param step
     * @param buffer
     * @return {@link}
     * @author luyanan
     * @since 2019/9/18
     */
    public static void output(String step, Buffer buffer) {
        System.out.println(step + " : ");
        //  容量, 数组大小
        System.out.print("capacity: " + buffer.capacity() + ",");
        // 当前操作数据所在的位置, 也可以叫做游标
        System.out.print("position:" + buffer.position() + ", ");
        // 锁定锁, flip, 数据操作范围索引只能在position - limit 之间
        System.out.println("limit: " + buffer.limit() + ",");
        System.out.println();
    }

}

```



结果

```
初始化 : 
capacity: 10,position:0, limit: 10,

调用read : 
capacity: 10,position:4, limit: 10,

调用flip() : 
capacity: 10,position:0, limit: 4,

调用get() : 
capacity: 10,position:4, limit: 4,

调用clear() : 
capacity: 10,position:0, limit: 10,
```

运行结果我们已经看到, 下面对以上的结果进行图解, 四个属性值分别如图所示:

![](http://files.luyanan.com//img/20190918100913.png)



我们可以从通道中读取一些数据到缓冲区中， 注意从通道读取数据, 相当于往缓冲区写入数据, 如果读取4个字节的数据, 则此时 position 的值为4, 即下一个将要被写入到字节索引为4, 而limit 仍然为10, 如下图所示：

![](http://files.luyanan.com//img/20190918101114.png)

下一步把读取的数据写入到输出通道中, 相当于从缓冲区读取数据, 在此之前, 必须调用 flip() 方法, 该方法会完成两件事情：

1. 把limit 设置为当前的 position 值
2. 把positio 设置为0

由于position 被设置为0， 所以可以保证在下一步输出的时候读取到的缓冲区的第一个字节， 而limit 被设置为当前的 position, 可以保证读取的数据正好是之前写入到缓冲区的数据, 如下图所示：

![](http://files.luyanan.com//img/20190918101445.png)

现在调用get()方法从 缓冲区中读取数据写入到输出管道, 这会导致position 的增加而 limit 不变,  但是 position 不会超过 limit 的值, 所以在读取我们之前写入到缓冲区中的4个字节之后, position和 limti 都设置为4, 如下图所示：

![](http://files.luyanan.com//img/20190918101829.png)

在从缓冲区中读取数据完毕后, limit的值仍然保持在我们调用 flip() 方法时的值, 调用clear() 方法能够把所有的状态变化设置为初始化时的值,  如下图所示：

![](http://files.luyanan.com//img/20190918102106.png)

#### 3. 缓冲区的分配

在前面的例子中， 我们已经看到了, 在创建一个缓冲区对象时, 会调用静态方法 allocate() 来指定缓冲区的容量,  其实调用 allocate()  相当于创建了一个指定大小的数组, 并把它包装为缓冲区对象。 或者我们也可以直接把一个现用的数组,包装为缓冲区对象, 代码如下：

```java
package com.formula.netty.base.buffer;

import java.nio.ByteBuffer;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p>手动分配缓冲区</p>
 **/
public class BufferWrap {

    public static void main(String[] args) {
        // 分配指定大小的缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(10);
        //包装一个现有的数组
        byte array[] = new byte[10];
        ByteBuffer wrap = ByteBuffer.wrap(array);
    }

}

```

#### 4. 缓冲区分片

在NIO中， 处了可以分配或者包装一个缓冲区对象外， 还可以根据现有的缓冲区对象来创建一个子缓冲区, 即在现有的缓冲区上切出一片来作为一个新的缓冲区, 但现有的缓冲区与创建的子缓冲区再底层数组层面上是数据共享的, 也就是说, 子缓冲区相当于是现有缓冲区的一个视图窗口, 调用 slice() 方法可以创建一个子缓冲区, 让我们通过例子来看一下 

```java
package com.formula.netty.base.buffer;

import java.nio.ByteBuffer;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p>缓冲区分片</p>
 **/
public class BufferSlice {


    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);
        // 缓冲区的数据 0-9
        for (int i = 0; i < buffer.capacity(); i++) {
            buffer.put((byte) i);
        }
        //创建子缓冲区
        buffer.position(3);
        buffer.limit(7);
        ByteBuffer slice = buffer.slice();

        //改变子缓冲区的内容
        for (int i = 0; i < slice.capacity(); i++) {
            byte b = slice.get(i);
            b *= 10;
            slice.put(i, b);
        }
        buffer.position(0);
        buffer.limit(buffer.capacity());

        while (buffer.remaining() > 0) {
            System.out.println(buffer.get());
        }

    }

}

```

在该示例中, 分配了一个容量大小为10的缓冲区, 并在其中放入了数据 0-9, 而在该缓冲区基础上又创建了一个子缓冲区, 并改变子缓冲区中的内容, 从最后输出的结果来看, 只有子缓冲区"可见的"那部分数据发生了变化,  并且说明子缓冲区和原缓冲区是数据共享的, 输出结果如下所示：

```
0
1
2
30
40
50
60
7
8
9
```

####  5. 只读缓冲区

只读缓冲区非常简单, 可以读取他们, 但是不能向他们写入数据, 可以通过调用缓冲区的 asReadOnlyBuffer() 方法， 将任何常规缓冲区转换为只读缓冲区, 这个方法返回一个与原缓冲区完全相同的缓冲区, 并与原缓冲区共享数据, 只不过它是只读的. 如果原缓冲区的内容发生了变化, 只读缓冲区的内容也随之发生了变化。 

```java
package com.formula.netty.base.buffer;

import java.nio.ByteBuffer;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p>只读缓冲区</p>
 **/
public class ReadOnlyBuffer {


    public static void main(String[] args) {
        ByteBuffer buffer = ByteBuffer.allocate(10);
        //缓冲区的数据 0-9
        for (int i = 0; i < buffer.capacity(); i++) {
            buffer.put((byte) i);
        }

        //创建只读缓冲区
        ByteBuffer onlyBuffer = buffer.asReadOnlyBuffer();

        //改变原有缓冲区的内容
        for (int i = 0; i < buffer.capacity(); i++) {
            byte b = buffer.get(i);
            b *= 10;
            buffer.put(i, b);
        }
        onlyBuffer.position(0);
        onlyBuffer.limit(buffer.capacity());

        // 只读缓冲区的内容也随着变化
        for (int i = 0; i < onlyBuffer.capacity(); i++) {
            System.out.println(onlyBuffer.get());
        }

    }


}

```

结果

```
0
10
20
30
40
50
60
70
80
90
```



如果尝试修改只读缓冲区的内容, 则会报 ReadOnlyBufferException  异常, 只读缓冲区对于保护数据很有用. 在将缓冲区传递给某个对象的方法时， 无法知道这个方法是否会修改缓冲区的数据. 创建一个只读的缓冲区可以保证该缓冲区不会被修改, 只可以将常规的缓冲区转换为只读缓冲区, 不可以将只读缓冲区转化为可写的 缓冲区.

#### 6. 直接缓冲区

直接缓冲区是为了加快IO速度， 使用一种特殊的方式为其分配内存的缓冲区, JDK文档中的描述为:给定一个直接字节缓冲区，java虚拟机将尽最大的努力字节对它执行本机I/O操作，也就说,	 他会在每一次调用底层操作系统的本机I/O操作之前(或之后) ,尝试避免将缓冲区的内容拷贝到一个中间缓冲区或者从一个中间缓冲区中拷贝数据. 要分配直接缓冲区, 需要调用 allocateDirect()方法, 而不是 allocate() 方法, 使用方式与普通缓冲区并无区别,如下面的拷贝文件示例：

```java
package com.formula.netty.base.buffer;

import java.io.*;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p>直接缓冲区</p>
 **/
public class DirectBuffer {

    public static void main(String[] args) throws IOException {
        // 首先我们从磁盘上读取我们刚才写出的文件内容
        String inFile = "D://test.txt";
        FileInputStream fis = new FileInputStream(inFile);

        FileChannel fcin = fis.getChannel();
        //把刚刚读取的内容写入到一个新的文件中
        String outFile = new String("D://textcopy.txt");
        FileOutputStream fos = new FileOutputStream(outFile);

        FileChannel fcon = fos.getChannel();


        ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
        while (true) {
            buffer.clear();
            int read = fcin.read(buffer);
            if (read == -1) {
                break;
            }
            buffer.flip();
            fcon.write(buffer);
        }
    }

}


```

####  7.  内存映射

内存映射是一种读和写文件数据的方法, 他可以比常规的基于流或者基于通道的I/O快的多, 内存映射文件I/O 是通过使文件中的数据神奇的出现为内存数组的内容来完成的.这其实初听起来似乎不过就是将整个文件读取到内存中， 但是事实上并不是这样的, 一般来说, 只有文件中世纪读取或者写入的部分才会映射到内存中. 如下面的示例代码：

```java
package com.formula.netty.base.buffer;

import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p>IO映射缓冲区</p>
 **/
public class MappedBuffer {

    static private final int start = 0;
    static private final int size = 1024;

    public static void main(String[] args) throws IOException {

        RandomAccessFile raf = new RandomAccessFile("D://text.txt", "rw");

        FileChannel channel = raf.getChannel();
        // 把缓冲区跟文件系统进行一个映射关联
        //  只要操作 缓冲区里面的内容, 文件内容也会跟着改变
        MappedByteBuffer mbb = channel.map(FileChannel.MapMode.READ_WRITE, start, size);
        mbb.put(0, (byte) 97);
        mbb.put(1023, (byte) 122);
        raf.close();
        
    }

}

```



### 2.  选择器 Selector

传统的Server/Client 模式会基于TPR(Thread per Request), 服务器会为每个客户端请求建立一个线程, 由该线程单独负责处理一个客户请求, 这种模式带来的一个问题就是线程数量的剧增, 大量的线程会增大服务器的开销, 大多数的实现为了避免这个问题都采用了线程池模型, 并设置线程池线程的最大数量, 这又带来了新的问题, 如果线程池中有200个线程， 而有200个用户都在进行大文件下载, 会导致第201个用户的请求无法及时处理, 即便第201个用户只想请求一个几KB大小的页面, 传统的Server/Client 模式如下图所示:

![](http://files.luyanan.com//img/20190918150236.png)

NIO中非阻塞I/O 采用了基于Reactor 模式的工作方式 , i/o 调用不会被阻塞, 相反是注册感兴趣的特定I/O 事情, 如可读数据到达, 新的套接字连接等等, 在发生特定事件时, 系统再通知我们.NIO 中实现非阻塞I/O 的核心对象就是Selector, Selector 就是注册各种I/O 事件地方, 而且当那些事件发生时, 就是这个对象告诉我们所发生的事件, 如下图所示:

![](http://files.luyanan.com//img/20190918150551.png)

从图中我们可以看出,当有读或者写等任何注册的事件发生的时候,可以从Selector 中获得相应的 SelectionKey, 同时从 SelectionKey 中可以找到发生的事件和该时间发生的具体的 SelectableChannel, 以获得客户端发送的数据.

使用NIO中非阻塞I/O 编写服务器处理程序,大体上可以分为下面三个步骤:

1. 向Selector 对象中注册感兴趣的事件
2. 从 Selector 中获取感兴趣的事件
3. 根据不同的事件进行不同的处理.

接下来我们用一个简单的示例来说明整个过程, 首先是向Selector 中注册感兴趣事件:

```java
package com.formula.netty.base.selector;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;

/**
 * @author luyanan
 * @since 2019/9/18
 * <p></p>
 **/
public class SelectorDemo {

    /**
     * <p>端口号</p>
     *
     * @author luyanan
     * @since 2019/9/18
     */
    private int port;

    /**
     * <p>注册事件</p>
     *
     * @author luyanan
     * @since 2019/9/18
     */
    private Selector getSelector() throws IOException {

        //创建Selector  对象
        Selector selector = Selector.open();

        // 创建可选择通道, 并且配置为非阻塞模式
        ServerSocketChannel server = ServerSocketChannel.open();
        server.configureBlocking(false);
        //  绑定通道到指定的端口
        ServerSocket socket = server.socket();
        InetSocketAddress inetSocketAddress = new InetSocketAddress(this.port);
        socket.bind(inetSocketAddress);

        // 向Selector 中注册感兴趣的事件

        server.register(selector, SelectionKey.OP_ACCEPT);
        return selector;
    }

}

```

创建了ServerSocketChannel 对象,并且调用 configureBlocking() 方法配置为非阻塞模式, 接下来的三行代码把该通道绑定到指定的端口, 最后向Selector 中注册对象, 此处指定的参数为 `OP_ACCEPT`, 即指定我们想要监听 accept 事件, 也就是新的连接发生时所产生的事件, 对于ServerSocketChannel  通道来说, 我们唯一可以指定的参数就是 `OP_ACCEPT`

从 Selector 中获取感兴趣的事件, 即开始监听, 进入内部循环

```java
  /**
     * <p>开始监听</p>
     *
     * @author luyanan
     * @since 2019/9/18
     */
    public void listen(Selector selector) throws IOException {

        System.out.println("listen  on :" + this.port);
        while (true) {
            //  该调用会阻塞，直到至少有一个事件发生
            selector.select();
            Set<SelectionKey> selectionKeys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = selectionKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey selectionKey = iterator.next();
                iterator.remove();;
                process(selectionKey);
            }
        }
    }

```

在非阻塞I/O 中, 内部循环模式基本都是遵循这种方式, 首先调用 select() 方法, 该方法会阻塞, 直到至少有一个事件发生， 然后再使用 selectedKeys 方法获取发生事件的selectedKey, 再使用迭代器进行循环.,

最后一步就是根据不同的事件, 编写响应的处理代码.

```java
 /**
     * <p>根据不同的事件做处理</p>
     *
     * @author luyanan
     * @since 2019/9/18
     */
    private void process(SelectionKey selectionKey) throws IOException {

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        // 接受请求
        if (selectionKey.isAcceptable()) {

            ServerSocketChannel server = (ServerSocketChannel) selectionKey.channel();
            SocketChannel accept = server.accept();
            accept.configureBlocking(false);
            accept.register(selector, SelectionKey.OP_READ);
        }
        //  读信息
        else if (selectionKey.isReadable()) {
            SocketChannel channel = (SocketChannel) selectionKey.channel();
            int len = channel.read(buffer);
            if (len > 0) {
                buffer.flip();
                String content = new String(buffer.array(), 0, len);
                SelectionKey key = channel.register(selector, SelectionKey.OP_WRITE);
                key.attach(content);
            } else {
                channel.close();
                ;
            }
            buffer.clear();
        }
        // 写事件
        else if (selectionKey.isWritable()) {

            SocketChannel channel = (SocketChannel) selectionKey.channel();
            String content = (String) selectionKey.attachment();
            ByteBuffer byteBuffer = ByteBuffer.wrap(("输出内容:" + content).getBytes());
            if (byteBuffer != null) {
                channel.write(byteBuffer);
            } else {
                channel.close();
            }
        }

    }
```

此处分别是判断接受请求,读数据还是鞋事件, 分别做不同的处理, 在Java1.4之前的I/O系统中, 提供的都是面向流的I/O系统, 系统一次一个字节的处理数据, 一个输入流产生一个字节的数据, 一个输出流消费一个字节的数据, 面向流的I/O速度非常慢, 而在java1.4 中推出了NIO,这是一个面向块的I/O系统,系统以块的方式处理, 每一个操作在一步中产生或者消费一个数据库, 按块处理要比按字节处理数据快的多.

### 3. 通道Channel

通道是一个对象, 通过它可以读取或者写入数据, 当然了所有数据都通过Buffer 对象来处理, 我们永远不会将字节直接写入到通道中, 相反是将数据写入包含一个或者多个字节的缓冲区. 同样不会直接从通道中读取字节, 而是将数据从通道读取缓冲区, 再从缓冲区获取这个字节.

在NIO中, 提供了多种通道对象, 而所有的通道对象都实现了Channel 接口, 他们之间的继承关系如下图所示:

![](http://files.luyanan.com//img/20190919100356.png)

####  1. 使用NIO 读取数据

在前面我们说过, 任何时候读取数据, 都不是直接从通道读取, 而是从通道读取到缓冲区, 所以使用NIO读取数据可以分为下面三个步骤：

1. 从 FileInputStream 获取Channel
2. 创建Buffer
3. 将数据从Channel 读取到Buffer

下面是一个简单的使用NIO 从文件中读取数据的例子:

```java
package com.formula.netty.base.channel;

import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p>从文件中读取数据的例子</p>
 **/
public class FileInputDemo {


    public static void main(String[] args) throws IOException {
        FileInputStream fis = new FileInputStream("D://test.txt");

        //  获取通道
        FileChannel channel = fis.getChannel();
        //创建缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        // 读取数据到缓冲区
        channel.read(buffer);

        buffer.flip();
        while (buffer.remaining() > 0) {
            byte b = buffer.get();
            System.out.println((char) b);
        }
        fis.close();
    }
}

```

#### 2. 使用NIO写入数据

使用NIO 写入数据与读取数据的过程类似, 同样数据不是直接写入通道， 而是写入到缓冲区, 可以分为下面三个步骤:

1.  从FileOutputStream 获取Channel
2. 创建buffer
3. 将数据从 Channel 吸入到Buffer

```java
package com.formula.netty.base.channel;

import java.io.FileOutputStream;
import java.nio.Buffer;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

/**
 * @author luyanan
 * @since 2019/9/19
 * <p>使用NIO写出数据</p>
 **/
public class FileOutputDemo {
    static private final byte message[] = {83, 111, 109, 101, 32,
            98, 121, 116, 101, 115, 46};

    public static void main(String[] args) throws Exception {
        FileOutputStream fos = new FileOutputStream("D:/text1.txt");
        FileChannel channel = fos.getChannel();

        // 创建缓冲区
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

        for (int i = 0; i < message.length; i++) {

            byteBuffer.put(i, message[i]);
        }
        byteBuffer.flip();
        channel.write(byteBuffer);
        fos.close();
    }

}

```

####  3.  IO 多路复用

我们试想这样一个现实场景:

一个 餐厅同时有100位客人到店, 当然到店后做的第一件事情就是点菜了, 但是问题来了,餐厅老板为了节约人力成本目前只有一位大堂服务员拿着唯一的一本菜单等待客人进行服务.

那么最笨的(但是最简单的)方法是(方法A) ,无论有多少客人等着点餐, 服务员都把仅有的一份菜单递给其中的一位客人,然后站在客人身旁等待这个客人完成点菜过程. 在记录客人点菜内容后, 把点菜记录交给后堂厨师, 然后是第二位客人。。。然后是第三位客人, 很明显, 只有脑袋被门夹过的老板才会设置这样的服务流程, 以为随后的80位客人, 再等待超时后就会离店(还会给差评)

于是还有一种办法(方法B) ,老板马上新雇佣99位服务员, 同时印制99本新的菜单, 每一名服务员 手持一本菜单负责一位客人(关键不只在服务员, 还在于菜单. 因为没有菜单客人也无法点菜). 在客人点菜完后, 记录点菜内容给后堂厨师(当然为了高效, 后堂厨师最好也有100位). 这样每一位客人享受的就是VIP服务, 当然客人不会走, 但是人力成本就是一个大头哦

![](http://files.luyanan.com//img/20190919111514.png)

另一种方法(方法C) 就是改进点菜的方式, 当客人到店后,自己申请一本菜单, 想好自己想点的菜后, 才呼叫服务员,  服务员记录好客人的菜单内容, 将菜单交给厨师的过程也要改进. 并不是每一份菜单记录好了之后都要交给厨师, 服务员可以记录好多分菜单后， 同时交给厨师就行了, 那么这种方式, 对于老板来说是人力成本最低的, 对于客人来说, 虽然不再享受VIP过程 并且要求进行一定的等待, 但是这些都是可以接受的. 对于服务员来说, 基本上他的时间都没有浪费， 基本上被老板压榨了最后一滴油水.

![](http://files.luyanan.com//img/20190919112017.png)

如果你是老板, 会采用哪种方式呢?

到店情况： 并发量, 到店情况不理想时， 一个服务员 一个菜单当然足够了, 所以不同的老板在不同的场合下将会灵活的选择服务员和菜单的配置。

客人： 客户端请求

点餐内容: 客户端发送的实际数据

老板: 操作系统

人力成本: 系统资源

菜单： 文件操作描述符(FD), 操作系统对于一个进程能够同时持有的文件的状态描述符的个数是由限制的, 在Linux 系统中 `$ulimit -n` 查看这个限制, 当然也是可以进行内核参数调整的.

服务员: 操作系统内核对于IO操作的线程(内核线程)

厨师: 应用程序线程(当然厨房就是应用程序进程了)

方法A: 同步IO

方法B: 同步IO

方法C: 多路复用

目前流行的多路复用IO实现主要包括四种: select、poll、epoll、kqueue . 下表是他们的一些重要特性的比较.



| IO模型 | 相对性能 | 关键思路         | 操作系统     | Java支持                                                     |
| ------ | -------- | ---------------- | ------------ | ------------------------------------------------------------ |
| seelct | 较高     | Reactor          | window/linux | 支持Reactor模式(反应器设计模式), Linux 操作系统的 kernels2.4 内核版本之前, 默认使用 select, 而目前的window 系统下对同步IO的支持, 都是select模型. |
| poll   | 较高     | Reactor          | Linux        | Linux 下的 Java NIO 框架, Linux kernels 2.6 内核版本之前使用poll 进行支持, 也是使用 Reactor模型. |
| epoll  | 高       | Reactor/Proactor | Linux        | Linux kernels 2.6 内核版本以及以后使用的 epoll 进行支持, Linux kernels 2.6 内核版本之前使用 poll 进行支持, 另外一定要注意, 由于linux 下没有Window 下的IOCP 技术提供真正的异步IO支持, 所以linux 下使用epoll 模拟异步IO |
| kqueue | 高       | Proactor         | Linux        | 目前JAVA版本不支持                                           |

多路复用IO技术适用于高并发场景, 所以高并发是指1毫秒内至少同时有上千个连接请求准备好, 其他情况下使用多路复用IO技术发挥不出来它的优势.  另一方面, 使用Java NIO 进行功能实现, 相对于传统的 Socket 套接字实现要复杂一些, 所以实际应用中, 需要根据自己的业务需求进行技术选择. 

##  2.NIO源码初探

说到源码,先从 Selector 的open 方法开始说起, 

```java
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }
```

然后再看看 `SelectorProvider.provider()` 做了什么:

```java
public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                            if (loadProviderFromProperty())
                                return provider;
                            if (loadProviderAsService())
                                return provider;
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```

其中 `provider = sun.nio.ch.DefaultSelectorProvider.create();` 会根据操作系统来返回不同的实现类, window 平台就会返回 `WindowsSelectorProvider` , 而 ` if (provider != null) return provider;`  则保证了整个server程序中只有一个 `WindowsSelectorProvider`  对象. 

再看看`WindowsSelectorProvider.openSelector()`  方法

```java
   public AbstractSelector openSelector() throws IOException {
        return new WindowsSelectorImpl(this);
    }
```

`new WindowsSelectorImpl(this);` 代码

```java
    WindowsSelectorImpl(SelectorProvider var1) throws IOException {
        super(var1);
         private final Pipe wakeupPipe = Pipe.open();
        private PollArrayWrapper pollWrapper = new PollArrayWrapper(8);
        this.wakeupSourceFd = ((SelChImpl)this.wakeupPipe.source()).getFDVal();
        SinkChannelImpl var2 = (SinkChannelImpl)this.wakeupPipe.sink();
        var2.sc.socket().setTcpNoDelay(true);
        this.wakeupSinkFd = var2.getFDVal();
        this.pollWrapper.addWakeupSocket(this.wakeupSourceFd, 0);
    }
```

其中,Pipe.open() 是关键,这个方法的调用过程是:

```java
    public static Pipe open() throws IOException {
        return SelectorProvider.provider().openPipe();
    }
```

SelectorProvider 中:

```java
   public Pipe openPipe() throws IOException {
        return new PipeImpl(this);
    }
```



---------------------





ServerSocketChannel.open(）的实现:

```java
   public static ServerSocketChannel open() throws IOException {
        return SelectorProvider.provider().openServerSocketChannel();
    }
```

SelectorProvider : 

```JAVA
  public ServerSocketChannel openServerSocketChannel() throws IOException {
        return new ServerSocketChannelImpl(this);
    }
```

可见创建的 ServerSocketChannelImpl 也有WindowsSelectorImpl 的引用

```java
    ServerSocketChannelImpl(SelectorProvider var1) throws IOException {
        super(var1);
        this.fd = Net.serverSocket(true);
        this.fdVal = IOUtil.fdVal(this.fd);
        this.state = 0;
    }
```

然后通过ServerSocketChanne.register(selector, SelectionKey.OP_ACCEPT);  把 selector和 channel 绑定在一起, 也即是把new ServerSocketChanne 时创建的 FD 与 selector  绑定在一起.

到此为止, server 段已经启动完成了, 主要创建了以下对象

WindowsSelectorImpl: 单例

WindowsSelectorImpl中包含了:

- pollWrapper: 保存 selector 上注册的FD,包括 pipe 的 write端FD 和 ServerSocketChanne 所用的FD，
- wakeupPipe: 通道,(其实就是两个FD, 一个 write, 一个read)

再到Server 的run()

 selector.select() 主要调用了WindowsSelectorImpl中的这个方法

```java
protected int doSelect(long var1) throws IOException {
        if (this.channelArray == null) {
            throw new ClosedSelectorException();
        } else {
            this.timeout = var1;
            this.processDeregisterQueue();
            if (this.interruptTriggered) {
                this.resetWakeupSocket();
                return 0;
            } else {
                this.adjustThreadsCount();
                this.finishLock.reset();
                this.startLock.startThreads();

                try {
                    this.begin();

                    try {
                        this.subSelector.poll();
                    } catch (IOException var7) {
                        this.finishLock.setException(var7);
                    }

                    if (this.threads.size() > 0) {
                        this.finishLock.waitForHelperThreads();
                    }
                } finally {
                    this.end();
                }

                this.finishLock.checkForException();
                this.processDeregisterQueue();
                int var3 = this.updateSelectedKeys();
                this.resetWakeupSocket();
                return var3;
            }
        }
    }
```

其中, subSelector.poll() 是核心, 也就是轮询  pollWrapper 中保存的FD. 具体实现是调用 native 方法的 poll():

```java
 private int poll() throws IOException {
            return this.poll0(WindowsSelectorImpl.this.pollWrapper.pollArrayAddress, Math.min(WindowsSelectorImpl.this.totalChannels, 1024), this.readFds, this.writeFds, this.exceptFds, WindowsSelectorImpl.this.timeout);
        }


    private native int poll0(long var1, int var3, int[] var4, int[] var5, int[] var6, long var7);


```



这个poll() 会监听 pollWrapper 中的FD 有没有数据进去, 这会造成IO阻塞, 直到有数据读写事件发生, 比如, 由于 pollWrapper 中保存的也有 ServerSocketChannel 的FD， 所以只要ClientSocket 发一份数据到ServerSocket , 那么 poll() 就会返回, 如果这两种情况 都没有发生, 那么 poll0() 就一直阻塞, 也就是 selector.select() 会一直阻塞, 如果有任何一种情况发生, 那么 selector.select() 就会返回, 所有在 OperationServer的 run() 里面要用 while(true), 这样就可以保证 在  selector 接收到数据并处理完后继续监听 poll();

这时再来看 WindowsSelectorImpl.wakeup() 方法

```java

    public Selector wakeup() {
        synchronized(this.interruptLock) {
            if (!this.interruptTriggered) {
                this.setWakeupSocket();
                this.interruptTriggered = true;
            }

            return this;
        }
    }

   private void setWakeupSocket() {
        this.setWakeupSocket0(this.wakeupSinkFd);
    }

    private native void setWakeupSocket0(int var1);

```



可见. wakeup() 是通过pipe() 的 write 端 send(scoutFD,&byte,1,0), 发生一个字节1, 来唤醒 poll() , 所以在需要的时候就可以调用 selector.wakeup() 来唤醒 selector.

## 3. 反应堆Reactor

现在我们已经对阻塞IO 有了有一定的了解了. 我们直到阻塞IO 在调用 inputStream.read() 方法时是阻塞的 它会一直等待到数据到来(或超时)才会返回. 同样, 在调用 ServerSocket.accept() 方法时, 也会一直阻塞到有客户端连接才会返回, 每个客户端连接过来后, 服务端都会启动一个线程去处理该客户端的请求, 阻塞IO的通信模型示意图如下：
![](http://files.luyanan.com//img/20190919162031.png)

如果你细细分析, 一定会发现阻塞IO存在一些缺点,根据阻塞io 通信模型, 我总结了它的两点缺点：

1. 当客户端多时,会创建大量的处理线程, 且每个线程都要占用栈空间和一些CPU时间
2. 阻塞可能带来频繁的上下文且换, 且大部分的上下文切换可能是无意义的, 在这种情况下非阻塞IO 就有了它的应用场景,

Java NIO 是在JDK1.4 开始使用的. 它既可以说成是新IO , 也可以说成是非阻塞式IO， 下面是java NIO的工作原理:

1. 由一个专门的线程来处理所有的IO 事件， 并负责分发.
2. 事件驱动机制, 事件到的时候触发, 而不是同步的去监视事件
3. 线程通讯 ,线程之间通过wait、notify 等方式通讯, 保证每次上下文切换都是有意义的, 减少无谓的线程切换, 

下面贴出我理解的Java NIO反应堆的工作原理

![](http://files.luyanan.com//img/20190919163256.png)

(注：每个线程的处理流程大概都是读取数据,解码, 计算处理, 编码, 发送响应.)



##  4. Netty和NIO

### 1. netty 支持的特性与功能

按照定义来说, Netty是一个异步、事件驱动的用来做高性能、高可靠性的网络应用框架， 主要的优点有:

1. 框架设计优雅, 底层模型随意切换适应不同的网络协议要求。
2. 根据很多标准的协议、安全、编码解码的支持.
3. 解决了很多NIO不易用的问题
4. 社区更为活跃, 在很多开源框架中使用, 比如Dubbo、RocketMQ、Spark 等.

![](http://files.luyanan.com//img/20190919171130.png)

上图体现的主要是 Netty 支持的功能 或者特性:

1. 底层核心有Zero-copy-Capable Buffer, 非常已用的零拷贝Buffer, 统一的API；标准可扩展的时间模型.
2. 传输方面的支持有: 管道通信, Http隧道、TCP与UDP
3. 协议方面的支持有: 基于原始文本和二进制的协议;解压缩;大文本传输;流媒体传输; protobuf 编解码;安全认证; http和 websocket

### 2. Netty 采用NIO而非AIO的理由

1. Netty 不看重 window 上的使用, 在Linux系统上, AIO的底层实现仍然使用EPOLL,没有很好的实现AIO， 因此在性能上没有明显的优势, 而且在JDK封装了一层不容易深度优化.
2. Netty 整体架构是Reactor 模型,. 而AIO是 Proactor 模型, 混合在一起会非常混乱, 把AIO也改造成 ractor 模型 看起来是把 epoll 绕个弯又绕回来了.
3. AIO还有个缺点是接受数据需要预先分配缓存, 而不是NIO那种需要接受时才需要分配缓存, 所以对连接数量非常大但是流量小的情况, 内存浪费很多。
4. Linux 上AIO不够成熟, 处理回调结果速度跟不上处理需求， 比如外卖员太少, 顾客太多, 供不应求,. 造成处理速度有瓶颈.

