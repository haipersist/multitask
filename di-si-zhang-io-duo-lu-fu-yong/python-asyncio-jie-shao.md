# Python asyncio介绍

自从Python3.4.开始真正引入了异步的概念。之前我也只是看过简单的应用，没仔细研究。最近自己在写商城的时候，写到人工客服，用到了Django channels，里面用到了大量的异步IO操作（其实里面的服务器最终还是依赖于Twisted）。并发的开发之简便基本等同于golang了（当然和golang比，还是差远了。且python的GIL值得吐槽，不过对于IO密集型的任务，还是有效果滴）。

说实话，Netty的思想和asyncio还差不多，不知道是否也是借鉴了，采用的线程模型都是事件驱动模型，即Reactor。当然,asyncio也有Proactor模型。下面是一个事件驱动模型示意图：

![](http://blog-1251509264.costj.myqcloud.com/eventdrivers.jpeg)

Netty中的基本思想就是一个主线程负责轮询accept外部请求（BossGroup NioEventLoop)，并通过chanel注册到子线程WorkGroup的NioEventLoop。workGroup负责处理IO事件。当然Netty的主从线程都是在同一个线程池里的，用ChannelFuture进行结果收集。

其实asyncio差不多也是这个意思。

它的核心组件是事件循环、协程、任务和Future对象。

事件循环就是不停地监听着事件，如果有新事件过来，那么就会将协程封装成Task，并注册到事件循环中。通过Task可以异步获取执行后的状态和结果。在Task中我们可以加入回调函数。当事件执行完毕后，通过绑定回调函数，可以在执行完毕后自动执行回调函数。其实这个Task也是Future对象的子集。

Task有点像Netty中的Channel.我现在越来越觉得Python的异步IO的设计充分借鉴了Netty的思想。

和Netty不同的主从线程Reactor模型不同的是，asyncio默认使用单线程Reactor模型。其实也可以实现多线程的，但其有GIL，也没有什么意义。后面再说。

一个近乎原生协程的最简单的例子：

```python
       async def main():
            await asyncio.sleep(1)
            print('hello')

        asyncio.run(main())
```

上面的代码就实现了异步编程，async表示该函数是一个协程对象；await表示在可能出现阻塞的地方挂起。入口就是asyncio.run。

看一下其入口代码：

```
def run(main, *, debug=False):
    """Execute the coroutine and return the result.

    This function runs the passed coroutine, taking care of
    managing the asyncio event loop and finalizing asynchronous
    generators.

    This function cannot be called when another asyncio event loop is
    running in the same thread.

    If debug is True, the event loop will be run in debug mode.

    This function always creates a new event loop and closes it at the end.
    It should be used as a main entry point for asyncio programs, and should
    ideally only be called once.

    Example:

        async def main():
            await asyncio.sleep(1)
            print('hello')

        asyncio.run(main())
    """
    #当前线程必须只有一个事件循环执行
    if events._get_running_loop() is not None:
        raise RuntimeError(
            "asyncio.run() cannot be called from a running event loop")

    #这必须是个协程函数或者协程对象。对象就是通过Task(Futrue的子集)封装的协程函数。
    if not coroutines.iscoroutine(main):
        raise ValueError("a coroutine was expected, got {!r}".format(main))

    #创建了事件循环对象。
    loop = events.new_event_loop()
    try:
        events.set_event_loop(loop)
        loop.set_debug(debug)
        #真正的执行函数。字面意思就是执行完就停止运行了。最后事件循环是必须终止的
        return loop.run_until_complete(main)
    finally:
        try:
            _cancel_all_tasks(loop)
            loop.run_until_complete(loop.shutdown_asyncgens())
        finally:
            events.set_event_loop(None)
            loop.close()
```

看到上面的main就是我们传入的协程函数或者协程对象。如果传入的是协程函数，它会再将其封装成协程对象Task。我们可以将main函数通过asyncio.create\_task封装成Task对象。

它的事件循环启动程序：

```python
 class BaseEventLoop(events.AbstractEventLoop):

   def __init__(self):
        ...
        # 存放 ready 状态 callbacks 的队列
        self._ready = collections.deque()
        # 存放通过 call_later 等方法注册的 callbacks 的优先队列
        self._scheduled = []
        ...

    def run_forever(self):
        """Run until stop() is called."""
        ...

        try:
            ...
            while True:
                self._run_once()
                if self._stopping:
                    break
        finally:
            ...
```

