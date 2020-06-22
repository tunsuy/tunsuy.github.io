# Kazoo源码剖析

## 启动ZK连接
调用方可以如下创建一个zk连接：
```python
zookeeper = KazooClient(hosts=get_address(address), timeout=timeout,
                            client_id=client_id, handler=handler,
                            default_acl=default_acl, auth_data=auth_data,
                            read_only=read_only,
                            randomize_hosts=randomize_hosts,
                            connection_retry=con_retry,
                            command_retry=command_retry, logger=logger)
  
try:
	scheme, credential = kazooACL.get_auth_acl()
	zookeeper.start(timeout=90)
	zookeeper.add_auth(scheme, credential)
	zookeeper.default_acl = (kazooACL.get_set_acl_val(),)
	return zookeeper
except Exception:
	zookeeper.stop()
    zookeeper.close()
	raise
```

### KazooClient类
上面首先实例化了一个KazooClient对象，
实例化KazooClient对象的时候，生成了一个ConnectionHandler实例
```python
self._connection = ConnectionHandler(
            self, self._conn_retry.copy(), logger=self.logger)
```

初始化了这个连接对象，进入该连接对象定义：
kazoo.protocol.connection.py
```python
class ConnectionHandler(object):
    """Zookeeper connection handler"""
    def __init__(self, client, retry_sleeper, logger=None):
        self.client = client
        self.handler = client.handler
        self.retry_sleeper = retry_sleeper
        self.logger = logger or log

        # Our event objects
        self.connection_closed = client.handler.event_object()
        self.connection_closed.set()
        self.connection_stopped = client.handler.event_object()
        self.connection_stopped.set()
        self.ping_outstanding = client.handler.event_object()

        self._read_sock = None
        self._write_sock = None

        self._socket = None
        self._xid = None
        self._rw_server = None
        self._ro_mode = False
        self._ro = False

        self._connection_routine = None

        self.sasl_cli = None
```

### 启动zk连接
再启动ZKClient的时候，也就是调用了该ConnectionHandler对象的start方法：
```python
def start(self):
	"""Start the connection up"""
	if self.connection_closed.is_set():
		rw_sockets = self.handler.create_socket_pair()
		self._read_sock, self._write_sock = rw_sockets
		self.connection_closed.clear()
	if self._connection_routine:
		raise Exception("Unable to start, connection routine already "
						"active.")
	self._connection_routine = self.handler.spawn(self.zk_loop)
```

这里是新起了一个协程进行执行轮训：
```python
def zk_loop(self):
	"""Main Zookeeper handling loop"""
	self.logger.log(BLATHER, 'ZK loop started')

	self.connection_stopped.clear()

	retry = self.retry_sleeper.copy()
	try:
		while not self.client._stopped.is_set():
			# If the connect_loop returns STOP_CONNECTING, stop retrying
			if retry(self._connect_loop, retry) is STOP_CONNECTING:
				break
	except RetryFailedError:
		self.logger.warning("Failed connecting to Zookeeper "
							"within the connection retry policy.")
	finally:
		self.connection_stopped.set()
		self.client._session_callback(KeeperState.CLOSED)
		self.logger.log(BLATHER, 'Connection stopped')

def _connect_loop(self, retry):
	# Iterate through the hosts a full cycle before starting over
	status = None
	host_ports = self._expand_client_hosts()

	# Check for an empty hostlist, indicating none resolved
	if len(host_ports) == 0:
		return STOP_CONNECTING

	for host, port in host_ports:
		if self.client._stopped.is_set():
			status = STOP_CONNECTING
			break
		status = self._connect_attempt(host, port, retry)
		if status is STOP_CONNECTING:
			break

	if status is STOP_CONNECTING:
		return STOP_CONNECTING
	else:
		raise ForceRetryError('Reconnecting')
```

