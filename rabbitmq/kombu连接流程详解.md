# kombu连接流程详解

## kombu的mq模型
因为 Kombu 是对 AMQP 进行抽象，所以它必定有抽象的模型，事实上，它大体上和 RabbitMQ 差不多，但是，不完全一样，有一些差别，下面就介绍一下 Konbu 的抽象模型。

在 Kombu 中，存在多个概念，其实我们在前边简单的生产/消费者样例中已经看到了了一些，他们分别是：

Message：生产消费的基本单位，其实就是我们所谓的一条条消息
Connection：对 MQ 连接的抽象，一个 Connection 就对应一个 MQ 的连接
Transport：真实的 MQ 连接，也是真正连接到 MQ(redis/rabbitmq) 的实例
Producers: 发送消息的抽象类
Consumers：接受消息的抽象类
Exchange：MQ 路由，这个和 RabbitMQ 差不多，支持 5 类型
Queue：对应的 queue 抽象，其实就是一个字符串的封装

## Connection
使用方式：
```python
self.connection = kombu.connection.Connection(
            self._url, ssl=self._fetch_ssl_params(),
            login_method=self.login_method,
            heartbeat=self.heartbeat_timeout_threshold,
            failover_strategy=self.kombu_failover_strategy,
            transport_options={
                'confirm_publish': True,
                'client_properties': {
                    'capabilities': {
                        'authentication_failure_close': True,
                        'connection.blocked': True,
                        'consumer_cancel_notify': True
                    },
                    'connection_name': self.name},
                'on_blocked': self._on_connection_blocked,
                'on_unblocked': self._on_connection_unblocked,
            },
        )
```

进入Connection的init方法:
```python
def __init__(self, hostname='localhost', userid=None,
			 password=None, virtual_host=None, port=None, insist=False,
			 ssl=False, transport=None, connect_timeout=5,
			 transport_options=None, login_method=None, uri_prefix=None,
			 heartbeat=0, failover_strategy='round-robin',
			 alternates=None, **kwargs):
	alt = [] if alternates is None else alternates
	# have to spell the args out, just to get nice docstrings :(
	params = self._initial_params = {
		'hostname': hostname, 'userid': userid,
		'password': password, 'virtual_host': virtual_host,
		'port': port, 'insist': insist, 'ssl': ssl,
		'transport': transport, 'connect_timeout': connect_timeout,
		'login_method': login_method, 'heartbeat': heartbeat
	}

	if hostname and not isinstance(hostname, string_t):
		alt.extend(hostname)
		hostname = alt[0]
	if hostname and '://' in hostname:
		if ';' in hostname:
			alt.extend(hostname.split(';'))
			hostname = alt[0]
		if '+' in hostname[:hostname.index('://')]:
			# e.g. sqla+mysql://root:masterkey@localhost/
			params['transport'], params['hostname'] = \
				hostname.split('+', 1)
			self.uri_prefix = params['transport']
		else:
			transport = transport or urlparse(hostname).scheme
			if not get_transport_cls(transport).can_parse_url:
				# we must parse the URL
				url_params = parse_url(hostname)
				params.update(
					dictfilter(url_params),
					hostname=url_params['hostname'],
				)

			params['transport'] = transport

	self._init_params(**params)
```
注意这里的:
`self.transport_cls = transport`
后续会通过transport名称调用相应的transport方法，比如：如果transport是rabbitmq，则使用librabbitmq.py中的方法

## Transport
基类
transport/base.py
```python
class Transport(object):
    """Base class for transports."""
```

