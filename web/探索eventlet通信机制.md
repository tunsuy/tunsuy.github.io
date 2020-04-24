## 探索eventlet通信机制
### 一、源码解析
对python原生文件打补丁：
```python
import eventlet
eventlet.monkey_patch()
```

跟踪进入该模块方法：eventlet.patcher#monkey_patch
```python
def monkey_patch(**on):
......

modules_to_patch = []
for name, modules_function in [
	('os', _green_os_modules),
	('select', _green_select_modules),
	('socket', _green_socket_modules),
	('thread', _green_thread_modules),
	('time', _green_time_modules),
	('MySQLdb', _green_MySQLdb),
	('builtins', _green_builtins),
	('subprocess', _green_subprocess_modules),
]:
	if on[name] and not already_patched.get(name):
		modules_to_patch += modules_function()
		already_patched[name] = True
		
......
```
该方法对某些系统模块进行全局打补丁，使其对Greenthread友好。
关键字参数用于指定哪些模块需要打补丁，如果未提供关键字参数，则会对所有默认的模块（如代码所示）打补丁，例如：
```monkey_patch（socket = True，select = True）```
仅对socket和select模块打补丁。
大多数参数都是对同名的单个模块进行打补丁，比如操作系统，时间，选择。
但是socket例外，它也会对ssl模块（如果存在）打补丁，thread用于对threading、thread、Queue打补丁。
说明：多次调用monkey_patch是安全的。

以socket为例：`('socket', _green_socket_modules)`，进入该方法：
```python
def _green_socket_modules():
    from eventlet.green import socket
    try:
        from eventlet.green import ssl
        return [('socket', socket), ('ssl', ssl)]
    except ImportError:
        return [('socket', socket)]
```

进入socket模块：eventlet.green.socket
```python
__import__('eventlet.green._socket_nodns')
__socket = sys.modules['eventlet.green._socket_nodns']

__all__ = __socket.__all__
__patched__ = __socket.__patched__ + [
    'create_connection',
    'getaddrinfo',
    'gethostbyname',
    'gethostbyname_ex',
    'getnameinfo',
]
```
在进入eventlet.green._socket_nodns：
```python
__socket = __import__('socket')

__all__ = __socket.__all__
__patched__ = ['fromfd', 'socketpair', 'ssl', 'socket', 'timeout']
```
可以看到是对python的原生socket模块进行了打补丁：pythonx.x/Lib/socket.py
以socket类为例：
python原生的socket.socket()类并替换为了eventlet.greenio.base#GreenSocket类
该补丁类完全兼容原生socket类的API，它还可以识别关键字参数`set_nonblocking = True`。用来设置socket为非阻塞模式。
```python
class GreenSocket(object):
    # This placeholder is to prevent __getattr__ from creating an infinite call loop
    fd = None

    def __init__(self, family=socket.AF_INET, *args, **kwargs):
        should_set_nonblocking = kwargs.pop('set_nonblocking', True)
        if isinstance(family, six.integer_types):
            fd = _original_socket(family, *args, **kwargs)
            # Notify the hub that this is a newly-opened socket.
            notify_opened(fd.fileno())
        else:
            fd = family

        # import timeout from other socket, if it was there
        try:
            self._timeout = fd.gettimeout() or socket.getdefaulttimeout()
        except AttributeError:
            self._timeout = socket.getdefaulttimeout()

        # Filter fd.fileno() != -1 so that won't call set non-blocking on
        # closed socket
        if should_set_nonblocking and fd.fileno() != -1:
            set_nonblocking(fd)
        self.fd = fd
        # when client calls setblocking(0) or settimeout(0) the socket must
        # act non-blocking
        self.act_non_blocking = False
		
	......
```

