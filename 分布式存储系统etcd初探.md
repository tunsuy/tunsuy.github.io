# 分布式存储系统etcd初探
## etcd是什么
简单来说，etcd是一个高可用，强一致性的分布式kv存储数据库。由此可以衍生出很多其他功能需求，比如：
* 服务注册
* 服务发现
* 配置管理
* 分布式锁
* 。。。

下面就简单来操作操作

## 集群搭建
### 1、目录结构
```sh
[root@karbor1 etcd]# ll
total 29044
drwxr-x---  2 root root     4096 Oct 19 19:52 conf
drwxr-x---  3 root root     4096 Oct 19 20:02 data
drwxrwxr-x 11 root root     4096 Jun 10  2017 Documentation
-rwxrwxr-x  1 root root 15764288 Jun 10  2017 etcd
-rwxrwxr-x  1 root root 13909728 Jun 10  2017 etcdctl
drwxr-x---  2 root root     4096 Oct 19 20:02 log
-rw-rw-r--  1 root root    32632 Jun 10  2017 README-etcdctl.md
-rw-rw-r--  1 root root     5878 Jun 10  2017 README.md
-rw-rw-r--  1 root root     7892 Jun 10  2017 READMEv2-etcdctl.md
```
当然，这里只是演示，生产环境需要严格按照规范来规划

### 2、增加配置文件
```sh
[root@karbor1 etcd]# cat conf/etcd.conf 
name: etcd-1
data-dir: /opt/etcd/data
listen-client-urls: http://30.1.3.42:2379
advertise-client-urls: http://30.1.3.42:2379
listen-peer-urls: http://30.1.3.42:2380
initial-advertise-peer-urls: http://30.1.3.42:2380
initial-cluster: etcd-1=http://30.1.3.42:2380,etcd-2=http://30.1.3.43:2380
initial-cluster-token: etcd-cluster-token
initial-cluster-state: new
```
具体的参数含义，在后面一系列的etcd原理剖析文章中再做详细讲解

### 3、启动服务
```sh
nohup etcd --config-file=/opt/etcd/conf/etcd.conf > /opt/etcd/log/etcd.log 2>&1 &
```
这样就后台启动了一个服务，当然，生产环境一般不这样做，生产环境一般使用systemctl系统来做，需要定义xxx.service等服务文件，里面包含服务的启停操作等等。

在另外一个节点上也执行如上步骤

### 4、查看集群
```sh
[root@karbor2 etcd]# etcdctl member list
Error:  client: etcd cluster is unavailable or misconfigured; error #0: dial tcp 127.0.0.1:2379: getsockopt: connection refused
; error #1: dial tcp 127.0.0.1:4001: getsockopt: connection refused

error #0: dial tcp 127.0.0.1:2379: getsockopt: connection refused
error #1: dial tcp 127.0.0.1:4001: getsockopt: connection refused
```
这是因为我们在配置文件的advertise-client-urls字段中，没有添加回环地址，那么我们就指定访问，如下所示：
```sh
[root@karbor2 etcd]# etcdctl --endpoints "http://30.1.3.43:2379" member list
6f840018249f939b: name=etcd-2 peerURLs=http://30.1.3.43:2380 clientURLs=http://30.1.3.43:2379 isLeader=false
e8e5aa3e0acdf16b: name=etcd-1 peerURLs=http://30.1.3.42:2380 clientURLs=http://30.1.3.42:2379 isLeader=true
```
可以看到，两个节点中，一个节点是主节点，一个是备节点。

## KV存储
### 1、写入数据
```sh
[root@karbor2 etcd]#  etcdctl --endpoints "http://30.1.3.42:2379" set /ts/test/name 'ts1'
ts1
```

### 2、读取数据
```sh
[root@karbor2 etcd]#  etcdctl --endpoints "http://30.1.3.43:2379" get /ts/test/name
ts1
```
可以看到，即使访问另外一个节点，也是能够获取到数据的。

其他命令就暂时不演示了

## 集群验证
### 1、异常构造

因为etcd也是一个强一致性系统，采用的raft协议，部署节点数一般是奇数个，即3个挂1个可用，5个挂2个可用。
我们将其中一个节点停掉，查看另外一个节点，日志如下：

