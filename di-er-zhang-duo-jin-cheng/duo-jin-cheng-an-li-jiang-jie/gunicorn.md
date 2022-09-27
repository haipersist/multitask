# Gunicorn

&#x20;      如果你是做Python开发的，对Gunicorn一定很熟悉。其是在Unix环境下使用的WSGI HTTP Server。WSGI全称是Python Web Server Gateway Interface，是一种接口规范，其本质是一种Web服务器和应用的通信协议。最早的时候是CGI，后来又衍生出FastCGI。WSGI是Python应用和Web服务器的通信协议，Gunicorn是一个具有高性能的Web服务器，类似于JAVA中的Tomcat。通常我们在部署Python应用时，都不会用框架自带的Server，比如Django，Flask，而这完全出于性能考虑。

&#x20;      Gunicorn和Nginx一样是采用了多进程模型，pre-fork，master也同样负责worker管理，worker用来进行请求处理。不过和Nginx不同的是，Gunicorn worker实现了多种处理模型，包括多线程、协程、IO多路复用、异步等方式处理请求。

![](<../../.gitbook/assets/9 (1)>)

上面的stat中的l表示的是多线程。

&#x20;      在实际应用场景中，Nginx和Gunicorn会配合使用，各司其职，Nginx是反向代理的角色，可以做静态资源代理，可以实现负载均衡，而Gunicorn充当的就是Web Server的角色，解析Http请求，调用Python应用处理请求，返回请求。

![IMG\_256](<../../.gitbook/assets/10 (1)>)

接下来一起看下gunicorn的源码，看他是如何实现pre-fork的。

gunicorn的启动入口在工程目录下arbiter.py中的run方法，我保留了其中比较重要的部分。

```
def run(self): 
#这里面就是启动了master进程。 
     self.start()  

   #管理进程
 self.manage_workers()  
  
#随后无限循环，主要用来处理外界信号。
    while True:  
        self.maybe_promote_master()  
  
        sig = self.SIG_QUEUE.pop(0) if self.SIG_QUEUE else None  
        if sig is None:  
            self.sleep()  
            self.murder_workers()  
            self.manage_workers()  
            continue  
  
        signame = self.SIG_NAMES.get(sig)  
        handler = getattr(self, "handle_%s" % signame, None)  
      
        handler()  
```

在上面的代码中，看下几个重要的操作。

1、start()用来设置主进程信息

```python
def start(self):
    """\
    Initialize the arbiter. Start listening and set pidfile if needed.
    """
    self.log.info("Starting gunicorn %s", __version__)
    if 'GUNICORN_PID' in os.environ:
        self.master_pid = int(os.environ.get('GUNICORN_PID'))
        self.proc_name = self.proc_name + ".2"
        self.master_name = "Master.2"
    #初始化信号        
    self.init_signals()

```

上面的init\_signal是用来初始化信号的，Master 和worker 之间的通信是通过信号来进行的。Master接受到的信号都会放到一个队列中去。一旦队列满了，就不在对信号做出任何反应。

2、manage\_workers方法就是用来实际管理worker的，我们往下看：

```
def manage_workers(self):  
      #数量小于配置的num，就创建worker
       if len(self.WORKERS) < self.num_workers:  
           self.spawn_workers()  
  
       workers = self.WORKERS.items()  
       workers = sorted(workers, key=lambda w: w[1].age)  
#数量大于配置的num，就kill掉相对老的
       while len(workers) > self.num_workers:  
           (pid, _) = workers.pop(0)  
           self.kill_worker(pid, signal.SIGTERM)  
```

可以看到，上面的主要目的就是控制当前的worker数量。

3、spawn\_worker就是创建子进程用的。

```
// Some code
def spawn_worker(self):  
      self.worker_age += 1  
      worker = self.worker_class(self.worker_age, self.pid, self.LISTENERS, 
                                 self.app, self.timeout / 2.0,  
                                 self.cfg, self.log)  
#提前fork出子进程
      self.cfg.pre_fork(self, worker)  
      pid = os.fork()  
#这里主要是做一些worker的初始化，具体做什么还要取决于用了哪种worker
      worker.init_process()  
```



创建worker进程后，会经历如下过程：

* 首先会调用init\_process方法，即初始化进程，包括创建管道；
* 然后会加载定义的应用，是你django，flask或其他框架写的app;
* 执行run方法，开始在指定socket监听；
* 一旦有连接后会调用handle方法处理请求。

&#x20;     Gunicorn通过多个Worker处理请求，而master本身不处理任何的客户端请求，只用于管理worker。实际上Worker内部也并不是单线程阻塞处理请求，Gunicorn内部实现了多个不同类型的worker，支持IO多路复用、异步IO、多线程等等。



```
SUPPORTED_WORKERS = {    
    "sync": "gunicorn.workers.sync.SyncWorker",  
    "eventlet": "gunicorn.workers.geventlet.EventletWorker",  
    "gevent": "gunicorn.workers.ggevent.GeventWorker",   
    "gevent_wsgi": "gunicorn.workers.ggevent.GeventPyWSGIWorker",  
    "gevent_pywsgi": "gunicorn.workers.ggevent.GeventPyWSGIWorker",  
    "tornado": "gunicorn.workers.gtornado.TornadoWorker", 
    "gthread": "gunicorn.workers.gthread.ThreadWorker",
}
```

