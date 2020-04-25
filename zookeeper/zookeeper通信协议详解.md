# zookeeper通信协议详解

## 通信协议
基于TCP/IP协议，zk实现了自己的通信协议来完成客户端与服务端，服务端与服务端之间的网络通信，zk的通信协议整体上的设计非常简单，

客户端发起连接,发送握手包进行timeout协商,协商成功后会返回一个session id和timeoout值.随后就可以进行正常通信,通信过程中要在timeout范围内发送ping包. 
zookeeper client和server之间的通信协议基本规则就是发送请求获取响应.并根据响应做不同的动作.

发送数据格式为:

* 消息长度+xid+request. xid每次请求必须是唯一的.消息长度和xid同为4字节,命令长度为4字节且必须为request的开始4字节.
* 命令是从1到11的数字表示,close的命令为-11.不同类型请求request有些差异
* 特殊请求具有固定的xid:watch_xid固定为-1,ping_xid为-2,auth_xid固定为-4.普通请求一般从0开始每次请求累加一次xid.

响应数据为:

* 消息长度+header+response.消息长度为4字节,表明header+response的总长度.
* header为xid,zxid,err对应长度为4,8,4.response根据请求类型不同具有差别
* 根据header里xid的区别分为watch,ping,auth,data这四种类型
* 根据这四种类型来区分返回消息是事件,还是认证,心跳和请求数据.client并以此作出不同响应.


## 消息结构
### 握手消息
#### request消息体
protocol_version+zxid+timeout+session_id+passwd_len+passwd+read_only.对应的字节长度为4,8,4,8,4,16,1
取值除timeout外其他几个皆可为0,password可以为任意16个字符.read_only为0或1(是布尔值).
注：握手包没有xid和命令

#### response消息体
protocol_version+timeout+session_id+passwd_len+passwd+read_only.
注：握手响应包没有header.

#### 效果展示
```sh
2020-04-21 19:26:53.990 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:646 _connect]  Connecting to 30.3.3.60:9888, use_ssl: False
2020-04-21 19:26:53.990 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:650 _connect]      Using session_id: 144131667822575626 session_passwd: b'41f366ef7005bc5c859b7fc56fa40872'
2020-04-21 19:26:53.990 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:299 _submit]  Sending request(xid=None): Connect(protocol_version=0, last_zxid_seen=12884901993, time_out=30000, session_id=144131667822575626, passwd=b'A\xf3f\xefp\x05\xbc\\\x85\x9b\x7f\xc5o\xa4\x08r', read_only=None)
2020-04-21 19:26:53.991 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:285 _invoke]  Read response Connect(protocol_version=0, last_zxid_seen=0, time_out=30000, session_id=144131667822575626, passwd=b'A\xf3f\xefp\x05\xbc\\\x85\x9b\x7f\xc5o\xa4\x08r', read_only=False)
2020-04-21 19:26:53.991 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:694 _connect]  Session created, session_id: 144131667822575626 session_passwd: b'41f366ef7005bc5c859b7fc56fa40872'
    negotiated session timeout: 30000
    connect timeout: 10000.0
    read timeout: 20000.0
2020-04-21 19:26:53.991 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [client.py:463 _session_callback]  test: cur state CONNECTED, old state CONNECTING
```

### ping消息
#### request消息体
type (ping包只有一个字段就是命令值是11,它完整的发送包是4字节长度,4字节xid,4字节命令.)

#### response消息体
res_len+header+res (ping响应包一般只拆到header即可通过xid确认)

#### 效果展示
```sh
2020-04-21 20:05:03.971 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:603 _connect_attempt]  test: send ping
2020-04-21 20:05:03.971 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:490 _send_ping]  test: send ping
2020-04-21 20:05:03.971 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:299 _submit]  Sending request(xid=-2): Ping()
2020-04-21 20:05:03.973 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:606 _connect_attempt]  test: read socket
2020-04-21 20:05:03.973 localhost cms_watcher info  INFO cms_watcher [pid:240] [Thread-4] [connection.py:415 _read_socket]  test: Received Ping

```

### getdata消息
#### request消息体
type+path_len+path+watcher
type=4.
path_len,是4字节,为path的长度
path为需要查询的路径,支持utf8
watcher为布尔值.判断是否有事件注册.为1或0. 1字节