## Message
这是 Kombu 中生产/消费的基本单位
```python
    def __init__(self, body=None, delivery_tag=None,
                 content_type=None, content_encoding=None, delivery_info=None,
                 properties=None, headers=None, postencode=None,
                 accept=None, channel=None, **kwargs):
        delivery_info = {} if not delivery_info else delivery_info
        self.errors = [] if self.errors is None else self.errors
        self.channel = channel
        self.delivery_tag = delivery_tag
        self.content_type = content_type
        self.content_encoding = content_encoding
        self.delivery_info = delivery_info
        self.headers = headers or {}
        self.properties = properties or {}
        self._decoded_cache = None
        self._state = 'RECEIVED'
        self.accept = accept

        compression = self.headers.get('compression')
        if not self.errors and compression:
            try:
                body = decompress(body, compression)
            except Exception:
                self.errors.append(sys.exc_info())

        if not self.errors and postencode and isinstance(body, text_t):
            try:
                body = body.encode(postencode)
            except Exception:
                self.errors.append(sys.exc_info())
        self.body = body
```

## 流程详解
回顾下oslo_messaging之RPC详解中提到的：
如果是调用create()新创建一个连接，就会实例化一个Connection 实例，该Connection类的实现在消息队列驱动类所在的模块，对于Rabbit Driver，Connection 类位于oslo_messaging/_drivers/impl_rabbit.py。Connection 类封装了RabbitMQ 的相关操作，如：direct_send,topic_send，fanout_send, notify_send 等。这几种方法对应了RabbitMQ 中几种消息分发模式。

client调用的时候最终会执行到下面的代码：
oslo_messaging/_drivers/impl_rabbit.py
```python
def ensure(self, method, retry=None, 
               recoverable_error_callback=None, error_callback=None,
               timeout_is_error=True):

        if retry is None:
            retry = self.max_retries # 虽然retry 没有接收任何传入值，但在此被赋值
        if retry is None or retry < 0:
            retry = None

        def execute_method(channel):
            self._set_current_channel(channel)
            # 正常情况，稍后会走到此，运行调用ensure() 传递进来的参数
            method()

        try:
           # self.connection 即kombu 库下的Connection 类实例
           # autoretry 调用较复杂，将在下文详细说明
            autoretry_method = self.connection.autoretry(
                #self.channel 只会在调用ensure_connection()方法时，被置为None
                execute_method, channel=self.channel,
                max_retries=retry,
                errback=on_error,
                interval_start=self.interval_start or 1,
                interval_step=self.interval_stepping,
                interval_max=self.interval_max,
                on_revive=on_reconnection)
            # autoretry_method 实际是一个_ensured 方法
            ret, channel = autoretry_method()
            self._set_current_channel(channel) #设置self.channel 
            return ret
        except rpc_amqp.AMQPDestinationNotFound:
            # NOTE(sileht): we must reraise this without
            # trigger error_callback
            raise
        except Exception as exc:
            error_callback and error_callback(exc)
            self._set_current_channel(None)
            # NOTE(sileht): number of retry exceeded and the connection
            # is still broken
            info = {'err_str': exc, 'retry': retry}
            info.update(self.connection.info())
            msg = _('Unable to connect to AMQP server on '
                    '%(hostname)s:%(port)s after %(retry)s '
                    'tries: %(err_str)s') % info
            LOG.error(msg)
            raise exceptions.MessageDeliveryFailure(msg)
```
下面看一下self.connection.autoretry 调用流程，这个流程用法稍微用到了Python的高级特性——可调用对象。

首先，明确self.connection 为在上述Connection 类初始化中赋值的，实际为kombu/connection.py 中的Connection实例，在kombu 库中的Connection类中，同样有ensure_connection() 和ensure() 方法。

