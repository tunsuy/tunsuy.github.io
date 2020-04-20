## redis事件驱动

### Reactor 模式
它要解决什么问题呢？传统的 thread per connection 用法中，线程在真正处理请求之前首先需要从 socket 中读取网络请求，而在读取完成之前，线程本身被阻塞，不能做任何事，这就导致线程资源被占用，而线程资源本身是很珍贵的，尤其是在处理高并发请求时。

而 Reactor 模式指出，在等待 IO 时，线程可以先退出，这样就不会因为有线程在等待 IO 而占用资源。但是这样原先的执行流程就没法还原了，因此，我们可以利用事件驱动的方式，要求线程在退出之前向 event loop 注册回调函数，这样 IO 完成时 event loop 就可以调用回调函数完成剩余的操作。

### 事件类型
Redis 中事件主要有两种类型：
* 文件事件 ( file event ) Redis 文件事件一般都是 套接字的 读写状态监控，然后回调相应接口进行处理
* 时间事件 ( time event ) 时间事件 则是维护一个定时器，每当满足预设的时间要求，就将该时间事件标记为待处理，然后在 Redis 的事件循环中进行处理。Redis 对于这两种事件的处理优先级是 文件事件优先于时间事件

### 文件事件
文件事件的结构体为
```c++
typedef struct aeFileEvent {
    // 文件事件类型 AE_READABLE|AE_WRITABLE|AE_BARRIER
    // AE_READABLE 可读
    // AE_WRITABLE 可写
    // AE_BARRIER 通常情况下我们都是先执行读事件再执行写事件，如果设置了这个状态，程序永远不会在可读事件后执行可写事件
    int mask;
    // 可读事件回调函数
    aeFileProc *rfileProc;
    // 可写事件回调函数
    aeFileProc *wfileProc;
    // 客户端传入的数据
    void *clientData;
} aeFileEvent;
```
就绪的文件事件结构体为
```c++
typedef struct aeFiredEvent {
    // 就绪文件句柄
    int fd;
    // 文件事件的 读写 类型
    int mask;
} aeFiredEvent;
```

### 时间事件
时间事件的结构体为
```c++
typedef struct aeTimeEvent {
    // 时间事件 id
    long long id;
    // 执行时间结点 秒
    long when_sec;
    // 执行时间结点 毫秒
    long when_ms;
    // 时间事件处理函数
    aeTimeProc *timeProc;
    // 时间事件终结函数
    aeEventFinalizerProc *finalizerProc;
    // 事件数据
    void *clientData;
    // 上一个时间事件
    struct aeTimeEvent *prev;
    // 下一个时间事件
    struct aeTimeEvent *next;
} aeTimeEvent;
```
时间事件为一个双向链表


这些事件都存在于一个aeEventLoop的结构体内
### 事件池
事件池状态结构体为
```c++
typedef struct aeEventLoop {
    // 当前已经注册的事件的最大文件描述符
    int maxfd;
    // 文件描述符监听集合的大小
    int setsize;
    // 下一个时间事件的 ID
    long long timeEventNextId;
    // 上一次事件的执行时间
    time_t lastTime;
    // 已经注册的文件事件表
    aeFileEvent *events;
    // 已经就绪的文件事件表
    aeFiredEvent *fired;
    // 时间事件链表的表头
    aeTimeEvent *timeEventHead;
    // 事件处理开关
    int stop;
    // 事件状态数据，这里使用的为一个万能指针，保存的主要为 底层 IO多路事件状态数据，因为之前我们提到过根据不同系统可能选择不同的 多路事件库 select、epoll、evport、kqueue
    void *apidata;
    // 执行处理事件之前运算的函数
    aeBeforeSleepProc *beforesleep;
    // 执行处理事件之后运算的函数
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```

