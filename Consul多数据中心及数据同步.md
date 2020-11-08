# Consul多数据中心及数据同步

## 多数据中心
### 1、搭建多数据中心
在上一篇文章中，我们讲解了单数据中心的搭建流程，这边文章将在其基础之上构建多数据中心。
我们另选一个region的两个节点，按照单数据中心的方式搭建好，然后执行如下命令，先查看下数据中心情况：
```sh
[root@karbor2 consul]# ./consul members -wan
Node                 Address         Status  Type    Build  Protocol  DC        Segment
ts-test-38.ts-test1  30.3.3.38:8302  alive   server  1.8.4  2         ts-test1  <all>
ts-test-39.ts-test1  30.3.3.39:8302  alive   server  1.8.4  2         ts-test1  <all>
```
可以看到，当前wan只有本region数据中心的两个节点

### 2、关联数据中心
下面我们开始将两个region的数据中心关联起来，我们在dc1的任一server节点执行如下命令：
```sh
[root@karbor1 consul]# ./consul join -wan 30.3.3.38
Successfully joined cluster by contacting 1 nodes.
```
这里的30.3.3.38可以是dc2的任一server节点

再次查看wan，我们分别在dc1和dc2的任意节点执行如下命令：
```sh
[root@karbor2 consul]# ./consul members -wan
Node                 Address         Status  Type    Build  Protocol  DC        Segment
ts-test-38.ts-test1  30.3.3.38:8302  alive   server  1.8.4  2         ts-test1  <all>
ts-test-39.ts-test1  30.3.3.39:8302  alive   server  1.8.4  2         ts-test1  <all>
ts-test-42.ts-test   30.1.3.42:8302  alive   server  1.8.4  2         ts-test   <all>
ts-test-43.ts-test   30.1.3.43:8302  alive   server  1.8.4  2         ts-test   <all>
```
显示结果都一样，同时我们可以发现，在将dc1的任一server节点加入到dc2中，dc1和dc2下所有的server节点都会自动加入该wan。

### 3、验证数据同步
我们在dc1的节点上增加数据key：
```sh
[root@karbor1 consul]# ./consul kv put ts/test/sex 'boy'
Success! Data written to: ts/test/sex
```
在dc2数据中心查询该key：
```sh
[root@karbor2 consul]# ./consul kv get ts/test/sex
Error! No key exists at: ts/test/sex
```
可以发现，两个数据中心的数据是不同步的，也就是说集群间不相互影响。

## 数据同步
对于有在多个数据中心进行数据同步的需求来说，官方也给出了两种解决方案，下面具体来说说

### 1、ACL同步复制
官方文档为：https://learn.hashicorp.com/tutorials/consul/access-control-token-migration?in=consul/day-2-agent-authentication

设置ACL复制涉及如下几个步骤：
* 1、在主数据中心中的所有Consul代理上设置primary_datacenter参数。
* 2、创建复制令牌。
* 3、在辅助数据中心中的所有Consul代理上配置primary_datacenter参数。
* 4、在辅助数据中心中的服务器上启用令牌复制。
* 5、将复制令牌应用于辅助数据中心中的所有服务器。

#### 1.1 配置主数据中心
默认情况下在主数据中心上启用了复制，但是我们还是建议在所有server和client上显式设置primary_datacenter参数。配置应类似于以下示例：
```sh
{
  "datacenter": "primary_dc",
  "primary_datacenter": "primary_dc",
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "down_policy": "extend-cache",
    "enable_token_persistence": true
  }
}

```
primary_datacenter参数将主数据中心设置为具有所有ACL信息的权限。
配置完成之后，需要重启agent。

接下来，使用以下特权创建用于管理ACL的复制令牌。

#### 1.2 创建用于ACL管理的复制令牌
创建文件replication-policy.hcl，内容如下：
```sh
acl = "write"

operator = "write"

service_prefix "" {
  policy = "read"
  intentions = "read"
}

```
* 1、acl = "write"表示允许复制token
* 2、operator = "write"用于复制代理默认配置条目并在辅助数据中心中启用CA证书签名。
* 3、service_prefix 用于复制服务默认配置条目，CA和intentions数据。

下面我们采用上面定义的rule创建acl策略：
```sh
consul acl policy create -name replication -rules @replication-policy.hcl
```
最后，使用新创建的策略来创建复制令牌。
```sh
consul acl token create -description "replication token" -policy-name replication
```

#### 1.3 配置副数据中心
需要在所有服务器上将primary_datacenter参数设置为主要数据中心的名称，并将enable_token_replication设置为true。
```sh
{
  "datacenter": "dc_secondary",
  "primary_datacenter": "primary_dc",
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "down_policy": "extend-cache",
    "enable_token_persistence": true,
    "enable_token_replication": true
  }
}

```
配置完成之后，需要重启agent。

