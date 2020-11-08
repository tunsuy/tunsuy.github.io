# 微服务系统consul初探
## consul是什么

简单来说，consul就是一个用于微服务治理的系统，什么是微服务，这个大家自行了解。consul的主要功能有如下几个：

* 服务注册
* 服务发现
* KV数据存储
* 配置管理
* 分布式锁
* 服务监控
* 。。。



下面就简单来实际操作一把



## 搭建集群

### 1、创建目录
```sh
[root@ts consul]# ll /opt/consul/
total 110464
drwxr-x--- 2 root root      4096 Oct 19 15:45 conf
-rwxr-x--- 1 root root 113093117 Oct 19 14:33 consul
-rw-r----- 1 root root         5 Oct 19 14:54 consul.pid
drwxr-x--- 4 root root      4096 Oct 19 14:54 data
drwxr-x--- 2 root root      4096 Oct 19 15:28 log
drwxr-x--- 2 root root      4096 Oct 19 14:54 ui
```
上面只是一个示例，大家可以根据自己的目录结构来。

### 2、配置文件
```json
{
        "bootstrap": true,
        "server": true,
        "datacenter": "ts-test",
        "data_dir": "/opt/consul/data",
        "pid_file": "/opt/consul/consul.pid",
        "ui_dir": "/opt/consul/ui",
        "client_addr": "0.0.0.0",
        "node_name": "ts-test-42",
        "rejoin_after_leave": true,
        "log_level": "INFO",
        "bind_addr":"30.1.3.42"
}
```
根据上面创建的目录文件路径、节点等信息，编写如上配置文件信息，保存文件为main.json，并放入conf文件夹下。

### 3、启动服务
执行如下命令：
```sh
nohup /opt/consul/consul agent -config-file=/opt/consul/conf/main.json > /opt/consul/log/consul.log 2>&1 &
```

同样的，在节点2上也进行如上步骤操作，注意，一定要将bootstrap改为false，因为一个集群只能有一个bootstrap为true。
否则，会出现如下错误：
```sh
    2020-10-19T15:01:36.327+0800 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: ts-test-43 30.1.3.43
    2020-10-19T15:01:36.327+0800 [INFO]  agent.server: Adding LAN server: server="ts-test-43 (Addr: tcp/30.1.3.43:8300) (DC: ts-test)"
    2020-10-19T15:01:36.327+0800 [INFO]  agent.server: New leader elected: payload=ts-test-43
    2020-10-19T15:01:36.327+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42
    2020-10-19T15:01:36.327+0800 [INFO]  agent.server: member joined, marking health alive: member=ts-test-43
    2020-10-19T15:01:36.328+0800 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: ts-test-43.ts-test 30.1.3.43
    2020-10-19T15:01:36.328+0800 [INFO]  agent.server: Handled event for server in area: event=member-join server=ts-test-43.ts-test area=wan
    2020-10-19T15:01:51.343+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-42 other=ts-test-43
    2020-10-19T15:01:51.343+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42
    2020-10-19T15:02:51.341+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-42 other=ts-test-43
    2020-10-19T15:02:51.341+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42
    2020-10-19T15:03:51.341+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-42 other=ts-test-43
    2020-10-19T15:03:51.341+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42
    2020-10-19T15:04:51.342+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-42 other=ts-test-43
    2020-10-19T15:04:51.342+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42
    2020-10-19T15:05:51.342+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-42 other=ts-test-43
    2020-10-19T15:05:51.342+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42
    2020-10-19T15:06:51.343+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-42 other=ts-test-43
    2020-10-19T15:06:51.343+0800 [ERROR] agent.server: Two nodes are in bootstrap mode. Only one node should be in bootstrap mode, not adding Raft peer.: node_to_add=ts-test-43 other=ts-test-42

```

### 4、加入集群
在第二个节点上执行如下命令：
```sh
consul join 30.1.3.42
```