kombu/connection.py ,并且有很多方法使用了property 装饰器
```python
class Connection(object):   
    def autoretry(self, fun, channel=None, **ensure_options):#fun 为execute_method
        channels = [channel]
        class Revival(object): #通过__call__()实现了一个可调用对象
            __name__ = getattr(fun, '__name__', None) # 将参数fun 的名字置为Revival 类的名字
            __module__ = getattr(fun, '__module__', None)
            __doc__ = getattr(fun, '__doc__', None)

            def __init__(self, connection):
                self.connection = connection # 将Connection 类实例引用赋值给了self.connection

            def revive(self, channel):
                channels[0] = channel

            def __call__(self, *args, **kwargs): #将在调用Revival 类时被执行
                if channels[0] is None:
                     #default_channel 实际为Connection类中一个通过property 装饰的方法
                    self.revive(self.connection.default_channel)
                return fun(*args, channel=channels[0], **kwargs), channels[0]

        revive = Revival(self) #注意传递进了参数为自身的类实例引用
        return self.ensure(revive, revive, **ensure_options) 
```
autoretry_method变量的赋值过程如下：
```python
Connection.autoretry()->Connection.ensure()-> return ensure._ensure
```
autoretry_method变量的执行过程：

autoretry_method() [实际执行的是_ensure()方法]->Revival.__call__()。下面详细分析一下__call__() 的执行流程：
```python
class Revival(object):
        __name__ = getattr(fun, '__name__', None)
        __module__ = getattr(fun, '__module__', None)
        __doc__ = getattr(fun, '__doc__', None)

        def __init__(self, connection):
            self.connection = connection

        def revive(self, channel):
            channels[0] = channel

        def __call__(self, *args, **kwargs):
            if channels[0] is None: # channel 存在，就不去判断是否存在TCP连接，及生成channel了。
                self.revive(self.connection.default_channel) #给channels[0]赋值
            return fun(*args, channel=channels[0], **kwargs), channels[0]
```
在__call__方法中,调用self.revive()方法给channels[0] 赋值，revive() 方法传入的参数实际上是通过执行了self.connection.default_channel 获取，default_channel 是一个被property 装饰的方法。

kombu/connetion.py
```python
@property
def default_channel(self):
    conn_opts = {}
	transport_opts = self.transport_options
	if transport_opts:
		if 'max_retries' in transport_opts:
			conn_opts['max_retries'] = transport_opts['max_retries']
		if 'interval_start' in transport_opts:
			conn_opts['interval_start'] = transport_opts['interval_start']
		if 'interval_step' in transport_opts:
			conn_opts['interval_step'] = transport_opts['interval_step']
		if 'interval_max' in transport_opts:
			conn_opts['interval_max'] = transport_opts['interval_max']

	# make sure we're still connected, and if not refresh.
	self.ensure_connection(**conn_opts)
	if self._default_channel is None:
		self._default_channel = self.channel()
	return self._default_channel

def ensure_connection(self, errback=None, max_retries=None,
					  interval_start=2, interval_step=2, interval_max=30,
					  callback=None, reraise_as_library_errors=True,
					  timeout=None):
	def on_error(exc, intervals, retries, interval=0):
		round = self.completes_cycle(retries)
		if round:
			interval = next(intervals)
		if errback:
			errback(exc, interval)
		self.maybe_switch_next()  # select next host

		return interval if round else 0

	ctx = self._reraise_as_library_errors
	if not reraise_as_library_errors:
		ctx = self._dummy_context
	with ctx():
		retry_over_time(self.connect, self.recoverable_connection_errors,
						(), {}, on_error, max_retries,
						interval_start, interval_step, interval_max,
						callback, timeout=timeout)
	return self
```

