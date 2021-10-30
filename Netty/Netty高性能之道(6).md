# Netty高性能之道

## 1. 背景介绍

### 1. Netty惊人的性能数据

通过使用Netty(NIO框架) 相比于传统基于Java 序列化+ BIO(同步阻塞IO)的通信框架， 性能提升了8倍多.

### 2. 传统RPC 调用性能差的三宗罪

网络传输方式问题: 传统的RPC框架或者基于RMI等方式的远程服务(过程) 调用采用了同步阻塞IO,当客户端的并发压力或者网络时延增大后, 同步阻塞IO 会由于频繁的wait 导致IO线程经常性的阻塞, 由于线程无法高效的工作，io处理能力自然下降. 下面, 我们通过BIO 通信模型看下BIO通信的弊端.

![](http://files.luyanan.com//img/20190921143042.png)

采用BIO通信模型的服务端，通常由一个独立的Acceptor 线程负责客户端的连接, 接收到客户端连接后, 为客户端连接创建一个新的线程处理请求消息. 处理完成后, 返回应答消息给客户端, 线程销毁。 这就是典型的一线程一应答模型. 该架构最大的问题就是不具备弹性伸缩能力, 当并发访问量增加后, 服务端的线程个数和并发访问数呈线性正比, 由于线程是JAVA 虚拟机非常宝贵的资源, 当线程数膨胀后，系统的性能急剧下降, 随着并发量的继续增加，可能会发生句柄溢出、线程堆栈溢出等问题, 并最终导致服务端宕机.

序列化方式问题: Java 序列化存在如下几个典型的问题:

1. Java 序列化机制是Java内部的一种对象编解码技术, 无法跨语言使用, 例如对于异构系统之间的对接,Java序列化后的码流需要能过够通过其他语言反序列化成原始对象(副本),目前很难支持.
2. 相比于其他开源的序列化框架，Java序列化后的码流太大, 无论是网络传输还是持久化到磁盘, 都会导致额外的资源占用.
3. 序列化性能差(CPU资源占用高)
4. 线程模型问题: 由于采用同步阻塞IO，这会导致每个TCP 连接都占用一个线程, 由于线程资源是JVM 虚拟机最宝贵的资源, 当IO读写阻塞导致线程无法及时释放时, 会导致系统性能急剧下降, 严重的甚至会导致虚拟机无法创建新的线程.

### 3. 高性能的三个主题

1. 传输: 用什么样的通道将数据发送给对方, BIO、NIO 或者AIO,IO模型在很大程度上决定了框架的性能.
2. 协议: 采用什么样的通信协议, HTTP或者内部私有协议. 协议的选择不同,性能模型也不同, 相比于公有协议, 内部私有协议的性能通常可以被设计的更优.
3. 线程: 数据包如何读取, 读取之后的百年编解码在哪个线程进行, 编解码后的消息如何派发, Peactor 线程模型的不同, 对性能的影响也非常大.

## 2. Netty 高性能之道

###  1 异步非阻塞通信

在IO编写过程中, 当需要同时处理多个客户端接入请求时， 可以利用多线程或者IO多路复用技术进行处理。 IO多路复用通过把多个IO阻塞复用到同一个select 的阻塞上, 从而使得系统在单线程的情况下可以同时处理多个客户端请求. 与传统的多线程/多进程模型相比, io 多路复用的最大优势是系统开销小. 系统不需要创建新的额外的进程或者线程, 也不需要维护这些进程和线程的运行, 降低了系统的维护工作量, 节省了系统资源。

JDK1.4 提供了对非阻塞IO(NIO)的支持, JDK1.5_update10 版本使用epoll 替代了传统的 select/poll , 极大的提升了NIO通信的性能. JDK NIO 通信模型如下所示：

![](http://files.luyanan.com//img/20190923100808.png)

于Socket类和ServerSocket类相比, NIO 也提供了SocketChannel 和ServerSocketChannel 两种不同的套接字通道实现, 这两种新增的通道都支持阻塞和非阻塞两种模式, 阻塞模式使用非常简单, 但是性能和可靠性都不好, 非阻塞模型正好相反. 开发人员一般可以根据自己的需要来选择合适的开发模型, 一般来说, 低负载、低并发的应用程序可以选择同步阻塞IO 以降低编程复杂度. 但是对于高负载、高并发的网络应用, 需要使用NIO的非阻塞模型进行开发, Netty 架构按照Reactor模式设计和实现, 他的服务端通信序列图如下:

![](http://files.luyanan.com//img/20190923101835.png)

客户端通信序列图如下:

![](http://files.luyanan.com//img/20190923101901.png)

Netty的IO线程 NioEventLoop 聚合了多路复用器 Selector, 可以同时并发处理成百上千个客户端 Channel, 由于读写操作都是非阻塞的, 这就可以充分提升IO线程的运行效率, 避免由于频繁IO阻塞导致的线程挂起. 另外， 由于Netty 采用了异步通信模型, 一个IO线程可以并发处理N个客户端连接和读写操作， 这就从根据上解决了传统同步IO阻塞 `一连接一线程` 模型, 架构的性能, 弹性伸缩能力和可靠性都得到了极大的提升.

### 2. 零拷贝

Netty的`零拷贝` 主要体现在如下三个方面:

1. Netty的接收和发送ByteBuffer 采用 `DIRECT BUFFERS`, 使用堆外直接内存进行Socket 读写， 不需要进行字节缓存区的二次拷贝, 如果使用传统的堆内存(HEAP BUFFERS) 使用Socket读写,  JVM 会将堆内存Buffer 拷贝一份到直接内存中, 然后才写入到Socket, 相比于堆外直接内存, 消息在发送过程中多了一次缓冲区的内存拷贝。 
2. Netty 提供了组合Buffer对象, 可以聚合多个ByteBuffer 对象, 用户可以像操作一个Buffer 那样方便的对组合Buffer 进行操作, 避免了传统使用内存拷贝的方式将几个小Buffer 合并成一个大的Buffer .
3. Netty 的文件传输采用了 transferTo() 方法, 它可以直接将文件缓冲区的数据发送到目标Channel, 避免了传统通过 循环 write() 方式导致的内存拷贝问题.

下面, 我们对上述的三种`零拷贝` 进行说明, 先看Netty 接受Buffer的创建

打开 AbstractNioByteChannel的 NioByteUnsafe

```java
     @Override
        public final void read() {
            final ChannelConfig config = config();
            final ChannelPipeline pipeline = pipeline();
            final ByteBufAllocator allocator = config.getAllocator();
            final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle();
            allocHandle.reset(config);

            ByteBuf byteBuf = null;
            boolean close = false;
            try {
                do {
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (close) {
                    closeOnRead(pipeline);
                }
            } catch (Throwable t) {
                handleReadException(pipeline, byteBuf, t, close, allocHandle);
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
        }
```

再找到  do while 中的 `allocHandle.allocate(allocator)` 方法, 实际上调用的是

DefaultMaxMessagesRecvByteBufAllocator中的MaxMessageHandle的 allocate 方法

```java
   @Override
        public ByteBuf allocate(ByteBufAllocator alloc) {
            return alloc.ioBuffer(guess());
        }
```

相当于每次循环读取一次消息, 就通过 AbstractByteBufAllocator的 ioBuffer 方法获取 ByteBuf 对象， 下面继续看他的接口定义

```java
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf> {
}
```

当进行 Socket IO 读写的时候, 为了避免从堆内存拷贝一份副本到直接内存, Netty的ByteBuf 分配器直接创建非堆内存避免缓冲区的二次拷贝, 通过`零拷贝` 来提升读写性能.

下面我们继续看第二种 `零拷贝` 的 实现 CompositeByteBuf, 他对外将多个 ByteBuf 封装成一个ByteBuf, 对外提供统一封装后的 ByteBuf接口, 他的类定义如下:

![](http://files.luyanan.com//img/20190923110501.png)

通过继承关系我们可以看出CompositeByteBuf  实际上就是个 ByteBuf的包装器, 它将多个 ByteBuf 的包装器, 他将多个ByteBuf 组合成一个集合, 然后对外提供统一的ByteBuf 接口, 相关定义如下:

```java
    private static final ByteBuffer EMPTY_NIO_BUFFER = Unpooled.EMPTY_BUFFER.nioBuffer();
    private static final Iterator<ByteBuf> EMPTY_ITERATOR = Collections.<ByteBuf>emptyList().iterator();

    private final ByteBufAllocator alloc;
    private final boolean direct;
    private final List<Component> components;
    private final int maxNumComponents;

    private boolean freed;
```

添加ByteBuf ,  不需要做内存拷贝, 相关代码如下：

```java
  private int addComponent0(boolean increaseWriterIndex, int cIndex, ByteBuf buffer) {
        assert buffer != null;
        boolean wasAdded = false;
        try {
            checkComponentIndex(cIndex);

            int readableBytes = buffer.readableBytes();

            // No need to consolidate - just add a component to the list.
            @SuppressWarnings("deprecation")
            Component c = new Component(buffer.order(ByteOrder.BIG_ENDIAN).slice());
            if (cIndex == components.size()) {
                wasAdded = components.add(c);
                if (cIndex == 0) {
                    c.endOffset = readableBytes;
                } else {
                    Component prev = components.get(cIndex - 1);
                    c.offset = prev.endOffset;
                    c.endOffset = c.offset + readableBytes;
                }
            } else {
                components.add(cIndex, c);
                wasAdded = true;
                if (readableBytes != 0) {
                    updateComponentOffsets(cIndex);
                }
            }
            if (increaseWriterIndex) {
                writerIndex(writerIndex() + buffer.readableBytes());
            }
            return cIndex;
        } finally {
            if (!wasAdded) {
                buffer.release();
            }
        }
    }
```

最后, 我们看一下文件传输的 `零拷贝`

```java
    @Override
    public long transferTo(WritableByteChannel target, long position) throws IOException {
        long count = this.count - position;
        if (count < 0 || position < 0) {
            throw new IllegalArgumentException(
                    "position out of range: " + position +
                    " (expected: 0 - " + (this.count - 1) + ')');
        }
        if (count == 0) {
            return 0L;
        }
        if (refCnt() == 0) {
            throw new IllegalReferenceCountException(0);
        }
        // Call open to make sure fc is initialized. This is a no-oop if we called it before.
        open();

        long written = file.transferTo(this.position + position, count, target);
        if (written > 0) {
            transferred += written;
        }
        return written;
    }
```

Netty文件传输  `DefaultFileRegion` 通过 transferTo 方法将文件 发送到目标Channel, 下面重点看 FileChannel 的 transferTo  方法， 它的API DOC 说明如下:

```java

    /**
     * Transfers bytes from this channel's file to the given writable byte
     * channel.
     *
     * <p> An attempt is made to read up to <tt>count</tt> bytes starting at
     * the given <tt>position</tt> in this channel's file and write them to the
     * target channel.  An invocation of this method may or may not transfer
     * all of the requested bytes; whether or not it does so depends upon the
     * natures and states of the channels.  Fewer than the requested number of
     * bytes are transferred if this channel's file contains fewer than
     * <tt>count</tt> bytes starting at the given <tt>position</tt>, or if the
     * target channel is non-blocking and it has fewer than <tt>count</tt>
     * bytes free in its output buffer.
     *
     * <p> This method does not modify this channel's position.  If the given
     * position is greater than the file's current size then no bytes are
     * transferred.  If the target channel has a position then bytes are
     * written starting at that position and then the position is incremented
     * by the number of bytes written.
     *
     * <p> This method is potentially much more efficient than a simple loop
     * that reads from this channel and writes to the target channel.  Many
     * operating systems can transfer bytes directly from the filesystem cache
     * to the target channel without actually copying them.  </p>
     *
     * @param  position
     *         The position within the file at which the transfer is to begin;
     *         must be non-negative
     *
     * @param  count
     *         The maximum number of bytes to be transferred; must be
     *         non-negative
     *
     * @param  target
     *         The target channel
     *
     * @return  The number of bytes, possibly zero,
     *          that were actually transferred
     *
     * @throws IllegalArgumentException
     *         If the preconditions on the parameters do not hold
     *
     * @throws  NonReadableChannelException
     *          If this channel was not opened for reading
     *
     * @throws  NonWritableChannelException
     *          If the target channel was not opened for writing
     *
     * @throws  ClosedChannelException
     *          If either this channel or the target channel is closed
     *
     * @throws  AsynchronousCloseException
     *          If another thread closes either channel
     *          while the transfer is in progress
     *
     * @throws  ClosedByInterruptException
     *          If another thread interrupts the current thread while the
     *          transfer is in progress, thereby closing both channels and
     *          setting the current thread's interrupt status
     *
     * @throws  IOException
     *          If some other I/O error occurs
     */
    public abstract long transferTo(long position, long count,
                                    WritableByteChannel target)
        throws IOException;
```

对于很多操作系统它直接将文件缓冲区的内容发送到目标Channel, 而不需要通过拷贝的方式, 这是一种更加高效的传输方式, 它实现了文件传输的`零拷贝`

###  3. 内存池

随着JVM 虚拟机和JIT 及时编译技术的发展, 对象的分配和回收是个非常轻量级的工作. 但是对于缓存区Buffer,情况却稍有不同, 特别是对于堆外内存的分配和回收， 是一件耗时的操作. 为了尽量重用缓冲区, Netty 提供了基于内存池的缓冲区重用机制, 下面我们一起看下Netty ByteBuf 的实现

![](http://files.luyanan.com//img/20190923112727.png)

Netty 提供了多种内存管理策略, 通过在启动辅助类中配置相关参数, 可以实现差异化的定制， 

下面通过性能测试, 我们看下基于内存池循环利用的ByteBuf 和普通的 ByteBuf 的性能差异

```java
package com.formula.netty.base.buffer;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.PooledByteBufAllocator;
import io.netty.buffer.Unpooled;

/**
 * @author luyanan
 * @since 2019/9/23
 * <p> 基于内存池循环利用的 ByteBuf 和普通 ByteBuf 的性能差异</p>
 **/
public class ByteBufDemo {

    public static void main(String[] args) {
        final byte[] CONTENT = new byte[1024];
        int loop = 1800000;
        long startTime = System.currentTimeMillis();
        ByteBuf poolBuffer = null;
        for (int i = 0; i < loop; i++) {
            poolBuffer = PooledByteBufAllocator.DEFAULT.directBuffer(1024);
            poolBuffer.writeBytes(CONTENT);
            poolBuffer.release();
        }
        long endTime = System.currentTimeMillis();
        System.out.println("内存池分配缓冲区耗时：" + (endTime - startTime) + "ms");


        long startTime2 = System.currentTimeMillis();

        ByteBuf buf = null;

        for (int i = 0; i < loop; i++) {
            buf = Unpooled.directBuffer(1024);
            buf.writeBytes(CONTENT);
        }
        long endTime2 = System.currentTimeMillis();
        System.out.println("非内存池分配缓冲区耗时: " + (endTime2 - startTime2) + "ms");
    }
}

```

各执行 180万次, 性能对比结果如下:

```
内存池分配缓冲区耗时：773ms
非内存池分配缓冲区耗时: 1705ms
```

性能测试表明, 采用内存池的ByteBuf 相比于 于朝生夕灭的ByteBuf,性能高了23倍左右(性能数据与使用场景强相关), 下面我们一起简单分析下 Netty 内存池的内存分配

```java
    @Override
    public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
        if (initialCapacity == 0 && maxCapacity == 0) {
            return emptyBuf;
        }
        validate(initialCapacity, maxCapacity);
        return newDirectBuffer(initialCapacity, maxCapacity);
    }
```

继续看 newDirectBuffer 方法, 我们发现他是一个抽象方法, 由  AbstractByteBufAllocator的子类 负责具体实现. 代码如下:

![](http://files.luyanan.com//img/20190923114708.png)

代码跳转到 PooledByteBufAllocator的 newDirectBuffer 方法. 从Cache 中获取内存区域 PoolArena, 调用它的 allocate 方法进行内存分配

```java
 @Override
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        ByteBuf buf;
        if (directArena != null) {
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            if (PlatformDependent.hasUnsafe()) {
                buf = UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
            } else {
                buf = new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
            }
        }

        return toLeakAwareBuffer(buf);
    }
```

PoolArena的 allocate 方法如下： 

```java
    PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
        PooledByteBuf<T> buf = newByteBuf(maxCapacity);
        allocate(cache, buf, reqCapacity);
        return buf;
    }
```

我们重点分析newByteBuf 的实现, 他同样是个抽象方法, 由子类 DirectArena 和 HeapArena 来实现不同类型的缓冲区分配, 由于测试用例使用的是 堆外内存

![](http://files.luyanan.com//img/20190923115636.png)

因此重点分析 DirectArena 的实现, 如果没有开启使用 sun的 unsafe ,则

```java
 @Override
        protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
            if (HAS_UNSAFE) {
                return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
            } else {
                return PooledDirectByteBuf.newInstance(maxCapacity);
            }
        }
```

执行 PooledDirectByteBuf.newInstance 的方法，代码如下:

```java
    static PooledDirectByteBuf newInstance(int maxCapacity) {
        PooledDirectByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }
```

通过 RECYCLER的get 方法循环使用ByteBuf 对象， 如果是非内存池实现, 则直接创建一个新的ByteBuf 对象, 从缓冲池中获取ByteBuf 之后, 调用 AbstractReferenceCountedByteBuf的 setRefCnt 方法设置引用计数器, 用于对象的引用计数和内存回收(类似于JVM垃圾回收机制)

### 4. 高效的Reactor 线程模型

常见的Reactor模型有三种, 分别如下:

1. Reactor 单线程模型

     Reactor 单线程模型,指的是所有的IO操作都在同一个NIO 线程上面完成, NIO线程的职责如下:

   1. 作为NIO服务端， 接受客户端的TCP连接
   2. 作为NIO客户端, 想服务端发起TCP连接
   3. 读取通信对端的请求或者应答消息
   4. 想通信对端发送消息请求或者应答请求.

   Reactor 单线程模型示意图如下所示

   ![](http://files.luyanan.com//img/20190923133948.png)

   由于Reactor 模式使用的是异步非阻塞IO， 所有的IO操作都不会导致阻塞, 理论上一个线程可以独立处理所有的IO相关的操作. 从架构层面上, 一个NIO线程确实可以完成其承担的职责, 例如, 通过Accepter 接受客户端的TCP 连接请求消息, 链路 建立成功后,通过Dispatch 将对用的ByteBuf 派发到指定的 Handler 上进行消息解码. 用户的Handler 可以通过NIO线程 将消息发送至客户端.

   对于一个小容量应用场景, 可以使用单线程模型. 但是对于高负载、大并发的应用却不合适, 主要原因如下：

   1. 一个NIO线程同时处理成百上千的链路, 性能上无法支持, 及时NIO的线程的CPU负荷达到100%, 也无法满足海量消息的解码、编码、读取和发送,
   2. 当NIO 线程负载过重后， 处理速度将变慢,这会导致大量客户端连接超时, 超时之后往往会进行重发， 这更加重了NIO线程的负载, 最终会导致大量消息积压和处理超时, NIO线程会成为系统的性能瓶颈,
   3. 可靠性问题: 一旦NIO线程 意外跑飞, 或者进入死循环, 会导致整个系统通信模块不可用, 不能接受和处理外部消息， 造成节点故障.

   为了解决这些问题, 演进出了 Reactor 多线程模型, 接下来我们一起来学习一下Reactor 多线程模型

2. Reactor 多线程模型

    Reactor 多线程模型与单线程模型最大的区别就是有一组NIO线程处理IO操作, 它的原理图如下：

   ![](http://files.luyanan.com//img/20190923135226.png)

   Reactor 多线程模型的特点如下：

   1. 专门有一个NIO线程 Acceptor 线程用于监听服务端, 接受客户端TCP连接请求.
   2. 网络IO操作-读、写等由一个NIO线程池负责, 线程池可以采用标准的JDK 线程池实现, 它包含了一个任务队列和N个可用的线程, 由这些线程负责消息的读取、解码、编码和发送.
   3. 1个NIO线程可以同时处理N个链路, 但是1个链路只对用1个线程, 防止发生并发操作问题.

   ​       在绝大数场景下, Reactor 多线程模型都可以满足性能需求, 但是, 在极特殊应用场景中, 一个NIO 线程负责监听和处理所有的客户端连接可能会存在性能问题, 例如百万客户端并发链接，或者服务端需要对客户端的握手消息进行安全认证, 认证本身非常耗损性能, 这这些场景下， 单独一个Accepter 线程可能会存在性能不足问题, 为了解决性能问题, 产生了第三种Reactor线程模型 - 主从Reactor 多线程模型.

3. 主从Reactor  多线程模型

    主从Reactor 线程模型的特点是: 服务端用于接收客户端连接的不再是一个单独的NIO线程, 而是一个独立的NIO线程池, Acceptor 接收到客户端TCP 连接请求处理完成后(可能包含接入认证), 将新创建的 SocketChannel 注册到IO线程池(sub reactor 线程池) 的某个IO 线程上, 由他负责 SocketChannel 的读写和编解码工作. Acceptor 线程池仅仅只用于客户端的登录、握手和安全认证, 一旦链路建立成功, 就将链路注册到后端 subReactor 线程池的IO线程上, 由IO线程负责后续的IO操作.

   它的线程模型如下图所示:

   ![](http://files.luyanan.com//img/20190923140323.png)

   利用主从IO线程模型, 可以解决1 个服务端监听线程无法有效处理所有客户端连接的性能不足问题, 因此, 在Netty 的官方demo中, 推荐使用该线程模型.

   事实上, Netty的线程模型并非固定不变, 通过在启动辅助类中创建不同的EventLoopGroup 实例并通过适当的参数配置， 就可以支持上述三种Reactor 线程模型, 正是因为Netty 对Reactor 线程模型的支持提供了灵活的定制能力, 所以可以满足不同业务场景的性能诉求.

###  5. 无锁化的串行设计理念.

   在大多数场景下, 并行多线程处理可以提升系统的并发性能. 但是, 如果对于共享资源的并发访问处理不当, 会带来严重的锁竞争, 这最终会导致性能的下降. 为了尽可能的避免锁竞争带来的性能损耗，可以通过串行化设计, 即消息的处理尽可能在同一个线程内完成, 期间不进行线程切换, 这样就避免了多线程竞争和同步锁。

为了尽可能的提升性能, Netty采用了串行无锁化设计, 在IO线程内部进行串行操作, 避免多线程竞争导致的性能下降, 表面上看, 串行化设计似乎CPU利用率不高， 并发程度不够, 但是, 通过调整NIO线程池的线程参数, 可以同时启动多个串行化的线程并行运行, 这种局部无锁化的串行线程设计相比一个队列-多个工作线程模型性能更优.

Netty 的串行化设计工作原理如下:

![](http://files.luyanan.com//img/20190923141322.png)

Netty的 NioEventLoop  读取到消息后, 直接调用 ChannelPipeline 的 fireChannelRead(Object msg)，只要用户不主动切换线程, 一直会由 NioEventLoop 调用到用户的handler, 期间不进行线程切换, 这种串行化处理方式避免了多线程操作导致的锁的竞争, 从性能角度看是最优的. 

### 6. 高效的并发编程.

Netty的高效并发编程主要体现在如下几点:

1. volatile 的大量、正确使用
2. CAS 和原子类的广泛使用
3. 线程安全容器的使用
4. 通过读写锁提升并发性能.

### 7. 高性能的序列化框架

影响序列化性能的关键因素总结如下:

1. 序列化后的码流大小(网络带宽的占用)
2. 序列化&反序列化的性能(CPU资源占用)
3. 是否支持跨语言(异构系统的对接和开发语言切换)
4. Netty 默认提供了对Google Protobuf的支持， 通过扩展 Netty的编解码接口, 用户可以实现其他的高性能序列化框架, 例如 Thrift 的压缩二进制编解码框架, 下面我们一起来看一下 不同序列化&反序列化框架序列化后的字节数组大小.

![](http://files.luyanan.com//img/20190923142442.png)

从上图可以看出, Protobuf 序列化后的码流只有Java 序列化后的1/4左右, 正是由于Java原生序列化性能表现太差, 才催生了各种高性能的开源序列化技术和框架(性能差只是其中的一个原因, 还有跨语言、IDL定义等其他因素)

### 8. 灵活的TCP参数配置能力

合理设置TCP参数在某些场景下对于性能的提升可以起到显著的效果, 例如SO_RCVBUF 和 SO_SNDBUF。如果设置不当, 对性能的影响是非常大的, 下面我们总结一下对性能影响比较大的几个配置项.

1. SO_RCVBUF 和 SO_SNDBUF : 通常建议值设置为128K 或者256K
2. SO_TCPNODELAY：NAGLE算法通过将缓冲区内的小封包自动相连. 组成较大的封包, 阻止大量小封包的发送阻塞网络, 从而提高网络应用效率. 但是对于时延敏感的应用场景来说需要关闭该优化算法.
3. 软中断: 如果linux 内核版本支持 RPS(2.6.35 以上版本),开启 RPS 后可以实现软中断, 提升网络吞吐量, RPS 根据数据包的源地址, 目的地址,以及目的源地址和端口,计算出一个hash值, 然后根据这个hash值来选择软中断 运行的CPU， 从上层来看, 也就是将每个链接和cpu绑定, 并通过这个hash值, 来均衡软中断在多个cpu上, 提升网络并行处理性能.

Netty在启动辅助类中可以灵活的配置TCP参数, 满足不同的用户场景, 相关配置接口定义如下：

![](http://files.luyanan.com//img/20190923143722.png)



