# memcached启动分析

入口文件:
memcached.c

入口函数：
main()

参数校验就直接略过

## 初始化主线程的libevent
```c++
main_base = event_init();
```

## 初始化stats信息
在文本协议的memcached中，我们nc/telent后输入stats命令，会很快地输出一些当前memcached的信息的。这些就是stats信息。并不是输入stats的时候才遍历统计出来的。而是已经保存好了这份信息。代码调用在main函数中的：
```c++
static void stats_init(void) {
    memset(&stats, 0, sizeof(struct stats));
    memset(&stats_state, 0, sizeof(struct stats_state));
    stats_state.accepting_conns = true; /* assuming we start in this state. */

    /* make the time we started always be 2 seconds before we really
       did, so time(0) - time.started is never zero.  if so, things
       like 'settings.oldest_live' which act as booleans as well as
       values are now false in boolean context... */
    process_started = time(0) - ITEM_UPDATE_INTERVAL - 2;
    stats_prefix_init(settings.prefix_delimiter);
}
```

## 初始化连接信息
```c++
static void conn_init(void) {
    /* We're unlikely to see an FD much higher than maxconns. */
    int next_fd = dup(1);
    if (next_fd < 0) {
        perror("Failed to duplicate file descriptor\n");
        exit(1);
    }
    int headroom = 10;      /* account for extra unexpected open FDs */
    struct rlimit rl;

    max_fds = settings.maxconns + headroom + next_fd;

    /* But if possible, get the actual highest FD we can possibly ever see. */
    if (getrlimit(RLIMIT_NOFILE, &rl) == 0) {
        max_fds = rl.rlim_max;
    } else {
        fprintf(stderr, "Failed to query maximum file descriptor; "
                        "falling back to maxconns\n");
    }

    close(next_fd);

    if ((conns = calloc(max_fds, sizeof(conn *))) == NULL) {
        fprintf(stderr, "Failed to allocate connection structures\n");
        /* This is unrecoverable so bail out early. */
        exit(1);
    }
}
```
为了更快地找到connection的fd（文件描述符），实际上申请的connection会比配置的更大一点。

## hash桶初始化
```c++
void assoc_init(const int htable_init) {
    if (hashtable_init) {
        hashpower = hashtable_init;
    }
    primary_hashtable = calloc(hashsize(hashpower), sizeof(void *));
    if (! primary_hashtable) {
        fprintf(stderr, "Failed to init hashtable.\n");
        exit(EXIT_FAILURE);
    }
    STATS_LOCK();
    stats_state.hash_power_level = hashpower;
    stats_state.hash_bytes = hashsize(hashpower) * sizeof(void *);
    STATS_UNLOCK();
}

```
在memcached中，保存着一份hash表用来存放memcached key。默认这个hash表是2^16(65536)个key。后续会根据规则动态扩容这个hash表的。如果希望启动的时候，这个hash表更大，可以-o 参数调节。
hash表中， memcached key作为key，value是item指针，并不是item value。

