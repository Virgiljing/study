# Netty入门教程——认识Netty



Netty

### 什么是Netty？

> Netty 是一个利用 Java 的高级网络的能力，隐藏其背后的复杂性而提供一个易于使用的 API 的客户端/服务器框架。
> Netty 是一个广泛使用的 Java 网络编程框架（Netty 在 2011 年获得了Duke's Choice Award，见<https://www.java.net/dukeschoice/2011>）。它活跃和成长于用户社区，像大型公司 Facebook 和 Instagram 以及流行 开源项目如 Infinispan, HornetQ, Vert.x, Apache Cassandra 和 Elasticsearch 等，都利用其强大的对于网络抽象的核心代码。

*以上是摘自《Essential Netty In Action》这本书，本文的内容也是本人读了这本书之后的一些整理心得，如有不当之处欢迎大虾们指正*

### Netty和Tomcat有什么区别？

Netty和Tomcat最大的区别就在于通信协议，Tomcat是基于Http协议的，他的实质是一个基于http协议的web容器，但是Netty不一样，他能通过编程自定义各种协议，因为netty能够通过codec自己来编码/解码字节流，完成类似redis访问的功能，这就是netty和tomcat最大的不同。

*有人说netty的性能就一定比tomcat性能高，其实不然，tomcat从6.x开始就支持了nio模式，并且后续还有APR模式——一种通过jni调用apache网络库的模式，相比于旧的bio模式，并发性能得到了很大提高，特别是APR模式，而netty是否比tomcat性能更高，则要取决于netty程序作者的技术实力了。*

### 为什么Netty受欢迎？

如第一部分所述，netty是一款收到大公司青睐的框架，在我看来，netty能够受到青睐的原因有三：

1. **并发高**
2. **传输快**
3. **封装好**

### Netty为什么并发高

Netty是一款基于NIO（Nonblocking I/O，非阻塞IO）开发的网络通信框架，对比于BIO（Blocking I/O，阻塞IO），他的并发性能得到了很大提高，两张图让你了解BIO和NIO的区别：

