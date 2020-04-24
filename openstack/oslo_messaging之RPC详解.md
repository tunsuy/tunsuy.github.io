# oslo_messaging之RPC详解



## Transport

Transport 就是 RPC 调用过程中，使用的消息通信介质，如果我们使用 rabbitmq，那么需要指定 rabbitmq 服务器的连接地址，以及用户名，密码等参数。
RPC 调用的 client 和 server 端都需要指定一个 transport 作为消息的 broker.
oslo.messaging 中通过 oslo_messaging.get_transport 函数返回一个 transport 对象，如：
```python
def get_rpc_transport(conf, url=None,
                      allowed_remote_exmods=None):
    return msg_transport._get_transport(
        conf, url, allowed_remote_exmods,
        transport_cls=msg_transport.RPCTransport)
```
这里使用`msg_transport.RPCTransport`类对消息队列类进行初始化
```python
def _get_transport(conf, url=None, allowed_remote_exmods=None,
                   transport_cls=RPCTransport):
    allowed_remote_exmods = allowed_remote_exmods or []
    conf.register_opts(_transport_opts)

    if not isinstance(url, TransportURL):
        url = TransportURL.parse(conf, url)

    kwargs = dict(default_exchange=conf.control_exchange,
                  allowed_remote_exmods=allowed_remote_exmods)

    try:
        mgr = driver.DriverManager('oslo.messaging.drivers',
                                   url.transport.split('+')[0],
                                   invoke_on_load=True,
                                   invoke_args=[conf, url],
                                   invoke_kwds=kwargs)
    except RuntimeError as ex:
        raise DriverLoadFailure(url.transport, ex)

    return transport_cls(mgr.driver)
```
返回值mgr 的driver 属性为某一消息队列的驱动（或具体消息队列调用的封装）。该driver的具体值和传入的URL 有关系。如果指明使用的消息队列为RibbitMQ。所以，此处driver 的值为RabbitDriver 类的一个实例。RabbitDriver 类的实现在oslo_messaging/_drivers/impl_rabbit.py 中。具体如何在driver.DriverManager（）方法中调用到RabbitDriver 类暂不研究。

### TransportURL
消息传输的URL，格式如下：
`driver://[user:pass@]host:port[,[userN:passN@]hostN:portN]/virtual_host?query`
该类主要是将url解析成Transport的url格式
```python
TransportURL.parse(conf, url)
```
例如，如果是使用rabbitmq作为mq驱动，则format中的driver为rabbit

## DriverManager
继承自`NamedExtensionManager`，通过stevedore库的能力从名字空间中加载消息驱动插件

### RabbitDriver
RabbitDriver 类继承了amqpdriver.AMQPDriverBase 及base.BaseDriver 方法，如：send， _send, send_notification,listen,cleanup 等方法。

先看RabbitDriver 的初始化：
oslo_messaging/_drivers/impl_rabbit.py
```python
class RabbitDriver(amqpdriver.AMQPDriverBase):
    """RabbitMQ Driver

    The ``rabbit`` driver is the default driver used in OpenStack's
    integration tests.

    The driver is aliased as ``kombu`` to support upgrading existing
    installations with older settings.

    """

    def __init__(self, conf, url,
                 default_exchange=None,
                 allowed_remote_exmods=None):
        opt_group = cfg.OptGroup(name='oslo_messaging_rabbit',
                                 title='RabbitMQ driver options')
        conf.register_group(opt_group)
        conf.register_opts(rabbit_opts, group=opt_group)
        conf.register_opts(rpc_amqp.amqp_opts, group=opt_group)
        conf.register_opts(base.base_opts, group=opt_group)
        conf = rpc_common.ConfigOptsProxy(conf, url, opt_group.name)

        self.missing_destination_retry_timeout = (
            conf.oslo_messaging_rabbit.kombu_missing_consumer_retry_timeout)

        self.prefetch_size = (
            conf.oslo_messaging_rabbit.rabbit_qos_prefetch_count)

        # the pool configuration properties
        max_size = conf.oslo_messaging_rabbit.rpc_conn_pool_size
        min_size = conf.oslo_messaging_rabbit.conn_pool_min_size
        ttl = conf.oslo_messaging_rabbit.conn_pool_ttl

        connection_pool = pool.ConnectionPool(
            conf, max_size, min_size, ttl,
            url, Connection)

        super(RabbitDriver, self).__init__(
            conf, url,
            connection_pool,
            default_exchange,
            allowed_remote_exmods
        )
```
重点看pool.ConnectionPool() 的初始化过程。在pool.ConnectionPool 类中，实现了建立连接，获取连接，归还连接，清空连接池等方法。该类初始化过程中，传入了连接池的TCP 连接数量的上 ，下限值，及具体的连接类。