我们再来看下ssl模块。python原生的ssl模块被替换为了evenlet.green.ssl模块
该模块提供了一个方法用来包装socket：
```python
def wrap_socket(sock, *a, **kw):
    return GreenSSLSocket(sock, *a, **kw)
```
直接进入GreenSSLSocket类:
```python
class GreenSSLSocket(_original_sslsocket):
	......
```
可以看出该补丁模块继承了原生socket，将原生socket的api都重写了，但是基本都是直接调用原生api。
注：Python3.x版本中，如果socket的另一端已关闭时，非阻塞模式的sslsocket对象不会再抛出错误（虽然它们会在另一端关闭时发出通知）。
如果另一端的socket已经关闭，任何的写/读操作都会被简单地挂起。
这个问题目前没有好的解决方案。它看起来是Python的sslsocket对象实现的一个限制。
一个解决方法是使用命令`settimeout（）`在socket上设置合理的超时时间，并在超时时关闭/重新打开连接。

下面看下原生ssl模块：pythonx.x/Lib/ssl.py
```python
def wrap_socket(sock, keyfile=None, certfile=None,
                server_side=False, cert_reqs=CERT_NONE,
                ssl_version=PROTOCOL_TLS, ca_certs=None,
                do_handshake_on_connect=True,
                suppress_ragged_eofs=True,
                ciphers=None):

    if server_side and not certfile:
        raise ValueError("certfile must be specified for server-side "
                         "operations")
    if keyfile and not certfile:
        raise ValueError("certfile must be specified")
    context = SSLContext(ssl_version)
    context.verify_mode = cert_reqs
    if ca_certs:
        context.load_verify_locations(ca_certs)
    if certfile:
        context.load_cert_chain(certfile, keyfile)
    if ciphers:
        context.set_ciphers(ciphers)
    return context.wrap_socket(
        sock=sock, server_side=server_side,
        do_handshake_on_connect=do_handshake_on_connect,
        suppress_ragged_eofs=suppress_ragged_eofs
    )
```
可以看到该调用了SSLContext.wrap_socket方法，进入该方法：
```python
class SSLContext(_SSLContext):
   ......
   
    sslsocket_class = None  # SSLSocket is assigned later.
    sslobject_class = None  # SSLObject is assigned later.

	......

    def wrap_socket(self, sock, server_side=False,
                    do_handshake_on_connect=True,
                    suppress_ragged_eofs=True,
                    server_hostname=None, session=None):
        # SSLSocket class handles server_hostname encoding before it calls
        # ctx._wrap_socket()
        return self.sslsocket_class._create(
            sock=sock,
            server_side=server_side,
            do_handshake_on_connect=do_handshake_on_connect,
            suppress_ragged_eofs=suppress_ragged_eofs,
            server_hostname=server_hostname,
            context=self,
            session=session
        )


```
该类中类属性sslobject_class定义如下：
```python
# Python does not support forward declaration of types.
SSLContext.sslsocket_class = SSLSocket
SSLContext.sslobject_class = SSLObject
```
进入SSLSocket类：
```python
class SSLSocket(socket):
    
	......
	
    @classmethod
    def _create(cls, sock, server_side=False, do_handshake_on_connect=True,
                suppress_ragged_eofs=True, server_hostname=None,
                context=None, session=None):
        
		......
		
        if connected:
            # create the SSL object
            try:
                self._sslobj = self._context._wrap_socket(
                    self, server_side, self.server_hostname,
                    owner=self, session=self._session,
                )
                if do_handshake_on_connect:
                    timeout = self.gettimeout()
                    if timeout == 0.0:
                        # non-blocking
                        raise ValueError("do_handshake_on_connect should not be specified for non-blocking sockets")
                    self.do_handshake()
            except (OSError, ValueError):
                self.close()
                raise
        return self
```
最终该self._sslobj实例就是cpython中定义的对象，所有后续的所有操作都是调用的cpython方法。


