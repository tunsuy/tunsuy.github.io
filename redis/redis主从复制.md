## redis主从复制



## 前言

一个服务器节点可以部署多个redis实例，每个实例都有独立的配置文件

例如：如下配置
```sh
include /opt/redis/data/tstest-9-99/redis_commom.conf
port 29999
dir /opt/redis/data/
bind 30.1.3.29
dbfilename tstest-9-99.rdb
pidfile /opt/redis/data/tstest-9-99/tstest-9-99.pid
logfile /opt/redis/data/tstest-9-99/tstest-9-99.log
requiredbname tstest
requirepass "000000033RcdAFS2Gg1Eujn7UX8lPOWhjQWfwX78QZJEWePQlcc="
requirereaduserpass "00000003SGKbkilo9O0NU9TwFau6/yYt9fOnGpqPawl3RFeyNq8="
requireuserpass "00000003tGnRBFAHm7y3zCEcsjkOfz/RuAhBUkLiemM6ezGTycg="
requiresalt 8cMpo+VxGo+hb+xF5RDJGw==
cipher-dir  /opt/redis/etc/cipher
syslog-ident tstest-9-99
requirereadusersalt M7IsVvb2boKD6PSBYw5I/g==
requireusersalt OB6HNaPOr/tOL+IsxABRTQ==
maxmemory 4096mb
include /opt/redis/data/tstest-9-99/redis_paramgroup.conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command EVAL ""
rename-command KEYS ""
```

进入某个redis实例数据库：
```sh
redis-cli -h 30.1.3.29 -p 29999 -cipherdir /opt/redis/etc/cipher -a tstest@dbuser@Admin@123
```

## 主从复制
主从复制的配置还是比较简单的，下面来了解下主从复制的实现原理

Redis的主从复制过程大体上分3个阶段：**建立连接**、**数据同步**、**命令传播**

#### 建立连接

这个阶段主要是从服务器发出`slaveof`命令之后，与主服务器如何建立连接，为数据同步做准备的过程。

1）在`slaveof`命令执行之后，从服务器根据设置的master的ip地址和端口，创建连向主服务器的socket套接字连接，连接成功后，从服务器会为这个套接字关联一个专门的处理器，用于处理后续的复制工作

2）建立连接之后，从服务器会向主服务器发送`ping`命令，确认主服务器是否可用，以及当前是否可用接受处理命令。如果收到主服务器的`pong`回复说明是可用的，否则有可能是网络超时或主服务器阻塞，从服务器会断开连接发起重连

3）身份验证。如果主服务器设置了`requirepass`选项，那么从服务器必须配置`masterauth`选项，且保证密码一致才能通过验证

4）身份验证完成之后，从服务器会发送自己的监听端口，主服务器会保存下来

```sh
12376:C 08 Apr 10:26:05.952 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
12376:C 08 Apr 10:26:05.953 # Redis version=5.0.6, bits=64, commit=00000000, modified=0, pid=12376, just started
12376:C 08 Apr 10:26:05.953 # Configuration loaded
12377:M 08 Apr 10:26:05.956 * Running mode=standalone, port=26661.
12377:M 08 Apr 10:26:05.956 # Server initialized
12377:M 08 Apr 10:26:05.956 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
12377:M 08 Apr 10:26:05.956 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
12377:M 08 Apr 10:26:05.957 * Ready to accept connections
12377:S 08 Apr 10:26:06.012 * Before turning into a replica, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
12377:S 08 Apr 10:26:06.012 * REPLICAOF 30.1.3.29:26661 enabled (user request from 'id=3 addr=30.1.3.30:58031 fd=7 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=43 qbuf-free=32725 obl=0 oll=0 omem=0 events=r cmd=slaveof')
12377:S 08 Apr 10:26:06.959 * Connecting to MASTER 30.1.3.29:26661
12377:S 08 Apr 10:26:06.959 * MASTER <-> REPLICA sync started
12377:S 08 Apr 10:26:06.959 * Non blocking connect for SYNC fired the event.
12377:S 08 Apr 10:26:06.959 * Master replied to PING, replication can continue...
```

#### 数据同步

在主从服务器建立连接确认各自身份之后，就开始数据同步，从服务器向主服务器发送`PSYNC`命令，执行同步操作，并把自己的数据库状态更新至主服务器的数据库状态

Redis的主从同步分为：**完整重同步（full resynchronization）和部分重同步（partial resynchronization）**

##### 完整重同步

有两种情况下是完整重同步，一是slave连接上master第一次复制的时候；二是如果当主从断线，重新连接复制的时候有可能是完整重同步

- 从服务器连接主服务器，发送SYNC命令

