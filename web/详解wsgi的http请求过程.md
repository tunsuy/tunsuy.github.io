## 详解wsgi的http请求过程

### 一、概述
wsgi服务启动并监听http请求的流程：
* 1.利用paste.deploy模块的loadapp函数加载指定服务（如proxy）的配置文件,获取到用户的application，即业务程序
* 2.调用wsgi.server,其中wsgi.server会绑定IP和端口，监听来自客户端的消息。并由process_request函数负责处理消息。


下面主要说下处理http请求的过程（其他的在另外的文章中已有讲解）

我们都知道wsgi application都需要实现__call__()方法，并且参数必须为environ, start_response；
那么这两个参数是从哪里传递进来的呢？下面就来解密下这个。

### 二、http请求处理
模块：../site-packages/eventlet/wsgi.py

#### 1、入口函数server
wsgi.py文件中定义了Server类，用于开启一个服务端socket，处理socket通信；
文件中还定义了一个入口函数：server，源码如下：
```python
def server():
   serv = Server(sock, sock.getsockname(),
			  site, log,
			  environ=environ,
			  max_http_version=max_http_version,
			  protocol=protocol,
			  minimum_chunk_size=minimum_chunk_size,
			  log_x_forwarded_for=log_x_forwarded_for,
			  keepalive=keepalive,
			  log_output=log_output,
			  log_format=log_format,
			  url_length_limit=url_length_limit,
			  debug=debug,
			  socket_timeout=socket_timeout,
			  capitalize_response_headers=capitalize_response_headers,
			  )
    if server_event is not None:
        server_event.send(serv)
    if max_size is None:
        max_size = DEFAULT_MAX_SIMULTANEOUS_REQUESTS
    if custom_pool is not None:
        pool = custom_pool
    else:
        pool = greenpool.GreenPool(max_size)
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
```
该函数首先创建一个服务器对象（可以指定协议）；
通过外部指定的协程池大小初始化协程池；
最后循环监听来自客户端的连接：每次收到一个请求，就新开一个协程去处理该请求。

#### 2、请求处理函数process_request
在入口函数server中，调用了process_request方法来处理请求，源码如下：
```python
def process_request(self, sock_params):
	# The actual request handling takes place in __init__, so we need to
	# set minimum_chunk_size before __init__ executes and we don't want to modify
	# class variable
	sock, address = sock_params
	proto = new(self.protocol)
	if self.minimum_chunk_size is not None:
		proto.minimum_chunk_size = self.minimum_chunk_size
	proto.capitalize_response_headers = self.capitalize_response_headers
	try:
		proto.__init__(sock, address, self)
	except socket.timeout:
		# Expected exceptions are not exceptional
		sock.close()
		# similar to logging "accepted" in server()
		self.log.debug('(%s) timed out %r' % (self.pid, address))
```
该方法实例化了协议类（由外部指定协议），调用了该协议类的__init__方法。
注：HttpProtocol对象自身没有init函数，所以会调用父类BaseRequestHandler（PythonXX/Lib/SocketServer.py）的init 函数 ，源码如下：
```python
def __init__(self, request, client_address, server):
	self.request = request
	self.client_address = client_address
	self.server = server
	self.setup()
	try:
		self.handle()
	finally:
		self.finish()
```
此函数中，执行了setup, handle, finish 三个操作，下面依次进行分析。

##### I、setup函数
HttpProtocal 类重写了setup函数，根据client socket生成rfile和wfile。

##### II、handle函数
BaseHTTPRequestHandler（../site-packages/eventlet/green/http/server.py） 重写了handle函数，此函数主要调用 handle_one_request (HttpProtocl类重写)函数。

handle_one_request函数，调用parse_request（BaseHTTPRequestHandler类）函数，通过mimetools.Message类生成headers,以及command,path,version等参数；
调用get_environ (HttpProtocol类)函数生成environ字典；
将self.server.app赋值self.application，其中self.server.app就是外部初始化wsgi.Server()时传递进来的，具体后面讲解；
调用handle_one_response函数。

handle_one_response函数，调用self.application函数，传递environ参数和start_response函数，执行用户业务流程并返回响应数据；
调用write函数，重新构造响应数据，调用wfile.writelines函数返回给客户端。
在self.application（即__call__函数）执行过程中，根据envrion 会生成request对象。
注：这就回答了文章开头提出的那个问题。

##### III、finish函数

HttpProtocol类重写，核心功能由StreamRequestHandler类完成。刷新缓存，关闭wfile，rfile，释放协程和socket资源。
start_response函数解析

在多个中间件项目中，start_response函数会被作为参数继续传递，直到最后一个app时，利用app过程中执行的status和headers参数，重写handle_one_response内部的header_set变量，作为执行结果返回给客户端。

#### 3、Application对象
在上一节中我们知道在handle_one_response方法中会执行application()方法，而这个application是怎么来的呢？
它是通过../site-packages/oslo_service/wsgi.py中load_app来的，源码如下：
```python
def load_app(self, name):
	"""Return the paste URLMap wrapped WSGI application.

	:param name: Name of the application to load.
	:returns: Paste URLMap object wrapping the requested application.
	:raises: PasteAppNotFound

	"""
	try:
		LOG.debug("Loading app %(name)s from %(path)s",
				  {'name': name, 'path': self.config_path})
		return deploy.loadapp("config:%s" % self.config_path, name=name)
	except LookupError:
		LOG.exception("Couldn't lookup app: %s", name)
		raise PasteAppNotFound(name=name, path=self.config_path)
```
这里的name就是在api-paste.ini文件中定义的入口section，例如下面的app_name
```python
[composite:app_name]
use = egg:Paste#urlmap
/: apiversions
/v1: xxxx
```
可以看到use则为实际的application执行的入口类，即paste.urlmap.URLMap，也就是调用其__call__方法，下面来看下源码：
```python
def __call__(self, environ, start_response):
	host = environ.get('HTTP_HOST', environ.get('SERVER_NAME')).lower()
	if ':' in host:
		host, port = host.split(':', 1)
	else:
		if environ['wsgi.url_scheme'] == 'http':
			port = '80'
		else:
			port = '443'
	path_info = environ.get('PATH_INFO')
	path_info = self.normalize_url(path_info, False)[1]
	for (domain, app_url), app in self.applications:
		if domain and domain != host and domain != host+':'+port:
			continue
		if (path_info == app_url
			or path_info.startswith(app_url + '/')):
			environ['SCRIPT_NAME'] += app_url
			environ['PATH_INFO'] = path_info[len(app_url):]
			return app(environ, start_response)
	environ['paste.urlmap_object'] = self
	return self.not_found_application(environ, start_response)
```
可以看出该方法会从environ中解析出访问地址及url路径，然后遍历所有的配置文件中的路径，找到匹配访问的url，执行其application。
也就是调用了下一个section的中间件的use指定的方法：每个中间件都会返回application。最后走到我们定义的业务应用route这个application里面，

#### 4、业务route
从上面我们得知，请求经过一系列的filter之后，就是真正到达我们的业务application中，这里我们的route是通过routes模块来实现的。
所以我们的route的__call__必须实例化RoutesMiddleware类（../site-packages/routes/middleware.py），该类也是一个可调用对象。
实例化该RoutesMiddleware类时，我们需要传递两个参数：
* 一个就是我们业务中定义的route mapper，即每个url对应的具体处理的controller类。
* 一个就是持有controller引用的资源分发处理类，该类必须也是一个可调用对象：在__call__中，解析出url对应的处理方法、参数，以及response封装等等。