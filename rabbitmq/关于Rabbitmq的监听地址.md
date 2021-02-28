# 关于rabbitmq的监听地址

## listener
为了让RabbitMQ接受客户端连接，它需要绑定到一个或多个接口并侦听（特定于协议的）端口。在RabbitMQ中，一个这样的接口/端口对称为listener。listeners是使用`listeners.tcp.*`配置选项配置的。
TCP侦听器同时配置接口和端口。以下示例演示如何配置AMQP 0-9-1和AMQP 1.0侦听器以使用特定的IP和标准端口：
```sh
listeners.tcp.1 = 192.168.1.99:5672
```

默认情况下，RabbitMQ将在所有可用接口上侦听端口5672。可以将客户端连接限制为接口的子集，或者甚至仅一个接口（例如仅IPv6的接口）。以下几节演示了如何执行此操作。

### 1、侦听双栈（IPv4和IPv6）接口
以下示例演示了如何配置RabbitMQ以仅在本地主机上侦听IPv4和IPv6：
```sh
listeners.tcp.1 = 127.0.0.1:5672
listeners.tcp.2 = ::1:5672
```
在现代Linux内核和Windows发行版中，当指定端口并将RabbitMQ配置为侦听所有IPv6地址但未明确禁用IPv4时，将包括IPv4地址。
```sh
listeners.tcp.1 = :::5672
```
等同于
```sh
listeners.tcp.1 = 0.0.0.0:5672
listeners.tcp.2 = :::5672
```

### 2、只监听IPv6接口
在此示例中，RabbitMQ将仅在IPv6接口上侦听：
```sh
listeners.tcp.1 = fe80::2acf:e9ff:fe17:f97b:5672
```
在仅IPv6的环境中，还必须将节点配置为使用IPv6进行节点间通信和CLI工具连接。

### 3、只监听IPv4接口
在此示例中，RabbitMQ将仅在IPv6接口上侦听：
```sh
listeners.tcp.1 = 192.168.1.99:5672
```
可以通过禁用所有常规TCP侦听器来禁用非TLS连接。只有启用了TLS的客户端才能连接：
```sh
# disables non-TLS listeners, only TLS-enabled clients will be able to connect
listeners.tcp = none

listeners.ssl.default = 5671

ssl_options.cacertfile = /path/to/ca_certificate.pem
ssl_options.certfile   = /path/to/server_certificate.pem
ssl_options.keyfile    = /path/to/server_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false
```

## epmd
epmd（用于Erlang端口映射守护程序）是一个小的附加守护程序，它与每个RabbitMQ节点一起运行，运行时使用它来发现特定节点侦听哪个端口以进行节点间通信。然后，对等节点和CLI工具将使用该端口。
当节点或CLI工具需要联系节点rabbit@hostname2时，它将执行以下操作：
* 使用标准操作系统解析程序或inetrc文件中指定的自定义解析程序将hostname2解析为IPv4或IPv6地址
* 使用上述地址联系在hostname2上运行的epmd
* 向epmd询问节点rabbit使用的端口
* 使用解析的IP地址和发现的端口连接到节点
* 开始进行交互

### 1、epmd绑定接口
epmd默认会在所有接口上监听。可以使用ERL_EPMD_ADDRESS环境变量将其限制为多个接口：
```sh
# makes epmd listen on loopback IPv6 and IPv4 interfaces
export ERL_EPMD_ADDRESS="::1"
```
更改ERL_EPMD_ADDRESS时，必须停止主机上的RabbitMQ节点和epmd。对于epmd，请使用
```sh
# Stops local epmd process.
# Use after shutting down RabbitMQ.
epmd -kill
```
终止它。该服务将由本地RabbitMQ节点在启动时自动启动。
回环接口将隐式添加到该列表（换句话说，epmd将始终绑定到回环接口）。

### 2、epmd绑定端口
缺省epmd端口为4369，但可以使用ERL_EPMD_PORT环境变量来更改此端口：
```sh
# makes epmd bind to port 4369
export ERL_EPMD_PORT="4369"
```
群集中的所有主机必须使用相同的端口。

更改ERL_EPMD_PORT时，必须停止主机上的RabbitMQ节点和epmd。对于epmd，请使用
```sh
# Stops local epmd process.
# Use after shutting down RabbitMQ.
epmd -kill
```
终止它。该服务将由本地RabbitMQ节点在启动时自动启动。

## 节点间通讯端口范围
RabbitMQ节点将使用某个范围内的端口，该范围称为节点间通信端口范围。 CLI工具需要联系节点时使用相同的端口。范围可以修改。

RabbitMQ节点使用称为distribution端口的端口与CLI工具和其他节点通信。它是从一系列值中动态分配的。对于RabbitMQ，默认范围限于通过RABBITMQ_NODE_PORT（AMQP 0-9-1和AMQP 1.0端口）+ 20000计算得出的单个值，这导致使用端口25672。可以使用RABBITMQ_DIST_PORT环境变量来配置此单个端口。

