# rabbitmq札记
## 启动脚本
/etc/rabbitmq/rabbitmq-env.conf 文件包含变量设置，可以覆盖默认的内置RabbitMQ启动脚本中的默认变量值。
该文件被系统shell解释，所以应该由一系列的shell环境变量组成。普通的shell语法是允许的（因为该文件是使用shell操作符 "." 来执行的），包括使用 "#" 开头的行注释。
/usr/local/lib/rabbitmq/sbin/rabbitmq-server
```sh
set -e

# Get default settings with user overrides for (RABBITMQ_)<var_name>
# Non-empty defaults should be set in rabbitmq-env
. `dirname $0`/rabbitmq-env
```

启动脚本获取变量的值，优先从环境变量获取，其次是文件 /etc/rabbitmq/rabbitmq-env.conf，最后是从内置默认值中获取。例如对于 RABBITMQ_NODENAME 变量设置，
首先从环境中检查 RABBITMQ_NODENAME，如果不存在或者等于一个空字符串，然后从 /etc/rabbitmq/rabbitmq-env.conf文件中检查 NODENAME，如果也不存在或者等于一个空字符串，就使用从启动脚本中的默认值。
该文件中的变量名称，总是等于去掉了 RABBITMQ_ 这个前缀的环境变量名称。例如来自环境中的变量 RABBITMQ_NODE_PORT在该文件中就成为了 NODE_PORT(实际上也可以带上RABBITMQ_前缀)。

## 服务启动
1、非守护进程启动
```sh
$RABBITMQ_HOME/sbin/rabbitmq-server
```

2、以守护进程启动
```sh
$RABBITMQ_HOME/sbin/rabbitmq-server -detached
```

如果成功的话，它会显示类似以下的信息:
```sh
completed with [n] plugins
```

3、停止
```sh
$RABBITMQ_HOME/sbin/rabbitmqctl stop
```

4、查看状态
```sh
$RABBITMQ_HOME/sbin/rabbitmqctl status
```

## rabbitmq-env.conf
rabbitmq-env.conf
```sh
export ERL_EPMD_ADDRESS=30.1.3.41
RABBITMQ_NODE_IP_ADDRESS=30.1.3.41
RABBITMQ_NODE_PORT=5671
RABBITMQ_LOG_BASE=/var/log/rabbitmq
RABBITMQ_MNESIA_BASE=/usr/local/lib/rabbitmq/var/lib/rabbitmq/mnesia
RABBITMQ_NODENAME=rabbit@mq301341
IO_THREAD_POOL_SIZE=128
RABBITMQ_MAX_NUMBER_OF_PROCESSES=2048000
CTL_ERL_ARGS="-setcookie rabbitmq_server_cookie"
```

ERL_EPMD_ADDRESS: epmd使用的接口，它是节点间和CLI工具通信中的组件。
默认值：所有可用接口，IPv6和IPv4。

RABBITMQ_NODE_IP_ADDRESS: 如果只想绑定到一个网络接口，请更改此设置。可以在配置文件中设置绑定到两个或多个接口。
默认值：一个空字符串，表示“绑定到所有网络接口”。

RABBITMQ_MNESIA_BASE: 该目录包含RabbitMQ服务器的节点数据库，消息存储和集群状态文件的子目录，除非明确设置了RABBITMQ_MNESIA_DIR。通常有效的RabbitMQ用户必须有足够的权限随时读取，写入和创建此目录中的文件和子目录，这一点很重要。通常不会覆盖此变量。通常会改写RABBITMQ_MNESIA_DIR。

RABBITMQ_MNESIA_DIR：该RabbitMQ节点的数据存储目录。这包括模式数据库，消息存储，集群成员信息和其他持久节点状态。

RABBITMQ_NODENAME：每个Erlang-node-and-machine组合的节点名称应该唯一。
默认：rabbit@$HOSTNAME

RABBITMQ_IO_THREAD_POOL_SIZE：运行时用于I/O的线程数。不建议使用低于32的值。
默认值：128（Linux），64（Windows）

## 主机名解析要求
RabbitMQ节点彼此之间使用域名，要么是简短的，要么是全限定的(FQDNs). 因此，集群中所有成员的主机名都必须是可解析的，也可用于机器上的命令行工具，如rabbitmqctl.

