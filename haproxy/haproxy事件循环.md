## haproxy事件循环

在haproxy启动的时候，main方法会在socket建立连接之后调用run_poll_loop方法进行事件循环处理：
```c++
static void run_poll_loop()
{
	int next, wake;

	tv_update_date(0,1);
	while (1) {
		wake_expired_tasks();

		/* Process a few tasks */
		process_runnable_tasks();

		/* check if we caught some signals and process them in the
		 first thread */
		if (tid == 0)
			signal_process_queue();

		/* stop when there's nothing left to do */
		if ((jobs - unstoppable_jobs) == 0)
			break;

		/* also stop  if we failed to cleanly stop all tasks */
		if (killed > 1)
			break;

		/* expire immediately if events are pending */
		wake = 1;
		if (thread_has_tasks())
			activity[tid].wake_tasks++;
		else if (signal_queue_len && tid == 0)
			activity[tid].wake_signal++;
		else {
			_HA_ATOMIC_OR(&sleeping_thread_mask, tid_bit);
			__ha_barrier_atomic_store();
			if ((global_tasks_mask & tid_bit) || thread_has_tasks()) {
				activity[tid].wake_tasks++;
				_HA_ATOMIC_AND(&sleeping_thread_mask, ~tid_bit);
			} else
				wake = 0;
		}

		/* If we have to sleep, measure how long */
		next = wake ? TICK_ETERNITY : next_timer_expiry();

		/* The poller will ensure it returns around <next> */
		cur_poller.poll(&cur_poller, next, wake);

		activity[tid].loops++;
	}
}
```

### 处理信号
haproxy 封装了自己的信号处理机制。接受到信号之后，将该信号放到信号队列中。signal_register_fct，signal_register_task接口提供了注册函数回调和任务类型回调两种方式。
在程序运行到signal_process_queue() 时处理所有位于信号队列中的信号。
```c++
void __signal_process_queue()
{
	int sig, cur_pos = 0;
	struct signal_descriptor *desc;
	sigset_t old_sig;

	/* block signal delivery during processing */
	ha_sigmask(SIG_SETMASK, &blocked_sig, &old_sig);

	/* It is important that we scan the queue forwards so that we can
	 * catch any signal that would have been queued by another signal
	 * handler. That allows real signal handlers to redistribute signals
	 * to tasks subscribed to signal zero.
	 */
	for (cur_pos = 0; cur_pos < signal_queue_len; cur_pos++) {
		sig  = signal_queue[cur_pos];
		desc = &signal_state[sig];
		if (desc->count) {
			struct sig_handler *sh, *shb;
			list_for_each_entry_safe(sh, shb, &desc->handlers, list) {
				if ((sh->flags & SIG_F_TYPE_FCT) && sh->handler)
					((void (*)(struct sig_handler *))sh->handler)(sh);
				else if ((sh->flags & SIG_F_TYPE_TASK) && sh->handler)
					task_wakeup(sh->handler, TASK_WOKEN_SIGNAL);
			}
			desc->count = 0;
		}
	}
	signal_queue_len = 0;

	/* restore signal delivery */
	ha_sigmask(SIG_SETMASK, &old_sig, NULL);
}
```
信号注册时注册SIG_F_TYPE_FCT标识则直接调用信号回调处理；SIG_F_TYPE_TASK标识说明注册时回调函数是一个task指针，这时需要唤醒task，并指明任务状态为TASK_WOKEN_SIGNAL，此后对应处理函数将在task管理下处理。下面来看看task管理。

###  唤醒超时任务
haproxy 的顶层处理逻辑是 task，task 上存储着要处理的任务的全部信息。task 的管理是采用ebtree树形队列方式，同时分为 wait queue 和 run queue。顾名思义，wait queue 是需要等 待一定时间的 task 的集合，而 run queue 则代表需要立即执行的 task 的集合。