### 连接状态变更
```python
def _connect_attempt(self, host, port, retry):
	client = self.client
	KazooTimeoutError = self.handler.timeout_exception
	close_connection = False

	self._socket = None

	# Were we given a r/w server? If so, use that instead
	if self._rw_server:
		self.logger.log(BLATHER,
						"Found r/w server to use, %s:%s", host, port)
		host, port = self._rw_server
		self._rw_server = None

	if client._state != KeeperState.CONNECTING:
		client._session_callback(KeeperState.CONNECTING)

	try:
		self._xid = 0
		read_timeout, connect_timeout = self._connect(host, port)
		read_timeout = read_timeout / 1000.0
		connect_timeout = connect_timeout / 1000.0
		retry.reset()
		self.ping_outstanding.clear()
		with self._socket_error_handling():
			while not close_connection:
				# Watch for something to read or send
				jitter_time = random.randint(0, 40) / 100.0
				# Ensure our timeout is positive
				timeout = max([read_timeout / 2.0 - jitter_time,
							   jitter_time])
				s = self.handler.select([self._socket, self._read_sock],
										[], [], timeout)[0]

				if not s:
					if self.ping_outstanding.is_set():
						self.ping_outstanding.clear()
						raise ConnectionDropped(
							"outstanding heartbeat ping not received")
					self._send_ping(connect_timeout)
				elif s[0] == self._socket:
					response = self._read_socket(read_timeout)
					close_connection = response == CLOSE_RESPONSE
				else:
					self._send_request(read_timeout, connect_timeout)
		self.logger.info('Closing connection to %s:%s', host, port)
		client._session_callback(KeeperState.CLOSED)
		return STOP_CONNECTING
	except (ConnectionDropped, KazooTimeoutError) as e:
		if isinstance(e, ConnectionDropped):
			self.logger.warning('Connection dropped: %s', e)
		else:
			self.logger.warning('Connection time-out: %s', e)
		if client._state != KeeperState.CONNECTING:
			self.logger.warning("Transition to CONNECTING")
			client._session_callback(KeeperState.CONNECTING)
	except AuthFailedError:
		retry.reset()
		self.logger.warning('AUTH_FAILED closing')
		client._session_callback(KeeperState.AUTH_FAILED)
		return STOP_CONNECTING
	except SessionExpiredError:
		retry.reset()
		self.logger.warning('Session has expired')
		client._session_callback(KeeperState.EXPIRED_SESSION)
	except RWServerAvailable:
		retry.reset()
		self.logger.warning('Found a RW server, dropping connection')
		client._session_callback(KeeperState.CONNECTING)
	except Exception:
		self.logger.exception('Unhandled exception in connection loop')
		raise
	finally:
		if self._socket is not None:
			self._socket.close()
```
当连接的状态变化的时候，都会调用`client._session_callback`回调方法

## 监听器
zk的客户端持有一个监听器的集合属性
```python
class KazooClient


self.state_listeners = set()
```

### 增加listener
```python
def add_listener(self, listener):
	"""Add a function to be called for connection state changes.

	This function will be called with a
	:class:`~kazoo.protocol.states.KazooState` instance indicating
	the new connection state on state transitions.

	.. warning::

		This function must not block. If its at all likely that it
		might need data or a value that could result in blocking
		than the :meth:`~kazoo.interfaces.IHandler.spawn` method
		should be used so that the listener can return immediately.

	"""
	if not (listener and callable(listener)):
		raise ConfigurationError("listener must be callable")
	self.state_listeners.add(listener)
```

### 执行listener
从上面的分析我们知道，在client跟zk建立连接之后，client会监控session的状态
```python
def _session_callback(self, state):
	if state == self._state:
		return

	# Note that we don't check self.state == LOST since that's also
	# the client's initial state
	dead_state = self._state in LOST_STATES
	self._state = state

	# If we were previously closed or had an expired session, and
	# are now connecting, don't bother with the rest of the
	# transitions since they only apply after
	# we've established a connection
	if dead_state and state == KeeperState.CONNECTING:
		self.logger.log(BLATHER, "Skipping state change")
		return

	if state in (KeeperState.CONNECTED, KeeperState.CONNECTED_RO):
		self.logger.info("Zookeeper connection established, "
						 "state: %s", state)
		self._live.set()
		self._make_state_change(KazooState.CONNECTED)
	elif state in LOST_STATES:
		self.logger.info("Zookeeper session lost, state: %s", state)
		self._live.clear()
		self._make_state_change(KazooState.LOST)
		self._notify_pending(state)
		self._reset()
	else:
		self.logger.info("Zookeeper connection lost")
		# Connection lost
		self._live.clear()
		self._notify_pending(state)
		self._make_state_change(KazooState.SUSPENDED)
		self._reset_watchers()
```

每个状态的变更都调用了`_make_state_change`方法
```python
def _make_state_change(self, state):
	# skip if state is current
	if self.state == state:
		return

	self.state = state

	# Create copy of listeners for iteration in case one needs to
	# remove itself
	for listener in list(self.state_listeners):
		try:
			remove = listener(state)
			if remove is True:
				self.remove_listener(listener)
		except Exception:
			self.logger.exception("Error in connection state listener")
```
该方法遍历所有的listener，然后执行每个回调方法，比如，DataWatcher的get_data方法