主机名解析可使用任何一种标准的操作系统提供方法:
* DNS 记录
* 本地主机文件(e.g. /etc/hosts)
  在更加严格的环境中，DNS记录或主机文件修改是受限的,不可能的或不受欢迎的, Erlang VM可通过使用替代主机名解析方法来配置, 如一个替代的DNS服务器,一个本地文件，一个非标准的主机文件位置或一个混合方法. 这些方法可以与标准操作主机名解析方法一起协同工作。
```sh
[root@karbor1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
30.1.3.41 mq301341
30.1.3.42 mq301342
```

## 集群

### 1、集群构成
集群可以通过多种方式来构建:
* 手动使用rabbitmqctl
* 通过在配置文件列举集群节点来声明
* 通过使用rabbitmq-autocluster (插件)来声明
* 使用rabbitmq-clusterer（插件）来声明

一个集群的构成可以动态修改. 所有RabbitMQ brokers开始都是以单个节点来运行的. 这些节点可以加入到集群中, 随后也可以脱离集群再次成为单一节点。
加入集群会隐式地重置节点, 因此这会删除此节点上先前存在的所有资源和数据.

这里有一些重要的警告:
当整个集群崩溃的时候， 最后一个崩溃的节点必须第一个上线.如果不是这样，节点将会等待最后一个磁盘节点30秒以确认其重新上线，否则就会失败. 如果最后一个下线的节点，不能再重新上线，那么它可能会使用forget_cluster_node命令来从集群中删除 - 查阅 rabbitmqctl页面来了解更多信息.
如果所有集群节点都在同一个时间内停止且不受控制（如断电）。在这种情况下，你可以在某个节点上使用force_boot命令使其再次成为可启动的－查阅 rabbitmqctl页面来了解更多信息.

### 2、脱离集群
当节点不再是集群的一部分时，可以明确地将其从集群中删除. 首先我们将节点rabbit@rabbit3从集群中删除, 以使其回归独立操作.要做到这一点，需要在rabbit@rabbit3节点上停止RabbitMQ 应用程序,重设节点，并重启RabbitMQ应用程序.
我们也可以远程地删除节点，这是相当有用的，举例来说，当处理无反应的节点时.举例来说，我们可以从 rabbit@rabbit2中删除rabbit@rabbi1.
```sh
rabbitmqctl forget_cluster_node rabbit@rabbit1 
```

### 3、节点(以及CLI工具)之间如何来认证: Erlang Cookie
RabbitMQ 节点和CLI 工具(如rabbitmqctl) 使用cookie来确定每个节点之间是否可以通信. 两个节点之间要能通信，它们必须要有相同的共享密钥Erlang cookie. cookie只是具有字母数字特征的字符串。只要你喜欢，它可长可短. 每个集群节点必须有相同的cookie.

当RabbitMQ 服务器启动时，Erlang VM 会自动地创建一个随机的cookie文件. 最简单的处理方式是允许一个节点来创建文件，然后再将这个文件拷贝到集群的其它节点中。
在 Unix 系统中, cookie的通常位于/var/lib/rabbitmq/.erlang.cookie 或$HOME/.erlang.cookie.
作为替代方案，你可以在 rabbitmq-server 和 rabbitmqctl 脚本中调用erl时,插入"-setcookie cookie"选项.

当cookie未配置时 (例如，不相同), RabbitMQ 会记录这样的错误"Connection attempt from disallowed node" and "Could not auto-cluster".

### 4、怎样识别rabbitmq节点
RabbitMQ节点由节点名称标识。节点名称由两部分组成，前缀（通常是Rabbit）和主机名。例如，rabbit@node1.messaging.svc.local是一个节点名称，其前缀为Rabbit，主机名为node1.messaging.svc.local。
集群中的节点名称必须唯一。如果在给定主机上运行多个节点（在开发和QA环境中通常是这种情况），则它们必须使用不同的前缀，例如rabbit1@hostname和rabbit2@hostname。
在集群中，节点使用节点名称进行标识并相互联系。这意味着必须解析每个节点名称的主机名部分。 CLI工具还使用节点名称标识和寻址节点。
节点启动时，它将检查是否已为其分配了节点名称。这是通过RABBITMQ_NODENAME环境变量完成的。如果未显式配置任何值，则节点解析其主机名，并在其前面添加Rabbit以计算其节点名。