最后，使用CLI将复制令牌应用于所有服务器。
```sh
consul acl set-agent-token replication <token>
```

同样的，对于客户端，您需要将primary_datacenter参数设置为主数据中心的名称，并将enable_token_replication设置为true。
```sh
{
  "datacenter": "dc_secondary",
  "primary_datacenter": "primary_dc",
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "down_policy": "extend-cache",
    "enable_token_persistence": true,
    "enable_token_replication": true
  }
}

```
配置完成之后，需要重启agent。

### 2、通过consul-replication工具同步
#### 2.1 下载
在git上下载一个最新的二进制包：https://github.com/hashicorp/consul-replicate
将consul-replication工具上传到需要同步其他数据中心的consul节点上

#### 2.2 启动服务
```sh
[root@karbor1 consul]# consul-replicate -log-level info -prefix "ts@ts-test1"
2020/10/23 06:31:39.371080 [INFO] consul-replicate v0.4.0 (886abcc)
2020/10/23 06:31:39.371110 [INFO] (runner) creating new runner (once: false)
2020/10/23 06:31:39.371485 [INFO] (runner) creating watcher
2020/10/23 06:31:39.371582 [INFO] (runner) starting
2020/10/23 06:31:39.374341 [INFO] (runner) running

```
参数含义如下：
* 1、-log-level: 表示打印日志级别
* 2、-prefix： ts@ts-test1——表示该数据中心会时刻监控数据中心ts-test1中key前缀为ts的所有key

注：consul-replicate是一个常驻进程，它会时刻监控key的变化，并立刻更新本数据中心的数据

#### 2.3 配置文件
该工具除了支持命令行参数之外，还支持配置文件，这对于多个配置来说是非常方便的，下面是官方给的一个配置文件样例：
```sh
# This denotes the start of the configuration section for Consul. All values
# contained in this section pertain to Consul.
consul {
  # This block specifies the basic authentication information to pass with the
  # request. For more information on authentication, please see the Consul
  # documentation.
  auth {
    enabled  = true
    username = "test"
    password = "test"
  }

  # This is the address of the Consul agent. By default, this is
  # 127.0.0.1:8500, which is the default bind and port for a local Consul
  # agent. It is not recommended that you communicate directly with a Consul
  # server, and instead communicate with the local Consul agent. There are many
  # reasons for this, most importantly the Consul agent is able to multiplex
  # connections to the Consul server and reduce the number of open HTTP
  # connections. Additionally, it provides a "well-known" IP address for which
  # clients can connect.
  address = "127.0.0.1:8500"

  # This is the ACL token to use when connecting to Consul. If you did not
  # enable ACLs on your Consul cluster, you do not need to set this option.
  #
  # This option is also available via the environment variable CONSUL_TOKEN.
  token = "abcd1234"

  # This controls the retry behavior when an error is returned from Consul.
  # Consul Replicate is highly fault tolerant, meaning it does not exit in the
  # face of failure. Instead, it uses exponential back-off and retry functions
  # to wait for the cluster to become available, as is customary in distributed
  # systems.
  retry {
    # This enabled retries. Retries are enabled by default, so this is
    # redundant.
    enabled = true

    # This specifies the number of attempts to make before giving up. Each
    # attempt adds the exponential backoff sleep time. Setting this to
    # zero will implement an unlimited number of retries.
    attempts = 12

    # This is the base amount of time to sleep between retry attempts. Each
    # retry sleeps for an exponent of 2 longer than this base. For 5 retries,
    # the sleep times would be: 250ms, 500ms, 1s, 2s, then 4s.
    backoff = "250ms"

    # This is the maximum amount of time to sleep between retry attempts.
    # When max_backoff is set to zero, there is no upper limit to the
    # exponential sleep between retry attempts.
    # If max_backoff is set to 10s and backoff is set to 1s, sleep times
    # would be: 1s, 2s, 4s, 8s, 10s, 10s, ...
    max_backoff = "1m"
  }

  # This block configures the SSL options for connecting to the Consul server.
  ssl {
    # This enables SSL. Specifying any option for SSL will also enable it.
    enabled = true

    # This enables SSL peer verification. The default value is "true", which
    # will check the global CA chain to make sure the given certificates are
    # valid. If you are using a self-signed certificate that you have not added
    # to the CA chain, you may want to disable SSL verification. However, please
    # understand this is a potential security vulnerability.
    verify = false

    # This is the path to the certificate to use to authenticate. If just a
    # certificate is provided, it is assumed to contain both the certificate and
    # the key to convert to an X509 certificate. If both the certificate and
    # key are specified, Consul Replicate will automatically combine them into an
    # X509 certificate for you.
    cert = "/path/to/client/cert"
    key  = "/path/to/client/key"

    # This is the path to the certificate authority to use as a CA. This is
    # useful for self-signed certificates or for organizations using their own
    # internal certificate authority.
    ca_cert = "/path/to/ca"

    # This is the path to a directory of PEM-encoded CA cert files. If both
    # `ca_cert` and `ca_path` is specified, `ca_cert` is preferred.
    ca_path = "path/to/certs/"

    # This sets the SNI server name to use for validation.
    server_name = "my-server.com"
  }
}

# This is the list of keys to exclude if they are found in the prefix. This can
# be specified multiple times to exclude multiple keys from replication.
exclude {
  source = "my-key"
}

# This is the signal to listen for to trigger a graceful stop. The default value
# is shown below. Setting this value to the empty string will cause Consul
# Replicate to not listen for any graceful stop signals.
kill_signal = "SIGINT"

# This is the log level. If you find a bug in Consul Replicate, please enable
# debug logs so we can help identify the issue. This is also available as a
# command line flag.
log_level = "warn"

# This is the maximum interval to allow "stale" data. By default, only the
# Consul leader will respond to queries; any requests to a follower will
# forward to the leader. In large clusters with many requests, this is not as
# scalable, so this option allows any follower to respond to a query, so long
# as the last-replicated data is within these bounds. Higher values result in
# less cluster load, but are more likely to have outdated data.
max_stale = "10m"

# This is the path to store a PID file which will contain the process ID of the
# Consul Replicate process. This is useful if you plan to send custom signals
# to the process.
pid_file = "/path/to/pid"

# This is the prefix and datacenter to replicate and the resulting destination.
prefix {
  source      = "global"
  datacenter  = "nyc1"
  destination = "default"
}

# This is the signal to listen for to trigger a reload event. The default value
# is shown below. Setting this value to the empty string will cause Consul
# Replicate to not listen for any reload signals.
reload_signal = "SIGHUP"

# This is the path in Consul to store replication and leader status.
status_dir = "service/consul-replicate/statuses"

# This block defines the configuration for connecting to a syslog server for
# logging.
syslog {
  # This enables syslog logging. Specifying any other option also enables
  # syslog logging.
  enabled = true

  # This is the name of the syslog facility to log to.
  facility = "LOCAL5"
}

# This is the quiescence timers; it defines the minimum and maximum amount of
# time to wait for the cluster to reach a consistent state before rendering a
# replicating. This is useful to enable in systems that have a lot of flapping,
# because it will reduce the the number of times a replication occurs.
wait {
  min = "5s"
  max = "10s"
}
```
上面的配置不是所有的都需要的，可以根据自己的情况删除，从上面我们可以看出，该工具也是比较智能的，支持的功能比较多：
* 1、支持数据中心之间的认证
* 2、支持重试机制
* 3、支持日志转储
* 4、...

