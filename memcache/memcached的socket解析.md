# memcached的socket解析

Memcached一共用到了3种套接字(即: TCP, UDP和NUIX域套 接字)

## UNIX Domain Socket 与 TCP/IP Socket 对比
* socket API原本是为网络通讯设计的，但后来在socket的框架上发展出一种IPC机制，就是UNIX Domain Socket。
* 虽然网络socket也可用于同一台主机的进程间通讯（通过loopback地址127.0.0.1），  
  但是UNIX Domain Socket用于IPC更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。
* UNIX域套接字与TCP套接字相比较，在同一台主机的传输速度前者是后者的两倍。  
  这是因为，IPC机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。
* UNIX Domain Socket也提供面向流和面向数据包两种API接口，类似于TCP和UDP，但是面向消息的UNIX Domain Socket也是可靠的，消息既不会丢失也不会顺序错乱。

## TCP Socket选项
文件：memcached.c##server_socket方法中：
```c++
#ifdef IPV6_V6ONLY
	if (next->ai_family == AF_INET6) {
		error = setsockopt(sfd, IPPROTO_IPV6, IPV6_V6ONLY, (char *) &flags, sizeof(flags));
		if (error != 0) {
			perror("setsockopt");
			close(sfd);
			continue;
		}
	}
#endif

setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, (void *)&flags, sizeof(flags));
if (IS_UDP(transport)) {
	maximize_sndbuf(sfd);
} else {
	error = setsockopt(sfd, SOL_SOCKET, SO_KEEPALIVE, (void *)&flags, sizeof(flags));
	if (error != 0)
		perror("setsockopt");

	error = setsockopt(sfd, SOL_SOCKET, SO_LINGER, (void *)&ling, sizeof(ling));
	if (error != 0)
		perror("setsockopt");

	error = setsockopt(sfd, IPPROTO_TCP, TCP_NODELAY, (void *)&flags, sizeof(flags));
	if (error != 0)
		perror("setsockopt");
}
```
### IPV6_V6ONLY

设定IPV6的选项值，设置了IPV6_V6ONLY，表示只收发IPV6的数据包，此时IPV4和IPV6可以绑定到同一个端口而不影响数据的收发

### SO_REUSEADDR 重用地址信息
经常在开启一个服务器之后，处于某些原因需要重启，我们经常做的是关闭服务器，然后马上开启，这时经常会出现＂Port is already in use＂的错误，这是因为，计算机上不允许两个进程绑定到同一个端口．上述出现错误的原因是服务器刚关闭时，还处于time_wait状态，还没有完全释放端口，所以重用会报错．但是tcp提供一个选项SO_REUSEADDR来设置处于time_wait的端口重用．  
注：必须在bind操作之前设置

### SO_KEEPALIVE 保活
对于高并发的服务器，服务器会有很多客户端连接，如果有些客户端突然断电，因为没有给服务器发送数据，所以服务器也不知道这个客户端已＂死＂，这样会占着服务器一个文件描述符，如果有很多这样的客户端，这样必然会降低服务器端的并发性．  
于是tcp套接字就有了这样一个保持存活的选项．即如果在２小时(/proc/sys/net/ipv4/tcp_keepalive_time 7200 即2小时)内该套接字的任何一方向上都没有数据交换，TCP就自动给对方发送一个保持存活探测分节(keep-alive probe)，这是对端必须响应的一个TCP分节．保活分节会导致以下三种情况发生：
* client端连接正常,返回一个ACK.server端收到ACK后重置计时器,在2小时后在发送探测.如果2小时内连接上有数据传输,那么在该时间的基础上向后推延2小时发送探测包;
* 客户端异常关闭,或网络断开。client无响应,server收不到ACK,在一定时间(/proc/sys/net/ipv4/tcp_keepalive_intvl 75 即75秒)后重发keepalive packet, 并且重发一定次数(/proc/sys/net/ipv4/tcp_keepalive_probes 9 即9次);，如果还是没有回应，则放弃，套接字关闭；
* 客户端曾经崩溃,但已经重启.server收到的探测响应是一个复位,该套接字被置为ECONNREST，套接字本身则被关闭． 

### SO_LINGER

在讲这个选项之前，可以先了解下shutdown和close这两个函数的区别．