### 5、数据怎么复制
所有的data/state在节点间都会同步，除了消息队列。
消息队列默认情况下只存在一个节点上，尽管它们在所有节点上都是可见且可访问的。要在群集中的节点之间复制队列，请参阅有关高可用性的文档（注意：本指南是镜像的先决条件）。

一些分布式系统具有领导者和跟随者节点。对于RabbitMQ，集群中的所有节点都是对等的：RabbitMQ中没有特殊的节点。当考虑队列镜像和插件时，会有一些细微差别，但出于大多数目的，应将所有群集节点视为相等
可以针对任何节点执行许多CLI工具操作。 HTTP API客户端可以针对任何节点执行。

假设所有群集成员均可用，则客户端可以连接到任何节点并执行任何操作。节点会将操作路由到队列主节点，这对client是透明的。
使用所有受支持的消息传递协议，客户端一次只能连接到一个节点。

### 6、集群间怎样认证
RabbitMQ节点和CLI工具（例如Rabbitmqctl）使用cookie来确定是否允许它们彼此通信。为了使两个节点能够通信，它们必须具有称为Erlang cookie的相同共享密钥。 Cookie只是一串字母数字字符，最大长度为255个字符。它通常存储在本地文件中。必须只能所有者能够访问该文件（例如，具有UNIX权限600或类似权限）。每个群集节点必须具有相同的cookie。
如果该文件不存在，则在RabbitMQ服务器启动时，Erlang VM将尝试使用随机生成的值创建一个文件。这仅在开发环境中是可行的，因为由于每个节点将独立产生自己的cookie，因此这将导致无法组成集群。
Erlang cookie的生成应在集群部署阶段完成，理想情况下使用自动化和编排工具，例如Chef，Puppet，BOSH，Docker或类似工具。

### 7、节点的数量
由于若干功能（例如仲裁队列，MQTT中的客户端跟踪）需要集群成员之间达成共识，因此强烈建议奇数个集群节点：1、3、5、7等。
强烈建议不要使用两个节点的群集，因为群集节点无法确定多数节点，这就导致无法在失去连接的情况下达成共识。例如，当两个节点失去连接时，将不接受MQTT客户端连接，仲裁队列将失去其可用性，依此类推。
从共识的角度来看，四个或六个节点群集将具有与三个和五个节点群集相同的可用性特征，因此会浪费一个节点。

## 备份与恢复
### 1、数据
每个RabbitMQ节点都有一个数据目录，该目录存储该节点上的所有信息。
数据目录包含两种类型的数据：定义类（metadata, schema/topology）和消息存储类数据。
节点和集群总存储能够表示架构，元数据或拓扑的信息。 Users, vhosts, queues, exchanges, bindings, 运行时参数都属于此类。

可以通过HTTP API，CLI工具以及客户端库（应用程序）来导出和导入定义类数据。
定义存储在内部数据库中，并在所有群集节点之间复制。集群中的每个节点都有其自己的所有定义副本。当定义的一部分更改时，所有节点将在单个事务中执行更新。在备份的上下文中，这意味着在实践中可以从任何群集节点导出定义而获得相同的结果。

消息存放在专门的消息存储中，这是一个对用户透明的实体。

每个节点都有自己的数据目录，并存储其主节点托管在该节点上的队列的消息。可以使用队列镜像在节点之间复制消息。消息存储在节点数据目录的子目录中。

### 2、生命周期
定义通常大多是静态的，而消息却不断地从发布者流向消费者。
执行备份时，第一步是确定是仅备份定义还是备份消息存储。因为消息通常是短暂的并且可能是瞬态的，所以强烈建议不要从运行中的节点下备份消息，并且可能导致数据快照不一致。
只能从正在运行的节点上备份定义。

下面只介绍手动备份定义的情况：
在一个运行的rabbitmq节点上执行命令：
```sh
root@ts / > rabbitmqctl eval 'rabbit_mnesia:dir().'
warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
"/usr/local/lib/rabbitmq/var/lib/rabbitmq/mnesia/rabbit@mq303113"
```
上面的数据目录的子目录中还包含消息存储数据。如果不想备份消息，则跳过拷贝消息目录。