## 初始化slabs
```c++
void slabs_init(const size_t limit, const double factor, const bool prealloc, const uint32_t *slab_sizes, void *mem_base_external, bool reuse_mem) {
    int i = POWER_SMALLEST - 1;
    unsigned int size = sizeof(item) + settings.chunk_size;

    /* Some platforms use runtime transparent hugepages. If for any reason
     * the initial allocation fails, the required settings do not persist
     * for remaining allocations. As such it makes little sense to do slab
     * preallocation. */
    bool __attribute__ ((unused)) do_slab_prealloc = false;

    mem_limit = limit;

    if (prealloc && mem_base_external == NULL) {
        mem_base = alloc_large_chunk(mem_limit);
        if (mem_base) {
            do_slab_prealloc = true;
            mem_current = mem_base;
            mem_avail = mem_limit;
        } else {
            fprintf(stderr, "Warning: Failed to allocate requested memory in"
                    " one large chunk.\nWill allocate in smaller chunks\n");
        }
    } else if (prealloc && mem_base_external != NULL) {
        // Can't (yet) mix hugepages with mmap allocations, so separate the
        // logic from above. Reusable memory also force-preallocates memory
        // pages into the global pool, which requires turning mem_* variables.
        do_slab_prealloc = true;
        mem_base = mem_base_external;
        // _current shouldn't be used in this case, but we set it to where it
        // should be anyway.
        if (reuse_mem) {
            mem_current = ((char*)mem_base) + mem_limit;
            mem_avail = 0;
        } else {
            mem_current = mem_base;
            mem_avail = mem_limit;
        }
    }

    memset(slabclass, 0, sizeof(slabclass));

    while (++i < MAX_NUMBER_OF_SLAB_CLASSES-1) {
        if (slab_sizes != NULL) {
            if (slab_sizes[i-1] == 0)
                break;
            size = slab_sizes[i-1];
        } else if (size >= settings.slab_chunk_size_max / factor) {
            break;
        }
        /* Make sure items are always n-byte aligned */
        if (size % CHUNK_ALIGN_BYTES)
            size += CHUNK_ALIGN_BYTES - (size % CHUNK_ALIGN_BYTES);

        slabclass[i].size = size;
        slabclass[i].perslab = settings.slab_page_size / slabclass[i].size;
        if (slab_sizes == NULL)
            size *= factor;
        if (settings.verbose > 1) {
            fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
                    i, slabclass[i].size, slabclass[i].perslab);
        }
    }

    power_largest = i;
    slabclass[power_largest].size = settings.slab_chunk_size_max;
    slabclass[power_largest].perslab = settings.slab_page_size / settings.slab_chunk_size_max;
    if (settings.verbose > 1) {
        fprintf(stderr, "slab class %3d: chunk size %9u perslab %7u\n",
                i, slabclass[i].size, slabclass[i].perslab);
    }

    /* for the test suite:  faking of how much we've already malloc'd */
    {
        char *t_initial_malloc = getenv("T_MEMD_INITIAL_MALLOC");
        if (t_initial_malloc) {
            mem_malloced = (size_t)atol(t_initial_malloc);
        }

    }

    if (do_slab_prealloc) {
        if (!reuse_mem) {
            slabs_preallocate(power_largest);
        }
    }
}
```
hash桶中初始化的是key。slabs初始化的是这些key对应的value
在初始化slab的时候，下一个slab的size（chunk size）总是大于等于当前slab的size的。

## 初始化worker线程
```c++
void memcached_thread_init(int nthreads, void *arg) {
    int         i;
    int         power;

    for (i = 0; i < POWER_LARGEST; i++) {
        pthread_mutex_init(&lru_locks[i], NULL);
    }
    pthread_mutex_init(&worker_hang_lock, NULL);

    pthread_mutex_init(&init_lock, NULL);
    pthread_cond_init(&init_cond, NULL);

    pthread_mutex_init(&cqi_freelist_lock, NULL);
    cqi_freelist = NULL;

    /* Want a wide lock table, but don't waste memory */
    if (nthreads < 3) {
        power = 10;
    } else if (nthreads < 4) {
        power = 11;
    } else if (nthreads < 5) {
        power = 12;
    } else if (nthreads <= 10) {
        power = 13;
    } else if (nthreads <= 20) {
        power = 14;
    } else {
        /* 32k buckets. just under the hashpower default. */
        power = 15;
    }

    if (power >= hashpower) {
        fprintf(stderr, "Hash table power size (%d) cannot be equal to or less than item lock table (%d)\n", hashpower, power);
        fprintf(stderr, "Item lock table grows with `-t N` (worker threadcount)\n");
        fprintf(stderr, "Hash table grows with `-o hashpower=N` \n");
        exit(1);
    }

    item_lock_count = hashsize(power);
    item_lock_hashpower = power;

    item_locks = calloc(item_lock_count, sizeof(pthread_mutex_t));
    if (! item_locks) {
        perror("Can't allocate item locks");
        exit(1);
    }
    for (i = 0; i < item_lock_count; i++) {
        pthread_mutex_init(&item_locks[i], NULL);
    }

    threads = calloc(nthreads, sizeof(LIBEVENT_THREAD));
    if (! threads) {
        perror("Can't allocate thread descriptors");
        exit(1);
    }

    for (i = 0; i < nthreads; i++) {
        int fds[2];
        if (pipe(fds)) {
            perror("Can't create notify pipe");
            exit(1);
        }

        threads[i].notify_receive_fd = fds[0];
        threads[i].notify_send_fd = fds[1];
#ifdef EXTSTORE
        threads[i].storage = arg;
#endif
        setup_thread(&threads[i]);
        /* Reserve three fds for the libevent base, and two for the pipe */
        stats_state.reserved_fds += 5;
    }

    /* Create threads after we've done all the libevent setup. */
    for (i = 0; i < nthreads; i++) {
        create_worker(worker_libevent, &threads[i]);
    }

    /* Wait for all the threads to set themselves up before returning. */
    pthread_mutex_lock(&init_lock);
    wait_for_thread_registration(nthreads);
    pthread_mutex_unlock(&init_lock);
}
```
worker线程和main线程，组成了libevent的reactor模式

