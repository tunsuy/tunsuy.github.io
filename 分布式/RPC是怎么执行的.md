## RPC是怎么执行的
我们都知道rpc是远程过程调用的意思，通俗的说，就是可以跨节点调用其他节点上的方法。
当然这里要跟rmi区分开来，他们之间有类似的地方。rmi是远程方法调用，是java领域特有的。而rpc是不区分语言的，发送端和接收端可以是异构的。

这篇文章不会具体说rpc的整个过程，因为在之前的文章中，已经详细的讲解了rpc的发送逻辑，这里接着说rpc的接收逻辑。

### RPC发送端

回顾下之前说过的rpc的发送端
```python
def delete_v2(self, context, job_id, resources):
	cctxt = self.client.prepare(version='1.0')
	cctxt.cast(context, 'delete_v2', job_id=job_id,
			   resources=resources)
```
我们再看下cast方法：
```python
def cast(self, ctxt, method, **kwargs):
	"""Invoke a method and return immediately. See RPCClient.cast()."""
	msg = self._make_message(ctxt, method, kwargs)
	msg_ctxt = self.serializer.serialize_context(ctxt)

	self._check_version_cap(msg.get('version'))

	try:
		self.transport._send(self.target, msg_ctxt, msg,
							 retry=self.retry,
							 transport_options=self.transport_options)
	except driver_base.TransportDriverError as ex:
		raise ClientSendError(self.target, ex)
```
主要看下msg里面都是怎样的数据结构：
```python
def _make_message(self, ctxt, method, args):
	msg = dict(method=method)

	msg['args'] = dict()
	for argname, arg in args.items():
		msg['args'][argname] = self.serializer.serialize_entity(ctxt, arg)

	if self.target.namespace is not None:
		msg['namespace'] = self.target.namespace
	if self.target.version is not None:
		msg['version'] = self.target.version

	return msg
```

好了，下面开始进入正题，看下rpc的接收端，到底是怎样找到本地方法进行执行的？

### RPC接收端启动
我们先来看下rpc接收端是怎么启动，并跟rpc服务端进行连接的？
```python
target = messaging.Target(topic=self.topic, server=self.host)
endpoints = [RemoveClass()]
endpoints.extend(self.manager.additional_endpoints)
access_policy = dispatcher.DefaultRPCAccessPolicy
server = messaging.get_rpc_server(TRANSPORT,
								target,
								endpoints,
								executor='eventlet',
								access_policy=access_policy)
server.start()
```
上面的代码只是一个示例，主要要关注endpoints，也就是提供给rpc发送端调用的类实例

再来看下start方法：
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
上面的代码创建了一个监听器，由监听器来监控rpc请求的到来。

### 请求监听器
监听器是怎么来的呢？
```python
def _listen(self, target, batch_size, batch_timeout):
	if not (target.topic and target.server):
		raise exceptions.InvalidTarget('A server\'s target must have '
									   'topic and server names specified',
									   target)
	return self._driver.listen(target, batch_size,
							   batch_timeout)
```
有具体的Transport根据消息驱动类型得到，我们这里以rabbitmq为例：
```python
amqpdriver.py
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
我们进入PollStyleListenerAdapter类：
```python
super(PollStyleListenerAdapter, self).__init__(
		batch_size, batch_timeout, poll_style_listener.prefetch_size
	)
	self._poll_style_listener = poll_style_listener
	self._listen_thread = threading.Thread(target=self._runner)
	self._listen_thread.daemon = True
	self._started = False
```
该类启动了一个线程，我们看看这个现在主要做了什么事情：
```python
def _runner(self):
	while self._started:
		incoming = self._poll_style_listener.poll(
			batch_size=self.batch_size, batch_timeout=self.batch_timeout)

		if incoming:
			self.on_incoming_callback(incoming)

	# listener is stopped but we need to process all already consumed
	# messages
	while True:
		incoming = self._poll_style_listener.poll(
			batch_size=self.batch_size, batch_timeout=self.batch_timeout)

		if not incoming:
			return
		self.on_incoming_callback(incoming)

def poll(self, timeout=None):
	stopwatch = timeutils.StopWatch(duration=timeout).start()

	while not self._shutdown.is_set():
		self._message_operations_handler.process()

		if self.incoming:
			return self.incoming.pop(0)

		left = stopwatch.leftover(return_none=True)
		if left is None:
			left = self._current_timeout
		if left <= 0:
			return None

		try:
			self.conn.consume(timeout=min(self._current_timeout, left))
		except rpc_common.Timeout:
			self._current_timeout = max(self._current_timeout * 2,
										ACK_REQUEUE_EVERY_SECONDS_MAX)
		else:
			self._current_timeout = ACK_REQUEUE_EVERY_SECONDS_MIN

	# NOTE(sileht): listener is stopped, just processes remaining messages
	# and operations
	self._message_operations_handler.process()
	if self.incoming:
		return self.incoming.pop(0)

	self._shutoff.set()