### 5、查看集群信息
在节点1上执行：
```sh
[root@ts consul]# ./consul info
agent:
	check_monitors = 0
	check_ttls = 0
	checks = 0
	services = 0
build:
	prerelease = 
	revision = 12b16df3
	version = 1.8.4
consul:
	acl = disabled
	bootstrap = false
	known_datacenters = 1
	leader = true
	leader_addr = 30.1.3.43:8300
	server = true
raft:
	applied_index = 880
	commit_index = 880
	fsm_pending = 0
	last_contact = 0
	last_log_index = 880
	last_log_term = 135
	last_snapshot_index = 0
	last_snapshot_term = 0
	latest_configuration = [{Suffrage:Voter ID:e34ff4b2-5f71-8d7a-bc25-784827f27dab Address:30.1.3.42:8300} {Suffrage:Voter ID:082ea273-263c-2de8-9516-3eb6c223071c Address:30.1.3.43:8300}]
	latest_configuration_index = 0
	num_peers = 1
	protocol_version = 3
	protocol_version_max = 3
	protocol_version_min = 0
	snapshot_version_max = 1
	snapshot_version_min = 0
	state = Leader
	term = 135

```
关注consul信息，可以看到leader = true，表示是主节点

在另一个节点执行
```sh
[root@ts1 consul]# ./consul info
agent:
	check_monitors = 1
	check_ttls = 0
	checks = 1
	services = 1
build:
	prerelease = 
	revision = 12b16df3
	version = 1.8.4
consul:
	acl = disabled
	bootstrap = true
	known_datacenters = 1
	leader = false
	leader_addr = 30.1.3.43:8300
	server = true
raft:
	applied_index = 888
	commit_index = 888
	fsm_pending = 0
	last_contact = 23.765731ms
	last_log_index = 888
	last_log_term = 135
	last_snapshot_index = 0
	last_snapshot_term = 0
	latest_configuration = [{Suffrage:Voter ID:e34ff4b2-5f71-8d7a-bc25-784827f27dab Address:30.1.3.42:8300} {Suffrage:Voter ID:082ea273-263c-2de8-9516-3eb6c223071c Address:30.1.3.43:8300}]
	latest_configuration_index = 0
	num_peers = 1
	protocol_version = 3
	protocol_version_max = 3
	protocol_version_min = 0
	snapshot_version_max = 1
	snapshot_version_min = 0
	state = Follower
	term = 135
```
可以看到leader = false，说明是一个备节点


## 验证数据同步
### 1、增加kv存储
在一个节点上执行如下命令：
```sh
[root@ts consul]# ./consul kv put ts/test/name 'ts'
Success! Data written to: ts/test/name
```

### 2、查询kv存储
在另外一个节点上执行如下命令：
```sh
[root@ts consul]# ./consul kv get ts/test/name
ts
```

可以发现，节点间的数据是共享的。


## 验证集群可用性
因为consul使用的raft一致性协议，该协议是是强一致性的，因为一般要求部署单数节点，也就是说3个节点，挂1个可用，5个节点挂2个可用。
如果是两节点，挂1个，整个集群就不可用了。

