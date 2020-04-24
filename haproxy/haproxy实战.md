## haproxy实战
haproxy的简介就不说了，网上很详细，直接google。

### 一、配置
```sh
# this config needs haproxy-1.5.x

global
    daemon
    nbproc 1
    maxconn 4096
    log 127.0.0.1 local1 info
    tune.bufsize 1000000
    tune.ssl.default-dh-param 2048
    ssl-default-bind-options no-sslv3 no-tlsv10
    ssl-default-bind-ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-SHA256
    stats timeout 2m

defaults
    log     global
    maxconn 2000
    option  redispatch
    retries 3
    mode    http
    option  httplog
    option  splice-auto
    option  httpclose
    option  forwardfor
    option  log-health-checks
    stats   hide-version
    timeout http-request 30s
    timeout queue   1m
    timeout connect 5s
    timeout client  1h
    timeout server  1h
	

frontend ts_8081
    bind 30.7.3.124:8081
    mode http
    option httpclose
    default_backend b_def_ts_8081

backend b_def_ts_8081
    mode http
    balance roundrobin
    option tcpka
    stats hide-version
    option httpchk
    option httplog
    server controller1 30.7.3.126:28081 check inter 30s fastinter 30s downinter 30s rise 2 fall 4
```

### 二、客户端
```python
# coding: utf-8
import json
import sys
import threading

import requests


def test_haproxy(concurrent_num):
    threads = []
    results = []

    for i in range(concurrent_num):
        th = threading.Thread(target=send_req, args=(results,))
        th.start()
        threads.append(th)

    for i in threads:
        i.join()

    print("test finished")

    success_times = 0
    failed_times = 0
    for result in results:
        if result is True:
            success_times += 1
        elif result is False:
            failed_times += 1

    print("success times: %s, failed times: %s" % (success_times,
                                                   failed_times))


def send_req(results=list()):
    url = "http://30.7.3.124:8081/test_haproxy"
    headers = {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
    }
    body = {"ts": "hhhh"}
    kwargs = {
        "headers": headers,
        "data": json.dumps(body),
        "verify": False,
        "timeout": 60
    }

    try:
        response = requests.request('POST', url, **kwargs)
        if response.status_code == 200:
            print("recv server data: %s" % response.content)
            results.append(True)
            return True
        print("delete request failed, code: %s, msg: %s" %
              (response.status_code, response.content))
        results.append(False)
        return False
    except Exception as err:
        print("send request occur exc, msg: %s" % err.message)
        results.append(False)
        return False


if __name__ == "__main__":
    args = sys.argv
    if not args or len(args) != 2:
        print("Usage: python xx.py [concurrent_num]")
        exit(1)

    test_haproxy(int(args[1]))
```

### 三、服务端
```python
# coding: utf-8
import random

import eventlet
from eventlet import wsgi
from webob import Request, Response


class ResponseBuilder(object):
    def __init__(self):
        super(ResponseBuilder, self).__init__()

    @classmethod
    def success(cls, body=None):
        res = Response()
        res.status = "200 OK"
        res.content_type = "application/json"
        res.body = body
        return res


class TestApp(object):
    def __call__(self, environ, start_response):
        req = Request(environ)
        req.charset = "utf-8"
        req.decode("utf-8")

        url = req.url
        params = req.params
        body = req.body
        client_addr = req.client_addr
        print("recv client req, {url: %(url)s, params: %(params)s, "
              "body: %(body)s, client_addr: %(client_addr)s}" % {
                  "url": url, "params": params, "body": body,
                  "client_addr": client_addr
              })

        if req.path.startswith("/test_haproxy"):
            eventlet.sleep(random.randint(0, 60))
        response = ResponseBuilder.success(body="server response")
        return response(environ, start_response)


def start():
    socket = eventlet.listen(("30.7.3.126", 28081))
    app = TestApp()
    wsgi.server(socket, app)


if __name__ == "__main__":
    start()
```