oslo_messaging/_drivers/pool.py
```python
class ConnectionPool(Pool):
    """Class that implements a Pool of Connections."""

    def __init__(self, conf, max_size, min_size, ttl, url, connection_cls):
        self.connection_cls = connection_cls
        self.conf = conf
        self.url = url
        super(ConnectionPool, self).__init__(max_size, min_size, ttl,
                                             self._on_expire)
```
总的来说，ConnectionPool维护了一个连接池，保管连接实例，但目前连接池为空，没有建立好的连接实例。何时调用create() 建立连接？带着这个疑问继续往下走。

返回到_get_transport ，完成了driver.DriverManager() 方法的调用，接着执行transport_cls(mgr.driver) 实例化一个transport，该transport中还未建立TCP 连接。


## Target
Target 对象代表一个调用需要匹配的目标。RPC 客户端需要指定 Target 来进行 RPC 调用，RPC server 段也要指定 Target 来说明接收哪些 RPC 调用。Target 在底层被用来决定 RPC server 需要创建哪些队列，使用哪些 routing key 来绑定到 exchange 上，以及 RPC client 发送消息的 routing key。

Target 类的原型如下：
```python
class Target(object):
    def __init__(self, exchange=None, topic=None, namespace=None,
                 version=None, server=None, fanout=None,
                 legacy_namespaces=None):
        self.exchange = exchange
        self.topic = topic
        self.namespace = namespace
        self.version = version
        self.server = server
        self.fanout = fanout
        self.accepted_namespaces = [namespace] + (legacy_namespaces or [])
```
这里的 exchange, topic, namespace, server， fanout 等参数会被用于完成 exchange 的声明，队列的创建，binding 的创建以及 routing key 的选择等。而 namespace, version 等参数是 oslo.messaging 为了实现更精确的匹配规则创建的概念。

RPC 中的各个组件都需要使用这个 Target 对象，他们在使用时需要指定的参数如下：
RPC Server: 必须指定 topic 和 server，还可以指定 exchange
RPC endpoint: 可以指定 namespace 和 version
RPC client: 必须指定 topic，其他均为可选项
Notification Server：必须指定 topic，还可以指定 exchange
Notifier: 必须指定 topic，还可以指定 exchange

例如：在伪代码 `target= messaging.Target(topic='test',server='server1')` 中，指定了消息发往的服务器是监听 ’test’ topic 的server1 服务器。

## RPC Server
RPC Server 的构造函数为：
```python
def get_rpc_server(transport, target, endpoints,
                   executor='blocking', serializer=None, access_policy=None):
    dispatcher = rpc_dispatcher.RPCDispatcher(endpoints, serializer,
                                              access_policy)
    return RPCServer(transport, target, dispatcher, executor)
```
这里是 endpoints 参数是一个列表，包含所有的 endpoints 对象。
executor: 执行器，表示使用协程还是线程执行
access_policy：方法访问权限控制，默认不能访问私有方法(_开始)

### Endpoints
RPC Server 通过 Endpoint，将方法暴露出去，供 Client 端进行调用。一个 RPC Server 可以指定多个 Endpoint 对象。

