# Netty高性能介绍

Netty官网对自己的定位：

> Netty is an asynchronous event-driven network application framework for rapid development of maintainable high performance protocol servers & clients.

&#x20;     可以看到，Netty的主要目标就是提供高性能的服务或者客户端。此外，其支持多种协议的应用开发，比如HTTP，WebSocket等，功能较强大。

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

&#x20;     目前很多成熟的框架底层通信都使用了Netty，比如lettuce，RocketMQ，Dubbo，ES等。作为一名JAVA开发者，Netty还是很有必要学习的，不过，Netty的源码很多，所以我们学习的不应该是细节，而是其设计思想。

### 1、Reactor线程模型

&#x20;     在高性能网络I/O模式中介绍了几种Reactor模式，包括单Reactor单线程、单Reactor多线程、多Reactor多线程模型，实际上在Netty中，这三种模型都可以实现，只不过是多Reactor多线程的模型更加常用。

&#x20;   Netty通过NioEventLoop来实现IO多路复用，用于处理请求，其整体的模型如下：

&#x20; &#x20;

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

&#x20;   Netty在启动时会创建两个NioEventLoopGroup，一个用于处理连接，通常称为boss，一个用于处理I/O，通常称为work。

&#x20;通用的创建NettyServer的方法：

```java
 public void start(EventLoopGroup boss,EventLoopGroup work) throws InterruptedException {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(boss,work)
                .channel(NioServerSocketChannel.class)
                .localAddress(new InetSocketAddress(6666))
                .option(ChannelOption.SO_BACKLOG,1024)
                .childOption(ChannelOption.SO_KEEPALIVE,true)
                .childOption(ChannelOption.TCP_NODELAY,true)
                 //定义的channel handler
                .childHandler(new NettyServerInitializer());

        try {
            ChannelFuture channelFuture = bootstrap.bind(6666).sync();
            log.info("nettyServer开始监听接口:6666");
            channelFuture.channel().closeFuture().sync();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            boss.shutdownGracefully();
            work.shutdownGracefully();
        }
    }
```

ChannelPipeLine的定义：

```java
public class NettyServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    protected void initChannel(SocketChannel socketChannel) throws Exception {
        ChannelPipeline pipeline = socketChannel.pipeline();
        pipeline.addLast("framer", new DelimiterBasedFrameDecoder(8192, Delimiters.lineDelimiter()));
        pipeline.addLast("decoder", new StringDecoder(CharsetUtil.UTF_8));
        pipeline.addLast("encoder", new StringEncoder(CharsetUtil.UTF_8));
        pipeline.addLast("handler", new NettyServerHandler());
    }
}
```

如果想实现不同的Reactor模型，只要调整这两个对象参数即可。

&#x20;1）单Reactor模式。

```java
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup work = boss;
```

在该模式中,boss和work是同一个Group,且这个Group的线程数是1。

2）单Reactor多线程模式

<pre><code>EventLoopGroup boss = new NioEventLoopGroup(1);
<strong>EventLoopGroup work =  new NioEventLoopGroup();</strong></code></pre>

在该模式中，boss线程池的线程数是1，work是默认的线程池。

3）多Reactor多线程模式

```java
EventLoopGroup boss = new NioEventLoopGroup(); 
EventLoopGroup work = new NioEventLoopGroup();
```

&#x20;该模式是Netty推荐的模式。其架构图如下：

<figure><img src="../.gitbook/assets/image (26).png" alt=""><figcaption></figcaption></figure>

&#x20;       当客户端请求到达时，Boss Group主要用来accept请求，当客户端请求到来时，会首先创建Channel，然后在 NioServerSocketChannel 中触发 channelRead 事件传播，NioServerSocketChannel 中包含了一种特殊的处理器 ServerBootstrapAcceptor，最终通过 ServerBootstrapAcceptor 的 channelRead() 方法将新建的客户端 Channel 分配到 WorkerEventLoopGroup 中。可以看到Boss的角色就是Reactor；WorkGroup用来进行I/O事件的处理。Boss Group和WorkGroup本质都是NioEventLoopGroup，它内部维护了一个线程池，由多个EventLoop组成。而EventLoop除了IO处理外，还负责定时任务以及系统的Task。

