## 一、概览
haproxy有两种策略支持ssl。

### 1、SSL Termination
该策略是在haproxy处终止/解密SSL连接，并将未加密的连接发送到后端服务器的做法。
这意味着haproxy负责解密SSL连接 - 相对于接受非SSL请求而言，这是一个耗时且占用CPU的过程。

这与SSL Pass-Through相反，后者将SSL连接直接发送到代理服务器。

### 2、SSL-Pass-Through
SSL连接在每个代理服务器上终止，从而在这些服务器之间分配CPU负载。但是，这种方式将无法添加或编辑HTTP标头，因为连接只是通过负载平衡器路由到代理服务器。
这意味着server服务器将无法获取X-Forwarded-*标头，这可能包括客户端的IP地址，端口等。
选择哪种策略取决于应用程序需求。SSL Termination是最典型的用法，但是SSL-Pass-Through可能更安全。

## 二、SSL Termination
如上所述，我们需要让haproxy处理SSL连接。这意味着在haproxy服务器上存在SSL证书。
该证书一般是一个pem文件，该文件本质上只是证书，包含一个文件的密钥和可选的证书颁发机构。这是HAProxy读取SSL证书的首选方式。

要在HAProxy中处理SSL连接，需要绑定一个端口，比如443，并让HAProxy知道SSL证书的位置：
```sh
frontend ts_8799
    bind 30.7.20.109:8799 ssl crt /opt/ts/server-cert/haproxy/haproxy-cert.pem
    mode http
    option httpclose
    default_backend b_def_ts_8799
```
该配置就表示，haproxy自身监听在8799端口，在接收到https请求后，就会根据这个配置中的证书进行解密，然后将解密后的请求转发给后端

后端配置如下：
```sh
backend b_def_ts_8799
    mode http
    balance roundrobin
    option tcpka
    stats hide-version
    option httpchk
    option httplog
    server controller1 30.7.20.111:28799 check inter 15s fastinter 15s downinter 15s rise 2 fall 4
```
此时后台服务器收到的就是http类型的请求，数据都是没有加密的。

## 三、SSL-Pass-Through
通过SSL Pass-Through，将让后端服务器处理SSL连接，而不是haproxy。
然后，haproxy的工作就是将请求代理到其配置的后端服务器。由于连接仍然是加密的，因此除了将请求重定向到另一台服务器之外，HAProxy无法对其执行任何操作。

要在HAProxy中直接透传SSL连接，需要在前端和后端配置中使用TCP模式。HAProxy将连接视为代理服务器的信息流，而不是使用其可用于HTTP请求的功能。
前端配置：
```sh
frontend ts_8799
    bind 30.7.20.109:8799
    mode tcp
    default_backend b_def_ts_8799
```
如上所述，要将安全连接传递给后端服务器而不加密它，我们需要使用TCP模式（mode tcp）。这也意味着需要将日志记录设置为tcp而不是默认的http（option tcplog）。
后端配置：
```sh
backend b_def_ts_8799
    mode tcp
    balance roundrobin
    option tcpka
    stats hide-version
    server controller1 30.7.20.111:28799 check inter 15s fastinter 15s downinter 15s rise 2 fall 4
```
此设置为mode tcp - 需要将前端和后端配置都设置为此模式。
当然，后台服务器需要支持解密SSL。

## 四、同时使用两种策略
如果应用需要同时采用两种策略，即在console发送到haproxy，haproxy接收到请求，进行ssl验证之后；
在haproxy发送到后台服务器，后台服务器接收到请求，也需要再一次进行ssl验证。这就意味着haproxy在解密之后，还需要再次加密后才能传输给后台服务器。
前端配置：
```sh
frontend ts_8799
    bind 30.7.20.109:8799 ssl crt /opt/ts/server-cert/haproxy/haproxy-cert.pem
    mode http
    option httpclose
    default_backend b_def_ts_8799
```
后端配置如下：
```sh
backend b_def_ts_8799
    mode http
    balance roundrobin
    option tcpka
    stats hide-version
    option httpchk
    option httplog
    server controller1 30.7.20.111:28799 check inter 15s fastinter 15s downinter 15s rise 2 fall 4 ca-file /opt/ts/server-ca/ca-cert.pem ssl verify required
```
这就表示，haproxy在收到请求之后，通过frontend中配置的证书解密之后，还需要通过backend中配置的ca证书进行加密之后再发送给后台服务器。