#### response消息体
data_len+data+stat
data_len为data长度,4字节.
stat由8,8,8,8,4,4,4,8,4,4,8字节顺序组成.

#### 效果展示
```sh
2020-04-21 20:25:13.460 localhost cms_watcher info  INFO cms_watcher [pid:9078] [Thread-4] [connection.py:610 _connect_attempt]  test: send request
2020-04-21 20:25:13.460 localhost cms_watcher info  INFO cms_watcher [pid:9078] [Thread-4] [connection.py:482 _send_request]  test: send request xid 4
2020-04-21 20:25:13.460 localhost cms_watcher info  INFO cms_watcher [pid:9078] [Thread-4] [connection.py:299 _submit]  Sending request(xid=4): GetData(path='/cms/config/items/item.cts_cfg', watcher=<bound method DataWatch._watcher of <kazoo.recipe.watchers.DataWatch object at 0x7ff860ba3438>>)
2020-04-21 20:25:13.461 localhost cms_watcher info  INFO cms_watcher [pid:9078] [Thread-4] [connection.py:606 _connect_attempt]  test: read socket
2020-04-21 20:25:13.461 localhost cms_watcher info  INFO cms_watcher [pid:9078] [Thread-4] [connection.py:448 _read_socket]  test: Reading for header ReplyHeader(xid=4, zxid=17179869239, err=0)

```

## 序列化和反序列化
下面看下kazoo这个库是怎样根据zk的这个协议来组装数据和解析数据的

### request字节流
下面的代码展示了将请求对象序列化成socket字节流的过程
```python
def _submit(self, request, timeout, xid=None):
	"""Submit a request object with a timeout value and optional
	xid"""
	b = bytearray()
	if xid:
		b.extend(int_struct.pack(xid))
	if request.type:
		b.extend(int_struct.pack(request.type))
	b += request.serialize()
	self.logger.log(
		(BLATHER if isinstance(request, Ping) else logging.DEBUG),
		"Sending request(xid=%s): %s", xid, request)
	self._write(int_struct.pack(len(b)) + b, timeout)
```

从上面的代码可以看出，首先根据不同的请求，决定是否发送xid字段、type字段（也就是上面协议所说的），最后根据request对象序列化成字节流。
这里的request就是kazoo/protocol/serialization.py定义的各个类实例
比如连接类：
```python
class Connect(namedtuple('Connect', 'protocol_version last_zxid_seen'
                         ' time_out session_id passwd read_only')):
    type = None

    def serialize(self):
        b = bytearray()
        b.extend(int_long_int_long_struct.pack(
            self.protocol_version, self.last_zxid_seen, self.time_out,
            self.session_id))
        b.extend(write_buffer(self.passwd))
        b.extend([1 if self.read_only else 0])
        return b

    @classmethod
    def deserialize(cls, bytes, offset):
        proto_version, timeout, session_id = int_int_long_struct.unpack_from(
            bytes, offset)
        offset += int_int_long_struct.size
        password, offset = read_buffer(bytes, offset)

        try:
            read_only = bool_struct.unpack_from(bytes, offset)[0] is 1
            offset += bool_struct.size
        except struct.error:
            read_only = False
        return cls(proto_version, 0, timeout, session_id, password,
                   read_only), offset
```

### response字节流
下面的代码展示了将socket字节流反序列化成对象的过程
```python
def _read_header(self, timeout):
	b = self._read(4, timeout)
	length = int_struct.unpack(b)[0]
	b = self._read(length, timeout)
	header, offset = ReplyHeader.deserialize(b, 0)
	return header, b, offset
```

从上面的代码可以看出，首先从socket中读取4个字节，根据上面的协议，我们知道，这4个字节是data_len，即包的大小
然后在根据len继续读取该大小的字节流，最后解析成具体的对象。
```python
class ReplyHeader(namedtuple('ReplyHeader', 'xid, zxid, err')):
    @classmethod
    def deserialize(cls, bytes, offset):
        """Given bytes and the current bytes offset, return a
        :class:`ReplyHeader` instance and the new offset"""
        new_offset = offset + reply_header_struct.size
        return cls._make(
            reply_header_struct.unpack_from(bytes, offset)), new_offset
```