启动server: `server.start()`
```python
 def start(self, override_pool_size=None):
        """Start handling incoming messages.

        This method causes the server to begin polling the transport for
        incoming messages and passing them to the dispatcher. Message
        processing will continue until the stop() method is called.

        The executor controls how the server integrates with the applications
        I/O handling strategy - it may choose to poll for messages in a new
        process, thread or co-operatively scheduled coroutine or simply by
        registering a callback with an event loop. Similarly, the executor may
        choose to dispatch messages in a new thread, coroutine or simply the
        current thread.
        """
        if self._started:
            LOG.warning('The server has already been started. Ignoring '
                        'the redundant call to start().')
            return

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

        self.listener.start(self._on_incoming)
```
这里创建了监听，最终进入到transport里面的listen方法：
```python
    def _listen(self, target, batch_size, batch_timeout):
        if not (target.topic and target.server):
            raise exceptions.InvalidTarget('A server\'s target must have '
                                           'topic and server names specified',
                                           target)
        return self._driver.listen(target, batch_size,
                                   batch_timeout)
```
也就是各驱动自己的监听方法，如果是rabbitmq，则是RabbitDriver，而它又继承自AMQPDriverBase，所以，进入如下方法
```python
def listen(self, target, batch_size, batch_timeout):
	conn = self._get_connection(rpc_common.PURPOSE_LISTEN)

	listener = RpcAMQPListener(self, conn)

	conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
								topic=target.topic,
								callback=listener)
	conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
								topic='%s.%s' % (target.topic,
												 target.server),
								callback=listener)
	conn.declare_fanout_consumer(target.topic, listener)

	return base.PollStyleListenerAdapter(listener, batch_size,
										 batch_timeout)
```
这里可以看到创建三个队列，两个 topic 类型，分别以 “topic” 和 “topic.server” 为名称，同时也是队列绑定到 exchange 的 routing key. 
还有一个 fanout 类型的队列用作 notification.
```python
    def declare_fanout_consumer(self, topic, callback):
        """Create a 'fanout' consumer."""

        unique = uuid.uuid4().hex
        exchange_name = '%s_fanout' % topic
        queue_name = '%s_fanout_%s' % (topic, unique)

        consumer = Consumer(exchange_name=exchange_name,
                            queue_name=queue_name,
                            routing_key=topic,
                            type='fanout',
                            durable=False,
                            exchange_auto_delete=True,
                            queue_auto_delete=False,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues,
                            rabbit_queue_ttl=self.rabbit_transient_queues_ttl)
```
exchange名为topic_fanout，queue名为topic_fanout_uuid

注：因为使用rabbitmq做rpc时，server端就是消费者，client就是生产者，故这里的方法名是消费者队列

## RPC Client 
RPC 的调用都要通过 RPC Client 来完成。创建一个 RPC Client 需要指定 Transport 和 Target.
```python
def __init__(self, transport, target,
                 timeout=None, version_cap=None, serializer=None, retry=None,
                 call_monitor_timeout=None, transport_options=None):
```
RPCClient 的作用就是通过 Target 中设置的参数来找到 RPC 调用需要发送的 exchange 和 routing key。虽然 target 是在创建 RPC Client 的时候指定的，在某些调用中也可以通过 RPCCLient 的 prepare() 方法重载 target 中的属性。例如在某些调用中设置一个特殊的 Target namespace 或者 version.

### Call调用
RPCClient 可以发起 call 调用，此时线程会阻塞直至收到调用的返回结果。call() 调用会在调用时创建一个用于接收返回消息的 direct exchange 和队列，并监听在此队列上。
call() 方法接收的参数分别为请求的 context dict，需要调用的方法，和方法的参数。由于 call 调用是阻塞的，因此程序中的 call() 是保证按顺序执行的。

### Cast调用
cast 调用是以非阻塞的方式来进行 RPC 调用（例如 Nova 中的虚拟机重启）。cast 调用可以发送到 fanout exchange 中。由于 cast() 是非阻塞的，因此程序中的 cast 调用不会保证按顺序执行。

### 建立连接
深入分下RPClient的call() 或cast() 方法，会发现最终会调用_BaseCallContext类中的call() 或cast()方法，以call() 为例，看一下最后的call() 方法的实现。

oslo_messaging/rpc/client.py
```python
class _BaseCallContext(object):
    def call(self, ctxt, method, **kwargs):
        """Invoke a method and wait for a reply. See RPCClient.call()."""
        if self.target.fanout:
            raise exceptions.InvalidTarget('A call cannot be used with fanout',
                                           self.target)

        msg = self._make_message(ctxt, method, kwargs)
        msg_ctxt = self.serializer.serialize_context(ctxt)

        timeout = self.timeout
        if self.timeout is None:
            timeout = self.conf.rpc_response_timeout

        cm_timeout = self.call_monitor_timeout

        self._check_version_cap(msg.get('version'))

        try:
            result = \
                self.transport._send(self.target, msg_ctxt, msg,
                                     wait_for_reply=True, timeout=timeout,
                                     call_monitor_timeout=cm_timeout,
                                     retry=self.retry,
                                     transport_options=self.transport_options)
        except driver_base.TransportDriverError as ex:
            raise ClientSendError(self.target, ex)

        return self.serializer.deserialize_entity(ctxt, result)
```
前文说到：transport 为RPCTransport 类的实例，进入该类的_send() 方法

