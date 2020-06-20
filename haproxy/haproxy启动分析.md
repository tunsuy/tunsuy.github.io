## haproxy启动分析

### 主要数据结构
```sh
frontend test
    bind 21.57.0.212:8799 ssl crt /opt/haproxy/haproxy-cert.pem
    mode http
    default_backend b_def_ts_8888
```

1、proxy：一个proxy可以认为是一个客户，通过一个proxy的流量有着相同的转发规则。haproxy进程可以容纳多个proxy，对应于配置中的listener或frontend。例如上面配置中的test

2、listener：一个监听fd的封装，一个proxy可以有多个listener，对应于配置中的bind，每个listener可以有自己的最大连接数。例如上面配置中的bind

3、connection：一个具体连接fd的封装，可以通过fdtab[fd].owner找到。

4、task：haproxy的一个执行调度单位，想执行点东西一般先激活一个task去执行，比如当socket有事件的时候，把task加入run queue，然后执行task。
task有两类，一类是等待执行的task，比如3秒后做health check，挂在wait queue上，3秒后执行，另一类是马上需要执行的task，挂在run queue上，按『顺序』执行，决定task执行顺序的是task的nice值，类似于UniX进程调度的nice值。

5、stream：stream用于表示一个转发流，可以认为是一个http的transaction，包含前端连接和后端两部分，stream和task是1：1的关系。

6、stream_interface：stream的成员，一个stream有2个stream_interface，分别表示前端和后端，可以认为是connection的一种中转形式，因为haproxy都是异步操作，如果想发数据，不能直接发，需要调用stream_interface的发送，实际上并没有发送，只是注册了写事件，等poll触发后才可以通过connection发送。

7、session：以前版本里的session相当于现在的stream，现在的session已经简化了，只记录了该stream的一些基本属性信息，比如属于哪个frontend，listener，accept时间等，用于记录统计。

8、channel：对数据通道的封装，比如超时，状态，标志，统计等，都在这里设置，一个stream里有2个channel，分别对应请求和响应，channel并不负责发送和接收，只是维护数据及其状态，另外一个channel的作用就是封装http和tcp的channel为一个统一的接口，因为tcp和http不一样，需要统一接口。

9、buffer：channel操作的对象，真正存储数据的地方。


haproxy作为c语言项目，那么其启动入口肯定是main函数，我们找到它: haproxy.c##main

### 解析配置
```sh
main()
 |-init()                                  ← 所有的初始化操作，包括各个模块初始化、命令行解析等
 | |-cfgfiles_expand_directories()         ← 处理配置文件，参数解析会将配置文件保存在cfg_cfgfiles
 | |-init_default_instance()               ← 初始化默认配置
 | |-readcfgfile()                         ← 读取配置文件，并调用sections->section_parser()对应的函数
 | | |-cfg_parse_listen()                  ← 对于frontend、backend、listen段的参数解析验证
 | |   |-str2listener()
 | |     |-l->frontend=curproxy
 | |     |-tcpv4_add_listener()            ← 添加到proto_tcpv4对象中的链表，真正监听会在proto_XXX.c文件中
 | |       |-listener->proto=&proto_tcpv4  ← 会设置该变量，后续的接收链接也就对应了accept变量
 | |-check_config_validity()               ← 配置文件格式校验
 | | |-listener->frontend=curproxy         ← 在上面解析，实际上curporxy->accept=frontend_accept
 | | |-listener->accept=session_accept_fd
 | | |-listener->handler=process_stream

### 初始化listener
listener，顾名思义，就是用于负责处理监听相关的逻辑。

在 haproxy 解析 bind 配置的时候赋值给 listener 的 proto 成员。函数调用流程如下：
​```sh
main()
 |-init()                                  
 | |-readcfgfile()                         ← 读取配置文件，并调用sections->section_parser()对应的函数
 | | |-cfg_parse_listen()                  ← 对于frontend、backend、listen段的参数解析验证
 | |   |-str2listener()
 | |     |-l->frontend=curproxy
 | |     |-tcpv4_add_listener()            ← 添加到proto_tcpv4对象中的链表，真正监听会在proto_XXX.c文件中
 | |       |-listener->proto=&proto_tcpv4  ← 会设置该变量，后续的接收链接也就对应了accept变量