## Watcher
### DataWatcher

```python
class DataWatch(object):
	def __init__(self, client, path, func=None, *args, **kwargs):
        """Create a data watcher for a path

        :param client: A zookeeper client.
        :type client: :class:`~kazoo.client.KazooClient`
        :param path: The path to watch for data changes on.
        :type path: str
        :param func: Function to call initially and every time the
                     node changes. `func` will be called with a
                     tuple, the value of the node and a
                     :class:`~kazoo.client.ZnodeStat` instance.
        :type func: callable

        """
        self._client = client
        self._path = path
        self._func = func
        self._stopped = False
        self._run_lock = client.handler.lock_object()
        self._version = None
        self._retry = KazooRetry(max_tries=None,
                                 sleep_func=client.handler.sleep_func)
        self._include_event = None
        self._ever_called = False
        self._used = False

        if args or kwargs:
            warnings.warn('Passing additional arguments to DataWatch is'
                          ' deprecated. ignore_missing_node is now assumed '
                          ' to be True by default, and the event will be '
                          ' sent if the function can handle receiving it',
                          DeprecationWarning, stacklevel=2)

        # Register our session listener if we're going to resume
        # across session losses
        if func is not None:
            self._used = True
            self._client.add_listener(self._session_watcher)
            self._get_data()
```
在实例化该watch的时候，就会通过`_get_data`方法通知下所有的客户端执行

### 增加watcher
```python
@_ignore_closed
def _get_data(self, event=None):
	# Ensure this runs one at a time, possible because the session
	# watcher may trigger a run
	with self._run_lock:
		if self._stopped:
			return

		initial_version = self._version

		try:
			data, stat = self._retry(self._client.get,
									 self._path, self._watcher)
		except NoNodeError:
			data = None
```

这里调用了KazooClient的get方法，进而调用`get_async`
```python
def get_async(self, path, watch=None):
	"""Asynchronously get the value of a node. Takes the same
	arguments as :meth:`get`.

	:rtype: :class:`~kazoo.interfaces.IAsyncResult`

	"""
	if not isinstance(path, string_types):
		raise TypeError("Invalid type for 'path' (string expected)")
	if watch and not callable(watch):
		raise TypeError("Invalid type for 'watch' (must be a callable)")

	async_result = self.handler.async_result()
	self._call(GetData(_prefix_root(self.chroot, path), watch),
			   async_result)
	return async_result
```

这里将path、watch封装成了一个对象`GetData`，也就是下面的`request`
```python
def _call(self, request, async_object):
	"""Ensure there's an active connection and put the request in
	the queue if there is.

	Returns False if the call short circuits due to AUTH_FAILED,
	CLOSED, EXPIRED_SESSION or CONNECTING state.

	"""

	if self._state == KeeperState.AUTH_FAILED:
		async_object.set_exception(AuthFailedError())
		return False
	elif self._state == KeeperState.CLOSED:
		async_object.set_exception(ConnectionClosedError(
			"Connection has been closed"))
		return False
	elif self._state in (KeeperState.EXPIRED_SESSION,
						 KeeperState.CONNECTING):
		async_object.set_exception(SessionExpiredError())
		return False

	self._queue.append((request, async_object))

	# wake the connection, guarding against a race with close()
	write_sock = self._connection._write_sock
	if write_sock is None:
		async_object.set_exception(ConnectionClosedError(
			"Connection has been closed"))
	try:
		write_sock.send(b'\0')
	except:
		async_object.set_exception(ConnectionClosedError(
			"Connection has been closed"))
```
这里将watcher放入了队列中。

### 执行watcher
```python
def _send_request(self, read_timeout, connect_timeout):
	"""Called when we have something to send out on the socket"""
	client = self.client
	try:
		request, async_object = client._queue[0]
	except IndexError:
		# Not actually something on the queue, this can occur if
		# something happens to cancel the request such that we
		# don't clear the socket below after sending
		try:
			# Clear possible inconsistence (no request in the queue
			# but have data in the read socket), which causes cpu to spin.
			self._read_sock.recv(1)
		except OSError:
			pass
		return

	# Special case for testing, if this is a _SessionExpire object
	# then throw a SessionExpiration error as if we were dropped
	if request is _SESSION_EXPIRED:
		raise SessionExpiredError("Session expired: Testing")
	if request is _CONNECTION_DROP:
		raise ConnectionDropped("Connection dropped: Testing")

	# Special case for auth packets
	if request.type == Auth.type:
		xid = AUTH_XID
	else:
		self._xid = (self._xid % 2147483647) + 1
		xid = self._xid

	self._submit(request, connect_timeout, xid)
	client._queue.popleft()
	self._read_sock.recv(1)
	client._pending.append((request, async_object, xid))
```



