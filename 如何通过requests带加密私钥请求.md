# 如何通过requests带加密私钥请求

## 前序
在解决这个问题之前，我们需要弄清楚requests库或者urllib3库的http请求流程

request的使用
```python
requests.request(method, final_url, verify=verify,
                 headers=_headers, **args)
```

通过requests.api.request->requests.session.request->requests.session.send
```python
def send(self, request, **kwargs):
	"""Send a given PreparedRequest.

	:rtype: requests.Response
	"""
	# Set defaults that the hooks can utilize to ensure they always have
	# the correct parameters to reproduce the previous request.
	kwargs.setdefault('stream', self.stream)
	kwargs.setdefault('verify', self.verify)
	kwargs.setdefault('cert', self.cert)
	kwargs.setdefault('proxies', self.proxies)

	# It's possible that users might accidentally send a Request object.
	# Guard against that specific failure case.
	if isinstance(request, Request):
		raise ValueError('You can only send PreparedRequests.')

	# Set up variables needed for resolve_redirects and dispatching of hooks
	allow_redirects = kwargs.pop('allow_redirects', True)
	stream = kwargs.get('stream')
	hooks = request.hooks

	# Get the appropriate adapter to use
	adapter = self.get_adapter(url=request.url)

	# Start time (approximately) of the request
	start = preferred_clock()

	# Send the request
	r = adapter.send(request, **kwargs)
```

注意这里的get_adapter方法：
```python
def get_adapter(self, url):
	"""
	Returns the appropriate connection adapter for the given URL.

	:rtype: requests.adapters.BaseAdapter
	"""
	for (prefix, adapter) in self.adapters.items():

		if url.lower().startswith(prefix.lower()):
			return adapter

	# Nothing matches :-/
	raise InvalidSchema("No connection adapters were found for '%s'" % url)
```
这里的self.adapters实在实例化session时赋值的：
```python
class Session():
	def __init__():
	# Default connection adapters.
	self.adapters = OrderedDict()
	self.mount('https://', HTTPAdapter())
	self.mount('http://', HTTPAdapter())
```

进入到这个adapter
requests.adapters.HTTPAdapter
```python
def get_connection(self, url, proxies=None):
	"""Returns a urllib3 connection for the given URL. This should not be
	called from user code, and is only exposed for use when subclassing the
	:class:`HTTPAdapter <requests.adapters.HTTPAdapter>`.

	:param url: The URL to connect to.
	:param proxies: (optional) A Requests-style dictionary of proxies used on this request.
	:rtype: urllib3.ConnectionPool
	"""
	proxy = select_proxy(url, proxies)

	if proxy:
		proxy = prepend_scheme_if_needed(proxy, 'http')
		proxy_url = parse_url(proxy)
		if not proxy_url.host:
			raise InvalidProxyURL("Please check proxy URL. It is malformed"
								  " and could be missing the host.")
		proxy_manager = self.proxy_manager_for(proxy)
		conn = proxy_manager.connection_from_url(url)
	else:
		# Only scheme should be lower case
		parsed = urlparse(url)
		url = parsed.geturl()
		conn = self.poolmanager.connection_from_url(url)

	return conn
```

这里会调用到urllib3库方法
urllib3.poolmanager
```python
def connection_from_url(self, url, pool_kwargs=None):
	"""
	Similar to :func:`urllib3.connectionpool.connection_from_url`.

	If ``pool_kwargs`` is not provided and a new pool needs to be
	constructed, ``self.connection_pool_kw`` is used to initialize
	the :class:`urllib3.connectionpool.ConnectionPool`. If ``pool_kwargs``
	is provided, it is used instead. Note that if a new pool does not
	need to be created for the request, the provided ``pool_kwargs`` are
	not used.
	"""
	u = parse_url(url)
	return self.connection_from_host(u.host, port=u.port, scheme=u.scheme,
									 pool_kwargs=pool_kwargs)
```
这里会根据url构造url类实例，这个类实例包括实际的请求host、port和schema。

