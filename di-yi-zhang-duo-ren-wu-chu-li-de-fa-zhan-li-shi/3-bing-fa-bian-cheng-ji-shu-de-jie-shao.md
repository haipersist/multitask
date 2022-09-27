# 3、并发编程技术的介绍

**操作系统的并发**

在《深入理解计算机系统》这本书中，对于迸发的定义是如果逻辑控制流在时间上重叠，则可称之为迸发。如果学过操作系统原理的课程，应该可以知道，操作系统会通过不同的策略来运行多个应用。

在应用级，作为开发者，可以开发出迸发的程序。目前主要包括多进程、多线程、IO多路复用以及协程，其中前三个都是操作系统提供的功能，最后一个协程是在应用级别来实现的，其并不是操作系统本身提供的构造。

对于多任务技术，我们知道需要在不同任务间切换，那就涉及到调度的问题以及当前任务现场保护的问题。

**系统调度**

那当我们要从就绪队列中选择进程时是通过什么原则呢？这时候就需要CPU完成调度。

至于调度的时机，即该何时去选择，以及选择哪个进程去执行。这还要取决于当前系统是抢占式还是非抢占式系统。

非抢占式系统：当前进程会主动放弃CPU；

抢占式系统：

中断请求时，会将当前进程转成就绪状态。那通常情况下，如果时间片轮转，就会在时间片用完了，会被抢占。或者有一个进程从等待变就绪了，且其更急迫需要执行，就会抢占当前进程。

CPU的调度最重要的目标是能够提供最大的吞吐量以及CPU的利用率。

目前存在的几种CPU调度算法：

先来先服务算法；

短进程优先算法；

最高响应比优先算法；

时间片轮转算法；

多级反馈队列算法；

公平共享调度算法；

**先来先服务算法FCFS：**

在就绪队列中，先进入的进程会先被执行。看下面的队列：

![IMG\_256](../.gitbook/assets/1)

上面的队列，P1执行时间24，P2执行时间3，到达时间24，P3执行时间3，到达时间24.

上面队列的平均周转时间（24+27+30)/3。

看上面的例子，P1是一个执行时间很长的进程，在这种算法下，他会是先执行的，这就导致整个队列的平均周转时间可能比较长，这取决于长进程排在哪里，也就是周平均等待时间波动比较大。

另外一个问题是IO资源和CPU资源的利用率比较低。假如现在有一个CPU密集型任务的长进程正在执行，此时IO资源是空闲的，但是呢这种算法下，IO资源也并没有利用上，即使后面有IO密集型任务，也是只能干等着。这是一种非抢占式的算法。

**短进程优先算法：**

他是对FCFS的一种改进，它会优先执行短进程的，就绪队列会按照时间来排序。从而缩短了平均周转时间。

但怎么选择是一个问题，每个进程预计执行时间是多少呢？它要么希望用户告知，要么会通过某种算法预估执行时间。

上面是一个最优排序的算法，可以保证平均等待时间最短。

此外，短进程优先有一种算法改进，即加入当前有个进程执行，这时如果又到来一个进程，它执行时间要比正在执行进程剩余时间还短，那么它可以进行抢占。

那它有一个缺点，就是就绪队列不断地进入进程，由于它会把短的排在前面，可能不断地有短进程进入，就导致长进程永远得不到执行，导致长进程饥饿。这是一个问题。

**最高响应比优先算法：**

基于上面说的饥饿的问题，又有一种改进算法叫最高相应比优先算法。

相应比计算公式 rate = (w+s)/s， w表示等待时间，s表示预估执行时间。等待时间越长，rate越来越大，这样就可以避免长进程长时间等待出现饥饿的问题。该算法不允许抢占。

**时间片轮转算法：**

时间片是分批处理资源的基本执行单元。时间片轮转是为每个进程分配相同时间片，然后通过时钟中断，完成进程切换。

时间片轮转算法最重要的是时间片大小的设置。

设置太大，可能就变成先进先服务了。设置太小，可能会引起频繁的进程上下文切换。基于经验，一般都是设置成进程切换的1%.

**多级反馈队列算法**

把就绪队列分成多个独立的子队列，每个队列可以有不同的优先级，时间片的大小根据队列的优先级设置不同值。比如CPU密集型地可以放在低优先级，IO密集型地可以放在高优先级队列。进程可以放到不同的队列中，比如某个进程在一个队列的时间片内没有执行完，会把该进程放到低优先级队列中。

**公平共享调度算法FSS**

该算法会控制用户对资源的访问，可能让更重要的用户访问资源。

**多处理机调度**

目前计算机都是多处理器，因此都是多CPU组成的多处理器系统。在多处理器系统中，通常都是每个处理器有自己的调度程序，然后在访问共享资源时会进行同步。目前存在的算法：

**1、静态资源算法**

一个进程从开始到结束都在一个固定处理器，每个处理器有自己的就绪队列。这种算法开销小，但可能导致调度不均衡。

**2、动态进程分配**