### 二、遗留问题
问题堆栈：
```sh
Traceback (most recent call last):
  File "test.py", line 40, in <module>
    main()
  File "test.py", line 35, in main
    srv(listener)
  File "test.py", line 10, in srv
    r.readline(1<<10)
  File "/usr/lib/python3.7/socket.py", line 589, in readinto
    return self._sock.recv_into(b)
  File "/usr/lib/python3.7/site-packages/eventlet/green/ssl.py", line 241, in recv_into
    return self._base_recv(nbytes, flags, into=True, buffer_=buffer)
  File "/usr/lib/python3.7/site-packages/eventlet/green/ssl.py", line 256, in _base_recv
    read = self.read(nbytes, buffer_)
  File "/usr/lib/python3.7/site-packages/eventlet/green/ssl.py", line 176, in read
    super(GreenSSLSocket, self).read, *args, **kwargs)
  File "/usr/lib/python3.7/site-packages/eventlet/green/ssl.py", line 146, in _call_trampolining
    return func(*a, **kw)
  File "/usr/lib/python3.7/ssl.py", line 911, in read
    return self._sslobj.read(len, buffer)
ssl.SSLWantReadError: The operation did not complete (read) (_ssl.c:2488)
```
从这里我们可以看到系统调用的入口是python3.7/socket.py中的readinto方法，进入该方法：
```python
def readinto(self, b):
	self._checkClosed()
	self._checkReadable()
	if self._timeout_occurred:
		raise OSError("cannot read from timed out object")
	while True:
		try:
			return self._sock.recv_into(b)
		except timeout:
			self._timeout_occurred = True
			raise
		except error as e:
			if e.args[0] in _blocking_errnos:
				return None
			raise
```
最多将len（b）个字节读入可写缓冲区* b *并返回读取的字节数。如果套接字是非阻塞的并且没有字节可用，则返回None。
如果* b *为非空，则返回值为0表示该连接在另一端被关闭。
注：如果未设置默认超时并且侦听套接字具有（非零）超时，请强制新套接字处于阻塞模式，以覆盖特定于平台的套接字标志继承。

我们根据堆栈一步步进入最终报错的地方：`self._sslobj.read(len, buffer)`
根据我们上面说的，`self._sslobj`实际上是cpython对象，那read方法是怎么就进入到了cpython实际的方法里面的呢？
通过python代用C代码的机制可以找到如下代码：
```c
#define _SSL__SSLSOCKET_READ_METHODDEF    \
    {"read", (PyCFunction)_ssl__SSLSocket_read, METH_VARARGS, _ssl__SSLSocket_read__doc__},

static PyObject *
_ssl__SSLSocket_read_impl(PySSLSocket *self, int len, int group_right_1,
                          Py_buffer *buffer);

static PyObject *
_ssl__SSLSocket_read(PySSLSocket *self, PyObject *args)
{
    PyObject *return_value = NULL;
    int len;
    int group_right_1 = 0;
    Py_buffer buffer = {NULL, NULL};

    switch (PyTuple_GET_SIZE(args)) {
        case 1:
            if (!PyArg_ParseTuple(args, "i:read", &len)) {
                goto exit;
            }
            break;
        case 2:
            if (!PyArg_ParseTuple(args, "iw*:read", &len, &buffer)) {
                goto exit;
            }
            group_right_1 = 1;
            break;
        default:
            PyErr_SetString(PyExc_TypeError, "_ssl._SSLSocket.read requires 1 to 2 arguments");
            goto exit;
    }
    return_value = _ssl__SSLSocket_read_impl(self, len, group_right_1, &buffer);

exit:
    /* Cleanup for buffer */
    if (buffer.obj) {
       PyBuffer_Release(&buffer);
    }

    return return_value;
}
```
可以看出，`read`是映射到了`_ssl__SSLSocket_read`方法，而`_ssl__SSLSocket_read`则调用了`_ssl__SSLSocket_read_impl`方法。
我们进入`_ssl__SSLSocket_read_impl`的实现：
```c
static PyObject *
_ssl__SSLSocket_read_impl(PySSLSocket *self, int len, int group_right_1,
                          Py_buffer *buffer)
/*[clinic end generated code: output=00097776cec2a0af input=ff157eb918d0905b]*/
{
    ......

    do {
        PySSL_BEGIN_ALLOW_THREADS
        count = SSL_read(self->ssl, mem, len);
        err = _PySSL_errno(count <= 0, self->ssl, count);
        PySSL_END_ALLOW_THREADS
        self->err = err;

        if (PyErr_CheckSignals())
            goto error;

        if (has_timeout)
            timeout = deadline - _PyTime_GetMonotonicClock();

        if (err.ssl == SSL_ERROR_WANT_READ) {
            sockstate = PySSL_select(sock, 0, timeout);
        } else if (err.ssl == SSL_ERROR_WANT_WRITE) {
            sockstate = PySSL_select(sock, 1, timeout);
        } else if (err.ssl == SSL_ERROR_ZERO_RETURN &&
                   SSL_get_shutdown(self->ssl) == SSL_RECEIVED_SHUTDOWN)
        {
            count = 0;
            goto done;
        }
        else
            sockstate = SOCKET_OPERATION_OK;

        if (sockstate == SOCKET_HAS_TIMED_OUT) {
            PyErr_SetString(PySocketModule.timeout_error,
                            "The read operation timed out");
            goto error;
        } else if (sockstate == SOCKET_IS_NONBLOCKING) {
            break;
        }
    } while (err.ssl == SSL_ERROR_WANT_READ ||
             err.ssl == SSL_ERROR_WANT_WRITE);

    if (count <= 0) {
        PySSL_SetError(self, count, __FILE__, __LINE__);
        goto error;
    }
    if (self->exc_type != NULL)
        goto error;

......
}
```
从该模块的include也可以看出，该模块就是调用了系统的openssl库进行ssl通信
```c
/* Include OpenSSL header files */
#include "openssl/rsa.h"
#include "openssl/crypto.h"
#include "openssl/x509.h"
#include "openssl/x509v3.h"
#include "openssl/pem.h"
#include "openssl/ssl.h"
#include "openssl/err.h"
#include "openssl/rand.h"
#include "openssl/bio.h"
#include "openssl/dh.h"
```