然后会进入到connection_from_context方法
```python
def connection_from_context(self, request_context):
	"""
	Get a :class:`ConnectionPool` based on the request context.

	``request_context`` must at least contain the ``scheme`` key and its
	value must be a key in ``key_fn_by_scheme`` instance variable.
	"""
	scheme = request_context['scheme'].lower()
	pool_key_constructor = self.key_fn_by_scheme[scheme]
	pool_key = pool_key_constructor(request_context)

	return self.connection_from_pool_key(pool_key, request_context=request_context)
```

经过一系列的调用，进入_new_pool方法
```python
def _new_pool(self, scheme, host, port, request_context=None):
	"""
	Create a new :class:`ConnectionPool` based on host, port, scheme, and
	any additional pool keyword arguments.

	If ``request_context`` is provided, it is provided as keyword arguments
	to the pool class used. This method is used to actually create the
	connection pools handed out by :meth:`connection_from_url` and
	companion methods. It is intended to be overridden for customization.
	"""
	pool_cls = self.pool_classes_by_scheme[scheme]
	if request_context is None:
		request_context = self.connection_pool_kw.copy()

	# Although the context has everything necessary to create the pool,
	# this function has historically only used the scheme, host, and port
	# in the positional args. When an API change is acceptable these can
	# be removed.
	for key in ('scheme', 'host', 'port'):
		request_context.pop(key, None)

	if scheme == 'http':
		for kw in SSL_KEYWORDS:
			request_context.pop(kw, None)

	return pool_cls(host, port, **request_context)
```
这里注意下这段代码：
```python
if request_context is None:
		request_context = self.connection_pool_kw.copy()
```
这个connection_pool_kw是在实例化时初始化的
```python
class PoolManager(RequestMethods):
    """
    Allows for arbitrary requests while transparently keeping track of
    necessary connection pools for you.

    :param num_pools:
        Number of connection pools to cache before discarding the least
        recently used pool.

    :param headers:
        Headers to include with all requests, unless other headers are given
        explicitly.

    :param \\**connection_pool_kw:
        Additional parameters are used to create fresh
        :class:`urllib3.connectionpool.ConnectionPool` instances.

    Example::

        >>> manager = PoolManager(num_pools=2)
        >>> r = manager.request('GET', 'http://google.com/')
        >>> r = manager.request('GET', 'http://google.com/mail')
        >>> r = manager.request('GET', 'http://yahoo.com/')
        >>> len(manager.pools)
        2

    """

    proxy = None

    def __init__(self, num_pools=10, headers=None, **connection_pool_kw):
        RequestMethods.__init__(self, headers)
        self.connection_pool_kw = connection_pool_kw
        self.pools = RecentlyUsedContainer(num_pools,
                                           dispose_func=lambda p: p.close())

        # Locally set the pool classes and keys so other PoolManagers can
        # override them.
        self.pool_classes_by_scheme = pool_classes_by_scheme
        self.key_fn_by_scheme = key_fn_by_scheme.copy()
```
这里的`connection_pool_kw`就是让调用方传入其他参数的地方。

回到`_new_pool`方法， 这里会根据schema在pool_classes_by_scheme字典中获取处理请求的类
```python
pool_classes_by_scheme = {
    'http': HTTPConnectionPool,
    'https': HTTPSConnectionPool,
}
```

