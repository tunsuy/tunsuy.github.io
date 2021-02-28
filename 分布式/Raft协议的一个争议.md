# Raft协议的一个争议

### 问题的由来
##### 注：下面的raft实现库是：Hashicorp/raft

当我使用3个节点部署了一个consul集群的时候，在将主节点reboot之后，发现consul的集群状态异常了，异常信息如下：
```sh
2021-01-07T00:49:30.989+0800 [INFO]  agent.server.raft: initial configuration: index=5041 servers="[{Suffrage:Voter ID:5d3fec83-6111-d765-4fff-d28c7c194dc6 Address:51.6.196.225:8801} {Suffrage:Voter ID:05aff258-9064-91ad-f8fc-3f79c7a2fc26 Address:51.6.196.216:8801} {Suffrage:Voter ID:2852d4e7-9eac-186f-bf15-8ce8c89ba0e5 Address:51.6.196.235:8801}]"
2021-01-07T00:49:30.990+0800 [INFO]  agent.server.raft: entering follower state: follower="Node at 51.6.196.225:8801 [Follower]" leader=
2021-01-07T00:49:30.991+0800 [WARN]  agent.server.memberlist.wan: memberlist: Binding to public address without encryption!
2021-01-07T00:49:30.992+0800 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: rd1.hn 51.6.196.225
2021-01-07T00:49:30.992+0800 [INFO]  agent.server.serf.wan: serf: Attempting re-join to previously known node: rd2.hn: 51.6.196.235:8302
2021-01-07T00:49:30.992+0800 [WARN]  agent.server.memberlist.lan: memberlist: Binding to public address without encryption!
2021-01-07T00:49:30.992+0800 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: rd1 51.6.196.225
2021-01-07T00:49:30.993+0800 [INFO]  agent.server: Handled event for server in area: event=member-join server=rd1.hn area=wan
2021-01-07T00:49:30.994+0800 [INFO]  agent.server.serf.lan: serf: Attempting re-join to previously known node: rd3: 51.6.196.216:8802
2021-01-07T00:49:30.994+0800 [INFO]  agent.server: Adding LAN server: server="rd1 (Addr: tcp/51.6.196.225:8801) (DC: hn)"
2021-01-07T00:49:30.994+0800 [INFO]  agent.server: Raft data found, disabling bootstrap mode
2021-01-07T00:49:30.994+0800 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: rd3.hn 51.6.196.216
2021-01-07T00:49:30.994+0800 [INFO]  agent.server: Handled event for server in area: event=member-join server=rd3.hn area=wan
2021-01-07T00:49:30.995+0800 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: rd2.hn 51.6.196.235
2021-01-07T00:49:30.995+0800 [WARN]  agent.server.memberlist.wan: memberlist: Refuting an alive message for 'rd1.hn' (51.6.196.225:8302) meta:([255 140 162 100 99 162 104 110 167 118 115 110 95 109 97 120 161 51 165 98 117 105 108 100 174 49 46 55 46 52 58 100 49 52 57 100 55 101 57 164 112 111 114 116 164 56 56 48 49 166 101 120 112 101 99 116 161 51 164 97 99 108 115 161 48 164 114 111 108 101 166 99 111 110 115 117 108 167 115 101 103 109 101 110 116 160 162 105 100 218 0 36 53 100 51 102 101 99 56 51 45 54 49 49 49 45 100 55 54 53 45 52 102 102 102 45 100 50 56 99 55 99 49 57 52 100 99 54 163 118 115 110 161 50 167 118 115 110 95 109 105 110 161 50 168 114 97 102 116 95 118 115 110 161 51] VS [255 140 164 97 99 108 115 161 48 162 100 99 162 104 110 167 115 101 103 109 101 110 116 160 167 118 115 110 95 109 105 110 161 50 167 118 115 110 95 109 97 120 161 51 168 114 97 102 116 95 118 115 110 161 51 164 112 111 114 116 164 56 56 48 49 166 101 120 112 101 99 116 161 51 164 114 111 108 101 166 99 111 110 115 117 108 162 105 100 218 0 36 53 100 51 102 101 99 56 51 45 54 49 49 49 45 100 55 54 53 45 52 102 102 102 45 100 50 56 99 55 99 49 57 52 100 99 54 163 118 115 110 161 50 165 98 117 105 108 100 174 49 46 55 46 52 58 100 49 52 57 100 55 101 57]), vsn:([1 5 2 2 5 4] VS [1 5 2 2 5 4])
2021-01-07T00:49:30.995+0800 [INFO]  agent.server.serf.wan: serf: Re-joined to previously known node: rd2.hn: 51.6.196.235:8302
2021-01-07T00:49:30.995+0800 [INFO]  agent.server: Handled event for server in area: event=member-join server=rd2.hn area=wan
2021-01-07T00:49:30.995+0800 [WARN]  agent: Service name will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.: service=keepalived_51.6.196.225
2021-01-07T00:49:30.995+0800 [WARN]  agent.server.memberlist.lan: memberlist: Refuting a suspect message (from: rd1)
2021-01-07T00:49:30.995+0800 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: rd3 51.6.196.216
2021-01-07T00:49:30.995+0800 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: rd2 51.6.196.235
2021-01-07T00:49:30.995+0800 [WARN]  agent: Service name will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.: service=rd-service_51.6.196.225
2021-01-07T00:49:30.995+0800 [INFO]  agent.server: Adding LAN server: server="rd3 (Addr: tcp/51.6.196.216:8801) (DC: hn)"
2021-01-07T00:49:30.995+0800 [INFO]  agent.server: Adding LAN server: server="rd2 (Addr: tcp/51.6.196.235:8801) (DC: hn)"
2021-01-07T00:49:30.995+0800 [INFO]  agent.server.serf.lan: serf: Re-joined to previously known node: rd3: 51.6.196.216:8802
2021-01-07T00:49:30.995+0800 [WARN]  agent: Service name will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.: service=redis_51.6.196.225
2021-01-07T00:49:30.997+0800 [INFO]  agent: Started HTTP server: address=51.6.196.225:8800 network=tcp
2021-01-07T00:49:30.997+0800 [INFO]  agent: started state syncer
2021-01-07T00:49:30.997+0800 [INFO]  agent: Retry join is supported for the following discovery methods: cluster=LAN discovery_methods="aliyun aws azure digitalocean gce k8s linode mdns os packet scaleway softlayer tencentcloud triton vsphere"
2021-01-07T00:49:30.997+0800 [INFO]  agent: Joining cluster...: cluster=LAN
2021-01-07T00:49:30.997+0800 [INFO]  agent: (LAN) joining: lan_addresses=[51.6.196.225, 51.6.196.235, 51.6.196.216]
2021-01-07T00:49:31.001+0800 [INFO]  agent: (LAN) joined: number_of_nodes=3
2021-01-07T00:49:31.001+0800 [INFO]  agent: Join cluster completed. Synced with initial agents: cluster=LAN num_agents=3
2021-01-07T00:49:37.125+0800 [WARN]  agent.server.raft: heartbeat timeout reached, starting election: last-leader=
2021-01-07T00:49:37.125+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 51.6.196.225:8801 [Candidate]" term=294
2021-01-07T00:49:38.007+0800 [ERROR] agent.anti_entropy: failed to sync remote state: error="No cluster leader"
2021-01-07T00:49:40.642+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-225
2021-01-07T00:49:42.716+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election
2021-01-07T00:49:42.716+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 51.6.196.225:8801 [Candidate]" term=295
2021-01-07T00:49:49.792+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election
2021-01-07T00:49:49.792+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 51.6.196.225:8801 [Candidate]" term=296
2021-01-07T00:49:50.643+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-225
2021-01-07T00:49:56.357+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election

```
这个是主节点重启之后的日志，一直在尝试向其他集群节点发起投票选举，但是对方都没有响应，导致超时并重试。