进入openssl头文件，可以看到确实有定义这个错误码`SSL_ERROR_WANT_READ`：
```c
在openssl源码中我们可以找到这个定义include.openssl.ssl.h
​```c
# define SSL_AD_NO_APPLICATION_PROTOCOL  TLS1_AD_NO_APPLICATION_PROTOCOL
# define SSL_ERROR_NONE                  0
# define SSL_ERROR_SSL                   1
# define SSL_ERROR_WANT_READ             2
# define SSL_ERROR_WANT_WRITE            3
# define SSL_ERROR_WANT_X509_LOOKUP      4
# define SSL_ERROR_SYSCALL               5/* look at error stack/return
```

下面我们来看下`PySSL_SetError`方法：cpython->modules._ssl.c
```c
static PyObject *
PySSL_SetError(PySSLSocket *sslsock, int ret, const char *filename, int lineno)
{
    PyObject *type = PySSLErrorObject;
    char *errstr = NULL;
    _PySSLError err;
    enum py_ssl_error p = PY_SSL_ERROR_NONE;
    unsigned long e = 0;

    assert(ret <= 0);
    e = ERR_peek_last_error();

    if (sslsock->ssl != NULL) {
        err = sslsock->err;

        switch (err.ssl) {
        case SSL_ERROR_ZERO_RETURN:
            errstr = "TLS/SSL connection has been closed (EOF)";
            type = PySSLZeroReturnErrorObject;
            p = PY_SSL_ERROR_ZERO_RETURN;
            break;
        case SSL_ERROR_WANT_READ:
            errstr = "The operation did not complete (read)";
            type = PySSLWantReadErrorObject;
            p = PY_SSL_ERROR_WANT_READ;
            break;
        case SSL_ERROR_WANT_WRITE:
            p = PY_SSL_ERROR_WANT_WRITE;
            type = PySSLWantWriteErrorObject;
            errstr = "The operation did not complete (write)";
            break;
```
经过一步步跟进去，确实会发现返回了一个SSLError类型的错误。