## handler
kazoo.handlers.threading
默认为`SequentialThreadingHandler`
顺序执行回调的线程执行器，为每一个回调时间创建队列，它们分为两个队列，一个用于监视事件，一个用于异步结果完成回调
每种队列类型都有一个线程工作程序，该工作程序将回调事件从队列中拉出并按客户端看到的顺序运行。这种拆分有助于确保在Zookeeper客户端执行期间连接断开的情况下，watch回调不会阻止会话重建。
监视和完成回调应避免阻塞行为。如果需要阻止，请生成一个新线程并立即返回，以便继续进行回调。

### 队列
1、callback_queue
2、completion_queue

#### 放入队列
当连接的socket通道中收到请求时
```python
def _read_socket(self, read_timeout):
	"""Called when there's something to read on the socket"""
	client = self.client

	header, buffer, offset = self._read_header(read_timeout)
	if header.xid == PING_XID:
		self.logger.log(BLATHER, 'Received Ping')
		self.ping_outstanding.clear()
	elif header.xid == AUTH_XID:
		self.logger.log(BLATHER, 'Received AUTH')

		request, async_object, xid = client._pending.popleft()
		if header.err:
			async_object.set_exception(AuthFailedError())
			client._session_callback(KeeperState.AUTH_FAILED)
		else:
			async_object.set(True)
	elif header.xid == WATCH_XID:
		self._read_watch_event(buffer, offset)
```

这里根据请求header里面的xid判断，当xid为WATCH_XID的时候
```python
def _read_watch_event(self, buffer, offset):
	client = self.client
	watch, offset = Watch.deserialize(buffer, offset)
	path = watch.path

	self.logger.debug('Received EVENT: %s', watch)

	watchers = []

	if watch.type in (CREATED_EVENT, CHANGED_EVENT):
		watchers.extend(client._data_watchers.pop(path, []))
	elif watch.type == DELETED_EVENT:
		watchers.extend(client._data_watchers.pop(path, []))
		watchers.extend(client._child_watchers.pop(path, []))
	elif watch.type == CHILD_EVENT:
		watchers.extend(client._child_watchers.pop(path, []))
	else:
		self.logger.warn('Received unknown event %r', watch.type)
		return

	# Strip the chroot if needed
	path = client.unchroot(path)
	ev = WatchedEvent(EVENT_TYPE_MAP[watch.type], client._state, path)

	# Last check to ignore watches if we've been stopped
	if client._stopped.is_set():
		return

	# Dump the watchers to the watch thread
	for watch in watchers:
		client.handler.dispatch_callback(Callback('watch', watch, (ev,)))

		
def dispatch_callback(self, callback):
	self.callback_queue.put(lambda: callback.func(*callback.args))
```
从这里我们就可以看到，将watcher放到了`callback_queue`队列中

#### 取出队列
在实例化KazooClient的时候
```python
# Record the handler strategy used
self.handler = handler if handler else SequentialThreadingHandler()
if inspect.isclass(self.handler):
	raise ConfigurationError("Handler must be an instance of a class, "
							 "not the class: %s" % self.handler)
```
这里支持调用方自定义Handler并传入进来，默认采用`SequentialThreadingHandler`处理器

自动KazooClient的时候
```python
def start(self, timeout=15):
	"""Initiate connection to ZK.

	:param timeout: Time in seconds to wait for connection to
					succeed.
	:raises: :attr:`~kazoo.interfaces.IHandler.timeout_exception`
			 if the connection wasn't established within `timeout`
			 seconds.

	"""
	event = self.start_async()
	event.wait(timeout=timeout)
	if not self.connected:
		# We time-out, ensure we are disconnected
		self.stop()
		raise self.handler.timeout_exception("Connection time-out")

	if self.chroot and not self.exists("/"):
		warnings.warn("No chroot path exists, the chroot path "
					  "should be created before normal use.")


def start_async(self):
	"""Asynchronously initiate connection to ZK.

	:returns: An event object that can be checked to see if the
			  connection is alive.
	:rtype: :class:`~threading.Event` compatible object.

	"""
	# If we're already connected, ignore
	if self._live.is_set():
		return self._live

	# Make sure we're safely closed
	self._safe_close()

	# We've been asked to connect, clear the stop and our writer
	# thread indicator
	self._stopped.clear()
	self._writer_stopped.clear()

	# Start the handler
	self.handler.start()

	# Start the connection
	self._connection.start()
	return self._live
```