```
从上面代码可以看出，主要就是不断的拉取rpc请求消息，然后消费。
上面的on_incoming_callback怎么来的呢？是通过start传递进来的：
```python
def start(self, on_incoming_callback):
	super(PollStyleListenerAdapter, self).start(on_incoming_callback)
	self._started = True
	self._listen_thread.start()
```
这个start就是上面rpc启动的时候调用的：
```python
self.listener.start(self._on_incoming)
```

从上面我们知道监听器再监控到一个rpc请求时，会回调方法_on_incoming：
```python
def _on_incoming(self, incoming):
	"""Handles on_incoming event

	:param incoming: incoming request.
	"""
	self._work_executor.submit(self._process_incoming, incoming)
```
经过层层调用，最后进入到了这个方法中：
```python
def _process_incoming(self, incoming):
	message = incoming[0]

	# TODO(sileht): We should remove that at some point and do
	# this directly in the driver
	try:
		message.acknowledge()
	except Exception:
		LOG.exception("Can not acknowledge message. Skip processing")
		return

	failure = None
	try:
		res = self.dispatcher.dispatch(message)
	except rpc_dispatcher.ExpectedException as e:
		failure = e.exc_info
		LOG.debug(u'Expected exception during message handling (%s)', e)
	except Exception:
		# current sys.exc_info() content can be overridden
		# by another exception raised by a log handler during
		# LOG.exception(). So keep a copy and delete it later.
		failure = sys.exc_info()
		LOG.exception('Exception during message handling')

	try:
		if failure is None:
			message.reply(res)
		else:
			message.reply(failure=failure)
	except Exception:
		LOG.exception("Can not send reply for message")
	finally:
			# NOTE(dhellmann): Remove circular object reference
			# between the current stack frame and the traceback in
			# exc_info.
			del failure
```
这个方法就是在调用分发器，由请求分发器执行相应的请求方法，然后将结果返回给rpc发送端。

### 请求分发器
```python
def dispatch(self, incoming):
	"""Dispatch an RPC message to the appropriate endpoint method.

	:param incoming: incoming message
	:type incoming: IncomingMessage
	:raises: NoSuchMethod, UnsupportedVersion
	"""
	message = incoming.message
	ctxt = incoming.ctxt

	method = message.get('method')
	args = message.get('args', {})
	namespace = message.get('namespace')
	version = message.get('version', '1.0')

	# NOTE(danms): This event and watchdog thread are used to send
	# call-monitoring heartbeats for this message while the call
	# is executing if it runs for some time. The thread will wait
	# for the event to be signaled, which we do explicitly below
	# after dispatching the method call.
	completion_event = threading.Event()
	watchdog_thread = threading.Thread(target=self._watchdog,
									   args=(completion_event, incoming))
	if incoming.client_timeout:
		# NOTE(danms): The client provided a timeout, so we start
		# the watchdog thread. If the client is old or didn't send
		# a timeout, we just never start the watchdog thread.
		watchdog_thread.start()

	found_compatible = False
	for endpoint in self.endpoints:
		target = getattr(endpoint, 'target', None)
		if not target:
			target = self._default_target

		if not (self._is_namespace(target, namespace) and
				self._is_compatible(target, version)):
			continue

		if hasattr(endpoint, method):
			if self.access_policy.is_allowed(endpoint, method):
				try:
					return self._do_dispatch(endpoint, method, ctxt, args)
				finally:
					completion_event.set()
					if incoming.client_timeout:
						watchdog_thread.join()

		found_compatible = True

	if found_compatible:
		raise NoSuchMethod(method)
	else:
		raise UnsupportedVersion(version, method=method)
```
上面的代码非常明了了，就是从rpc的client请求中取出消息message，这个message的数据结构就是与文章开头我们回顾的client发送逻辑一一对应的。
我们注意到只有这样一句_do_dispatch：
```python
def _do_dispatch(self, endpoint, method, ctxt, args):
	ctxt = self.serializer.deserialize_context(ctxt)
	new_args = dict()
	for argname, arg in args.items():
		new_args[argname] = self.serializer.deserialize_entity(ctxt, arg)
	func = getattr(endpoint, method)
	result = func(ctxt, **new_args)
	return self.serializer.serialize_entity(ctxt, result)
```
到了这里就很清楚了，远程调用就是在这里触发的。