上面的\_ready非常重要，用于存储所有要执行的callback。

这样一个事件循环就启动了。它接下来要做的工作就是：

1、根据selector或者(proactor) 选择就绪的IO事件；

event\_list = self.\_selector.select(timeout)

selector可以选择，如果使用默认的，Python会根据操作系统帮助你选择和是的Selector。这个地方实际上就是调用操作的系统IO多路复用机制。

可以看以下Epoll的实现：

```python
 def select(self, timeout=None):
            if timeout is None:
                timeout = -1
            elif timeout <= 0:
                timeout = 0
            else:
                # epoll_wait() has a resolution of 1 millisecond, round away
                # from zero to wait *at least* timeout seconds.
                timeout = math.ceil(timeout * 1e3) * 1e-3

            # epoll_wait() expects `maxevents` to be greater than zero;
            # we want to make sure that `select()` can be called when no
            # FD is registered.
            max_ev = max(len(self._fd_to_key), 1)

            ready = []
            try:
                fd_event_list = self._selector.poll(timeout, max_ev)
            except InterruptedError:
                return ready
            for fd, event in fd_event_list:
                events = 0
                if event & ~select.EPOLLIN:
                    events |= EVENT_WRITE
                if event & ~select.EPOLLOUT:
                    events |= EVENT_READ

                key = self._key_from_fd(fd)
                if key:
                    ready.append((key, events & key.events))
            return ready
```

2、处理事件。

```
self._process_events(event_list)
```

```python
    def _process_events(self, event_list):
        for key, mask in event_list:
            fileobj, (reader, writer) = key.fileobj, key.data
            #读就绪
            if mask & selectors.EVENT_READ and reader is not None:
                if reader._cancelled:
                    self._remove_reader(fileobj)
                else:
                    self._add_callback(reader)
            #写就绪
            if mask & selectors.EVENT_WRITE and writer is not None:
                if writer._cancelled:
                    self._remove_writer(fileobj)
                else:
                    self._add_callback(writer)
```

不管是读就绪还是写就绪，它都是执行\_add\_callback方法，最终它调用的是

```
  self._ready.append(handle)  即将其加入到_ready双端队列中。
```

3、执行\_ready队列的callback。这个\_ready包含的包括selector监听的读写事件，还有我们自定义的异步任务以及\_scheduled中已经准备好的。 其实在Golang中也使用了队列，也有一个本地运行队列和全局运行队列，主要是存储运行中的协程G。M就相当于Python这里面的loop。当然了在Python中并没有对应的P,。

上面基本上把准备旧续的时间都加入到了队列中，后续就要去执行了：

```
  ntodo = len(self._ready)
        for i in range(ntodo):
            handle = self._ready.popleft()
            if handle._cancelled:
                continue
            if self._debug:
                try:
                    self._current_handle = handle
                    t0 = self.time()
                    handle._run()
                    dt = self.time() - t0
                    if dt >= self.slow_callback_duration:
                        logger.warning('Executing %s took %.3f seconds',
                                       _format_handle(handle), dt)
                finally:
                    self._current_handle = None
            else:
                handle._run()
        handle = None  # Needed to break cycles when an exception occurs.
```

那么我们自定义的任务是什么时候被加入的呢?其实就是我们在之前创建task的时候。Task类在初始化的时候会调用一个方法叫做call\_soon，它负责把其加入到事件循环中的\_ready队列中。然而，它并不是简单地将异步任务加入到队列中，而是传入Task内部实现的\_step方法。这个\_step方法就是异步实现的核心。上代码：