在每种事件内部，都有定义相应的处理函数，把函数当做变量一样存在结构体中。下面看下ae.c中的一些API的组成:
```c++
aeEventLoop *aeCreateEventLoop(int setsize); /* 创建aeEventLoop，内部的fileEvent和Fired事件的个数为setSize个 */
void aeDeleteEventLoop(aeEventLoop *eventLoop); /* 删除EventLoop，释放相应的事件所占的空间 */
void aeStop(aeEventLoop *eventLoop); /* 设置eventLoop中的停止属性为1 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData); /* 在eventLoop中创建文件事件 */
void aeDeleteFileEvent(aeEventLoop *eventLoop, int fd, int mask); /* 删除文件事件 */
int aeGetFileEvents(aeEventLoop *eventLoop, int fd); //根据文件描述符id，找出文件的属性，是读事件还是写事件
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc); /* 在eventLoop中添加时间事件，创建的时间为当前时间加上自己传入的时间 */
int aeDeleteTimeEvent(aeEventLoop *eventLoop, long long id); //根据时间id，删除时间事件，涉及链表的操作
int aeProcessEvents(aeEventLoop *eventLoop, int flags); /* 处理eventLoop中的所有类型事件 */
int aeWait(int fd, int mask, long long milliseconds); /* 让某事件等待 */
void aeMain(aeEventLoop *eventLoop); /* ae事件执行主程序 */
char *aeGetApiName(void);
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep); /* 每次eventLoop事件执行完后又重新开始执行时调用 */
int aeGetSetSize(aeEventLoop *eventLoop); /* 获取eventLoop的大小 */
int aeResizeSetSize(aeEventLoop *eventLoop, int setsize); /* EventLoop重新调整大小 */
```
无非涉及一些文件，时间事件的添加，修改等，都是在eventLoop内部的修改.

### 事件循环处理器
作为我们在之前的文章中提到的，redis会在启动的时候开启时间循环，下面让我们看下这个代码：
```c++
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    //如果eventLoop中的stop标志位不为1，就循环处理
    while (!eventLoop->stop) {
    	//每次eventLoop事件执行完后又重新开始执行时调用
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        //while循环处理所有的evetLoop的事件
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

```c++
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;
    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;
        // 获取最近的时间事件
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        // 如果时间事件存在的话，那么根据最近可执行时间事件和现在时间的时间差来决定文件事件的阻塞时间
        if (shortest) {
            long now_sec, now_ms;
            // 计算距今最近的时间事件还要多久才能达到，并将该时间距保存在 tv 结构中
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            /* How many milliseconds we need to wait for the next
             * time event to fire? */
            // 需要阻塞时间，时间差小于 0 ，说明事件已经可以执行了，将秒和毫秒设为 0 （不阻塞）
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;
            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            // 执行到这一步，说明没有时间事件，那么根据 AE_DONT_WAIT 是否设置来决定是否阻塞，以及阻塞的时间长
            if (flags & AE_DONT_WAIT) {
                // 设置文件事件不阻塞
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                // 文件事件可以阻塞直到有事件到达为止
                tvp = NULL; /* wait forever */
            }
        }
        /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
        // 处理文件事件，阻塞时间由 tvp 决定
        numevents = aeApiPoll(eventLoop, tvp);
        /* After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */
            /* Normally we execute the readable event first, and the writable
             * event laster. This is useful as sometimes we may be able
             * to serve the reply of a query immediately after processing the
             * query.
             *
             * However if AE_BARRIER is set in the mask, our application is
             * asking us to do the reverse: never fire the writable event
             * after the readable. In such a case, we invert the calls.
             * This is useful when, for instance, we want to do things
             * in the beforeSleep() hook, like fsynching a file to disk,
             * before replying to a client. */
            int invert = fe->mask & AE_BARRIER;
        /* Note the "fe->mask & mask & ..." code: maybe an already
             * processed event removed an element that fired and we still
             * didn't processed, so we check if the event is still valid.
             *
             * Fire the readable event if the call sequence is not
             * inverted. */
            // 读事件
            if (!invert && fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
            /* Fire the writable event. */
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }
            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }
            processed++;
        }
    }
    /* Check time events */
    // 执行时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);
    return processed; /* return the number of processed file/time events */
}
```

函数 aeApiPoll() 调用底层 IO 复用函数 epoll_wait 来获取准备好的事件描述符:
```c++
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    // 事件状态数据
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;
    // 监控是否有时间发生
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize, tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;
        // 就绪事件个数
        numevents = retval;
        // 遍历就绪的事件表，将其加入到 eventLoop 的就绪事件表中
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            // 描述符对应的事件
            struct epoll_event *e = state->events+j;
            // 将 epoll 事件状态转换为 aeEventLoop 对应的 mask 状态
            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            // 添加到就绪事件表中
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    // 返回就绪事件个数
    return numevents;
}
```

### 文件事件处理器
在之前的文章中有说到，在redis启动的时候main方法中:
```c++
for (j = 0; j < server.ipfd_count; j++) {
	if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
		acceptTcpHandler,NULL) == AE_ERR)
		{
			serverPanic(
				"Unrecoverable error creating server.ipfd file event.");
		}
}
```

这里会创建socket的文件事件，进入aeCreateFileEvent方法：
```c++
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```
我们可以知道socket文件事件的回调方法acceptTcpHandler赋值给了rfileProc或者wfileProc。

回到事件循环处理方法中：
```c++
// 读事件
if (!invert && fe->mask & mask & AE_READABLE) {
	// rfired 确保读/写事件只能执行其中一个
	fe->rfileProc(eventLoop,fd,fe->clientData,mask);
	fired++;
}
/* Fire the writable event. */
// 写事件
if (fe->mask & mask & AE_WRITABLE) {
	if (!fired || fe->wfileProc != fe->rfileProc) {
		fe->wfileProc(eventLoop,fd,fe->clientData,mask);
		fired++;
	}
}
```

那我们就知道了eventloop会遍历就绪事件，然后执行该事件的回调方法，也就是acceptTcpHandler：
```c++
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];
    UNUSED(el);
    UNUSED(mask);
    UNUSED(privdata);
    // 循环接受所有 TCP 连接
    while(max--) {
        // accept TCP 连接
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        // 连接错误
        if (cfd == ANET_ERR) {
            if (errno != EWOULDBLOCK)
                serverLog(LL_WARNING,
                    "Accepting client connection: %s", server.neterr);
            return;
        }
        // 记录日志
        serverLog(LL_VERBOSE,"Accepted %s:%d", cip, cport);
        // 初始化客户端连接操作
        acceptCommonHandler(cfd,0,cip);
    }
}
```

### 时间事件处理器
同样的，对于时间事件
在之前的文章中有说到，在redis启动的时候main方法中:
```c++
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
	serverPanic("Can't create event loop timers.");
	exit(1);
}
```

这里会创建时间事件，进入aeCreateTimeEvent方法：
```c++
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    long long id = eventLoop->timeEventNextId++;
    aeTimeEvent *te;

    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    te->id = id;
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->prev = NULL;
    te->next = eventLoop->timeEventHead;
    if (te->next)
        te->next->prev = te;
    eventLoop->timeEventHead = te;
    return id;
}
```

我们可以知道时间事件的回调方法serverCron赋值给了timeProc。

回到事件循环处理方法中：
```c++
// 执行时间事件
if (flags & AE_TIME_EVENTS)
	processed += processTimeEvents(eventLoop);
