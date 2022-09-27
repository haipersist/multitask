# 高性能网络I/O模式

&#x20;      目前的应用系统动辄上万、百万QPS，而我们的系统基本上都是I/O密集型应用，因此对于性能有较高的系统来说，选择较好的网络I/O模式尤为重要。网络I/O模式主要分为两种，一个是Reactor模式，是主流的使用模式；一个是Proactor模式。



### 一、Reactor模式

主要分为以下几种：

&#x20; 单Reacor单线程;

&#x20; 单Reactor多进程/线程；

&#x20;多Reactor多进程/线程；



#### 1、  单Reacor单线程



![](http://blog-1251509264.costj.myqcloud.com/reactorsinglethread.png)

上面示意图包括三个对象：

* Reactor
* Acceptor
* Handler   黄色部分，业务处理

&#x20;      Reactor对象通过select（可能是poll,epoll等函数）不断监听事件，收到事件后会进行dispatch分发；

&#x20;     当有新请求过来会分给acceptor并调用accept创建handler做后续处理；

&#x20;     当不是新请求，就直接分给对应的Hander进行处理。

&#x20;      通过多路复用，可以保证同时处理多个读写事件，尤其是epoll机制性能更好，由于其是单进程/线程，开发更简单，因此应用得比较广泛。但该模式最大的缺点就是无法充分利用多核CPU，因为其是单进程线程的，这就导致在handler在执行时，同时只能有一个handler执行，其他的handler需要阻塞等待。

目前Redis采用的是该模式。

Redis服务器中有两类事件，文件事件和时间事件。

* 文件事件（file event）：Redis客户端通过socket与Redis服务器连接，而文件事件就是服务器对套接字操作的抽象。例如，客户端发了一个GET命令请求，对于Redis服务器来说就是一个文件事件。Redis的协议本身发送的是文本。
* 时间事件（time event）：服务器定时或周期性执行的事件。例如，定期执行RDB持久化。

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

&#x20;    上面说到了单Reactor单线程得模式得缺点是无法利用多核，为什么Redis还要选择该模式呢？为什么性能依旧很强劲？单机QPS都可以达到10W+呢？

&#x20;     其实这还要得益于其全部都在内存中操作，不需要经过外部存储。此外，该模式正符合Redis追求的命令要串行执行的需求。

&#x20;     不过要补充一点，Redis4.0开始也引入了多线程。主要是为了解决一些耗时较长的一些操作，从而提高性能。比如删除大key啊。涉及到的命令有Unlink,FlushAll Async,FlushDB Async.从名字也可以看到，这些命令在执行时，会开启一个异步线程去做处理，而不会长时间阻塞到当前线程。

&#x20;    此外，在Redis6.0中也引入了IO多线程。

&#x20;      此外Nginx采用的是多进程+Reactor单线程的方式来处理的，它自己实现了一个Master进程+多个Worker进程，从而实现并发。

&#x20;      网上很多人都说Nginx是多Reactor多进程的模式，看着有些类似，Master是主Reactor，Worker是从Reactor，但实际上Nginx的多进程和多Reactor不是一个概念。因为主进程不会accept连接，而是由子进程的reactor来accept连接，通过互斥锁来控制一次只有一个子进程accept，避免惊群效应，子进程accept成功后就放到自己的reactor进行处理。Nginx的Master是为了管理这些Worker的。它只是将单Reactor单线程，fork了多个进程而已。

#### 2、单Reactor多进程/线程

![](http://blog-1251509264.costj.myqcloud.com/ioreactormulthread.png)

&#x20;     实际上，主要还是Reactor多线程模式，多进程几乎不怎么常用，因为多进程需要创建更多的资源，需要实现进程间通信。

&#x20;     多线程的Reactor也是只有一个，负责请求的接收和分发。多线程版本和单线程的区别是将实际上做处理业务的非IO操作的业务，如编解码，计算等，放到其他线程处理，这样能充分发挥多核的作用，也避免因为Handler的执行阻塞其他业务的处理。

&#x20;    虽然多线程版本能发挥多核的作用，但也由此带来数据竞争的问题，对于共享数据，并发处理是存在竞争问题，可能需要通过加锁来保证数据的线程安全性。当然，这也是多线程比较常见的问题，或者根本称不上是问题，保证线程安全性有非常成熟的方案了。

&#x20;     单Reactor多线程的模式相比于单Reactor单线程的模式，在处理业务数据时可以通过并发的方式提高执行效率。但由于Reactor只有一个，所有请求都要经过该Reactor，因此Reactor容易成为整个系统性能的瓶颈。基于此，又衍生出了另外一个模式：多Reactor多线程。

#### 3、多Reactor多线程

&#x20;      要解决单Reactor多线程模式的瓶颈，一个直白的做法就是增加Reactor，也就是接下来要介绍的模型。

<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

&#x20;    该模式的处理流程如下：

&#x20;     1、主线成的主Reactor通过select监听客户端的连接事件，当有新的连接事件时，会调用accept函数获取连接，并将连接分配给子Reactor处理；

&#x20;    2、subReactor接收到连接后，通过select监听就绪的读写事件，并在开始时创建Handler用于处理具体的业务；

&#x20;    3、当有读写事件就绪时，会调用对应的Handler函数；

&#x20;   4、Handler实际上会将业务丢到业务线程池中并发执行；

&#x20;   5、执行完后，handler将响应数据发送给客户端；

&#x20;    其中mainReactor和subRector都可以是个线程池，不一定只有一个线程。这种模式在性能上要优于上面的两种模式，因此很多高并发的框架都会使用该技术。比如著名的Netty，Netty可以称得上是JAVA并发界的佼佼者了，无人不知，无人不晓。下面是Netty的处理流程图：

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

上面的BossEventLoopGroup就是mainReactor，负责accept请求，并封装NioSocketChannel，将其注册到WorkGroup中的一个NioEventLoop上，且只能绑定NioEventLoop。因为NioEventLoop是线程隔离的，相互不影响。内部的事件都会串行的无锁化的执行。



&#x20;    EventLoop是一种事件驱动模型，在Reactor比较常用，即便是Python3中引入的asyncnio也是借助事件驱动机制来实现的。

<figure><img src="../.gitbook/assets/image (28).png" alt=""><figcaption></figcaption></figure>





### 二、Proactor模型



<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

&#x20;      该模式对应的就是在上一章节介绍的I/O模型中的异步IO，在上一章节已经详细介绍了异步I/O，本结不再赘述。不过要说的是Linux的异步I/O支持的不够友好，而真正实现异步I/O的是Windows，通过异步I/O接口IOCP实现，其是真正意义上的操作系统级别的异步实现方案。

&#x20;      目前很多框架所提到的异步也都是基于操作系统同步的方案来实现的异步，比如Netty，Nginx，这个异步是对于客户端来说的，虽然底层使用了epoll来进行处理，在执行时也需要同步等待数据拷贝过程，但此时服务端会先返回，比如Netty会返回一个ChannelFuture对象，只有当业务逻辑执行完之后，可以通过异步对象获取数据结果。