### 四、启动server
在命令行执行命令：
```sh
python socket_server.py
```
可以看到控制台在周期性的打印如下日志
```sh
(19761) accepted ('30.7.3.126', 53202)
recv client req, {url: http://30.7.3.126:28081/, params: NestedMultiDict([]), body: , client_addr: 30.7.3.126}
30.7.3.126 - - [20/Jun/2019 15:27:57] "OPTIONS / HTTP/1.0" 200 142 0.000472
(19761) accepted ('30.7.3.126', 53214)
recv client req, {url: http://30.7.3.126:28081/, params: NestedMultiDict([]), body: , client_addr: 30.7.3.126}
30.7.3.126 - - [20/Jun/2019 15:27:59] "OPTIONS / HTTP/1.0" 200 142 0.000567
```
这就是我们在配置文件中设置的`check inter 30s`参数，表示haproxy每隔30s使用tcp连接后台服务器端口，如果能建立连接，就认为存活且马上关闭连接。

服务端在成功启动后，haproxy检查日志如下：
```sh
2019-06-20 15:35:59.228 localhost haproxy [pid:19047]notice  Health check for server b_def_ts_8081/controller1 succeeded, reason: Layer7 check passed, code: 200, info: "OK", check duration: 1ms, status: 4/4 UP.
2019-06-20 15:35:59.228 localhost haproxy [pid:19047]notice  Server b_def_ts_8081/controller1 is UP. 1 active and 0 backup servers online. 0 sessions requeued, 0 total in queue.
```

### 五、启动client
在命令行执行命令：
```sh
python socket_client.py 1
```

haproxy的日志如下：
```sh
2019-06-20 02:39:18.371 localhost haproxy [pid:17464]info  30.7.3.126:51921 [20/Jun/2019:02:38:13.367] ts_8081 b_def_ts_8081/controller1 0/0/0/65002/65003 200 142 - - ---- 3/3/3/3/0 0/0 "POST /test_haproxy HTTP/1.1"
2019-06-20 02:39:18.373 localhost haproxy [pid:17464]info  30.7.3.126:51925 [20/Jun/2019:02:38:13.369] ts_8081 b_def_ts_8081/controller1 0/0/0/65002/65003 200 142 - - ---- 2/2/2/2/0 0/0 "POST /test_haproxy HTTP/1.1"
2019-06-20 02:39:18.376 localhost haproxy [pid:13414]info  30.7.3.126:51929 [20/Jun/2019:02:38:13.371] ts_8081 b_def_ts_8081/controller1 0/0/0/65002/65003 200 142 - - ---- 0/0/0/0/0 0/0 "POST /test_haproxy HTTP/1.1"
2019-06-20 02:39:18.387 localhost haproxy [pid:19047]info  30.7.3.126:51933 [20/Jun/2019:02:38:13.383] ts_8081 b_def_ts_8081/controller1 0/0/0/65003/65003 200 142 - - ---- 1/1/1/1/0 0/0 "POST /test_haproxy HTTP/1.1"
2019-06-20 02:39:18.387 localhost haproxy [pid:19047]info  30.7.3.126:51939 [20/Jun/2019:02:38:13.383] ts_8081 b_def_ts_8081/controller1 0/0/0/65003/65003 200 142 - - ---- 0/0/0/0/0 0/0 "POST /test_haproxy HTTP/1.1"
2019-06-20 02:39:18.389 localhost haproxy [pid:17464]info  30.7.3.126:51937 [20/Jun/2019:02:38:13.386] ts_8081 b_def_ts_8081/controller1 0/0/0/65002/65002 200 142 - - ---- 1/1/1/1/0 0/0 "POST /test_haproxy HTTP/1.1"
2019-06-20 02:39:18.395 localhost haproxy [pid:17464]info  30.7.3.126:51945 [20/Jun/2019:02:38:13.391] ts_8081 b_def_ts_8081/controller1 0/0/0/65002/65004 200 142 - - ---- 0/0/0/0/0 0/0 "POST /test_haproxy HTTP/1.1"
```
说明haproxy手动了请求并成功进行了转发。

server也成功收到了请求并回应。
```
(24134) accepted ('30.7.3.126', 56130)
recv client req, {url: http://30.7.3.124:8081/test_haproxy, params: NestedMultiDict([]), body: {"ts": "hhhh"}, client_addr: 30.7.3.126}
```

client收到的信息：
```sh
recv server data: server response
test finished
success times: 1, failed times: 0
```

### 六、启停haproxy
* 启动haproxy 
  `# /usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/conf/haproxy.cfg `

* 重启haproxy 
  `# /usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/conf/haproxy.cfg -st `cat /usr/local/haproxy/haproxy.pid`  `
* 停止haproxy 
  `# killall haproxy `