# Haproxy的安全配置
haproxy在安全上做了很大的努力，对外提供了大量的配置参数来控制请求的行为。
下面我们就一些常见的攻击行为，如何通过直接配置参数来达到防御效果。

### TCP SYN洪水攻击
这个直接修改linux参数达到效果。 
```sh
#vim /etc/sysctl.conf
# Protection SYN flood
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.tcp_max_syn_backlog = 1024 
```
`关于具体的syn攻击原理，请查看这篇博文：https://zhuanlan.zhihu.com/p/29539671`

### Slowloris攻击
Slowloris是在2009年由著名Web安全专家RSnake提出的一种攻击方法，其原理是以极低的速度往服务器发送HTTP请求。由于Web Server对于并发的连接数都有一定的上限，因此若是恶意地占用住这些连接不释放，那么Web Server的所有连接都将被恶意连接占用，从而无法接受新的请求，导致拒绝服务。比如一个字节一个字节的发送头部，服务器将一直处于 wating 状态，从而耗费服务器的资源。
Haproxy 通过配置 timeout http-request 参数，当一个用户的请求时间超过设定值时，Haproxy 断开与该用户的连接。
```sh
defaults
    log     global
    maxconn 2000
    option  redispatch
    retries 3
    mode    http
    option  httplog
    option  splice-auto
    option  http-server-close
    option  http-keep-alive
    option  forwardfor
    option  log-health-checks
    stats   hide-version
    timeout http-request 30s
    timeout queue   1m
    timeout connect 5s
    timeout client  1h
    timeout server  1h
```
上面配置的`timeout http-request 30s`则表示当连接的请求时间超过30s，则直接断开。

在介绍接下来的参数之前，我们需要先插入一个额外的知识点
### Stick table
stick table是haproxy的一个非常优秀的特性，它是一个haproxy用来存储数据的表，存储的是stickiness记录，这些记录包括如下信息：
* 1、stickiness记录了客户端和服务端1:1对应的引用关系。通过这个关系，haproxy可以将客户端的请求引导到之前为它服务过的后端服务器上，也就是实现了会话保持的功能。这种记录方式，俗称会话粘性(stickiness)，即将客户端和服务端粘连起来。stick table中使用key/value的方式映射客户端和后端服务器，key是客户端的标识符，可以使用客户端的源ip(50字节)、cookie以及从报文中过滤出来的部分String。value部分是服务端的标识符。
* 2、除了存储key/value实现最基本的粘性，stick table还可以额外存储每个stickiness记录对应的状态统计数据。比如stickiness记录1目前建立了多少和客户端的连接、平均建立连接的速度是多少、流入流出了多少字节的数据、建立会话的数量等等。

`注：具体的stick table特性请参考这篇博文：https://www.cnblogs.com/f-ck-need-u/p/8558514.html`

接下来我们要说的安全控制，就利用到了第二点的信息

### 用户连接数限制
当用户访问网站，或者从网站下载东西时，浏览器一般会建立多个 TCP 链接，或者短时间内创建了大量的TCP连接，就会耗费服务器大量资源，影响其它用户的访问，因此我们需要根据实际情况，限制同一个用户的链接数。
```sh
frontend test
  bind 0.0.0.0:8080

  # Table definition  
  stick-table type ip size 100k expire 30s store conn_cur
  # Allow clean known IPs to bypass the filter
  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
  # Shut the new connection as long as the client has already 300 opened 
  tcp-request connection reject if { src_conn_cur ge 300 }
  tcp-request connection track-sc1 src
```
* size：定义了这个table可以存储的entry最大数量
* expire：定义entry在stick-table中的过期时间,默认大概24天.
* store：用于存储其他额外信息，这些信息可以用于ACL，用于控制各种与客户端活动相关的标准。多种data type可以用逗号分隔，写在store后边 
  上面的配置表示：
  `如果每个用户的并发连接数（也就是同时连接还未关闭的http请求数）超过10，则直接拒接后续的请求`

###### 注：对于那种通过 NAT 访问网站的场景，因为外部的不同ip映射过来可能都是同一个ip，如果这样配置了，就会导致正常的用户请求也会被拒接，因此需要把 NAT 处的公网地址添加到 whitelist.lst 文件中。


### 用户连接速率限制
仅仅限制单个用户的并发连接数并不能完全解决恶意的DDoS攻击，如果用户在短时间内向服务器不断的发送建立和关闭链接请求，也会耗费服务器资源，影响服务器端的性能，因此需要控制单个用户建立连接的访问速率/频率。
```sh
frontend test
  bind 0.0.0.0:8080

  # Table definition  
  stick-table type ip size 100k expire 30s store conn_cur,conn_rate(3s) 
  # Allow clean known IPs to bypass the filter
  tcp-request connection accept if { src -f /etc/haproxy/whitelist.lst }
  # Shut the new connection as long as the client has already 300 opened or rate more than 200
  tcp-request connection reject if { src_conn_cur ge 300 } || { src_conn_rate ge 200}
  tcp-request connection track-sc1 src
```
上面的配置表示：
`如果每个用户在3s内的并发连接超过300，则直接拒接后续的请求`