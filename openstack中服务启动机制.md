# Karbor服务的启动

Karbor 依赖 evenlet 完成各种并发任务，它的进程可分为两类：  
1、 **WSGIService**: 接收和处理 http 请求，依赖 eventlet.wsgi 的 wsgi server 处理 http 请求，karbor-api 是唯一的 WSGIService 类型的进程  
2、 **Service**: 接收和处理 rpc 请求，非 karbor-api 以外的其它 karbor 进程，如 karbor-protection，karbor-operation等
无论是 WSGIService 还是 Service 类型的进程，每当接收到一个请求(http 或 rpc)，都会在线程池中分配一个协程处理该请求

## 一、WSGIService的启动
karbor-api 由 **karbor/cmd/api.py** 启动，它初始化一个 WSGIService(由 **karbor/service.py** 定义) 对象。
```python
def main():
    objects.register_all()
    CONF(sys.argv[1:], project='karbor',
         version=version.version_string())
    logging.setup(CONF, "karbor")

    rpc.init(CONF)
    launcher = service.get_launcher()
    server = service.WSGIService('osapi_karbor')
    launcher.launch_service(server, workers=server.workers)
    launcher.wait()
```
api中从service层获取一个启动器对象，最后将server对象传入启动器对象的launch_service方法中，launch_service（server, workers=server.workers）方法定义如下：
```python
class Launcher(object):
    def __init__(self):
        super(Launcher, self).__init__()
        self.launch_service = serve
        self.wait = wait
```
该方法被引用到serve方法，serve方法定义如下：
```python
def serve(server, workers=None):
    global _launcher
    if _launcher:
        raise RuntimeError(_('serve() can only be called once'))

    _launcher = service.launch(CONF, server, workers=workers)
```
最终调用了**oslo_service/service.py**下的launch方法，launch方法定义如下：
```python
def launch(conf, service, workers=1, restart_method='reload'):
    …

    if workers is not None and workers <= 0:
        raise ValueError(_("Number of workers should be positive!"))

    if workers is None or workers == 1:
        launcher = ServiceLauncher(conf, restart_method=restart_method)
    else:
        launcher = ProcessLauncher(conf, restart_method=restart_method)
    launcher.launch_service(service, workers=workers)
```
可以看到这里使用到了两种启动器，在进一步讲解启动的过程中先介绍下karbor中的启动器

## 二、Karbor中的Launcher
Openstack中有一个叫Launcher的概念，即专门用来启动服务的，这个类被放在了oslo_service这个包里面，Launcher分为两种:  
一种是**ServiceLauncher**；  
另一种为**ProcessLauncher**。  
ServiceLauncher用来启动单进程的服务，如karbor-protection，karbor-operation；  
而ProcessLauncher用来启动有多个worker子进程的服务，如各类api服务(nova-api、karbor-api)等

**oslo_service/service.py**  
### 1、ServiceLauncher  
ServiceLauncher继承自Launcher，启动服务的一个重要成员就是launcher_service，ServiceLauncher的该成员就是继承于Launcher
```python
def launch_service(self, service, workers=1):
 	…
    if workers is not None and workers != 1:
        raise ValueError(_("Launcher asked to start multiple workers"))
    _check_service_base(service)
    service.backdoor_port = self.backdoor_port
    self.services.add(service)
```
aucher_service就是将服务添加到self.services成员里面，services成员的类型是class Services，看看它的add方法
```python
class Services(object):

    def __init__(self):
        self.services = []
        self.tg = threadgroup.ThreadGroup()
        self.done = event.Event()

    def add(self, service):
        """Add a service to a list and create a thread to run it.

        :param service: service to run
        """
        self.services.append(service)
        self.tg.add_thread(self.run_service, service, self.done)
```
Services这个类的初始化很简单，即创建一个ThreadGroup，ThreadGroup其实是eventlet的GreenPool，Openstack利用eventlet实现并发，add方法，将self.run_service这个方法放入pool中，而service就是它的参数。run_service方法很简单，就是调用service的start方法，这样就完成了服务的启动

