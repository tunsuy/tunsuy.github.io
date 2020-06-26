# memcached防DDOS攻击

## 前言
针对memcached的DDoS攻击与所有DDoS放大攻击（例如NTP或DNS放大）的运行方式相似。该攻击通过将欺骗的请求发送到易受攻击的服务器来工作，该服务器随后会以比初始请求大的数据响应，从而放大了流量。

这种放大攻击的方法是可能存在的，因为memcache缓存服务器可以选择使用UDP协议进行操作。 UDP是一种网络协议，允许在不首先获得所谓握手的情况下发送数据，这是双方都同意进行通信的网络过程。UDP不需要询问目标主机是否愿意接收数据就可以将大量数据发送到目标主机。

## 攻击步骤
* 1、攻击者将大型项目设置到暴露的Memcached服务器中。  
* 2、攻击者用目标受害者的IP地址发送大量GET请求到memcache服务器。  
* 3、接收到请求的memcache缓存服务器会将大量响应发送到目标。  
* 4、目标服务器或其周围的基础服务无法处理从memcache缓存服务器发送的大量数据，从而导致过载和拒绝合法请求的服务。

## 阻止攻击
对于memcache缓存服务器，可以在不需要时禁用UDP支持。默认情况下，在1.5.6版和更高版本中，UDP默认是禁用的。
对于其他版本，可以通过在启动参数中添加`-U 0`来禁用它。如果使用配置文件，可以通过在/etc/memcached.conf中添加以下行来禁用UDP：
```sh
# Disable UDP protocol
-U 0
```
保存该/etc/memcached.conf文件然后重启服务

检查是否生效：
```sh
$ sudo netstat -plunt

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:11211         0.0.0.0:*               LISTEN      16079/memcached
```
可以看到，memcache只监听在tcp上

注：当然也可以通过防火墙来减轻DDoS攻击