以https为例，就会返回一个HTTPSConnectionPool实例
```python
class HTTPSConnectionPool(HTTPConnectionPool):
    """
    Same as :class:`.HTTPConnectionPool`, but HTTPS.

    When Python is compiled with the :mod:`ssl` module, then
    :class:`.VerifiedHTTPSConnection` is used, which *can* verify certificates,
    instead of :class:`.HTTPSConnection`.

    :class:`.VerifiedHTTPSConnection` uses one of ``assert_fingerprint``,
    ``assert_hostname`` and ``host`` in this order to verify connections.
    If ``assert_hostname`` is False, no verification is done.

    The ``key_file``, ``cert_file``, ``cert_reqs``, ``ca_certs``,
    ``ca_cert_dir``, ``ssl_version``, ``key_password`` are only used if :mod:`ssl`
    is available and are fed into :meth:`urllib3.util.ssl_wrap_socket` to upgrade
    the connection socket into an SSL socket.
    """

    scheme = 'https'
    ConnectionCls = HTTPSConnection

    def __init__(self, host, port=None,
                 strict=False, timeout=Timeout.DEFAULT_TIMEOUT, maxsize=1,
                 block=False, headers=None, retries=None,
                 _proxy=None, _proxy_headers=None,
                 key_file=None, cert_file=None, cert_reqs=None,
                 key_password=None, ca_certs=None, ssl_version=None,
                 assert_hostname=None, assert_fingerprint=None,
                 ca_cert_dir=None, **conn_kw):

        HTTPConnectionPool.__init__(self, host, port, strict, timeout, maxsize,
                                    block, headers, retries, _proxy, _proxy_headers,
                                    **conn_kw)

        self.key_file = key_file
        self.cert_file = cert_file
        self.cert_reqs = cert_reqs
        self.key_password = key_password
        self.ca_certs = ca_certs
        self.ca_cert_dir = ca_cert_dir
        self.ssl_version = ssl_version
        self.assert_hostname = assert_hostname
        self.assert_fingerprint = assert_fingerprint
```
这里可以看出，这里几乎支持所有request请求的参数的传入。

## 解密私钥
从以上我们就知道了， 有两种方式解决这个问题

### 1、自定义adapter
我们可以自定义一个adapter，根据情况继承HTTPAdapter或者BaseAdapter
```python
class XXHTTPAdapter(BaseAdapter):
	def __init__(xx, xx, **pool_kwargs)
	self.init_poolmanager(pool_connections, pool_maxsize, block=pool_block, **pool_kwargs)
	
	def init_poolmanager(self, connections, maxsize, block=DEFAULT_POOLBLOCK, **pool_kwargs):
	"""Initializes a urllib3 PoolManager.

	This method should not be called from user code, and is only
	exposed for use when subclassing the
	:class:`HTTPAdapter <requests.adapters.HTTPAdapter>`.

	:param connections: The number of urllib3 connection pools to cache.
	:param maxsize: The maximum number of connections to save in the pool.
	:param block: Block when no free connections are available.
	:param pool_kwargs: Extra keyword arguments used to initialize the Pool Manager.
	"""
	# save these values for pickling
	self._pool_connections = connections
	self._pool_maxsize = maxsize
	self._pool_block = block

	self.poolmanager = PoolManager(num_pools=connections, maxsize=maxsize,
								   block=block, strict=True, **pool_kwargs)
```

pool_kwargs不定参数中我们就可以传入`key_password`参数了，从而达到解密私钥的目的。
```python
s = requests.session()
xx_adapter = XXHTTPAdapter(key_password="xxx")
s.mount("https://", xx_adapter)
```

### 2、对ssl_文件打patch
比如：
```python
-    if keyfile and key_password is None and _is_key_file_encrypted(keyfile):
-        raise SSLError("Client private key is encrypted, password is required")
+    privkey_conf_path = "/opt/privkey/privkey.conf"
+    decrypted_password = None
+    if keyfile and _is_key_file_encrypted(keyfile):
+        if os.path.exists(privkey_conf_path):
+            try:
+                with open(privkey_conf_path, "r") as f:
+                    decrypted_password = module.decrypt(0, encrypt_password)
+            except Exception as e:
+                raise SSLError("Client private key is encrypted, password is required")

     if certfile:
-        if key_password is None:
+        if decrypted_password is None:
             context.load_cert_chain(certfile, keyfile)
         else:
-            context.load_cert_chain(certfile, keyfile, key_password)
+            context.load_cert_chain(certfile, keyfile, decrypted_password)
```