这个函数主要是完成n个子线程的初始化以及开启执行。其中setup_thread函数完成线程结构体LIBEVENT_THREAD成员初始化，

首先将给子线程分配一个libevent实例，然后将notify_receive_fd加入这个libevent的可读事件。
接着为这个子线程的消息队列分配内存，并初始化。
最后为这个子线程创建后缀缓存，暂时还不知道这缓存的用处。
create_worker函数用于开启子线程，第一个参数为回调函数，第二个参数为回调函数的参数。回调函数即线程执行函数。在这个回调函数worker_libevent又调用了register_thread_initialized这个函数，注册这个子线程。最后调用libevent的event_base_loop函数开启子线程的事件循环。

```c++
static void setup_thread(LIBEVENT_THREAD *me) {
    me->base = event_init();//分配一个libevent实例
    if (! me->base) {
        fprintf(stderr, "Can't allocate event base\n");
        exit(1);
    }
    /* Listen for notifications from other threads */
    event_set(&me->notify_event, me->notify_receive_fd,
              EV_READ | EV_PERSIST, thread_libevent_process, me);
    event_base_set(me->base, &me->notify_event);
    //将管道添加libevent事件循环中
    if (event_add(&me->notify_event, 0) == -1) {
        fprintf(stderr, "Can't monitor libevent notify pipe\n");
        exit(1);
    }
    //给消息队列分配内存
    me->new_conn_queue = malloc(sizeof(struct conn_queue));
    if (me->new_conn_queue == NULL) {
        perror("Failed to allocate memory for connection queue");
        exit(EXIT_FAILURE);
    }
    //初始化消息队列，将head和tail初始化为NULL
    cq_init(me->new_conn_queue);
    if (pthread_mutex_init(&me->stats.mutex, NULL) != 0) {
        perror("Failed to initialize mutex");
        exit(EXIT_FAILURE);
    }
   //分配缓存
    me->suffix_cache = cache_create("suffix", SUFFIX_SIZE, sizeof(char*),
                                    NULL, NULL);
    if (me->suffix_cache == NULL) {
        fprintf(stderr, "Failed to create suffix cache\n");
        exit(EXIT_FAILURE);
    }
}
/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
static void create_worker(void *(*func)(void *), void *arg) {
    pthread_t       thread;
    pthread_attr_t  attr;
    int             ret;
    pthread_attr_init(&attr);
    //开启子线程执行，子线程函数为回调函数func
    if ((ret = pthread_create(&thread, &attr, func, arg)) != 0) {
        fprintf(stderr, "Can't create thread: %s\n",
                strerror(ret));
        exit(1);
    }
}
/*++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*/
static void *worker_libevent(void *arg) {
    LIBEVENT_THREAD *me = arg;
    /* Any per-thread setup can happen here; memcached_thread_init() will block until
     * all threads have finished initializing.
     */
    register_thread_initialized();
    //开启事件循环
    event_base_loop(me->base, 0);
    return NULL;
}
```
到此为止，子线程就建立完毕，此时，子线程都已经处于libevent的事件循环当中，而且只有管道的可读一个事件。


## 启动主线程socket监听
```c++
/* create the listening socket, bind it, and init */
if (settings.socketpath == NULL) {
	
	...

	errno = 0;
	if (settings.port && server_sockets(settings.port, tcp_transport,
									   portnumber_file)) {
		vperror("failed to listen on TCP port %d", settings.port);
		exit(EX_OSERR);
	}

	...
```
server_sockets中调用了server_socket()函数
核心代码如下：
```c++
static int server_socket(const char *interface,
                         int port,
                         enum network_transport transport,
                         FILE *portnumber_file, bool ssl_enabled) {
    // ...

    for (next= ai; next; next= next->ai_next) {
        conn *listen_conn_add;
        if ((sfd = new_socket(next)) == -1) {
            // ...
            continue;
        }

        // 省略ipv6逻辑...
        if (bind(sfd, next->ai_addr, next->ai_addrlen) == -1) {
            // ...
            close(sfd);
            continue;
        } else {
            success++;
            if (!IS_UDP(transport) && listen(sfd, settings.backlog) == -1) {
                // ...
            }
            // ...
        }

        // 省略udp逻辑...
            if (!(listen_conn_add = conn_new(sfd, conn_listening,
                                             EV_READ | EV_PERSIST, 1,
                                             transport, main_base, NULL))) {
            // ...
        }

    // ...
}

```
代码中使用socket()和bind()方法启动了tcp服务器，并得到这个代表memcached server的fd