```sh
2020-10-19 20:37:26.303167 W | rafthttp: lost the TCP streaming connection with peer 6f840018249f939b (stream MsgApp v2 reader)
2020-10-19 20:37:26.303248 E | rafthttp: failed to read 6f840018249f939b on stream MsgApp v2 (unexpected EOF)
2020-10-19 20:37:26.303261 I | rafthttp: peer 6f840018249f939b became inactive
2020-10-19 20:37:26.303279 W | rafthttp: lost the TCP streaming connection with peer 6f840018249f939b (stream Message reader)
2020-10-19 20:37:26.537149 W | rafthttp: lost the TCP streaming connection with peer 6f840018249f939b (stream Message writer)
2020-10-19 20:37:27.342378 W | raft: e8e5aa3e0acdf16b stepped down to follower since quorum is not active
2020-10-19 20:37:27.342457 I | raft: e8e5aa3e0acdf16b became follower at term 284
2020-10-19 20:37:27.342473 I | raft: raft.node: e8e5aa3e0acdf16b lost leader e8e5aa3e0acdf16b at term 284
2020-10-19 20:37:28.435635 I | raft: e8e5aa3e0acdf16b is starting a new election at term 284
2020-10-19 20:37:28.435727 I | raft: e8e5aa3e0acdf16b became candidate at term 285
2020-10-19 20:37:28.435747 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 285
2020-10-19 20:37:28.435765 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 285
2020-10-19 20:37:29.535703 I | raft: e8e5aa3e0acdf16b is starting a new election at term 285
2020-10-19 20:37:29.535789 I | raft: e8e5aa3e0acdf16b became candidate at term 286
2020-10-19 20:37:29.535812 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 286
2020-10-19 20:37:29.535831 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 286
2020-10-19 20:37:31.315784 W | rafthttp: lost the TCP streaming connection with peer 6f840018249f939b (stream MsgApp v2 writer)
2020-10-19 20:37:31.435665 I | raft: e8e5aa3e0acdf16b is starting a new election at term 286
2020-10-19 20:37:31.435725 I | raft: e8e5aa3e0acdf16b became candidate at term 287
2020-10-19 20:37:31.435744 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 287
2020-10-19 20:37:31.435763 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 287
2020-10-19 20:37:32.835767 I | raft: e8e5aa3e0acdf16b is starting a new election at term 287
2020-10-19 20:37:32.835831 I | raft: e8e5aa3e0acdf16b became candidate at term 288
2020-10-19 20:37:32.835850 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 288
2020-10-19 20:37:32.835867 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 288
2020-10-19 20:37:33.935675 I | raft: e8e5aa3e0acdf16b is starting a new election at term 288
2020-10-19 20:37:33.935734 I | raft: e8e5aa3e0acdf16b became candidate at term 289
2020-10-19 20:37:33.935749 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 289
2020-10-19 20:37:33.935762 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 289
2020-10-19 20:37:35.535656 I | raft: e8e5aa3e0acdf16b is starting a new election at term 289
2020-10-19 20:37:35.535714 I | raft: e8e5aa3e0acdf16b became candidate at term 290
2020-10-19 20:37:35.535728 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 290
2020-10-19 20:37:35.535739 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 290
2020-10-19 20:37:37.339813 I | raft: e8e5aa3e0acdf16b is starting a new election at term 290
2020-10-19 20:37:37.339897 I | raft: e8e5aa3e0acdf16b became candidate at term 291
2020-10-19 20:37:37.339918 I | raft: e8e5aa3e0acdf16b received MsgVoteResp from e8e5aa3e0acdf16b at term 291
2020-10-19 20:37:37.339936 I | raft: e8e5aa3e0acdf16b [logterm: 284, index: 12] sent MsgVote request to 6f840018249f939b at term 291
2020-10-19 20:37:38.435656 I | raft: e8e5aa3e0acdf16b is starting a new election at term 291
2020-10-19 20:37:38.435719 I | raft: e8e5aa3e0acdf16b became candidate at term 292

```
可以看到，该节点一直在处于选举状态

### 2、读取kv数据
```sh
[root@karbor2 etcd]#  etcdctl --endpoints "http://30.1.3.42:2379" get /ts/test/name
ts
```
可以看到，虽然集群状态异常了，但是还是可以从可用的节点上读取数据

### 3、写入kv数据：
```sh
[root@karbor2 etcd]#  etcdctl --endpoints "http://30.1.3.42:2379" set /ts/test/name 'ts1'
Error:  client: etcd cluster is unavailable or misconfigured; error #0: client: endpoint http://30.1.3.42:2379 exceeded header timeout
; error #1: dial tcp 30.1.3.43:2379: getsockopt: connection refused

error #0: client: endpoint http://30.1.3.42:2379 exceeded header timeout
error #1: dial tcp 30.1.3.43:2379: getsockopt: connection refused


```
可以看到，即使有一个节点可用，但是也不能写入了。另外，集群相关的其他命令也不可用，比如
```sh
[root@karbor2 etcd]# etcdctl --endpoints "http://30.1.3.42:2379" member list
Failed to get leader:  client: etcd cluster is unavailable or misconfigured; error #0: client: etcd member http://30.1.3.42:2379 has no leader
; error #1: dial tcp 30.1.3.43:2379: getsockopt: connection refused

```

其他的功能验证就需要结合代码来验收了，这里暂时先不做了，后续有时间再说，比如服务注册，服务发现，分布式锁等等