return processed; /* return the number of processed file/time events */
```

进入processTimeEvents方法：
```c++
static int processTimeEvents(aeEventLoop *eventLoop) {
    int processed = 0;
    aeTimeEvent *te;
    long long maxId;
    time_t now = time(NULL);
    // 这里尝试发现时间混乱的情况，上一次处理事件的时间比当前时间还要大，重置最近一次处理事件的时间
    if (now < eventLoop->lastTime) {
        te = eventLoop->timeEventHead;
        while(te) {
            te->when_sec = 0;
            te = te->next;
        }
    }
    // 设置上一次时间事件处理的时间为当前时间
    eventLoop->lastTime = now;
    // 遍历时间事件链表
    te = eventLoop->timeEventHead;
    // 最大的 时间事件 id
    maxId = eventLoop->timeEventNextId-1;
    while(te) {
        long now_sec, now_ms;
        long long id;
        // 如果时间事件已经被删除，从链表中移除
        if (te->id == AE_DELETED_EVENT_ID) {
            aeTimeEvent *next = te->next;
            if (te->prev)
                te->prev->next = te->next;
            else
                eventLoop->timeEventHead = te->next;
            if (te->next)
                te->next->prev = te->prev;
            // 调用时间事件终结方法清除该事件
            if (te->finalizerProc)
                te->finalizerProc(eventLoop, te->clientData);
            zfree(te);
            te = next;
            continue;
        }
        // 如果当前 id 大于最大 id，忽略，因为我们插入事件的时候都是头插法
        if (te->id > maxId) {
            te = te->next;
            continue;
        }
        // 获取当前时间
        aeGetTime(&now_sec, &now_ms);
        // 找到到期时间事件
        if (now_sec > te->when_sec ||
            (now_sec == te->when_sec && now_ms >= te->when_ms))
        {
            int retval;
            id = te->id;
            // 回调时间事件函数
            retval = te->timeProc(eventLoop, id, te->clientData);
            processed++;
            // 如果不是定时事件，周期性继续循环
            if (retval != AE_NOMORE) {
                aeAddMillisecondsToNow(retval,&te->when_sec,&te->when_ms);
            } else {
            // 如果是定时事件，执行一次就可以删除了
                te->id = AE_DELETED_EVENT_ID;
            }
        }
        te = te->next;
    }
    return processed;
}
```

其中`retval = te->timeProc(eventLoop, id, te->clientData);`就是在执行时间事件的回调方法，也就是serverCron