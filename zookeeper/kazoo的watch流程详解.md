# kazoo的watch流程详解

## 前言
关于watch，zk做如下保证：
* 1、atch是针对其他事件、其他watch和异步答复而排序的。 ZooKeeper客户端库可确保按顺序分派所有内容。
* 2、客户端将看到它正在监视的znode的watch事件，然后才能看到与该znode对应的新数据。
* 3、ZooKeeper中监视事件的顺序与ZooKeeper服务所看到的更新顺序相对应。

注意事项：
* 1、watch是一次触发。如果收到监视事件，并且希望收到有关将来更改的通知，则必须设置另一个watch。
* 2、由于监视是一次触发，并且在获取事件和发送新请求以获取新的watch之间存在延迟，因此无法可靠地看到ZooKeeper中节点发生的每项更改。需要注意znode在获取事件和重新设置watch之间多次更改的情况。 （你可能不在乎，但至少意识到可能会发生。）
* 3、对于给定的通知、watch对象或功能/上下文将仅触发一次。例如，如果为一个文件注册了相同的watch对象，并且对同一文件进行了getData调用，然后删除了该文件，则watch对象将仅在该文件被删除时被调用一次。
* 4、与服务器断开连接时（例如，服务器发生故障时），直到重新建立连接后，才能获得任何watch。因此，会话事件将发送到所有的监视处理程序。这种情况下，会话事件将进入安全模式：断开连接后，将不会收到事件，因此进程应在该模式下谨慎行事。

### watcher重连

* 1.和server主动关闭连接一样，client抛出EndOfStreamException异常，此时客户端状态还是CONNECTED
* 2.SendThread处理异常，清理连接，将当前所有请求置为失败，错误码是CONNECTIONLOSS
* 3.发送Disconnected状态通知
* 4.选下一个server重连
* 5.连上之后发送ConnectRequest，sessionid和password是当前session的数据
* 6.server端处理，分leader和follower，由于此时client端重试比较快，session还没超时，所以leader和follower端session校验成功。如果这个时候session正好超时了，则校验失败，client会抛出sessionExpired异常并退出
* 7.server端返回成功的ConnectResponse
* 8.client收到相应，发送SyncConnected状态通知给watcher
* 9.client发送SetWatches包，重建watch



以下是基于zookeeper的python客户端kazoo进行讲解

## 注册监控
通过kazoo提供的装饰器进行注册
```python
@zookeeper.DataWatch(path)
def changed(data, stat):
	fun(data)
	type(stat)
```

简单看下这个DataWatch类，在实例化这个类的时候
```python
def __init__(xxx):
	if func is not None:
		self._used = True
		self._client.add_listener(self._session_watcher)
		self._get_data()
```

通过方法来到这里:
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
根据我在zk的通信协议中提到的，GetData请求的参数中如果watch为1，则表示客户端希望收到zk的数据监控回调
而这里就是带了watch=1.

## 执行回调
```python
def _read_socket(self, read_timeout):
	"""Called when there's something to read on the socket"""
	client = self.client

	header, buffer, offset = self._read_header(read_timeout)
	if header.xid == PING_XID:
		...
	elif header.xid == WATCH_XID:
		self._read_watch_event(buffer, offset)
	...

		return self._read_response(header, buffer, offset)
```

通过读取socket字节流，并解析字节流，根据xid判断是否是watch回调请求，
也即是上面所说的，在我们向zk注册了节点监控以后，后续的GetData请求，只要数据有变化，zk返回的response中的header的xid就是watch类型
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
```

这里通过遍历`_data_watchers`或者`_child_watchers`中的监控回调
将其加入回调队列中等待执行
`注：那这两个队列的元素是怎么来的呢？在下一节的再次注册监控可以看到。`
```python
def dispatch_callback(self, callback):
	"""Dispatch to the callback object

	The callback is put on separate queues to run depending on the
	type as documented for the :class:`SequentialThreadingHandler`.

	"""
	self.callback_queue.put(lambda: callback.func(*callback.args))
```

通过在之前的文章中介绍到的，这个`callback_queue`队列是在client的handler中轮训遍历执行的
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

def _create_thread_worker(self, queue):
	def _thread_worker():  # pragma: nocover
		while True:
			try:
				func = queue.get()
				try:
					if func is _STOP:
						break
					func()
				except Exception:
					log.exception("Exception in worker queue thread")
				finally:
					queue.task_done()
			except self.queue_empty:
				continue
	t = self.spawn(_thread_worker)
	return t
```
这个handler是在客户端跟zk建立连接的时候，启动的一个轮训线程：不断的从队列中取出func进行执行。

## 再次注册监控
因为zk的watch是一次性的，所以在当次watch回调执行完之后，想要在zk的节点再次变更的时候被通知，需要再次注册监控