### 2、ProcessLauncher
ProcessLauncher直接继承于Object，同样也有launch_service方法
```python
def launch_service(self, service, workers=1):
 	…
    _check_service_base(service)
    wrap = ServiceWrapper(service, workers)

    LOG.info('Starting %d workers', wrap.workers)
    while self.running and len(wrap.children) < wrap.workers:
        self._start_child(wrap)
```
lauch_service除了接受service以外，还需要接受一个workers参数，即子进程的个数，然后调用_start_child启动多个子进程
```python
def _start_child(self, wrap):
    if len(wrap.forktimes) > wrap.workers:
        # Limit ourselves to one process a second (over the period of
        # number of workers * 1 second). This will allow workers to
        # start up quickly but ensure we don't fork off children that
        # die instantly too quickly.
        if time.time() - wrap.forktimes[0] < wrap.workers:
            LOG.info('Forking too fast, sleeping')
            time.sleep(1)

        wrap.forktimes.pop(0)

    wrap.forktimes.append(time.time())

    pid = os.fork()
    if pid == 0:
        self.launcher = self._child_process(wrap.service)
        while True:
            self._child_process_handle_signal()
            status, signo = self._child_wait_for_exit_or_signal(
                self.launcher)
            if not _is_sighup_and_daemon(signo):
                self.launcher.wait()
                break
            self.launcher.restart()

        os._exit(status)

    LOG.debug('Started child %d', pid)

    wrap.children.add(pid)
    self.children[pid] = wrap
```
看见熟悉的fork没有，只是简单的调用了一个os.fork()，然后子进程开始运行，子进程调用_child_process
```python
def _child_process(self, service):
    self._child_process_handle_signal()

    # Reopen the eventlet hub to make sure we don't share an epoll
    # fd with parent and/or siblings, which would be bad
    eventlet.hubs.use_hub()

    # Close write to ensure only parent has it open
    os.close(self.writepipe)
    # Create greenthread to watch for parent to close pipe
    eventlet.spawn_n(self._pipe_watcher)

    # Reseed random number generator
    random.seed()

    launcher = Launcher(self.conf, restart_method=self.restart_method)
    launcher.launch_service(service)
    return launcher
```
_child_process其实很简单，创建一个Launcher，调用Laucher.launch_service方法，前面介绍过，其实ServiceLauncher继承自Launcher，也是调用的launcher_service方法，将服务启动，因此接下来的步骤可以参考前面，最终都将调用service.start方法启动服务

## 三、WSGIService的启动—续
回到前面的启动部分，从launcher节的说明，我们知道服务的启动最终调用了service的start方法，而这里的service就是我们最开始在api.py中创建的service，然后一层层传进后面的启动器中的，我们继续回到WSGIService类中的start(self)方法
```python
def start(self):
    …
    if self.manager:
        self.manager.init_host()
    self.server.start()
    self.port = self.server.port
```
这里调用了**oslo_service/wsgi.py**中的start(self)方法
```python
def start(self):
    …
    self.dup_socket = self.socket.dup()

    if self._use_ssl:
        self.dup_socket = sslutils.wrap(self.conf, self.dup_socket)

    wsgi_kwargs = {
        'func': eventlet.wsgi.server,
        'sock': self.dup_socket,
        'site': self.app,
        'protocol': self._protocol,
        'custom_pool': self._pool,
        'log': self._logger,
        'log_format': self.conf.wsgi_log_format,
        'debug': False,
        'keepalive': self.conf.wsgi_keep_alive,
        'socket_timeout': self.client_socket_timeout
        }

    if self._max_url_len:
        wsgi_kwargs['url_length_limit'] = self._max_url_len

    self._server = eventlet.spawn(**wsgi_kwargs)
```
注意 wsgi_kwargs 中的参数 func，它的值为 eventlet.wsgi.server，在 **eventlet/wsgi.py** 的定义如下：
```python
def server(sock, site,
    …
     try:
        serv.log.info("(%s) wsgi starting up on %s" % (
            serv.pid, socket_repr(sock)))
        while is_accepting:
            try:
                client_socket = sock.accept()
                client_socket[0].settimeout(serv.socket_timeout)
                serv.log.debug("(%s) accepted %r" % (
                    serv.pid, client_socket[1]))
                try:
                    pool.spawn_n(serv.process_request, client_socket)
                except AttributeError:
                    warnings.warn("wsgi's pool should be an instance of "
                                  "eventlet.greenpool.GreenPool, is %s. Please convert your"
                                  " call site to use GreenPool instead" % type(pool),
                                  DeprecationWarning, stacklevel=2)
                    pool.execute_async(serv.process_request, client_socket)
            except ACCEPT_EXCEPTIONS as e:
                if support.get_errno(e) not in ACCEPT_ERRNO:
                    raise
            except (KeyboardInterrupt, SystemExit):
                serv.log.info("wsgi exiting")
                break
    finally:
        pool.waitall()
        …
```
看，是不是看到熟悉的一幕了！sock.accept() 监听请求，每当接收到一个新请求，调用 pool.spawn_n() 启动一个协程处理该请求

