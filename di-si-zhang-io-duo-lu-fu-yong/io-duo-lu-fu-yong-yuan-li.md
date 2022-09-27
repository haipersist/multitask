# I/O多路复用原理

&#x20;       在上一章节提到I/O多路复用的实现方式通过某种机制去监听所有的文件描述符。一旦某个或某些文件描述符就绪了，就会通知应用程序进行相应的读写操作。目前Linux实现的I/O多路复用机制包括select、poll、epoll以及kqueue等。

&#x20;      介绍I/O多路复用原理之前，不得不介绍一下文件描述符，因为其是多路复用的基础。

&#x20;      在Linux中，一切都是文件，目录是文件，块设备是文件，Socket也是文件。每个文件都会对应一个inode，inode内部维护了文件类型、访问权限、大小等文件元信息。当进程打开文件时，会在进程内部的文件描述符表中增加该文件描述符，此外系统级也会维护相应的文件描述符（File Descriptor)。后续所有对文件的操作都是基于文件描述符进行的。

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

### 1、Select机制

&#x20;     select的实现方式监听所有的文件描述符集合，通过select系统调用将所有监听的文件描述符拷贝到内核中检查就绪的文件描述符；随后将就绪的文件描述符再拷贝到用户空间，程序对其进行处理。

&#x20;    select 函数监视的文件描述符分3类，分别是读文件描述符、写描述符和异描述符。

&#x20; 调用select函数后会阻塞，直到有描述符就绪或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

readfd : 监视的可读描述符集合，只要有文件描述符即将进行读操作，这个文件描述符就存储到这。

writefds : 监视的可写描述符集合。

exceptfds : 监视的错误异常描述符集合

```
/**
 * select()系统调用
 *
 * 参数列表：
 *     nfds       - 值为最大的文件描述符+1
 *    *readfds    - 用户检查可读性
 *    *writefds   - 用户检查可写性
 *    *exceptfds  - 用于检查外带数据
 *    *timeout    - 超时时间的结构体指针
 */
int select（int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout）;
```

&#x20;    我们看下Python3中SelectSelector封装的高级APi:

```python
   
   def select(self, timeout=None):
        timeout = None if timeout is None else max(timeout, 0)
        ready = []
        try:
            //get file descriptor
            r, w, _ = self._select(self._readers, self._writers, [], timeout)
        except InterruptedError:
            return ready
        r = set(r)
        w = set(w)
        for fd in r | w:
            events = 0
            if fd in r:
                events |= EVENT_READ
            if fd in w:
                events |= EVENT_WRITE

            key = self._key_from_fd(fd)
            if key:
                ready.append((key, events & key.events))
        return ready
```

上面例子较清晰，即通过select调用获取读写文件描述符，并返回已就绪描述符集合。

### 2、Poll机制

&#x20;    Poll和Select一样都需要遍历文件描述符集合，并进行文件描述符的拷贝。但其最大的区别是其不再像select使用三个位图来表示三个fdset的描述符集合，其自定义了一个结构体pollfd，以链表的形式组织起来。

```c
/**
 * poll()系统调用
 *
 * 参数列表：
 *    *fds         - pollfd结构体
 *     nfds        - 要监视的描述符的数量
 *     timeout     - 等待时间
 */
int poll（struct pollfd *fds, nfds_t nfds, int *timeout）;
 
 
### pollfd的结构体
struct pollfd{
　int fd；// 文件描述符
　short event；// 请求的事件
　short revent；// 返回的事件
}
```

&#x20;   看下python3中PollSelector的实现：

```python
   def select(self, timeout=None):
        ready = []
        try:
            fd_event_list = self._selector.poll(timeout)
        except InterruptedError:
            return ready
        for fd, event in fd_event_list:
            events = 0
            if event & ~self._EVENT_READ:
                events |= EVENT_WRITE
            if event & ~self._EVENT_WRITE:
                events |= EVENT_READ
            key = self._key_from_fd(fd)
            if key:
                ready.append((key, events & key.events))
        return ready
```

&#x20;      由于poll机制使用链表，因此其没有select的描述符数量限制（当然，还是要受到系统最大描述符限制的）。不过其依然存在着select的性能问题。

### 3、epoll机制

&#x20;      随着描述符的增大，select和poll的性能会不断地下降，因为 每次调用都需要做一次从内核空间到用户空间的拷贝 。

&#x20;      相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，此外其 利用了mmap ，这样在用户空间和内核空间的copy只需一次。

&#x20;     &#x20;

```c
struct eventpoll {
　　...
　　struct rb_root rbr;

　　struct list_head rdllist;
　　...
};
```

rb\_root是一个红黑树，存储了所有的事件，可以理解为所有的连接；rdlist是一个链表，存储了所有已经就绪的事件。

```c
int epoll_create(int size);  
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);  
```

&#x20;     在程序启动时，其经历如下几步：

&#x20;  1、首先会调用epoll\_create创建一个epoll对象；

&#x20;  2、随后通过epoll\_ctl添加进来的事件红黑树中；

&#x20;  3、当调用epoll\_wait时会检查双向链表里是否有数据，如果有数据就执行内核到用户数据的拷贝。没有就sleep，等到了超时时间，没有数据也会返回的。

<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

简单看一下Netty中的EpollEventLoop，在初始化时会进行如下操作（简化了代码）：

{% code overflow="wrap" %}
```java
this.epollFd = epollFd = Native.newEpollCreate();
this.eventFd = eventFd = Native.newEventFd();
Native.epollCtlAdd(epollFd.intValue(), eventFd.intValue(), Native.EPOLLIN | Native.EPOLLET);
```
{% endcode %}

随后执行时会进入死循环，调用epollWait：

```java
    protected void run() {
        long prevDeadlineNanos = NONE;
        for (;;) {
            if (!hasTasks()) {
                if (curDeadlineNanos == prevDeadlineNanos) {
                    // No timer activity needed
                    strategy = epollWaitNoTimerChange();
                } else {
                    // Timerfd needs to be re-armed or disarmed
                    prevDeadlineNanos = curDeadlineNanos;
                    strategy = epollWait(curDeadlineNanos);
                }
        }
```

&#x20;      看到epoll的流程，可以看出，相比于select/poll，其处理起来更高效，主要得益于以下几点：

1、不必轮询所有的文件描述符，就绪的会被添加到链表中；

2、不必每次都做内存拷贝，select/poll要来回拷贝。当然不是说epoll没有拷贝，其也需要将就绪链表中的事件拷贝到用户空间的；

3、使用了红黑树的数据结构存储所有的待检测的文件描述符，查询更高效，且没有文件描述符限制（只要不超过系统级限制）。



&#x20;      正因为epoll的高效，其在实际应用中使用得更广泛，很多框架都使用了epoll机制，如Netty、Redis、Nginx甚至Python3得asyncio也是基于多路复用机制来实现的。

#### epoll的触发方式

&#x20;epoll支持水平触发和边缘触发两种触发模式。LT水平触发的意思就是只有fd有数据可发读操作；ET边缘触发是只有新数据到来，才会触发读，因为它每次是要所有数据都读取走的。对于ET来说，就算是缓冲区有未读尽的数据，也不会触发epoll\_wait返回的。边缘触发帮我解决了大量我们不关心的读写就绪文件描述符。
