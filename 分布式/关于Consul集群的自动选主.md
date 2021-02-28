# 关于Consul集群的自动选主

### 问题
在实验环境下，如果我们搭建一个consul集群，要让所有节点组成集群，一般需要如下步骤：
##### 1、在其中一个节点的配置文件中指定主节点：
```json
{
  "bootstrap": true,
}
```

##### 2、在其他节点指定备节点：
```json
{
  "bootstrap": false,
}
```
注：这里只是指定初始启动时的主备状态。

##### 3、然后在其他节点指定命令手动加入主节点：
```sh
consul join xxx
```

### 解决方案
对于生成环境下的consul来说，肯定是需要自动完成的，同时，在部署脚本中对每个节点都是一视同仁的，也就是说每个节点的配置应该是一样的，那么需要怎么做呢？

##### 1、配置文件
```json
{
  "bootstrap_expect": 2,
  "datacenter": "dc1",
  "data_dir": "/root/consul/data",
  "log_level": "INFO",
  "server": true,
  "node_name": "node2_201",
  "ui": true,
  "bind_addr": "51.6.196.201",
  "client_addr": "51.6.196.201",
  "advertise_addr": "51.6.196.201",
  "enable_script_checks": true,
  "retry_join": ["51.6.196.201","51.6.196.211"],
  "retry_interval": "5s",
  "ports":{
    "http": 8500,
    "dns": 8600,
    "server": 8300,
    "serf_lan": 8301,
    "serf_wan": 8302
    }
}

```
注意如下几个配置：
* `bootstrap_expect`：此选项指定Consul服务器节点的预期数量，并在有那么多服务器可用时自动进行集群选举
* `retry_join`：指定集群所有的节点ip，用于状态变更时自动进行选举（进程启动、进程重启、节点重启等）

##### 2、启动节点
在所有的节点都配置了同样的配置文件之后，开始对每个节点启动consul进程，此时我们就能发现，整个consul集群会自动进行选主，然后进入稳定状态。
```sh
bootstrap_expect = 2: A cluster with 2 servers will provide no failure tolerance. See https://www.consul.io/docs/internals/consensus.html#deployment-table
bootstrap_expect > 0: expecting 2 servers
==> Starting Consul agent...
           Version: 'v1.7.4'
           Node ID: '53c42a83-99bf-ebd4-708b-a26ba5eb6707'
         Node name: 'node2_201'
        Datacenter: 'dc1' (Segment: '<all>')
            Server: true (Bootstrap: false)
       Client Addr: [51.6.196.201] (HTTP: 8500, HTTPS: -1, gRPC: -1, DNS: 8600)
      Cluster Addr: 51.6.196.201 (LAN: 8301, WAN: 8302)
           Encrypt: Gossip: false, TLS-Outgoing: false, TLS-Incoming: false, Auto-Encrypt-TLS: false

==> Log data will now stream in as it occurs:

    2020-12-14T10:55:09.515-0500 [WARN]  agent: Node name will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.: node_name=node2_201
    2020-12-14T10:55:09.718-0500 [INFO]  agent.server.raft: initial configuration: index=0 servers=[]
    2020-12-14T10:55:09.718-0500 [INFO]  agent.server.raft: entering follower state: follower="Node at 51.6.196.201:8300 [Follower]" leader=
    2020-12-14T10:55:09.719-0500 [WARN]  agent.server.memberlist.wan: memberlist: Binding to public address without encryption!
    2020-12-14T10:55:09.719-0500 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: node2_201.dc1 51.6.196.201
    2020-12-14T10:55:09.719-0500 [WARN]  agent.server.memberlist.lan: memberlist: Binding to public address without encryption!
    2020-12-14T10:55:09.720-0500 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: node2_201 51.6.196.201
    2020-12-14T10:55:09.720-0500 [INFO]  agent.server: Handled event for server in area: event=member-join server=node2_201.dc1 area=wan
    2020-12-14T10:55:09.720-0500 [INFO]  agent.server: Adding LAN server: server="node2_201 (Addr: tcp/51.6.196.201:8300) (DC: dc1)"
    2020-12-14T10:55:09.720-0500 [INFO]  agent: Started DNS server: address=51.6.196.201:8600 network=udp
    2020-12-14T10:55:09.720-0500 [INFO]  agent: Started DNS server: address=51.6.196.201:8600 network=tcp
    2020-12-14T10:55:09.721-0500 [INFO]  agent: Started HTTP server: address=51.6.196.201:8500 network=tcp
    2020-12-14T10:55:09.721-0500 [INFO]  agent: started state syncer
==> Consul agent running!
    2020-12-14T10:55:09.721-0500 [INFO]  agent: Retry join is supported for the following discovery methods: cluster=LAN discovery_methods="aliyun aws azure digitalocean gce k8s linode mdns os packet scaleway softlayer tencentcloud triton vsphere"
    2020-12-14T10:55:09.721-0500 [INFO]  agent: Joining cluster...: cluster=LAN
    2020-12-14T10:55:09.721-0500 [INFO]  agent: (LAN) joining: lan_addresses=[51.6.196.201, 51.6.196.211]
    2020-12-14T10:55:09.723-0500 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: node3_211 51.6.196.211
    2020-12-14T10:55:09.723-0500 [INFO]  agent: (LAN) joined: number_of_nodes=2
    2020-12-14T10:55:09.723-0500 [INFO]  agent: Join cluster completed. Synced with initial agents: cluster=LAN num_agents=2
    2020-12-14T10:55:09.723-0500 [INFO]  agent.server: Adding LAN server: server="node3_211 (Addr: tcp/51.6.196.211:8300) (DC: dc1)"
    2020-12-14T10:55:09.726-0500 [INFO]  agent.server: Found expected number of peers, attempting bootstrap: peers=51.6.196.201:8300,51.6.196.211:8300
    2020-12-14T10:55:09.846-0500 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: node3_211.dc1 51.6.196.211
    2020-12-14T10:55:09.846-0500 [INFO]  agent.server: Handled event for server in area: event=member-join server=node3_211.dc1 area=wan




    2020-12-14T10:55:16.732-0500 [ERROR] agent.anti_entropy: failed to sync remote state: error="No cluster leader"
    2020-12-14T10:55:17.029-0500 [INFO]  agent.server: New leader elected: payload=node3_211
    2020-12-14T10:55:17.578-0500 [INFO]  agent: Synced node info
    2020-12-14T10:55:17.770-0500 [INFO]  agent: Synced service: service=nginx201

```