为什么对方不响应呢，下面我们来看看peer节点的日志：
```sh
2021-01-07T00:40:11.621+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:40:17.472+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-235
2021-01-07T00:40:20.936+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:40:25.765+0800 [ERROR] agent.server.memberlist.lan: memberlist: Push/Pull with rd1 failed: dial tcp 51.6.196.225:8802: connect: no route to host
2021-01-07T00:40:26.908+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:40:27.473+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-235
2021-01-07T00:40:36.019+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:40:37.473+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-235
2021-01-07T00:40:43.049+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:40:47.474+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-235
2021-01-07T00:40:50.734+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:40:57.475+0800 [WARN]  agent: Check is now critical: check=service:rd-server-51-6-196-235
2021-01-07T00:40:57.516+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
2021-01-07T00:41:03.435+0800 [WARN]  agent.server.raft: rejecting vote request since we have a leader: from=51.6.196.225:8801 leader=51.6.196.216:8801
```
可以看到，peer的确是收到了投票消息，但是拒绝了这个投票，因为他们认为他们已经有了一个主节点，所以不接受选举。

为什么会出现这样的情况，这不是正常的异常操作吗，那怎样才能恢复集群状态呢？带着这个疑问，我查询了很多文档，大家对这个问题确实都存在争议性，下面记录如下：