我们将其中一个节点停掉，然后我们可以看到另外一个节点：
```sh
    2020-10-19T15:32:59.680+0800 [ERROR] agent.server.raft: failed to appendEntries to: peer="{Nonvoter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:32:59.816+0800 [ERROR] agent.server.raft: failed to appendEntries to: peer="{Nonvoter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:00.058+0800 [ERROR] agent.server.raft: failed to appendEntries to: peer="{Nonvoter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:00.344+0800 [WARN]  agent: error getting server health from server: server=ts-test-43 error="context deadline exceeded"
    2020-10-19T15:33:00.463+0800 [ERROR] agent.server.raft: failed to appendEntries to: peer="{Nonvoter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:00.518+0800 [ERROR] agent.server.raft: failed to heartbeat to: peer=30.1.3.43:8300 error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:00.968+0800 [INFO]  agent.server.memberlist.lan: memberlist: Suspect ts-test-43 has failed, no acks received
    2020-10-19T15:33:01.035+0800 [ERROR] agent.server.raft: failed to heartbeat to: peer=30.1.3.43:8300 error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:01.197+0800 [ERROR] agent.server.raft: failed to appendEntries to: peer="{Nonvoter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:01.345+0800 [WARN]  agent: error getting server health from server: server=ts-test-43 error="rpc error getting client: failed to get conn: dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:01.604+0800 [ERROR] agent.server.raft: failed to heartbeat to: peer=30.1.3.43:8300 error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:01.716+0800 [WARN]  agent.server.raft: failed to contact: server-id=082ea273-263c-2de8-9516-3eb6c223071c time=2.500086438s
    2020-10-19T15:33:01.716+0800 [WARN]  agent.server.raft: failed to contact quorum of nodes, stepping down
    2020-10-19T15:33:01.716+0800 [INFO]  agent.server.raft: entering follower state: follower="Node at 30.1.3.42:8300 [Follower]" leader=
    2020-10-19T15:33:02.343+0800 [WARN]  agent: error getting server health from server: server=ts-test-43 error="context deadline exceeded"
    2020-10-19T15:33:02.344+0800 [INFO]  agent.server: cluster leadership lost
    2020-10-19T15:33:02.528+0800 [ERROR] agent.server.raft: failed to heartbeat to: peer=30.1.3.43:8300 error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:02.578+0800 [ERROR] agent.server.raft: failed to appendEntries to: peer="{Nonvoter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:03.968+0800 [INFO]  agent.server.memberlist.lan: memberlist: Suspect ts-test-43 has failed, no acks received
    2020-10-19T15:33:04.968+0800 [INFO]  agent.server.memberlist.lan: memberlist: Marking ts-test-43 as failed, suspect timeout reached (0 peer confirmations)
    2020-10-19T15:33:04.968+0800 [INFO]  agent.server.serf.lan: serf: EventMemberFailed: ts-test-43 30.1.3.43
    2020-10-19T15:33:04.968+0800 [INFO]  agent.server: Removing LAN server: server="ts-test-43 (Addr: tcp/30.1.3.43:8300) (DC: ts-test)"
    2020-10-19T15:33:06.967+0800 [INFO]  agent.server.memberlist.wan: memberlist: Suspect ts-test-43.ts-test has failed, no acks received
    2020-10-19T15:33:06.968+0800 [INFO]  agent.server.memberlist.lan: memberlist: Suspect ts-test-43 has failed, no acks received
    2020-10-19T15:33:11.702+0800 [WARN]  agent.server.raft: heartbeat timeout reached, starting election: last-leader=
    2020-10-19T15:33:11.702+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 30.1.3.42:8300 [Candidate]" term=3
    2020-10-19T15:33:11.706+0800 [WARN]  agent.server.raft: unable to get address for sever, using fallback address: id=082ea273-263c-2de8-9516-3eb6c223071c fallback=30.1.3.43:8300 error="Could not find address for server id 082ea273-263c-2de8-9516-3eb6c223071c"
    2020-10-19T15:33:11.707+0800 [ERROR] agent.server.raft: failed to make requestVote RPC: target="{Voter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:11.984+0800 [INFO]  agent.server.serf.lan: serf: attempting reconnect to ts-test-43 30.1.3.43:8301
    2020-10-19T15:33:13.784+0800 [ERROR] agent: Coordinate update error: error="No cluster leader"
    2020-10-19T15:33:16.968+0800 [INFO]  agent.server.memberlist.wan: memberlist: Suspect ts-test-43.ts-test has failed, no acks received
    2020-10-19T15:33:18.551+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election
    2020-10-19T15:33:18.551+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 30.1.3.42:8300 [Candidate]" term=4
    2020-10-19T15:33:18.555+0800 [WARN]  agent.server.raft: unable to get address for sever, using fallback address: id=082ea273-263c-2de8-9516-3eb6c223071c fallback=30.1.3.43:8300 error="Could not find address for server id 082ea273-263c-2de8-9516-3eb6c223071c"
    2020-10-19T15:33:18.555+0800 [ERROR] agent.server.raft: failed to make requestVote RPC: target="{Voter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:25.525+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election
    2020-10-19T15:33:25.525+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 30.1.3.42:8300 [Candidate]" term=5
    2020-10-19T15:33:25.541+0800 [WARN]  agent.server.raft: unable to get address for sever, using fallback address: id=082ea273-263c-2de8-9516-3eb6c223071c fallback=30.1.3.43:8300 error="Could not find address for server id 082ea273-263c-2de8-9516-3eb6c223071c"
    2020-10-19T15:33:25.542+0800 [ERROR] agent.server.raft: failed to make requestVote RPC: target="{Voter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:34.492+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election
    2020-10-19T15:33:34.492+0800 [INFO]  agent.server.raft: entering candidate state: node="Node at 30.1.3.42:8300 [Candidate]" term=6
    2020-10-19T15:33:34.496+0800 [WARN]  agent.server.raft: unable to get address for sever, using fallback address: id=082ea273-263c-2de8-9516-3eb6c223071c fallback=30.1.3.43:8300 error="Could not find address for server id 082ea273-263c-2de8-9516-3eb6c223071c"
    2020-10-19T15:33:34.496+0800 [ERROR] agent.server.raft: failed to make requestVote RPC: target="{Voter 082ea273-263c-2de8-9516-3eb6c223071c 30.1.3.43:8300}" error="dial tcp 30.1.3.42:0->30.1.3.43:8300: connect: connection refused"
    2020-10-19T15:33:36.967+0800 [INFO]  agent.server.memberlist.wan: memberlist: Suspect ts-test-43.ts-test has failed, no acks received
    2020-10-19T15:33:36.968+0800 [INFO]  agent.server.memberlist.wan: memberlist: Marking ts-test-43.ts-test as failed, suspect timeout reached (0 peer confirmations)
    2020-10-19T15:33:36.968+0800 [INFO]  agent.server.serf.wan: serf: EventMemberFailed: ts-test-43.ts-test 30.1.3.43
    2020-10-19T15:33:36.968+0800 [INFO]  agent.server: Handled event for server in area: event=member-failed server=ts-test-43.ts-test area=wan
    2020-10-19T15:33:39.937+0800 [WARN]  agent.server.raft: Election timeout reached, restarting election

```
可以看到，该节点虽然还在，但是一直处于选举状态，并且接口都不可用了
```sh
[root@ts consul]# ./consul kv get ts/test/name
Error querying Consul agent: Unexpected response code: 500
```