该函数就是检查 wait queue 中那些超时的任务，并将其放到 run queue 中。haproxy 在 执行的过程中，会因为一些情况导致需要将当前的任务通过调用 task_queue 等接口放到 wait queue 中。
```c++
while (1) {
lookup_next_local:
	eb = eb32_lookup_ge(&tt->timers, now_ms - TIMER_LOOK_BACK);
	if (!eb) {
		/* we might have reached the end of the tree, typically because
		* <now_ms> is in the first half and we're first scanning the last
		* half. Let's loop back to the beginning of the tree now.
		*/
		eb = eb32_first(&tt->timers);
		if (likely(!eb))
			break;
	}

	if (tick_is_lt(now_ms, eb->key))
		break;

	/* timer looks expired, detach it from the queue */
	task = eb32_entry(eb, struct task, wq);
	__task_unlink_wq(task);

	/* It is possible that this task was left at an earlier place in the
	 * tree because a recent call to task_queue() has not moved it. This
	 * happens when the new expiration date is later than the old one.
	 * Since it is very unlikely that we reach a timeout anyway, it's a
	 * lot cheaper to proceed like this because we almost never update
	 * the tree. We may also find disabled expiration dates there. Since
	 * we have detached the task from the tree, we simply call task_queue
	 * to take care of this. Note that we might occasionally requeue it at
	 * the same place, before <eb>, so we have to check if this happens,
	 * and adjust <eb>, otherwise we may skip it which is not what we want.
	 * We may also not requeue the task (and not point eb at it) if its
	 * expiration time is not set.
	 */
	if (!tick_is_expired(task->expire, now_ms)) {
		if (tick_isset(task->expire))
			__task_queue(task, &tt->timers);
		goto lookup_next_local;
	}
	task_wakeup(task, TASK_WOKEN_TIMER);
}
```

### 处理可运行的任务
处理位于 run queue 中的任务。

前面提到，wake_expired_tasks 可能将一些超时的任务放到 run queue 中。此外，haproxy 执行的过程中，还有可能通过调用 task_wakeup 直接讲某个 task 放到 run queue 中，这代表程序希望该任务下次尽可能快的被执行。

对于TCP或者HTTP业务流量的处理，该函数最终通过调用 process_session 来完成，包括解析已经接收到的数据， 并执行一系列 load balance 的特性，但不负责从 socket 收发数据，数据收发由poll完成。同时，也会因为一些情况导致需要将当前的任务通过调用 task_queue 等接口放到 wait queue 中，实现上在任务回调处理时返回非空任务则会把任务重新加入wait queue。

haproxy 中用 jobs 记录当前要处理的任务总数， 如果 jobs 为 0 的话，通常意味着 haproxy 要退出了，因为连 listener 都要释放了。jobs 的数值通常在 process_session 时更新。

### poll消息驱动
haproxy 启动阶段，会检测当前系统可以启用那种异步处理的机制，比如 select、poll、 epoll、kqueue 等，并注册对应 poller 的 poll 方法。epoll 的相关函数接口在 ev_epoll.c 中。

这里就是执行已经注册的 poller 的 poll 方法，主要功能就是获取所有活动的 fd，并 调用对应的 handler，完成接受新建连接、数据收发等功能。

poller的poll方法执行时，程序会将某些符合条件以便再次执行 IO 处理的的fd放到 fd_cache中，之后fd_process_cached_events () 函数会再次执行这些fd的io handler。

可以大家有个疑问，这个poll方法到底是执行的哪个？下面我们一一道来

我们注意到工程中有这样的问题ev_xx.c，这里以ev_epoll.c为例，有这样一段代码
```c++
__attribute__((constructor))
static void _do_register(void)
{
	struct poller *p;
	int i;

	if (nbpollers >= MAX_POLLERS)
		return;

	for (i = 0; i < MAX_THREADS; i++)
		epoll_fd[i] = -1;

	p = &pollers[nbpollers++];

	p->name = "epoll";
	p->pref = 300;
	p->flags = HAP_POLL_F_ERRHUP; // note: RDHUP might be dynamically added
	p->private = NULL;

	p->clo  = __fd_clo;
	p->test = _do_test;
	p->init = _do_init;
	p->term = _do_term;
	p->poll = _do_poll;
	p->fork = _do_fork;
}
```
该方法会在main方法之前被自动执行，这里我们可以看到`p->poll = _do_poll;`
那么上面的poll对应的就是这个_do_poll。