我们进入conn_new中，看到主线程的event_handler回调函数如下：
```c++
void event_handler(const int fd, const short which, void *arg) {
    conn *c;

    c = (conn *)arg;
    assert(c != NULL);

    c->which = which;

    /* sanity */
    if (fd != c->sfd) {
        if (settings.verbose > 0)
            fprintf(stderr, "Catastrophic: event fd doesn't match conn fd!\n");
        conn_close(c);
        return;
    }

    drive_machine(c);

    /* wait for next event */
    return;
}

```
可以看到这个event_handler里面只是做了一些错误检查，然后就立即交给drive_machine函数进行处理了
这个drive_machine逻辑如下:
```c+
addrlen = sizeof(addr);
#ifdef HAVE_ACCEPT4
    if (use_accept4) {
        sfd = accept4(c->sfd, (struct sockaddr *)&addr, &addrlen, SOCK_NONBLOCK);
    } else {
        sfd = accept(c->sfd, (struct sockaddr *)&addr, &addrlen);
    }
#else
    sfd = accept(c->sfd, (struct sockaddr *)&addr, &addrlen);
#endif
    // 省略流程无关代码...

#ifdef TLS
        // ssl相关逻辑省略...
#endif

        dispatch_conn_new(sfd, conn_new_cmd, EV_READ | EV_PERSIST,
                             DATA_BUFFER_SIZE, c->transport, ssl_v);
    }

    stop = true;

```
可以看到master线程的tcp服务器有新客户端连接进入后，drive_machine处理逻辑会立即获取这个memcached客户端连接fd为sfd，并交给了dispatch_conn_new函数进行处理
dispatch_conn_new函数核心逻辑如下:
```c++
void dispatch_conn_new(int sfd, enum conn_states init_state, int event_flags,
                       int read_buffer_size, enum network_transport transport, void *ssl) {
    CQ_ITEM *item = cqi_new();
    char buf[1];
    if (item == NULL) {
        close(sfd);
        /* given that malloc failed this may also fail, but let's try */
        fprintf(stderr, "Failed to allocate memory for connection object\n");
        return ;
    }

	//选择子线程
    int tid = (last_thread + 1) % settings.num_threads;

    LIBEVENT_THREAD *thread = threads + tid;

    last_thread = tid;

    item->sfd = sfd;
    item->init_state = init_state;
    item->event_flags = event_flags;
    item->read_buffer_size = read_buffer_size;
    item->transport = transport;
    item->mode = queue_new_conn;
    item->ssl = ssl;

	//将消息实例插入选择的子线程的消息队列中
    cq_push(thread->new_conn_queue, item);

    MEMCACHED_CONN_DISPATCH(sfd, thread->thread_id);
    buf[0] = 'c';
	//给这个子线程发送一个字符，激活管道可读事件。
    if (write(thread->notify_send_fd, buf, 1) != 1) {
        perror("Writing to thread notify pipe");
    }
}

```
选择一个worker线程用于处理这个客户端连接后续的读写操作
声明了一个CQ_ITEM结构体代表这个客户端的连接，并保存到worker线程的new_conn_queue属性上，这个属性是个数组，并且被当成队列使用了
向worker线程的notify_send_fd属性发送一个字符'c'通知worker线程有新客户端连接分配给它了
后续worker线程的event_base监听到notify_receive_fd的可读事件就开始工作了, 因为这里使用pipe线程通信技术，所以notify_send_fd写数据，触发notify_receive_fd的可读事件.

注：1、memcache的消息是预先分配的，默认先分配64个CQ_ITEM消息实例，这样可以避免较多的内存碎片产生。所以CQ_ITEM *item = cqi_new()如果是第一次调用，则先分配64个CQ_ITEM实例，然后返回第一个；之后再次调用这个函数时，则是直接从预先分配的CQ_ITEM链表中获取。

2、子线程的选择是采用轮询的方式，每次选择的线程总是上次选择线程的下一个线程。