单个NioEventLoop的执行示意图如下：

<figure><img src="../.gitbook/assets/image (25).png" alt=""><figcaption></figcaption></figure>

1、Select负责轮询所有的就绪事件；

2、如果有就绪的事件会通过processSelectedKeys执行I/O事件；

3、runAllTasks负责执行一些异步任务；    &#x20;

说到这里，要提到Netty的一个著名特性，即解决了JAVA NIO的空轮询导致的CPU被打满的问题。

&#x20;      空轮询产生原因：Netty默认使用的I/O多路复用机制是epoll，epoll对于突然中断的连接socket会对返回的eventSet事件集合置为EPOLLHUP或者EPOLLERR，此时eventSet集合会发生变化，导致select被唤醒，而此时并没有真正的事件需要执行，从而导致空轮询。

**epoll感兴趣的事件集合**

| 符号           | 描述                                                                     |
| ------------ | ---------------------------------------------------------------------- |
| EPOLLIN      | 表示对应的文件描述符可以读（包括对端SOCKET正常关闭)                                          |
| EPOLLOUT     | 表示对应的文件描述符可以写；                                                         |
| EPOLLPRI     | 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；                                     |
| EPOLLERR     | 表示对应的文件描述符发生错误；                                                        |
| EPOLLHUP     | 表示对应的文件描述符被挂断；                                                         |
| EPOLLET      | 将 EPOLL设为边缘触发(Edge Triggered)模式（默认为水平触发），这是相对水平触发(Level Triggered)来说的。 |
| EPOLLONESHOT | 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socke的话，需要再次把这个socket加入到EPOLL队列里         |

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

&#x20;       Netty的解决方案是使用一个计数器，记录无效的轮询次数，当在单个周期内，如果技术超过一定阈值（默认是512），会重新创建selector对象，并重新将SelectionKey注册到selector对象。

```java
 if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
            // The selector returned prematurely many times in a row.
            // Rebuild the selector to work around the problem.
            logger.warn("Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                    selectCnt, selector);
            rebuildSelector();
            return true;
        }
```

&#x20; &#x20;

&#x20;       接下来看一下Dubbo中的netty使用场景。Dubbo在启动Provider时，会创建一个NettyServer（默认是netty)，看下源码：&#x20;

```java
 protected void doOpen() throws Throwable {
        bootstrap = new ServerBootstrap();

        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        // FIXME: should we use getTimeout()?
                        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // bind
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();

    }
```

&#x20;    Dubbo采用的就是单Reactor多线程的模型，bossGroup的线程池的数量是1，workGroup是一个IO线程池。

&#x20;     当服务端收到客户端的请求时，首先Dubbo首先解码数据，解码过程不说了，可以参考官方文档，解码后会封装一个Request对象，然后将请求发送至DisPatcher分发调度器，然后分发器会将请求分发至线程池，线程池负责执行具体任务。

![](http://blog-1251509264.costj.myqcloud.com/dispatcher-location.jpg)

Netty4选择的默认分发策略是all，即将所有消息都分发到业务线程池中执行。

```
 public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, "DubboServerHandler")));
    }


 protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
        return new MultiMessageHandler(new HeartbeatHandler(((Dispatcher)ExtensionLoader.getExtensionLoader(Dispatcher.class).getAdaptiveExtension()).dispatch(handler, url)));
    }
```

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

### 2、高性能之道

&#x20; **高效的并发编程**

&#x20;    Reactor多线程模型

&#x20;    CAS的使用

&#x20;    volatile的使用；

&#x20;    读写锁提升读写性能

&#x20; **零拷贝**

&#x20;  零拷贝是提高I/O效率的比较好的方案，之前我也写过相关的问题。零拷贝技术包括sendfile,mmap等等。Netty也使用了零拷贝技术，通过transferTo，也就是sendfile进行文件传输。

&#x20;**对象池**

&#x20; 通过对象池避免对象频繁的创建或销毁

**无锁化的串行执行**

&#x20;线程内部的handler串行执行，不需要加索，但可以多个线程并行的执行。