下面进入该方法，看下具体是怎么执行的：
```c++
/*
 * Linux epoll() poller
 */
REGPRM3 static void _do_poll(struct poller *p, int exp, int wake)
{
	int status;
	int fd;
	int count;
	int updt_idx;
	int wait_time;
	int old_fd;

	/* first, scan the update list to find polling changes */
	for (updt_idx = 0; updt_idx < fd_nbupdt; updt_idx++) {
		fd = fd_updt[updt_idx];

		_HA_ATOMIC_AND(&fdtab[fd].update_mask, ~tid_bit);
		if (!fdtab[fd].owner) {
			activity[tid].poll_drop++;
			continue;
		}

		_update_fd(fd);
	}
	fd_nbupdt = 0;
	/* Scan the global update list */
	for (old_fd = fd = update_list.first; fd != -1; fd = fdtab[fd].update.next) {
		if (fd == -2) {
			fd = old_fd;
			continue;
		}
		else if (fd <= -3)
			fd = -fd -4;
		if (fd == -1)
			break;
		if (fdtab[fd].update_mask & tid_bit)
			done_update_polling(fd);
		else
			continue;
		if (!fdtab[fd].owner)
			continue;
		_update_fd(fd);
	}

	thread_harmless_now();

	/* now let's wait for polled events */
	wait_time = wake ? 0 : compute_poll_timeout(exp);
	tv_entering_poll();
	activity_count_runtime();
	do {
		int timeout = (global.tune.options & GTUNE_BUSY_POLLING) ? 0 : wait_time;

		status = epoll_wait(epoll_fd[tid], epoll_events, global.tune.maxpollevents, timeout);
		tv_update_date(timeout, status);

		if (status)
			break;
		if (timeout || !wait_time)
			break;
		if (signal_queue_len || wake)
			break;
		if (tick_isset(exp) && tick_is_expired(exp, now_ms))
			break;
	} while (1);

	tv_leaving_poll(wait_time, status);

	thread_harmless_end();
	if (sleeping_thread_mask & tid_bit)
		_HA_ATOMIC_AND(&sleeping_thread_mask, ~tid_bit);

	/* process polled events */

	for (count = 0; count < status; count++) {
		unsigned int n;
		unsigned int e = epoll_events[count].events;
		fd = epoll_events[count].data.fd;

		if (!fdtab[fd].owner) {
			activity[tid].poll_dead++;
			continue;
		}

		if (!(fdtab[fd].thread_mask & tid_bit)) {
			/* FD has been migrated */
			activity[tid].poll_skip++;
			epoll_ctl(epoll_fd[tid], EPOLL_CTL_DEL, fd, &ev);
			_HA_ATOMIC_AND(&polled_mask[fd].poll_recv, ~tid_bit);
			_HA_ATOMIC_AND(&polled_mask[fd].poll_send, ~tid_bit);
			continue;
		}

		n = ((e & EPOLLIN)    ? FD_EV_READY_R : 0) |
		    ((e & EPOLLOUT)   ? FD_EV_READY_W : 0) |
		    ((e & EPOLLRDHUP) ? FD_EV_SHUT_R  : 0) |
		    ((e & EPOLLHUP)   ? FD_EV_SHUT_RW : 0) |
		    ((e & EPOLLERR)   ? FD_EV_ERR_RW  : 0);

		if ((e & EPOLLRDHUP) && !(cur_poller.flags & HAP_POLL_F_RDHUP))
			_HA_ATOMIC_OR(&cur_poller.flags, HAP_POLL_F_RDHUP);

		fd_update_events(fd, n);
	}
	/* the caller will take care of cached events */
}
```
该函数可以粗略分为三部分：
* 检查 fd 更新列表，获取各个 fd event 的变化情况，并作 epoll 的设置
* 计算 epoll_wait 的 delay 时间，并调用 epoll_wait，获取活动的 fd
* 逐一处理所有有 IO 事件的 fd