kombu/utils/functional.py
```python
def retry_over_time(fun, catch, args=None, kwargs=None, errback=None,
                    max_retries=None, interval_start=2, interval_step=2,
                    interval_max=30, callback=None, timeout=None):
    """Retry the function over and over until max retries is exceeded.

    For each retry we sleep a for a while before we try again, this interval
    is increased for every retry until the max seconds is reached.

    Arguments:
        fun (Callable): The function to try
        catch (Tuple[BaseException]): Exceptions to catch, can be either
            tuple or a single exception class.

    Keyword Arguments:
        args (Tuple): Positional arguments passed on to the function.
        kwargs (Dict): Keyword arguments passed on to the function.
        errback (Callable): Callback for when an exception in ``catch``
            is raised.  The callback must take three arguments:
            ``exc``, ``interval_range`` and ``retries``, where ``exc``
            is the exception instance, ``interval_range`` is an iterator
            which return the time in seconds to sleep next, and ``retries``
            is the number of previous retries.
        max_retries (int): Maximum number of retries before we give up.
            If neither of this and timeout is set, we will retry forever.
            If one of this and timeout is reached, stop.
        interval_start (float): How long (in seconds) we start sleeping
            between retries.
        interval_step (float): By how much the interval is increased for
            each retry.
        interval_max (float): Maximum number of seconds to sleep
            between retries.
        timeout (int): Maximum seconds waiting before we give up.
    """
    kwargs = {} if not kwargs else kwargs
    args = [] if not args else args
    interval_range = fxrange(interval_start,
                             interval_max + interval_start,
                             interval_step, repeatlast=True)
    end = time() + timeout if timeout else None
    for retries in count():
        try:
            return fun(*args, **kwargs)
        except catch as exc:
            if max_retries and retries >= max_retries:
                raise
            if end and time() > end:
                raise
            if callback:
                callback()
            tts = float(errback(exc, interval_range, retries) if errback
                        else next(interval_range))
            if tts:
                for _ in range(int(tts)):
                    if callback:
                        callback()
                    sleep(1.0)
                # sleep remainder after int truncation above.
                sleep(abs(int(tts) - tts))
```
在retry_over_time() 函数中，我们看到了一个用于尝试连接的for 循环，如果本次连接失败，通过在异常处理中判断retries 的次数 是否超过max_retries 来决定是否终止连接尝试。再看max_retries 参数的来源，max_retries 实际上来源于ensure_connection()

再看 retry_over_time() 中的正确执行流程 ,进入fun(*args, **kwargs) 一句，fun 变量实际为connect 方法。该方法内的调用多处用到了property 装饰特性。

kombu/connection.py
```python
class Connection(object):    
    def connect(self):
        """Establish connection to server immediately."""
        self._closed = False
        return self.connection # 用到property 装饰器
    
    @property
    def connection(self):
        if not self._closed: # 在connect()函数中被无条件置为False
            # self.connected 同样是一个被property装饰的函数，执行该函数确认连接是否已经建立
            if not self.connected:  
                self.declared_entities.clear()
                self._default_channel = None
                self._connection = self._establish_connection() # self._connection 在此赋值
                self._closed = False
            return self._connection
    
    def _establish_connection(self): # 建立连接
        self._debug('establishing connection...')
        # self.transport 用到property 装饰器，实际为返回self._transport
        conn = self.transport.establish_connection() 
        self._debug('connection established: %r', self)
        return conn
        
    @property
    def transport(self):
        if self._transport is None:  
           # self._transport 的通过create_transport() 初始化，实际为kombu/transport 下某个
           # Transport 类实例。此处，为kombu/transport/pyampq.py 中的Transport 类实例。
           # 先不细看create_transport() 的实例化过程
            self._transport = self.create_transport()
        return self._transport 
```
返回到Revial 类的__call__()中 ，执行完self.revive() 方法后，接着执行autoretry 方法传递的函数fun。该函数是ensure() [impl_rabbit.py]方法内部定义的 execute_method().
```python
def execute_method(channel):
     
     #设置当前使用的self.channel 为传递进来的channel
     #并且，如果为消费者channel ，设置channel的 QoS prefetch count
     self._set_current_channel(channel) 
     
     #执行ensure()传递的fun,ensure在多处被调用，如topic_send, fanout_send,direct_sned,
     #ensure_connection等. 对于ensure_connection()，执行的方法还是上面的connect() 方法，但不会再次 
     #建立建立，是直接返回
     method() 
```
返回到impl_rabbit.py 中Connection 类的初始化方法，执行完self.ensure_connection()语句后，一个TCP连接和channel 就初始化完成了。