oslo_messaging/transport.py
```python
    def _send(self, target, ctxt, message, wait_for_reply=None, timeout=None,
              call_monitor_timeout=None, retry=None, transport_options=None):
        if not target.topic:
            raise exceptions.InvalidTarget('A topic is required to send',
                                           target)
        return self._driver.send(target, ctxt, message,
                                 wait_for_reply=wait_for_reply,
                                 timeout=timeout,
                                 call_monitor_timeout=call_monitor_timeout,
                                 retry=retry,
                                 transport_options=transport_options)
```
回到具体的RabbitDriver 类，看具体的send 方法。在RabbitDriver 类中，send方法继承于基类AMQPDriverBase中的send()方法，最后调用了该基类的_send() 方法.

oslo_messaging/_drivers/amqpdriver.py
```python
    def _send(self, target, ctxt, message,
              wait_for_reply=None, timeout=None, call_monitor_timeout=None,
              envelope=True, notify=False, retry=None, transport_options=None):

        msg = message

        if wait_for_reply:
            msg_id = uuid.uuid4().hex
            msg.update({'_msg_id': msg_id})
            msg.update({'_reply_q': self._get_reply_q()})
            msg.update({'_timeout': call_monitor_timeout})

        rpc_amqp._add_unique_id(msg)
        unique_id = msg[rpc_amqp.UNIQUE_ID]

        rpc_amqp.pack_context(msg, ctxt)

        if envelope:
            msg = rpc_common.serialize_msg(msg)

        if wait_for_reply:
            self._waiter.listen(msg_id)
            log_msg = "CALL msg_id: %s " % msg_id
        else:
            log_msg = "CAST unique_id: %s " % unique_id

        try:
            with self._get_connection(rpc_common.PURPOSE_SEND) as conn:
                if notify:
                    exchange = self._get_exchange(target)
                    LOG.debug(log_msg + "NOTIFY exchange '%(exchange)s'"
                              " topic '%(topic)s'", {'exchange': exchange,
                                                     'topic': target.topic})
                    conn.notify_send(exchange, target.topic, msg, retry=retry)
                elif target.fanout:
                    log_msg += "FANOUT topic '%(topic)s'" % {
                        'topic': target.topic}
                    LOG.debug(log_msg)
                    conn.fanout_send(target.topic, msg, retry=retry)
                else:
                    topic = target.topic
                    exchange = self._get_exchange(target)
                    if target.server:
                        topic = '%s.%s' % (target.topic, target.server)
                    LOG.debug(log_msg + "exchange '%(exchange)s'"
                              " topic '%(topic)s'", {'exchange': exchange,
                                                     'topic': topic})
                    conn.topic_send(exchange_name=exchange, topic=topic,
                                    msg=msg, timeout=timeout, retry=retry,
                                    transport_options=transport_options)

            if wait_for_reply:
                result = self._waiter.wait(msg_id, timeout,
                                           call_monitor_timeout)
                if isinstance(result, Exception):
                    raise result
                return result
        finally:
            if wait_for_reply:
                self._waiter.unlisten(msg_id)
```
无论是call()或cast() 方法，都会调用_get_connection()从连接池拿到一个连接，如果连接池为空，会建立连接。下面重点看一下_get_connection()方法的执行流程，这是建立通信的关键。

oslo_messaging/_drivers/amqpdriver.py
```python
    def _get_connection(self, purpose=rpc_common.PURPOSE_SEND):
        return rpc_common.ConnectionContext(self._connection_pool,
                                            purpose=purpose)
```

oslo_messaging/_drivers/common.py
```python
class ConnectionContext(Connection):
    def __init__(self, connection_pool, purpose):
        self.connection = None
        self.connection_pool = connection_pool
        pooled = purpose == PURPOSE_SEND # 改为pooled = (purpose == PURPOSE_SEND) 或许更明了

        if pooled:
            self.connection = connection_pool.get()# 创建新的连接或获取存在的连接
        else:
            # a non-pooled connection is requested, so create a new connection
            self.connection = connection_pool.create(purpose) #创建新的连接
        self.pooled = pooled
        self.connection.pooled = pooled
```
看了ConnectionContext类的初始化方法，我们还应当留意该类实现的__enter__()、__exit__()、__del__()方法，它们都默默做了一些工作。

再看self.connection = connection_pool.get()一句的具体调用：

oslo_messaging/_drivers/pool.py
```python
class Pool(object):    
    def get(self):
        with self._cond:
            while True:
                try:
                    ttl_watch, item = self._items.pop()
                    self.expire()
                    return item # 返回一个有效连接
                except IndexError:
                    pass

                if self._current_size < self._max_size:
                    self._current_size += 1
                    break
    
                wait_condition(self._cond)
    
        # We've grabbed a slot and dropped the lock, now do the creation
        try:
            return self.create() #创建连接，至此，看到了create()方法的调用
        except Exception:
            with self._cond:
                self._current_size -= 1
            raise
```
在调用call() 方法后，我们看到了create()方法的调用，可见在oslo.messaing 中建立连接采取了一种滞后的方法，即真正第一次有远程方法调用时，开始建立连接。