RabbitMQ命令行工具也使用一系列端口。通过获取RabbitMQ distribution端口值并将其加上10000，可以计算出默认范围。接下来的10个端口也属于此范围。因此，默认情况下，此范围是35672至35682。可以使用RABBITMQ_CTL_DIST_PORT_MIN和RABBITMQ_CTL_DIST_PORT_MAX环境变量来配置此范围。请注意，将范围限制为单个端口将阻止多个CLI工具在同一主机上同时运行，并且可能会影响需要并行连接到多个集群节点的CLI命令。因此，建议端口范围为10。

配置防火墙规则时，强烈建议允许每个群集成员和每个可能使用CLI工具的主机在节点间通信端口上进行远程连接。必须打开epmd端口，CLI工具和群集才能起作用。

RabbitMQ使用的范围也可以通过两个配置键来控制：
* 仅经典配置格式的kernel.inet_dist_listen_min
* 仅经典配置格式的kernel.inet_dist_listen_max
它们定义了范围的上下限（包括上下限）。

下面的示例使用具有单个端口的范围，但其值不同于默认值：
```sh
[
  {kernel, [
    {inet_dist_listen_min, 33672},
    {inet_dist_listen_max, 33672}
  ]},
  {rabbit, [
    ...
  ]}
].
```
要验证节点使用哪个端口进行节点间和CLI工具通信，请运行
```sh
epmd -names
```
在该节点的主机上。它将产生如下所示的输出：
```sh
epmd: up and running on port 4369 with data:
name rabbit at port 25672
```

## 节点间通信缓冲区大小限制
节点间连接使用缓冲区存储待发送的数据。当缓冲区达到最大允许容量时，将对节点间流量进行临时限制。通过RABBITMQ_DISTRIBUTION_BUFFER_SIZE环境变量（以千字节为单位）控制该限制。默认值为128 MB（128000 kB）。

在节点间流量较大的群集中，此值可能会对吞吐量产生积极影响。不建议使用低于64 MB的值。

## 使用IPv6进行节点间通信（和CLI工具）
除了专用于客户端连接的IPv6用于客户端连接之外，还可以将节点配置为专用于将IPv6用于节点间和CLI工具的连接。

这涉及一些地方的配置：
* 运行时的节点间通信协议设置
* 配置CLI工具要使用的IPv6
* epmd，涉及节点间通信（发现）的服务

可以将IPv6用于节点间和CLI工具通信，但可以将IPv4用于客户端连接，反之亦然。这样的配置可能很难进行故障排除和推理，因此建议全盘使用相同的IP版本（例如IPv6）或双堆栈设置。

### 1、节点间通信协议
要指示运行时使用IPv6进行节点间通信和相关任务，请使用RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS环境变量传递几个标志：
```sh
# these flags will be used by RabbitMQ nodes
RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="-kernel inetrc '/etc/rabbitmq/erl_inetrc' -proto_dist inet6_tcp"
# these flags will be used by CLI tools
RABBITMQ_CTL_ERL_ARGS="-proto_dist inet6_tcp"
```
上面的RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS使用两个紧密相关的标志：
* -kernel inetrc: 来配置控制主机名解析的inetrc文件的路径
* -proto_dist inet6_tcp: 告诉节点在连接到对等节点并侦听CLI工具连接时使用IPv6
/etc/rabbitmq/erl_inetrc中的erl_inetrc文件将控制主机名解析设置。对于仅IPv6的环境，它必须包含以下行：
```sh
%% Tells DNS client on RabbitMQ nodes and CLI tools to resolve hostnames to IPv6 addresses.
%% The trailing dot is not optional.
{inet6,true}.
```

### 2、CLI工具使用IPv6
使用CLI工具，使用与上面的RabbitMQ节点相同的运行时标志，但使用不同的环境变量RABBITMQ_CTL_ERL_ARGS来提供它：
```sh
RABBITMQ_CTL_ERL_ARGS="-proto_dist inet6_tcp"
```
请注意，一旦指示使用IPv6，CLI工具将无法连接到不使用IPv6进行节点间通信的节点。这涉及到与目标RabbitMQ节点在同一主机上运行的epmd服务。

### 3、epmd使用IP
epmd是一个小型帮助程序守护程序，它在RabbitMQ节点旁边运行，并允许其对等方和CLI工具发现应使用哪个端口与之通信。可以将其配置为绑定到特定接口，就像RabbitMQ侦听器一样。这是使用ERL_EPMD_ADDRESS环境变量完成的：
```sh
export ERL_EPMD_ADDRESS="::1"
```

在使用systemd的发行版上，epmd.socket服务控制epmd的网络设置。可以将epmd配置为仅侦听IPv6接口：
```sh
ListenStream=[::1]:4369
```

https://www.rabbitmq.com/networking.html