每个进程在运行时可分配到各个处理器上，所有CPU共享一个就绪队列。这种做法开销是比较大的，因为它每次都要选择到哪个处理器上，但它却可以实现负载均衡。

**任务切换**

除了系统调度之外，另外一个重要概念就是当前任务的现场保护。

就说线程上下文切换，CPU在处理多任务时，需要不断地完成不同的线程之间的调度，而每个线程都会有自己独立的上下文环境（程序计数器，寄存器，栈等等）。当某种因素引起了切换，就要执行相应过程、

通常引起线程的上下文切换的因素主要包括：

当前执行任务的时间片用完之后，系统CPU正常调度下一个任务；

当前执行任务碰到IO阻塞，调度器将此任务挂起，继续下一任务；

多个任务抢占锁资源，当前任务没有抢到锁资源，被调度器挂起，继续下一任务；

用户代码挂起当前任务，让出CPU时间；

硬件中断；

无论是哪种方式引起的切换，首先要做的就是现场保护，主要向寄存器，程序计数器等等内容。只是进程间的切换要比线程切换的代价更大。

在Linux中，可以用vmstat来统计上下文切换的次数。

vmstat主要是用来在给定时间间隔收集当前机器的一些性能指标，比如CPU的利用率，线程切换次数，内存使用情况等等。

![](<../.gitbook/assets/2 (1)>)

当然上下文的切换也不仅仅是局限于进程或者线程间的切换。由于我们应用程序都是通过系统调用来访问硬件资源的，所以还会涉及到用户态到内核态，以及内核态到用户态的上下文切换，这里就不再细说了，感兴趣的可以看我写过的Linux零拷贝技术讲解。

**多进程**

多进程是最容易实现的一种并发编程。通过多个进程来同时完成一个任务，由内核来调度和执行。

比如有个C/S场景，多个客户端请求服务器获取相应的资源。如果是非多进程的，可能是如图所示：

![](../.gitbook/assets/3)

此时，客户端2必须要等Server处理完客户端1的请求才能继续得到服务器的响应。如果在当我们服务端开了多个进程来服务客户端，整个系统的性能上就会有一定的提升。接着上个图，在Server端再fork出一个子进程，如图所示：

![](../.gitbook/assets/4)

由于不同进程使用不同的虚拟地址空间，导致进程间需要通过显示的方式实现进程间通信。

**多线程**

多线程，这应该是很多程序员看到最多的概念，因为任何一个JAVA程序员都会或多或少接触到。在JAVA中，多线程技术的应用得淋漓尽致。由于其是进程内的资源，多个线程可以共享同一个内存地址空间，导致其进行线程间切换所付出的代价更小。但也正因为共享同一个地址空间，也带来了很多安全性的问题以及性能问题。因此，多线程相对于多进程来说，其复杂度要更高。在JAVA中，你可以看到各种因为线程安全性设置的变量、锁以及同步对象。还有出于性能考虑而设计的各种线程池等等。在第三章节中会详细介绍。

**IO多路复用**

一个IO请求可能包括数据准备，数据拷贝和处理。通常情况，IO操作是很费时的。如果一个请求在准备阶段阻塞住了，应该再处理其他的请求，直到其数据准备好。

IO多路复用是在单个进程上下文完成的。内核一旦发现进程内的一个或者多个IO条件准备读取，它就通知该进程去处理。当然触发机制有很多种，比如select,poll,epoll等。在第四章节有详细介绍。

在这种并发编程中，应用程序在一个进程的上下文进行调度，所有的流共享同一套地址空间。

与多进程、多线程相比，不必做进程或线程的创建和切换，减少了很多不必要的开销。在实际应用中，IO多路复用的使用也比较广泛，诸如Redis,Netty以及Python3的异步IO也是基于IO多路复用实现的。可见，其性能上所具备的无可比拟的优势。当然，通常情况，其也要配合着多线程一起使用，比较经典的Reactor模型，在第四章节会详细讲述。

**协程**

最早接触协程的概念，是我在写Python2时，那时有yield,send等基于生成器的协程，还有gevent库。后来跳槽到了小米，接触了Golang，才算真正认识了协程。

这种并发技术并不是操作系统直接支持的，而是开发者在应用中实现的。当发生阻塞时，主动让出CPU，从而实现在不同的逻辑流进行切换。

目前协程在很多新兴势力中得到较大范围的应用，golang就是其中之一，协程是其唯一的并发编程技术，相对于其他并发技术，协程更加简洁、高效。在第五章会对协程有更详细的介绍。

**总结**

迸发编程带来性能提升的同时，也引出了其他问题，或者说我们需要重点关注并解决的问题，其中安全性是最关键的问题，对于全局或者共享变量可能要考虑其安全性。比如在多线程中，线程安全性是最重要的课题。当有多个线程同时修改同一个变量时，如何保证所得到的结果是可预见的、正确的就是线程安全性的问题了，这个后续会详细说明。

在迸发编程中，还有其他很多的问题。比如多进程可能要考虑的是进程间通信、死锁等等。