1、close函数主要是把描述符的引用计数减一，仅在该计数变为０时，才关闭这个套接字．当调用close(fd)时，这时就不能往这个fd读写数据了，然而tcp会尝试发送已排队等待发送到对端的任何数据，最后再发送FIN．
* 对于close减少引用计数，主要是用在多进程环境中，子进程继承父进程的fd，

2、shutdown函数依赖与参数howto，但是它不会将描述符引用计数减一而是直接切断连接．
* shutdown函数可以关闭一半，也可以全关闭，取决为howto
* SHUT_RD　关闭连接的读这一半－－套接字不再有数据可以接收，而且该套接字中现有的数据都被丢弃．进程不能对该套接字调用任何读函数．
* SHUT_WR　关闭连接的写一半－－对于TCP套接字，这称为半关闭．当前留在套接字发送缓冲区中的数据将被发送掉，后跟TCP正常终止序列．不管套接字引用计数是否为0，写半部照样关闭．进程不能对套接字调用任何写函数．
* SHUT_RDWR　连接的读半部和写半部都关闭．这等于调用两次shutdown，一次关闭读，一次关闭写．

好，下面来看下SO_LINGER：
```c++
struct linger {
     int l_onoff; /* 0 = off, nozero = on */
     int l_linger; /* linger time */
};
```
第一个参数为这个选项的开关，第二个参数为延迟时间 有三种情况:
* 置 l_onoff为0，则该选项关闭，l_linger的值被忽略，等于内核缺省情况，close调用会立即返回给调用者，如果可能将会传输任何未发送的数据；
* 设置l_onoff为非0，l_linger为0，则套接口关闭时，TCP将丢弃保留在套接口发送缓冲区中的任何数据并发送一个RST给对方，而不是通常的四分组终止序列，这避免了TIME_WAIT状态；
* 设置 l_onoff 为非0，l_linger为非0，当套接口关闭时内核将拖延一段时间（由l_linger决定）。如果套接口缓冲区中仍残留数据，进程将处于睡眠状态，直 到所有数据发送完且被对方确认，之后进行正常的终止序列（描述字访问计数为0）或者延迟时间到。此种情况下，应用程序检查close的返回值是非常重要的，如果在数据发送完并被确认前时间到，close将返回EWOULDBLOCK错误且套接口发送缓冲区中的任何数据都丢失。close的成功返回仅告诉我们发送的数据（和FIN）已由对方TCP确认，它并不能告诉我们对方应用进程是否已读了数据。如果套接口设为非阻塞的，它将不等待close完成。

### TCP_NODELAY
TCP_NODELAY是为了关闭Nagle's Algorithm．

Nagles Algorithm是为了提高带宽利用率设计的算法，其做法是合并小的TCP包为一个，避免了过多的小报文的TCP头头所浪费的宽带．如果开启了这个算法(默认),则协议栈会积累数据直到以下两个条件之一满足的时候才真正发送出去:
* 积累的数据量达到最大的TCP Segment Size
* 收到了一个Ack
  还有一个算法经常和Nagles Algorithm算法配合使用，称为TCP Delayed Acknoledgement，这个算法也是为了类似的目的被设计出来的，它的作用就是延迟Ack包的发送，使得协议栈有机会合并多个Ack，提高网络性能．

如果对端开启ack延滞算法，那么发送端就可以积累更多的包为一个tcp包．对端的ack会在以下两种情况下发回发送端
* 超时时间到，默认为40ms
* 对端有数据回显，则可以捎带上这个ack

有以下两种情况不适合使用Nagles算法，
* 对于其服务器不在相反方向产生数据以便携带ACK的客户来说，ACK延滞算法出现问题．客户会明显感到延迟，因为客户TCP需要等待服务器的ACK延滞定时器超时才能才继续给服务器发送数据．这些客户需要禁用Nagles算法，TCP_NODELAY算法就起到这个作用．
* 另一类不适用使用Nagles算法和TCP延滞算法的客户是以若干小片数据向服务器发送单个逻辑请求的客户．例如像memcache的set命令分为两次发送，而且每次都是小片数据，因为第一次发送set键值部分，服务器是不会回复数据，所以必须等待ack延滞超时才能发送set命令值部分，这就出现延迟了．所以memcache必须关闭Nagles算法．redis有些命令也是类似

注：
如果服务器有多个ip信息，可以在每个(ip,port)上面绑定一个Memcached实例