```python
def _read_response(self, header, buffer, offset):
	client = self.client
	request, async_object, xid = client._pending.popleft()
	if header.zxid and header.zxid > 0:
		client.last_zxid = header.zxid
	if header.xid != xid:
		exc = RuntimeError('xids do not match, expected %r '
						   'received %r', xid, header.xid)
		async_object.set_exception(exc)
		raise exc

	# Determine if its an exists request and a no node error
	exists_error = (header.err == NoNodeError.code and
					request.type == Exists.type)

	# Set the exception if its not an exists error
	if header.err and not exists_error:
		callback_exception = EXCEPTIONS[header.err]()
		self.logger.debug(
			'Received error(xid=%s) %r', xid, callback_exception)
		if async_object:
			async_object.set_exception(callback_exception)
	elif request and async_object:
		if exists_error:
			# It's a NoNodeError, which is fine for an exists
			# request
			async_object.set(None)
		else:
			try:
				response = request.deserialize(buffer, offset)
			except Exception as exc:
				self.logger.exception(
					"Exception raised during deserialization "
					"of request: %s", request)
				async_object.set_exception(exc)
				return
			self.logger.debug(
				'Received response(xid=%s): %r', xid, response)

			# We special case a Transaction as we have to unchroot things
			if request.type == Transaction.type:
				response = Transaction.unchroot(client, response)

			async_object.set(response)

		# Determine if watchers should be registered
		watcher = getattr(request, 'watcher', None)
		if not client._stopped.is_set() and watcher:
			if isinstance(request, (GetChildren, GetChildren2)):
				client._child_watchers[request.path].add(watcher)
			else:
				client._data_watchers[request.path].add(watcher)

	if isinstance(request, Close):
		self.logger.log(BLATHER, 'Read close response')
		return CLOSE_RESPONSE
```
从上面的代码可以看出：
1、如果接口正常返回，kazoo会执行注册到该节点上的回调函数，并且会重新将watcher放入监控集合中，等待下次节点变更再次调度。
2、如果接口发生错误，则不会执行回调函数，也不会再将watcher放入集合中，这就导致以后zk的路径节点变更，监控函数都不会再执行。

## 执行方式
kazoo提供了两种请求方式，一种是异步执行，一种是同步执行

异步类
```python
class AsyncResult(object):
    """A one-time event that stores a value or an exception"""
    def __init__(self, handler, condition_factory, timeout_factory):
        self._handler = handler
        self._exception = _NONE
        self._condition = condition_factory()
        self._callbacks = []
        self._timeout_factory = timeout_factory
        self.value = None

    ...
	
    def set(self, value=None):
        """Store the value. Wake up the waiters."""
        with self._condition:
            self.value = value
            self._exception = None
            for callback in self._callbacks:
                self._handler.completion_queue.put(
                    functools.partial(callback, self)
                )
            self._condition.notify_all()

	...
	
    def get(self, block=True, timeout=None):
        """Return the stored value or raise the exception.

        If there is no value raises TimeoutError.

        """
        with self._condition:
            if self._exception is not _NONE:
                if self._exception is None:
                    return self.value
                raise self._exception
            elif block:
                self._condition.wait(timeout)
                if self._exception is not _NONE:
                    if self._exception is None:
                        return self.value
                    raise self._exception

            # if we get to this point we timeout
            raise self._timeout_factory()
```

根据子类的不同，这里的`_condition`对象也就不同，下面以threading为例：
这里使用了python的Condition，Condition对象提供了对复杂线程同步问题的支持。Condition被称为条件变量，除了提供与Lock类似的acquire和release方法外，还提供了wait和notify方法。
线程首先acquire一个条件变量，然后判断一些条件。如果条件不满足则wait；如果条件满足，进行一些处理改变条件后，通过notify方法通知其他线程，其他处于wait状态的线程接到通知后会重新判断条件。不断的重复这一过程，从而解决复杂的同步问题。

该类提供了下面几种主要方法：

* wait():线程挂起，直到收到一个notify通知才会被唤醒继续运行
* notify():通知其他线程，那些挂起的线程接到这个通知之后会开始运行
* notify_all(): 如果wait状态线程比较多，notifyAll的作用就是通知所有线程（这个一般用得少）

在同步的请求中，就是调用了`AsyncResult`的get方法，从而阻塞起来
```python
self.get_async(path, watch=watch).get()
```
也就是调用了异步方法，通过执行异步方法返回的`AsyncResult`对象，调用其get方法达到阻塞的效果

##### 那是在哪里收到通知的呢？

```python
def _read_response(self, header, buffer, offset):
	...
	try:
		response = request.deserialize(buffer, offset)
	except Exception as exc:
		self.logger.exception(
			"Exception raised during deserialization "
			"of request: %s", request)
		async_object.set_exception(exc)
		return
	self.logger.debug(
		'Received response(xid=%s): %r', xid, response)

	# We special case a Transaction as we have to unchroot things
	if request.type == Transaction.type:
		response = Transaction.unchroot(client, response)

	async_object.set(response)
	...
```
就是在请求的response中，调用了`AsyncResult`对象的set方法，从而使get方法得到结果返回。

#### 注：这里的`async_object`，请求回复一定要是同一个对象，这也就是kazoo在创建请求的时候，生成了一个`async_object`对象，并将其同`request`对象一起放入队列的原因。