```

### 启动proxy
```c++
int start_proxies(int verbose)
{
	struct proxy *curproxy;
	struct listener *listener;
	int lerr, err = ERR_NONE;
	int pxerr;
	char msg[100];

	for (curproxy = proxies_list; curproxy != NULL; curproxy = curproxy->next) {
		if (curproxy->state != PR_STNEW)
			continue; /* already initialized */

		pxerr = 0;
		list_for_each_entry(listener, &curproxy->conf.listeners, by_fe) {
			if (listener->state != LI_ASSIGNED)
				continue; /* already started */

			lerr = listener->proto->bind(listener, msg, sizeof(msg));

			/* errors are reported if <verbose> is set or if they are fatal */
			if (verbose || (lerr & (ERR_FATAL | ERR_ABORT))) {
				if (lerr & ERR_ALERT)
					ha_alert("Starting %s %s: %s\n",
						 proxy_type_str(curproxy), curproxy->id, msg);
				else if (lerr & ERR_WARN)
					ha_warning("Starting %s %s: %s\n",
						   proxy_type_str(curproxy), curproxy->id, msg);
			}

			err |= lerr;
			if (lerr & (ERR_ABORT | ERR_FATAL)) {
				pxerr |= 1;
				break;
			}
			else if (lerr & ERR_CODE) {
				pxerr |= 1;
				continue;
			}
		}

		if (!pxerr) {
			curproxy->state = PR_STREADY;
			send_log(curproxy, LOG_NOTICE, "Proxy %s started.\n", curproxy->id);
		}

		if (err & ERR_ABORT)
			break;
	}

	return err;
}
```
流程如下：
1、遍历所有的proxy，也就是配置里的frontend
2、对每个proxy遍历listener，也就是frontend下的bind选项。
3、调用各协议bind接口，对TCPv4就是tcp_bind_listener()，把句柄加入IO事件驱动机制中去，这样有新连接进来就会调用event_accept函数接收连接了
注：haproxy对每种协议（udp、tcp）进行了封装，都定义了proto结构体，对外提供统一的结构体访问。

下面对第三点详细说下：
```c++
lerr = listener->proto->bind(listener, msg, sizeof(msg));
```
可以看到是采用的listener这个结构体中proto字段，我们进入该结构体：
下面以ipv4协议为例
```c++
/* Note: must not be declared <const> as its list will be overwritten */
static struct protocol proto_tcpv4 = {
	.name = "tcpv4",
	.sock_domain = AF_INET,
	.sock_type = SOCK_STREAM,
	.sock_prot = IPPROTO_TCP,
	.sock_family = AF_INET,
	.sock_addrlen = sizeof(struct sockaddr_in),
	.l3_addrlen = 32/8,
	.accept = &listener_accept,
	.connect = tcp_connect_server,
	.bind = tcp_bind_listener,
	.bind_all = tcp_bind_listeners,
	.unbind_all = unbind_all_listeners,
	.enable_all = enable_all_listeners,
	.get_src = tcp_get_src,
	.get_dst = tcp_get_dst,
	.pause = tcp_pause_listener,
	.add = tcpv4_add_listener,
	.listeners = LIST_HEAD_INIT(proto_tcpv4.listeners),
	.nb_listeners = 0,
};
```
可以看到bind对应的就是tcp_bind_listener方法.

tcp_bind_listener该方法主要就
```c++
int tcp_bind_listener(struct listener *listener, char *errmsg, int errlen)
{
	...
	
	ready_len = sizeof(default_tcp_maxseg);
	if (default_tcp_maxseg == -1) {
		default_tcp_maxseg = -2;
		fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
		if (fd < 0)
			ha_warning("Failed to create a temporary socket!\n");
		else {
			if (getsockopt(fd, IPPROTO_TCP, TCP_MAXSEG, &default_tcp_maxseg,
			    &ready_len) == -1)
				ha_warning("Failed to get the default value of TCP_MAXSEG\n");
		}
		close(fd);
	}
	if (default_tcp6_maxseg == -1) {
		default_tcp6_maxseg = -2;
		fd = socket(AF_INET6, SOCK_STREAM, IPPROTO_TCP);
		if (fd >= 0) {
			if (getsockopt(fd, IPPROTO_TCP, TCP_MAXSEG, &default_tcp6_maxseg,
			    &ready_len) == -1)
				ha_warning("Failed ot get the default value of TCP_MAXSEG for IPv6\n");
			close(fd);
		}
	}
#endif

	...

	if (!ext && bind(fd, (struct sockaddr *)&listener->addr, listener->proto->sock_addrlen) == -1) {
		err |= ERR_RETRYABLE | ERR_ALERT;
		msg = "cannot bind socket";
		goto tcp_close_return;
	}

	ready = 0;
	ready_len = sizeof(ready);
	if (getsockopt(fd, SOL_SOCKET, SO_ACCEPTCONN, &ready, &ready_len) == -1)
		ready = 0;

	if (!(ext && ready) && /* only listen if not already done by external process */
	    listen(fd, listener_backlog(listener)) == -1) {
		err |= ERR_RETRYABLE | ERR_ALERT;
		msg = "cannot listen to socket";
		goto tcp_close_return;
	}

	...
```
可以unix下网络编程经典的步骤：
1、创建socket: socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)
2、绑定端口：bind(fd, (struct sockaddr *) 
3、开始监听：listen(fd, listener_backlog(listener))

### 启动协议

在分析这部分之前，有一些关联的设计需要简单介绍一下，以便于理解该函数中 的一些代码。

#### 1、fd 更新列表
见 fd.c 中的全局变量：
```c++
/* FD status is defined by the poller's status and by the speculative I/O list */
int fd_nbupdt = 0;             // number of updates in the list
unsigned int *fd_updt = NULL;  // FD updates list
```
这两个全局变量用来记录状态需要更新的 fd 的数量及具体的 fd。

#### 2、fdtab 数据结构
struct fdtab 数据结构在 include/types/fd.h 中定义，内容如下：
```c++
/* info about one given fd */
struct fdtab {
	int (*iocb)(int fd);                 /* I/O handler, returns FD_WAIT_* */
	void *owner;                         /* the connection or listener associated with this fd, NULL if closed */
	unsigned int  spec_p;                /* speculative polling: position in spec list+1. 0=not in list. */
	unsigned char spec_e;                /* speculative polling: read and write events status. 4 bits */
	unsigned char ev;                    /* event seen in return of poll() : FD_POLL_* */
	unsigned char new:1;                 /* 1 if this fd has just been created */
	unsigned char updated:1;             /* 1 if this fd is already in the update list */
};
```
该结构的成员基本上都有注释，除了前两个成员，其余的都是和 fd IO 处理相关的。

src/fd.c 中还有一个全局变量：
```c++
struct fdtab *fdtab = NULL;     /* array of all the file descriptors */
```
fdtab[] 记录了 HAProxy 所有 fd 的信息，数组的每个成员都是一个 struct fdtab， 而且成员的 index 正是 fd 的值，这样相当于 hash，可以高效的定位到某个 fd 对应的 信息。

#### 3、fd event 的设置
include/proto/fd.h 中定义了一些设置 fd event 的函数：
```c++
/* event manipulation primitives for use by I/O callbacks */
static inline void fd_want_recv(int fd)
static inline void fd_stop_recv(int fd)
static inline void fd_want_send(int fd)
static inline void fd_stop_send(int fd)
static inline void fd_stop_both(int fd)
```
这些函数见名知义，就是用来设置 fd 启动或停止接收以及发送的。这些函数底层调用的 是一系列 fd_ev_XXX() 的函数真正的设置 fd。这里简单介绍一下 fd_ev_set() 的代码：
```c++
static inline void fd_ev_set(int fd, int dir)
{
	unsigned int i = ((unsigned int)fdtab[fd].spec_e) & (FD_EV_STATUS << dir);
	...
	if (i & (FD_EV_ACTIVE << dir))
		return; /* already in desired state */
	fdtab[fd].spec_e |= (FD_EV_ACTIVE << dir);
	updt_fd(fd); /* need an update entry to change the state */
}
```
该函数会判断一下 fd 的对应 event 是否已经设置了。没有设置的话，才重新设置。设置 的结果记录在 struct fdtab 结构的 spec_e 成员上，而且只是低 4 位上。然后调用 updt_fd() 将该 fd 放到 update list 中：
```c++
static inline void updt_fd(const int fd)
{
	if (fdtab[fd].updated)
		/* already scheduled for update */
		return;
	fdtab[fd].updated = 1;
	fd_updt[fd_nbupdt++] = fd;
}
```
从上面代码可以看出， struct fdtab 中的 updated 成员用来标记当前 fd 是否已经被放 到 update list 中了。没有的话，则更新设置 updated 成员，并且记录到 fd_updt[] 中， 并且增加需要跟新的 fd 的计数 fd_nbupdt。

至此，一些背景知识介绍完毕。

#### 启动协议监听处理

下面看下启动协议代码
```c++
int protocol_enable_all(void)
{
	struct protocol *proto;
	int err;

	err = 0;
	HA_SPIN_LOCK(PROTO_LOCK, &proto_lock);
	list_for_each_entry(proto, &protocols, list) {
		if (proto->enable_all) {
			err |= proto->enable_all(proto);
		}
	}
	HA_SPIN_UNLOCK(PROTO_LOCK, &proto_lock);
	return err;
}
```

这里是调用了各个协议自己的方法`proto->enable_all(proto)`，以tcp为例:
```c++
static void enable_listener(struct listener *listener)
{
	HA_SPIN_LOCK(LISTENER_LOCK, &listener->lock);
	if (listener->state == LI_LISTEN) {
		if ((global.mode & (MODE_DAEMON | MODE_MWORKER)) &&
		    !(proc_mask(listener->bind_conf->bind_proc) & pid_bit)) {
			/* we don't want to enable this listener and don't
			 * want any fd event to reach it.
			 */
			if (!(global.tune.options & GTUNE_SOCKET_TRANSFER))
				do_unbind_listener(listener, 1);
			else {
				do_unbind_listener(listener, 0);
				listener->state = LI_LISTEN;
			}
		}
		else if (!listener->maxconn || listener->nbconn < listener->maxconn) {
			fd_want_recv(listener->fd);
			listener->state = LI_READY;
		}
		else {
			listener->state = LI_FULL;
		}
	}
	/* if this listener is supposed to be only in the master, close it in the workers */
	if ((global.mode & MODE_MWORKER) &&
	    (listener->options & LI_O_MWORKER) &&
	    master == 0) {
		do_unbind_listener(listener, 1);
	}
	HA_SPIN_UNLOCK(LISTENER_LOCK, &listener->lock);
}
```
该方法会判断该fd是否已经处于监听，如果是则继续处理：
1、如果该fd没有任何的事件到来，则不启动该fd的监听
2、如果有事件到来，并且判断连接数是否达到最大，如果是，则不再处理
3、否则调用接收方法

我们再进入fd_want_recv方法：
```c++
static inline void fd_want_recv(int fd)
{
	if ((fdtab[fd].state & FD_EV_ACTIVE_R) ||
	    HA_ATOMIC_BTS(&fdtab[fd].state, FD_EV_ACTIVE_R_BIT))
		return;
	updt_fd_polling(fd);
}


void updt_fd_polling(const int fd)
{
	if ((fdtab[fd].thread_mask & all_threads_mask) == tid_bit) {

		/* note: we don't have a test-and-set yet in hathreads */

		if (HA_ATOMIC_BTS(&fdtab[fd].update_mask, tid))
			return;

		fd_updt[fd_nbupdt++] = fd;
	} else {
		unsigned long update_mask = fdtab[fd].update_mask;
		do {
			if (update_mask == fdtab[fd].thread_mask)
				return;
		} while (!_HA_ATOMIC_CAS(&fdtab[fd].update_mask, &update_mask,
		    fdtab[fd].thread_mask));
		fd_add_to_fd_list(&update_list, fd, offsetof(struct fdtab, update));
	}
}
```
这个就是对前面对背景知识里面说的fd相关数据结构进行更新