利用配置文件启动命令如下：
```sh
$ consul-replicate -config "/my/config.hcl"
```

注意：
1、可以多次指定此参数以加载多个配置文件。最右边的配置具有最高的优先级。如果提供了目录路径（而不是文件路径），则给定目录中的所有文件将以词法顺序递归合并。请注意，不遵循符号链接。
2、命令行参数会覆盖文件参数

#### 2.4 实测
1、我们在dc1任一节点启动该工具：
```sh
[root@karbor1 consul]# consul-replicate -log-level info -prefix "ts@ts-test1"
2020/10/23 06:31:39.371080 [INFO] consul-replicate v0.4.0 (886abcc)
2020/10/23 06:31:39.371110 [INFO] (runner) creating new runner (once: false)
2020/10/23 06:31:39.371485 [INFO] (runner) creating watcher
2020/10/23 06:31:39.371582 [INFO] (runner) starting
2020/10/23 06:31:39.374341 [INFO] (runner) running

```

2、在dc2任一节点更新或者增加参数
```sh
[root@karbor2 consul]# ./consul kv put ts/test/lq "sdsdsdfd"
Success! Data written to: ts/test/lq
```

3、可以查看dc1中的节点监控日志：
```sh
2020/10/23 06:46:04.698742 [INFO] (runner) running
2020/10/23 06:46:04.713276 [INFO] (runner) replicated 1 updates, 0 deletes

```

4、在dc1任一节点查询数据：
```sh
[root@karbor1 djmanager]# /opt/consul/consul kv get ts/test/lq
sdsdsdfd
```
可以看到两个不同的数据中心的数据已经是一样的了

##### 说明
注意，这个工具只会同步kv数据，不会同步服务，也就是说各个数据中心注册的服务是相互独立的，不会被这个工具工步

参考链接：https://www.jsonlong.com/2020/06/02/%E8%B7%A8%E6%9C%BA%E6%88%BF%E6%95%B0%E6%8D%AE%E5%90%8C%E6%AD%A5%E5%8F%AF%E8%A1%8C%E6%80%A7%E8%B0%83%E7%A0%94/