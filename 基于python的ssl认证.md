基于python的ssl认证

### 一、模块
xx/pythonxx/Lib/ssl.py
此模块为客户端和服务器端的网络套接字提供对传输层安全性（通常称为“安全套接字层”）加密和对等身份验证功能的访问。该模块使用OpenSSL库。只要OpenSSL安装在该平台上，就可以在所有现代Unix系统，Windows，Mac OS X以及其他平台上使用。


### 二、接口
提供了这样一个类
```python
class SSLSocket(socket):
    """This class implements a subtype of socket.socket that wraps
    the underlying OS socket in an SSL context when necessary, and
    provides read and write methods over that channel."""

    def __init__(self, sock=None, keyfile=None, certfile=None,
                 server_side=False, cert_reqs=CERT_NONE,
                 ssl_version=PROTOCOL_TLS, ca_certs=None,
                 do_handshake_on_connect=True,
                 family=AF_INET, type=SOCK_STREAM, proto=0, fileno=None,
                 suppress_ragged_eofs=True, npn_protocols=None, ciphers=None,
                 server_hostname=None,
                 _context=None):
```

### 三、单向认证
单向认证流程：
1.客户端say hello服务端
2.服务端将证书、公钥等发给客户端
3.客户端CA验证证书，成功继续、不成功弹出选择页面
4.客户端告知服务端所支持的加密算法
5.服务端选择最高级别加密算法明文通知客户端
6.客户端生成随机对称密匙key，使用服务端公钥加密发送给服务端
7.服务端使用私钥解密，获取对称密匙key
8.后续客户端与服务端使用该密匙key进行加密通信

#### 1、server端
对应于第二节中的类
keyfile和certfile参数则必填，指定服务端证书和公钥，用于传输给客户端进行校验
伪代码如下：
```python
socket = ssl.wrap_socket(sock=sock, keyfile=keyfile, certfile=certfile,
                     server_side=True, cert_reqs=ssl.CERT_NONE,
                     ssl_version="ssl.PROTOCOL_TLSv1_2", 
                     do_handshake_on_connect=do_handshake_on_connect,
                     suppress_ragged_eofs=suppress_ragged_eofs,
                     ciphers=ciphers)
启动
eventlet.wsgi.server(socket, socket.getsockname(),
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
```

#### 2、client端
request方法中的verify参数必须填，指定服务端证书的签名根证书，用于校验服务端传输过来的证书和公钥
伪代码如下：
```python
headers = {
	'Content-Type': 'application/json',
	'Accept': 'application/json'
}
kwargs = {
	"headers": headers,
	"verify": "/xxx/xxx.pem",
	"timeout": 60
}

response = requests.request('GET', url, **kwargs)
```

其中，client的verify参数和server端的certfile、keyfile参数必须保持一致


### 四、双向认证
双向认证流程：
1.客户端say hello服务端
2.服务端将证书、公钥等发给客户端
3.客户端CA验证证书，成功继续、不成功弹出选择页面
4.客户端将自己的证书和公钥发送给服务端
5.服务端验证客户端证书，如不通过直接断开连接
6.客户端告知服务端所支持的加密算法
7.服务端选择最高级别加密算法使用客户端公钥加密后发送给客户端
8.客户端收到后使用私钥解密并生成随机对称密匙key，使用服务端公钥加密发送给服务端
9.服务端使用私钥解密，获取对称密匙key
10.后续客户端与服务端使用该密匙key进行加密通信

#### 1、server端
对应于第二节中的类
keyfile和certfile参数必填，指定服务端证书和公钥，用于传输给客户端进行校验
ca_certs参数必填，指定客户端证书的签名根证书，用于校验客户端传输过来的证书和公钥
另外，cert_reqs必须为CERT_REQUIRED
伪代码如下：
```python
socket = ssl.wrap_socket(sock=sock, keyfile=keyfile, certfile=certfile,
                     server_side=True, cert_reqs=ssl.CERT_REQUIRED,
                     ssl_version="ssl.PROTOCOL_TLSv1_2", ca_certs=cacerts,
                     do_handshake_on_connect=do_handshake_on_connect,
                     suppress_ragged_eofs=suppress_ragged_eofs,
                     ciphers=ciphers)
启动
eventlet.wsgi.server(socket, socket.getsockname(),
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
```

#### 2、client端
request方法中的verify参数必须填，指定服务端证书的签名根证书，用于校验服务端传输过来的证书和公钥
request方法中的cert参数必须填，指定服务端证书的签名根证书，用于传输给服务端进行校验
伪代码如下：
```python
headers = {
	'Content-Type': 'application/json',
	'Accept': 'application/json'
}
kwargs = {
	"headers": headers,
	"verify": "/xxx/ca.pem",
	"cert": (
		"/xxx/cert.pem",
		"/xxx/key.pem"),
	"timeout": 60
}

response = requests.request('GET', url, **kwargs)
```

其中，client的verify参数和server端的certfile、keyfile参数必须保持一致
client的cert参数和server端的ca_certs参数必须保持一致

https://zhuanlan.zhihu.com/p/36527074
https://www.rddoc.com/doc/Python/3.6.0/zh/library/ssl/#ssl-contexts