oslo_messaging/_drivers/pool.py
```python
def create(self, purpose=common.PURPOSE_SEND):
    LOG.debug('Pool creating new connection')
    #self.connection_cls 是在驱动实例化时赋值，返回RabbitDriver类初始化函数，查看connection_cls的值
    return self.connection_cls(self.conf, self.url, purpose)
```
通过_get_connection得到一个pool.ConnectionContext 实例，返回到_send() 方法,继续执行分支。以 topic = target.topic 分支为例，继续往下看。进入该分支：

oslo_messaging/_drivers/amqpdriver.py
```python
topic = target.topic
exchange = self._get_exchange(target) #获得交换器名字
if target.server:
    topic = '%s.%s' % (target.topic, target.server)
log_msg += "exchange '%(exchange)s'" \
           " topic '%(topic)s'" % {
               'exchange': exchange,
               'topic': topic}
LOG.debug(log_msg)
conn.topic_send(exchange_name=exchange, topic=topic, 
                msg=msg, timeout=timeout, retry=retry)
```
初次看conn.topic_send 一句，发现ConnectionContext类并没有topic_send() 方法，实际上调用的还是impl_rabbit.py /Connection 类的方法。

oslo_messaging/_drivers/impl_rabbit.py
```python
class Connection(object):  
   def topic_send(self, exchange_name, topic, msg, timeout=None, retry=None):
        """Send a 'topic' message."""
        exchange = kombu.entity.Exchange(
            name=exchange_name,
            type='topic',
            durable=self.amqp_durable_queues,
            auto_delete=self.amqp_auto_delete)

        self._ensure_publishing(self._publish, exchange, msg,
                                routing_key=topic, timeout=timeout,
                                retry=retry)

          
    def _ensure_publishing(self, method, exchange, msg, routing_key=None,
                           timeout=None, retry=None):
        """Send to a publisher based on the publisher class."""
    
        def _error_callback(exc):
            log_info = {'topic': exchange.name, 'err_str': exc}
            LOG.error(_LE("Failed to publish message to topic "
                          "'%(topic)s': %(err_str)s"), log_info)
            LOG.debug('Exception', exc_info=exc)
    
        method = functools.partial(method, exchange, msg, routing_key, timeout)
    
        with self._connection_lock:
            self.ensure(method, retry=retry, error_callback=_error_callback) # 带入了retry 值
```
关于进入ensure() 方法后的执行流程，不同于从ensure_connection(）调用ensure() 的是，这次传给ensure () method 的值变了，并且ensure() 方法内部调用autoretry() 方法时， self.channel 也有值了。

回到oslo_messaging/_drivers/amqpdriver.py的 _send() 方法中， 如果是一个call 同步调用，还会单独建立一个TCP连接，等待回复消息。

oslo_messaging/_drivers/amqpdriver.py
```python
class ReplyWaiter(object):    
    def wait(self, msg_id, timeout, call_monitor_timeout):
        timer = rpc_common.DecayingTimer(duration=timeout)
        timer.start()
        if call_monitor_timeout:
            call_monitor_timer = rpc_common.DecayingTimer(
                duration=call_monitor_timeout)
            call_monitor_timer.start()
        else:
            call_monitor_timer = None
        final_reply = None
        ending = False
        while not ending:
            timeout = timer.check_return(self._raise_timeout_exception, msg_id)
            if call_monitor_timer and timeout > 0:
                cm_timeout = call_monitor_timer.check_return(
                    self._raise_timeout_exception, msg_id)
                if cm_timeout < timeout:
                    timeout = cm_timeout
            try:
                message = self.waiters.get(msg_id, timeout=timeout) #阻塞等待，超时抛异常
            except moves.queue.Empty:
                self._raise_timeout_exception(msg_id)

            reply, ending = self._process_reply(message)
            if reply is not None:
                # NOTE(viktors): This can be either first _send_reply() with an
                # empty `result` field or a second _send_reply() with
                # ending=True and no `result` field.
                final_reply = reply
            elif ending is False:
                LOG.debug('Call monitor heartbeat received; '
                          'renewing timeout timer')
                call_monitor_timer.restart() # 此处有bug ,如果 call_monitor_timer 为None
        return final_reply
```