## 服务注册
### 1、增加配置文件
在conf配置目录下增加ts.json配置文件
```sh
{
  "service": {
    "name": "ts42",
    "tags": ["ts"],
    "port": 28799
  }
}
```

### 2、查看服务
重启服务之后，执行如下命令：
```sh
[root@ts2 consul]#  curl http://30.1.3.43:8500/v1/catalog/service/ts42 | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   580  100   580    0     0   405k      0 --:--:-- --:--:-- --:--:--  566k
[
    {
        "ID": "e34ff4b2-5f71-8d7a-bc25-784827f27dab",
        "Node": "ts-test-42",
        "Address": "30.1.3.42",
        "Datacenter": "ts-test",
        "TaggedAddresses": {
            "lan": "30.1.3.42",
            "lan_ipv4": "30.1.3.42",
            "wan": "30.1.3.42",
            "wan_ipv4": "30.1.3.42"
        },
        "NodeMeta": {
            "consul-network-segment": ""
        },
        "ServiceKind": "",
        "ServiceID": "ts42",
        "ServiceName": "ts42",
        "ServiceTags": [
            "ts"
        ],
        "ServiceAddress": "",
        "ServiceWeights": {
            "Passing": 1,
            "Warning": 1
        },
        "ServiceMeta": {},
        "ServicePort": 28799,
        "ServiceEnableTagOverride": false,
        "ServiceProxy": {
            "MeshGateway": {},
            "Expose": {}
        },
        "ServiceConnect": {},
        "CreateIndex": 527,
        "ModifyIndex": 527
    }
]
```
可以看到该服务详细信息

查看所有服务：
```sh
[root@karbor2 consul]#  curl http://30.1.3.43:8500/v1/catalog/services | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    35  100    35    0     0  28112      0 --:--:-- --:--:-- --:--:-- 35000
{
    "consul": [],
    "ts42": [
        "ts"
    ]
}
```

注：这里的接口展示的都是所有的服务，包括异常的服务。


## 服务监控
consul执行本地脚本检查、http接口检查、tcp接口检查，这里只演示本地脚本检查。

### 1、增加配置
同样的，在conf配置目录下的ts.json配置文件中增加如下配置：
```json
{
  "service": {
    ...
    "checks": [{
      "id": "check_ts",
      "name": "test",
      "args": ["/home/ts/ts_check.sh"],
      "interval": "5s"
    }]
  }
}
```

其中，/home/ts/ts_check.sh脚本如下：
```sh
#!/bin/bash

echo 'check pass' >> /home/ts/ts_check.log
exit 0
```
注意这个脚本是用来检查服务状态的，根据自身情况编写相应代码，这里只是示例；
其中，脚本的返回码会被作为consul用来判断服务是否正常的依据：
* 0——服务正常
* 1——服务告警
* 其他——服务异常

监控日志记录：
```sh
check pass
check pass
check pass
check pass
check pass
check pass
check pass
check pass
check pass
```
可以发现consul在定时执行该脚本