- 主服务器接收到SYNC命名后，开始执行`bgsave`命令生成RDB文件并使用缓冲区记录此后执行的所有写命令

- 主服务器`bgsave`执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令

- 从服务器收到快照文件后丢弃所有旧数据，载入收到的快照

- 主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令

- 从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令

```sh
12377:S 08 Apr 10:26:06.987 * Full resync from master: 8b7764acf7438d3c02efeaf6c4d76a00afac67a3:0
12377:S 08 Apr 10:26:06.987 * Discarding previously cached master state.
12377:S 08 Apr 10:26:07.025 * MASTER <-> REPLICA sync: receiving 175 bytes from master
12377:S 08 Apr 10:26:07.025 * MASTER <-> REPLICA sync: Flushing old data
12377:S 08 Apr 10:26:07.025 * MASTER <-> REPLICA sync: Loading DB in memory
12377:S 08 Apr 10:26:07.025 * MASTER <-> REPLICA sync: Finished with success
```

##### 部分重同步

部分重同步是用于处理断线后重复制的情况，先介绍几个用于部分重同步的部分

- `runid`(replication ID)，主服务器运行id，Redis实例在启动时，随机生成一个长度40的唯一字符串来标识当前节点
- `offset`，复制偏移量。主服务器和从服务器各自维护一个复制偏移量，记录传输的字节数。当主节点向从节点发送N个字节数据时，主节点的offset增加N，从节点收到主节点传来的N个字节数据时，从节点的offset增加N
- `replication backlog buffer`，复制积压缓冲区。是一个固定长度的FIFO队列，大小由配置参数`repl-backlog-size`指定，默认大小1MB。需要注意的是该缓冲区由master维护并且有且只有一个，所有slave共享此缓冲区，其作用在于备份最近主库发送给从库的数据

当slave连接到master，会执行`PSYNC <runid> <offset>`发送记录旧的master的`runid`（replication ID）和偏移量`offset`，这样master能够只发送slave所缺的增量部分。但是如果master的复制积压缓存区没有足够的命令记录，或者slave传的`runid`(replication ID)不对，就会进行**完整重同步**，即slave会获得一个完整的数据集副本

```sh
12377:S 07 May 00:17:19.027 # MASTER timeout: no data nor PING received...
12377:S 07 May 00:17:19.027 # Connection with master lost.
12377:S 07 May 00:17:19.027 * Caching the disconnected master state.
12377:S 07 May 00:17:19.027 * Connecting to MASTER 30.1.3.29:26661
12377:S 07 May 00:17:19.027 * MASTER <-> REPLICA sync started
12377:S 07 May 00:17:19.036 * Non blocking connect for SYNC fired the event.
12377:S 07 May 00:17:19.046 * Master replied to PING, replication can continue...
12377:S 07 May 00:17:19.056 * Trying a partial resynchronization (request 8b7764acf7438d3c02efeaf6c4d76a00afac67a3:337474002).
12377:S 07 May 00:17:19.056 * Successful partial resynchronization with master.
12377:S 07 May 00:17:19.056 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.
```

#### 实操
在主服务器redis控制台执行：
```sh
30.1.3.29:26661> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=30.1.3.30,port=26661,state=online,offset=349732013,lag=1
master_replid:8b7764acf7438d3c02efeaf6c4d76a00afac67a3
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:349732013
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:348683438
repl_backlog_histlen:1048576
```

在从服务器redis控制台执行：
```sh
30.1.3.30:26661> info replication
# Replication
role:slave
master_host:30.1.3.29
master_port:26661
master_link_status:up
master_last_io_seconds_ago:4
master_sync_in_progress:0
slave_repl_offset:349734529
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:8b7764acf7438d3c02efeaf6c4d76a00afac67a3
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:349734529
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:348685954
repl_backlog_histlen:1048576
```

在主服务器redis控制台执行：
```sh
30.1.3.29:26661> set ts test
OK
30.1.3.29:26661> get ts
"test"
```

在从服务器redis控制台执行：
```sh
30.1.3.30:26661> get ts
"test"
```

然后我们试着在从服务器中写入数据，会提示不能在只读的从服务器中写入数据
```sh
30.1.3.30:26661> set tstest haha
(error) READONLY You can't write against a read only replica.
```

### 主从复制面临的问题

当主节点发生故障的时候，需要手动的将一个从节点晋升为主节点，同时通知应用方修改主节点地址并重启应用，同时需要命令其它从节点复制新的主节点，整个过程需要人工干预。
主节点的写能力受到单机的限制。
主节点的存储能力受到单机的限制。

[深入学习Redis（3）：主从复制](https://www.cnblogs.com/kismetv/p/9236731.html)