![img](https://upload-images.jianshu.io/upload_images/1089449-546a563c9822ce16.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/548/format/webp)

阻塞IO的通信方式

![img](https://upload-images.jianshu.io/upload_images/1089449-9eebe781fba495fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/572/format/webp)

非阻塞IO的通信方式

```
Selector
```

这个过程是阻塞的

![img](https://upload-images.jianshu.io/upload_images/1089449-6377fd47256970ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/584/format/webp)

BIO的处理流程

![img](https://upload-images.jianshu.io/upload_images/1089449-78814cbb3acc30bd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/525/format/webp)

NIO的处理流程

除了BIO和NIO之外，还有一些其他的IO模型，下面这张图就表示了五种IO模型的处理流程：

![img](https://upload-images.jianshu.io/upload_images/1089449-6ff240c3900e0aaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/614/format/webp)

五种常见的IO模型

- BIO，同步阻塞IO，阻塞整个步骤，如果连接少，他的延迟是最低的，因为一个线程只处理一个连接，适用于少连接且延迟低的场景，比如说数据库连接。
- NIO，同步非阻塞IO，阻塞业务处理但不阻塞数据接收，适用于高并发且处理简单的场景，比如聊天软件。
- 多路复用IO，他的两个步骤处理是分开的，也就是说，一个连接可能他的数据接收是线程a完成的，数据处理是线程b完成的，他比BIO能处理更多请求，但是比不上NIO，但是他的处理性能又比BIO更差，因为一个连接他需要两次system call，而BIO只需要一次，所以这种IO模型应用的不多。
- 信号驱动IO，这种IO模型主要用在嵌入式开发，不参与讨论。
- 异步IO，他的数据请求和数据处理都是异步的，数据请求一次返回一次，适用于长连接的业务场景。

*以上摘自Linux IO模式及 select、poll、epoll详解*

### Netty为什么传输快

Netty的传输快其实也是依赖了NIO的一个特性——*零拷贝*。我们知道，Java的内存有堆内存、栈内存和字符串常量池等等，其中堆内存是占用内存空间最大的一块，也是Java对象存放的地方，一般我们的数据如果需要从IO读取到堆内存，中间需要经过Socket缓冲区，也就是说一个数据会被拷贝两次才能到达他的的终点，如果数据量大，就会造成不必要的资源浪费。
Netty针对这种情况，使用了NIO中的另一大特性——零拷贝，当他需要接收数据的时候，他会在堆内存之外开辟一块内存，数据就直接从IO读到了那块内存中去，在netty里面通过ByteBuf可以直接对这些数据进行直接操作，从而加快了传输速度。
下两图就介绍了两种拷贝方式的区别，摘自[Linux 中的零拷贝技术，第 1 部分](https://www.ibm.com/developerworks/cn/linux/l-cn-zerocopy1/index.html)

![img](https://upload-images.jianshu.io/upload_images/1089449-014c9e07d56e4be5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/481/format/webp)

传统数据拷贝

![img](https://upload-images.jianshu.io/upload_images/1089449-50e5aa5eec7e86cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/554/format/webp)

零拷贝

上文介绍的ByteBuf是Netty的一个重要概念，他是netty数据处理的容器，也是Netty封装好的一个重要体现，将在下一部分做详细介绍。

### 为什么说Netty封装好？

要说Netty为什么封装好，这种用文字是说不清的，直接上代码：

- 阻塞I/O

```
public class PlainOioServer {

    public void serve(int port) throws IOException {
        final ServerSocket socket = new ServerSocket(port);     //1
        try {
            for (;;) {
                final Socket clientSocket = socket.accept();    //2
                System.out.println("Accepted connection from " + clientSocket);

                new Thread(new Runnable() {                        //3
                    @Override
                    public void run() {
                        OutputStream out;
                        try {
                            out = clientSocket.getOutputStream();
                            out.write("Hi!\r\n".getBytes(Charset.forName("UTF-8")));                            //4
                            out.flush();
                            clientSocket.close();                //5

                        } catch (IOException e) {
                            e.printStackTrace();
                            try {
                                clientSocket.close();
                            } catch (IOException ex) {
                                // ignore on close
                            }
                        }
                    }
                }).start();                                        //6
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

- 非阻塞IO

```
public class PlainNioServer {
    public void serve(int port) throws IOException {
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        ServerSocket ss = serverChannel.socket();
        InetSocketAddress address = new InetSocketAddress(port);
        ss.bind(address);                                            //1
        Selector selector = Selector.open();                        //2
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);    //3
        final ByteBuffer msg = ByteBuffer.wrap("Hi!\r\n".getBytes());
        for (;;) {
            try {
                selector.select();                                    //4
            } catch (IOException ex) {
                ex.printStackTrace();
                // handle exception
                break;
            }
            Set<SelectionKey> readyKeys = selector.selectedKeys();    //5
            Iterator<SelectionKey> iterator = readyKeys.iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                try {
                    if (key.isAcceptable()) {                //6
                        ServerSocketChannel server =
                                (ServerSocketChannel)key.channel();
                        SocketChannel client = server.accept();
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_WRITE |
                                SelectionKey.OP_READ, msg.duplicate());    //7
                        System.out.println(
                                "Accepted connection from " + client);
                    }
                    if (key.isWritable()) {                //8
                        SocketChannel client =
                                (SocketChannel)key.channel();
                        ByteBuffer buffer =
                                (ByteBuffer)key.attachment();
                        while (buffer.hasRemaining()) {
                            if (client.write(buffer) == 0) {        //9
                                break;
                            }
                        }
                        client.close();                    //10
                    }
                } catch (IOException ex) {
                    key.cancel();
                    try {
                        key.channel().close();
                    } catch (IOException cex) {
                        // 在关闭时忽略
                    }
                }
            }
        }
    }
}

```

- Netty

```
public class NettyOioServer {

    public void server(int port) throws Exception {
        final ByteBuf buf = Unpooled.unreleasableBuffer(
                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));
        EventLoopGroup group = new OioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();        //1

            b.group(group)                                    //2
             .channel(OioServerSocketChannel.class)
             .localAddress(new InetSocketAddress(port))
             .childHandler(new ChannelInitializer<SocketChannel>() {//3
                 @Override
                 public void initChannel(SocketChannel ch) 
                     throws Exception {
                     ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {            //4
                         @Override
                         public void channelActive(ChannelHandlerContext ctx) throws Exception {
                             ctx.writeAndFlush(buf.duplicate()).addListener(ChannelFutureListener.CLOSE);//5
                         }
                     });
                 }
             });
            ChannelFuture f = b.bind().sync();  //6
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();        //7
        }
    }
}

```

从代码量上来看，Netty就已经秒杀传统Socket编程了，但是这一部分博大精深，仅仅贴几个代码岂能说明问题，在这里给大家介绍一下Netty的一些重要概念，让大家更理解Netty。

- Channel
  数据传输流，与channel相关的概念有以下四个，上一张图让你了解netty里面的Channel。

  ​

  ​

  ![img](https://upload-images.jianshu.io/upload_images/1089449-afd9e14197e1ef11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/751/format/webp)

  Channel一览

  - Channel，表示一个连接，可以理解为每一个请求，就是一个Channel。
  - **ChannelHandler**，核心处理业务就在这里，用于处理业务请求。
  - ChannelHandlerContext，用于传输业务数据。
  - ChannelPipeline，用于保存处理过程需要用到的ChannelHandler和ChannelHandlerContext。

- ByteBuf
  ByteBuf是一个存储字节的容器，最大特点就是**使用方便**，它既有自己的读索引和写索引，方便你对整段字节缓存进行读写，也支持get/set，方便你对其中每一个字节进行读写，他的数据结构如下图所示：

![img](https://upload-images.jianshu.io/upload_images/1089449-b1ec677f253b692a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/558/format/webp)

ByteBuf数据结构

他有三种使用模式：

1. Heap Buffer 堆缓冲区
   堆缓冲区是ByteBuf最常用的模式，他将数据存储在堆空间。

2. Direct Buffer 直接缓冲区

   ​

   直接缓冲区是ByteBuf的另外一种常用模式，他的内存分配都不发生在堆，jdk1.4引入的nio的ByteBuffer类允许jvm通过本地方法调用分配内存，这样做有两个好处

   - 通过免去中间交换的内存拷贝, 提升IO处理速度; 直接缓冲区的内容可以驻留在垃圾回收扫描的堆区以外。
   - DirectBuffer 在 -XX:MaxDirectMemorySize=xxM大小限制下, 使用 Heap 之外的内存, GC对此”无能为力”,也就意味着规避了在高负载下频繁的GC过程对应用线程的中断影响.

3. Composite Buffer 复合缓冲区
   复合缓冲区相当于多个不同ByteBuf的视图，这是netty提供的，jdk不提供这样的功能。

除此之外，他还提供一大堆api方便你使用，在这里我就不一一列出了，具体参见[ByteBuf字节缓存](https://waylau.gitbooks.io/essential-netty-in-action/content/CORE%20FUNCTIONS/Buffers.html)。

- Codec
  Netty中的编码/解码器，通过他你能完成字节与pojo、pojo与pojo的相互转换，从而达到自定义协议的目的。
  在Netty里面最有名的就是HttpRequestDecoder和HttpResponseEncoder了。

[Netty入门教程2——动手搭建HttpServer](https://www.jianshu.com/p/ed0177a9b2e3)
[Netty入门教程3——Decoder和Encoder](https://www.jianshu.com/p/fd815bd437cd)
[Netty入门教程4——如何实现长连接](https://www.jianshu.com/p/9d89b2299ce4)

以上就是我对《Netty实战》这本书的一些心得和书外的一些相关知识整理，如果有不同的见解，欢迎讨论！