```python
  def __step(self, exc=None):
        if self.done():
            raise exceptions.InvalidStateError(
                f'_step(): already done: {self!r}, {exc!r}')
        if self._must_cancel:
            if not isinstance(exc, exceptions.CancelledError):
                exc = exceptions.CancelledError()
            self._must_cancel = False
        coro = self._coro
        self._fut_waiter = None

        _enter_task(self._loop, self)
        # Call either coro.throw(exc) or coro.send(None).
        try:
            if exc is None:
                # We use the `send` method directly, because coroutines
                # don't have `__iter__` and `__next__` methods.
                result = coro.send(None)
            else:
                result = coro.throw(exc)
        except StopIteration as exc:
            if self._must_cancel:
                # Task is cancelled right before coro stops.
                self._must_cancel = False
                super().cancel()
            else:
                super().set_result(exc.value)
        except exceptions.CancelledError:
            super().cancel()  # I.e., Future.cancel(self).
        except (KeyboardInterrupt, SystemExit) as exc:
            super().set_exception(exc)
            raise
        except BaseException as exc:
            super().set_exception(exc)
        else:
            blocking = getattr(result, '_asyncio_future_blocking', None)
            if blocking is not None:
                # Yielded Future must come from Future.__iter__().
                if futures._get_loop(result) is not self._loop:
                    new_exc = RuntimeError(
                        f'Task {self!r} got Future '
                        f'{result!r} attached to a different loop')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)
                elif blocking:
                    if result is self:
                        new_exc = RuntimeError(
                            f'Task cannot await on itself: {self!r}')
                        self._loop.call_soon(
                            self.__step, new_exc, context=self._context)
                    else:
                        result._asyncio_future_blocking = False
                        result.add_done_callback(
                            self.__wakeup, context=self._context)
                        self._fut_waiter = result
                        if self._must_cancel:
                            if self._fut_waiter.cancel():
                                self._must_cancel = False
                else:
                    new_exc = RuntimeError(
                        f'yield was used instead of yield from '
                        f'in task {self!r} with {result!r}')
                    self._loop.call_soon(
                        self.__step, new_exc, context=self._context)

            elif result is None:
                # Bare yield relinquishes control for one event loop iteration.
                self._loop.call_soon(self.__step, context=self._context)
            elif inspect.isgenerator(result):
                # Yielding a generator is just wrong.
                new_exc = RuntimeError(
                    f'yield was used instead of yield from for '
                    f'generator in task {self!r} with {result!r}')
                self._loop.call_soon(
                    self.__step, new_exc, context=self._context)
            else:
                # Yielding something else is an error.
                new_exc = RuntimeError(f'Task got bad yield: {result!r}')
                self._loop.call_soon(
                    self.__step, new_exc, context=self._context)
        finally:
            _leave_task(self._loop, self)
            self = None  # Needed to break cycles when an exception occurs.

    def __wakeup(self, future):
        try:
            future.result()
        except BaseException as exc:
            # This may also be a cancellation.
            self.__step(exc)
        else:
            # Don't pass the value of `future.result()` explicitly,
            # as `Future.__iter__` and `Future.__await__` don't need it.
            # If we call `_step(value, None)` instead of `_step()`,
            # Python eval loop would use `.send(value)` method call,
            # instead of `__next__()`, which is slower for futures
            # that return non-generator iterators from their `__iter__`.
            self.__step()
        self = None  # Needed to break cycles when an exception occurs.
```

上面代码看着挺吓人的。首先是判断Future是否已经完成或者取消。否则执行coro.send()，这个会执行到遇到yield为止。如果没有执行完，会继续将其放入到队列中，反复执行，直到最后完成。

对于每一个协程对象来说，使用await函数的具备下面特性：

```python
  def __await__(self):
        if not self.done():
            self._asyncio_future_blocking = True
            yield self  # This tells Task to wait for completion.
        if not self.done():
            raise RuntimeError("await wasn't used with future")
        return self.result()  # May raise too.

    __iter__ = __await__  # make compatible with 'yield from'.
```

如果没有完成，就会yield self。否则就返回结果。

其实，很多成熟的框架或服务都使用了Reactor模式，除了上面说的Netty，还有Redis也是。Redis的事件分为文件事件和时间事件。不过它不同的是IO操作是顺序执行的。

下面来一个异步编程实战：

```
async def start(url):
    async with aiohttp.ClientSession() as session:
        resp = await session.get(url)
        resp = await resp.text(encoding='gb18030')
        parser(resp)

urls = ['http://bang.dangdang.com/books/bestsellers/01.00.00.00.00.00-recent7-0-0-1-%d'%i for i in range(1,20)]



# 统计该爬虫的消耗时间
print('#' * 50)
t1 = time.time() # 开始时间
async def main():
    tasks = []
    for url in urls:
        rask = asyncio.create_task(start(url))
        tasks.append(rask)

    for task in tasks:
        await task

asyncio.run(main())

print(table)

t2 = time.time() # 结束时间
print('使用aiohttp，总共耗时：%s' % (t2 - t1))
print('#' * 50)
```
