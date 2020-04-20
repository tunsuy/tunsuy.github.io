## PostgreSQL连接问题

### 查看当前连接数：
```sh
=> SELECT count(*) FROM pg_stat_activity;
=> select client_addr, count(*) from pg_stat_activity group by client_addr;
```
可以查看总的连接数，以及每个客户端的连接数，判断是否有客户端泄露连接。

### 查看连接状态
```sh
=> SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```

#### 连接有以下几种状态：

* idle: 连接对应的后端进程处于空闲状态
* actvie: 连接对应的后端进程正在执行查询
* idle in transaction: 连接对应的后端进程在一个事务中，但当前并没有执行查询
* idle in transaction (aborted): 在一个事务中，但是事务中的语句出现了错误，事务没有正确回滚
* fastpath function call: 正在执行fastpath函数调用
* disabled: 后端进程被配置为禁止跟踪

注：
9.6版本 PostgreSQL 支持自动查杀超过指定时间的 `idle in transaction` 空闲事务连接，下面演示下:
1、修改 postgresql.conf 以下参数
```sh
idle_in_transaction_session_timeout = 20000
```
备注：参数单位为毫秒，这里设置 `idle in transaction` 超时空闲事务时间为 20 秒。

2、重载配置文件
```sh
[pg96@db1 pg_root]$ pg_ctl reload
server signaled
```
备注：此参数修改后对当前连接依然生效，应用不需要重连即能生效。

### 等待锁的连接
```sh
=> SELECT count(distinct pid) FROM pg_locks WHERE granted = false;
```
查看是否有连接在等待排它锁导致效率低下

### 查看事务最大执行时间
```sh
=> SELECT max(now() -xact_start) FROM pg_stat_activity WHERE state IN ('idle in transaction','active'); 
```
查看当前所有事务中最长事务的执行时间，一般来讲事务应该在数秒内结束，数分钟已经是很长的事务了。如果有事务的执行时间达到了小时的级别，那一定是出现了错误，但是事务还在占用系统资源，应该将其干掉。

### 查看事务执行时间
```sh
=> SELECT pid, xact_start FROM pg_stat_activity ORDER BY xact_start ASC;
```
以事务的起始时间为顺序，如果某事务的运行时间过长，比如达到了小时级别，应该查清原因，然后搞掉它。

### 杀掉后端进程
```sh
=> SELECT pg_cancel_backend(1234);
```
1234就是pg_stat_activity表里的pid,也是postgres进程的系统进程pid，不要用操作系统命令去搞死进程，要用postgresql提供的命令。


### PostgreSQL 缓存
除了常见的执行计划缓存、数据缓存，PostgreSQL为了提高生成执行计划的效率，还提供了catalog, relation等缓存机制。

PostgreSQL 9.5支持的缓存代码如下
```sh
ll src/backend/utils/cache/  
  
attoptcache.c  catcache.c  evtcache.c  inval.c  lsyscache.c  plancache.c  relcache.c  relfilenodemap.c  relmapper.c  spccache.c  syscache.c  ts_cache.c  typcache.c  
```

#### 长连接的缓存问题
这些缓存中，某些缓存是不会主动释放的，因此可能导致长连接霸占大量的内存不释放。

通常，长连接的应用，一个连接可能给多个客户端会话使用过，访问到大量catalog的可能性非常大。所以此类的内存占用比是非常高的。

有什么影响呢？

如果长连接很多，而且每个都霸占大量的内存，你的内存很快会被大量的连接耗光，出现OOM是避免不了的。

而实际上，这些内存可能大部分都是relcache的（还有一些其他的），要用到内存时，这些relcache完全可以释放出来，腾出内存空间，而没有必要被持久霸占。

在数据库中存在大量的表，PostgreSQL会缓存当前会话访问过的对象的元数据，如果某个会话从启动以来，对数据库中所有的对象都有过查询的动作，那么这个会话需要将所有的对象定义都缓存起来，会占用较大的内存，占用的内存大小与一共访问了多少站该对象有关。

https://www.postgresql.org/docs/9.6/monitoring-stats.html