##### 1、有的人查询了raft的协议文档，发现是raft协议的处理建议就是如此
该白皮书在第6节的末尾指出，当服务器认为存在当前的领导者时，服务器应忽略RequestVote RPC：

为什么raft协议建议如此呢：
具体来说，如果服务器在当前领导者的最小选举超时时间内收到RequestVote RPC，则它不会更新其任期或授予其投票权（这不会影响正常的选举，在正常选举中，每个服务器在开始选举之前至少等待最小选举超时时）。这有助于避免重新加入的服务器造成中断：如果领导者能够对其集群发出心跳信号，则不会因任期较长而被淘汰。

但是，它具有非常讨厌的副作用。考虑带有节点A，B和C的这种情况
* A是领导者，B和C是追随者
* C由于某些网络问题而错过了足够的心跳信号，无法开始选举，并开始向A和B发送RequestVote RPC
* 这些RequestVote RPC请求到达A和B，但是它们被忽略，因为A是领导者
* 来自A的心跳和AppendEntries RPC仍被发送到C，但是C忽略了它们，因为它们来自较早的term编号
* 这种状态一直持续到A和B之间的连接中断并且它们接受C的RequestVote RPC。


尽管这不会导致停机或服务中断，但确实会带来一些问题：
* C将无限期地落后，并且在重新加入集群时可能必须从快照中恢复。
* raft集群的性能下降。

##### 2、有的人的解决方案如下
当认为存在领导者时，节点不应忽略所有RequestVote RPC。相反，他们应该从不是其配置一部分的服务器中忽略RequestVote RPC。
这样，本身并不知道自己不在集群中的服务器就无法通过发送RequestVote RPC破坏群集，但是不会无限期地忽略尝试开始选举的集群成员。

##### 3、etcd的处理
etcd/raft解决此问题的方式有所不同：当节点从较旧的term接收到一条Append消息时，它发送带有当前term的响应，而不仅仅是忽略该消息。这将触发整个集群中具有更高term的新选举，从而摆脱这种困境。

当然，这在某种程度上具有破坏性，因此也建议使用pre-vote的RPC（raft协议第9.6节），以避免一开始就增加该term。

#### 相关文档
https://github.com/hashicorp/raft/pull/256
https://groups.google.com/g/raft-dev/c/JEtBYaPpHXo
https://www.openlife.cc/blogs/2015/august/3-modifications-raft-consensus-algorithm-paper
https://www.openlife.cc/sites/default/files/php_uploads/4-modifications-for-Raft-consensus.pdf