这里会调用handler的start方法，也就是`SequentialThreadingHandler`的start方法
```python
def start(self):
	"""Start the worker threads."""
	with self._state_change:
		if self._running:
			return

		# Spawn our worker threads, we have
		# - A callback worker for watch events to be called
		# - A completion worker for completion events to be called
		for queue in (self.completion_queue, self.callback_queue):
			w = self._create_thread_worker(queue)
			self._workers.append(w)
		self._running = True
		python2atexit.register(self.stop)
```
这里我们就看到，会对队列中的每一个watcher回调方法，启动一个协程进行执行。

## KazooRetry
当kazoo与zk服务端的请求中，出现以下几种异常时会一直重试
```python
RETRY_EXCEPTIONS = (
        ConnectionLoss,
        OperationTimeoutError,
        ForceRetryError
)

EXPIRED_EXCEPTIONS = (
        SessionExpiredError,
)
```

其中，session过期异常时可选的重试，根据实例化的参数决定
```python
self.retry_exceptions = self.RETRY_EXCEPTIONS
self.interrupt = interrupt
if ignore_expire:
	self.retry_exceptions += self.EXPIRED_EXCEPTIONS
	
def __call__(self, func, *args, **kwargs):
	"""Call a function with arguments until it completes without
	throwing a Kazoo exception

	:param func: Function to call
	:param args: Positional arguments to call the function with
	:params kwargs: Keyword arguments to call the function with

	The function will be called until it doesn't throw one of the
	retryable exceptions (ConnectionLoss, OperationTimeout, or
	ForceRetryError), and optionally retrying on session
	expiration.

	"""
	self.reset()

	while True:
		try:
			if self.deadline is not None and self._cur_stoptime is None:
				self._cur_stoptime = time.time() + self.deadline
			return func(*args, **kwargs)
		except ConnectionClosedError:
			raise
		except self.retry_exceptions:
			# Note: max_tries == -1 means infinite tries.
			if self._attempts == self.max_tries:
				raise RetryFailedError("Too many retry attempts")
			self._attempts += 1
			sleeptime = random.randint(0, 1 + int(self._cur_delay))

			if self._cur_stoptime is not None and \
			   time.time() + sleeptime >= self._cur_stoptime:
				raise RetryFailedError("Exceeded retry deadline")

			if self.interrupt:
				while sleeptime > 0:
					# Break the time period down and sleep for no
					# longer than 0.1 before calling the interrupt
					if sleeptime < 0.1:
						self.sleep_func(sleeptime)
						sleeptime -= sleeptime
					else:
						self.sleep_func(0.1)
						sleeptime -= 0.1
					if self.interrupt():
						raise InterruptedError()
			else:
				self.sleep_func(sleeptime)
			self._cur_delay = min(self._cur_delay * self.backoff,
								  self.max_delay)

```

## Lock
kazoo提供了zookeeper作为分布式锁的实现，也就是下面的Lock类
```python
class Lock(object):
    """Kazoo Lock

    Example usage with a :class:`~kazoo.client.KazooClient` instance:

    .. code-block:: python

        zk = KazooClient()
        zk.start()
        lock = zk.Lock("/lockpath", "my-identifier")
        with lock:  # blocks waiting for lock acquisition
            # do something with the lock

    Note: This lock is not *re-entrant*. Repeated calls after already
    acquired will block.

    This is an exclusive lock. For a read/write lock, see :class:`WriteLock`
    and :class:`ReadLock`.

    """
```


## AsyncResult

### 条件锁
条件同步机制是指：一个线程等待特定条件，而另一个线程发出特定条件满足的信号。 
解释条件同步机制的一个很好的例子就是生产者/消费者（producer/consumer）模型。生产者随机的往列表中“生产”一个随机整数，而消费者从列表中“消费”整数。

## 优先队列


### https://zhuanlan.zhihu.com/p/84836942