## worker线程获取客户端连接
worker线程的回调在master里就指定好了，参考：thread.c::setup_thread()，worker的回调函数被指定为了thread_libevent_process，结合master的处理流程，这个worker的回调函数只有在master得到新连接时被触发
```c++
static void thread_libevent_process(int fd, short which, void *arg) {
    LIBEVENT_THREAD *me = arg;
    CQ_ITEM *item;
    char buf[1];
    conn *c;
    unsigned int timeout_fd;

    if (read(fd, buf, 1) != 1) {
        if (settings.verbose > 0)
            fprintf(stderr, "Can't read from libevent pipe\n");
        return;
    }

    switch (buf[0]) {
    case 'c':
        item = cq_pop(me->new_conn_queue);

        if (NULL == item) {
            break;
        }
        switch (item->mode) {
            case queue_new_conn:
                c = conn_new(item->sfd, item->init_state, item->event_flags,
                                   item->read_buffer_size, item->transport,
                                   me->base, item->ssl);
                if (c == NULL) {
                    if (IS_UDP(item->transport)) {
                        fprintf(stderr, "Can't listen for events on UDP socket\n");
                        exit(1);
                    } else {
                        if (settings.verbose > 0) {
                            fprintf(stderr, "Can't listen for events on fd %d\n",
                                item->sfd);
                        }
#ifdef TLS
                        if (item->ssl) {
                            SSL_shutdown(item->ssl);
                            SSL_free(item->ssl);
                        }
#endif
                        close(item->sfd);
                    }
                } else {
                    c->thread = me;
#ifdef TLS
                    if (settings.ssl_enabled && c->ssl != NULL) {
                        assert(c->thread && c->thread->ssl_wbuf);
                        c->ssl_wbuf = c->thread->ssl_wbuf;
                    }
#endif
                }
                break;

            case queue_redispatch:
                conn_worker_readd(item->c);
                break;
        }
        cqi_free(item);
        break;
    /* we were told to pause and report in */
    case 'p':
        register_thread_initialized();
        break;
    /* a client socket timed out */
    case 't':
        if (read(fd, &timeout_fd, sizeof(timeout_fd)) != sizeof(timeout_fd)) {
            if (settings.verbose > 0)
                fprintf(stderr, "Can't read timeout fd from libevent pipe\n");
            return;
        }
        conn_close_idle(conns[timeout_fd]);
        break;
    /* asked to stop */
    case 's':
        event_base_loopexit(me->base, NULL);
        break;
    }
}
```
worker调用cq_pop从队列中获取客户端连接后，然后就交给了conn_new函数进行处理
conn_new函数把客户端的读写事件交给了event_handler函数进行处理，核心代码如下
```c++
event_set(&c->event, sfd, event_flags, event_handler, (void *)c);
```
master也使用event_handler函数来处理master的事件，worker也使用event_handler处理事件，是不会冲突的，因为在event_handler里的drive_machine执行体中，会通过这个事件参数conn结构体的state属性的进行分支判断, worker的conn的state从CQ_ITEM获取，值为conn_new_cmd

drive_machine核心代码如下：
```c++
while (!stop) {
    case conn_new_cmd:
        /* Only process nreqs at a time to avoid starving other
           connections */

        --nreqs;
        if (nreqs >= 0) {
            reset_cmd_handler(c);
        } else {
            // 异常处理逻辑...
        }
        break;
}

```
把请求交给了reset_cmd_handler()函数进行处理,核心代码如下:
```c++
static void reset_cmd_handler(conn *c) {
    // ...
    if (c->rbytes > 0) {
        conn_set_state(c, conn_parse_cmd);
    } else {
        conn_set_state(c, conn_waiting);
    }
}

```
如果还有数据可以读取，就设置state为conn_parse_cmd状态，否则，进入conn_waiting状态

我们进入conn_parse_cmd状态分支（同样是在当前的循环内）：
```c++
case conn_parse_cmd :
    if (c->try_read_command(c) == 0) {
        /* wee need more data! */
        conn_set_state(c, conn_waiting);
    }

    break;

```
进行数据读取，然后将状态设置为conn_waiting, 这个try_read_command里面就对客户端的输入进行了处理，并把结果返回了客户端，参考: try_read_command_ascii
```c++
static int try_read_command_ascii(conn *c) {
    char *el, *cont;

    if (c->rbytes == 0)
        return 0;

    el = memchr(c->rcurr, '\n', c->rbytes);
    if (!el) {
        // 略过异常处理逻辑...
        return 0;
    }
    cont = el + 1;
    if ((el - c->rcurr) > 1 && *(el - 1) == '\r') {
        el--;
    }
    *el = '\0';

    assert(cont <= (c->rcurr + c->rbytes));

    c->last_cmd_time = current_time;
    process_command(c, c->rcurr);

    c->rbytes -= (cont - c->rcurr);
    c->rcurr = cont;

    assert(c->rcurr <= (c->rbuf + c->rsize));

    return 1;
}

```
process_command就是具体的命令处理了。

总结：主线程启动及分配请求流程：
server_sockets——>
server_socket——>
conn_new——>
event_handler——>
drive_machine——>
try_read_command（这里会判定，是文本协议还是二进制协议）