### 2、查看服务状态
```sh
[root@ts consul]# curl http://30.1.3.43:8500/v1/health/service/ts42 | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1051  100  1051    0     0   697k      0 --:--:-- --:--:-- --:--:-- 1026k
[
    {
        "Node": {
            "ID": "e34ff4b2-5f71-8d7a-bc25-784827f27dab",
            "Node": "ts-test-42",
            "Address": "30.1.3.42",
            "Datacenter": "ts-test",
            "TaggedAddresses": {
                "lan": "30.1.3.42",
                "lan_ipv4": "30.1.3.42",
                "wan": "30.1.3.42",
                "wan_ipv4": "30.1.3.42"
            },
            "Meta": {
                "consul-network-segment": ""
            },
            "CreateIndex": 9,
            "ModifyIndex": 404
        },
        "Service": {
            "ID": "ts42",
            "Service": "ts42",
            "Tags": [
                "ts"
            ],
            "Address": "",
            "Meta": null,
            "Port": 28799,
            "Weights": {
                "Passing": 1,
                "Warning": 1
            },
            "EnableTagOverride": false,
            "Proxy": {
                "MeshGateway": {},
                "Expose": {}
            },
            "Connect": {},
            "CreateIndex": 527,
            "ModifyIndex": 527
        },
        "Checks": [
            {
                "Node": "ts-test-42",
                "CheckID": "serfHealth",
                "Name": "Serf Health Status",
                "Status": "passing",
                "Notes": "",
                "Output": "Agent alive and reachable",
                "ServiceID": "",
                "ServiceName": "",
                "ServiceTags": [],
                "Type": "",
                "Definition": {},
                "CreateIndex": 9,
                "ModifyIndex": 9
            },
            {
                "Node": "ts-test-42",
                "CheckID": "check_ts",
                "Name": "test",
                "Status": "passing",
                "Notes": "",
                "Output": "",
                "ServiceID": "ts42",
                "ServiceName": "ts42",
                "ServiceTags": [
                    "ts"
                ],
                "Type": "script",
                "Definition": {},
                "CreateIndex": 629,
                "ModifyIndex": 860
            }
        ]
    }
]

```
返回信息中的"Status": "passing"就表示该服务正常。

### 3、异常构造
我们将/home/ts/ts_check.sh脚本修改如下：
```sh
#!/bin/bash

echo 'check not pass' >> /home/ts/ts_check.log
exit -1
```

再次查看服务状态：
```sh
[root@ts consul]# curl http://30.1.3.43:8500/v1/health/service/ts42 | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1051  100  1051    0     0   697k      0 --:--:-- --:--:-- --:--:-- 1026k
[
    {
        "Node": {
            "ID": "e34ff4b2-5f71-8d7a-bc25-784827f27dab",
            "Node": "ts-test-42",
            "Address": "30.1.3.42",
            "Datacenter": "ts-test",
            "TaggedAddresses": {
                "lan": "30.1.3.42",
                "lan_ipv4": "30.1.3.42",
                "wan": "30.1.3.42",
                "wan_ipv4": "30.1.3.42"
            },
            "Meta": {
                "consul-network-segment": ""
            },
            "CreateIndex": 9,
            "ModifyIndex": 404
        },
        "Service": {
            "ID": "ts42",
            "Service": "ts42",
            "Tags": [
                "ts"
            ],
            "Address": "",
            "Meta": null,
            "Port": 28799,
            "Weights": {
                "Passing": 1,
                "Warning": 1
            },
            "EnableTagOverride": false,
            "Proxy": {
                "MeshGateway": {},
                "Expose": {}
            },
            "Connect": {},
            "CreateIndex": 527,
            "ModifyIndex": 527
        },
        "Checks": [
            {
                "Node": "ts-test-42",
                "CheckID": "serfHealth",
                "Name": "Serf Health Status",
                "Status": "passing",
                "Notes": "",
                "Output": "Agent alive and reachable",
                "ServiceID": "",
                "ServiceName": "",
                "ServiceTags": [],
                "Type": "",
                "Definition": {},
                "CreateIndex": 9,
                "ModifyIndex": 9
            },
            {
                "Node": "ts-test-42",
                "CheckID": "check_ts",
                "Name": "test",
                "Status": "critical",
                "Notes": "",
                "Output": "",
                "ServiceID": "ts42",
                "ServiceName": "ts42",
                "ServiceTags": [
                    "ts"
                ],
                "Type": "script",
                "Definition": {},
                "CreateIndex": 629,
                "ModifyIndex": 860
            }
        ]
    }
]

```
返回信息中的"Status": "critical"就表示该服务异常。