## 四、Service的启动
Service 类型的进程同样由 karbor/cmd/* 目录下某些文件创建：
- **Karbor-protection: karbor/cmd/protection.py**
- ……  
作为消息中间件的消费者，它们监听各自的 queue，每当有 rpc 请求来临时，它们创建一个新的协程处理 rpc 请求。以karbor-protection为例，启动时初始化一个 Server(由 **karbor/service.py** 定义) 对象。  
整个Launcher过程跟WSGIServer一样，只是service的start()有些区别而已
```python
def start(self):
    …
    target = messaging.Target(topic=self.topic, server=self.host)
    endpoints = [self.manager]
    endpoints.extend(self.manager.additional_endpoints)
    serializer = objects_base.KarborObjectSerializer()
    self.rpcserver = rpc.get_server(target, endpoints, serializer)
    self.rpcserver.start()
```
经过层层调用，最终生成了这样一个RPCServer对象
```python
class RPCServer(msg_server.MessageHandlingServer):
    def __init__(self, transport, target, dispatcher, executor='blocking'):
        super(RPCServer, self).__init__(transport, dispatcher, executor)
        self._target = target
```
该类继承自**MessageHandlingServer**；  
注：karbor 的各个组件都依赖 oslo.messaging 访问消息服务器，通过 **oslo/messaging/server.py** 初始化一个 MessageHandlingServer 的对象，监听消息队列。  
最终调用了该service的start方法
```python
def start(self, override_pool_size=None):
    … 
    if self._started:
        LOG.warning(_LW('Restarting a MessageHandlingServer is inherently '
                        'racy. It is deprecated, and will become a noop '
                        'in a future release of oslo.messaging. If you '
                        'need to restart MessageHandlingServer you should '
                        'instantiate a new object.'))
    self._started = True

    executor_opts = {}

    if self.executor_type in ("threading", "eventlet"):
        executor_opts["max_workers"] = (
            override_pool_size or self.conf.executor_thread_pool_size
        )
    self._work_executor = self._executor_cls(**executor_opts)

    try:
        self.listener = self._create_listener()
    except driver_base.TransportDriverError as ex:
        raise ServerListenError(self.target, ex)

    # HACK(sileht): We temporary pass the executor to the rabbit
    # listener to fix a race with the deprecated blocking executor.
    # We do this hack because this is need only for 'synchronous'
    # executor like blocking. And this one is deprecated. Making
    # driver working in an sync and an async way is complicated
    # and blocking have 0% tests coverage.
    if hasattr(self.listener, '_poll_style_listener'):
        l = self.listener._poll_style_listener
        if hasattr(l, "_message_operations_handler"):
            l._message_operations_handler._executor = (
                self.executor_type)

    self.listener.start(self._on_incoming)
```
上述的对象又初始化一个 EventletExecutor(由 **oslo/messaging/_executors/impl_eventlet.py**) 类型的 excuete 对象，它调用 **self.listener.poll()** 监听 rpc 请求，每当接收到一个请求，创建一个协程处理该请求。
```python
class EventletExecutor(base.ExecutorBase):
    ......

    def start(self):
        if self._thread is not None:
            return

        @excutils.forever_retry_uncaught_exceptions
        def _executor_thread():
            try:
                while True:
                    incoming = self.listener.poll()
                    spawn_with(ctxt=self.dispatcher(incoming),
                               pool=self._greenpool)
            except greenlet.GreenletExit:
                return

        self._thread = eventlet.spawn(_executor_thread)
