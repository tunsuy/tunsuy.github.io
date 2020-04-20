# rabbitmq网络分区

RabbitMQ的模型类似交换机模型，且采用erlang这种电信网络方面的专用语言实现。RabbitMQ集群是不能跨LAN部署（如果要WAN部署需要采用专门的插件）的，也就是基于网络情况良好的前提下运行的。这种假设就好比paxos并不解决拜占庭问题。
为什么RabbitMQ需要这种前提假设？这个它本身的数据一致性复制原理有关。RabbitMQ采用的镜像队列是一种环形的逻辑结构.

RabbitMQ除了发布（Publish）消息之外，所有的其余操作都是在master上完成，之后再将有影响的操作同步到slave节点上。如果客户端连接的是slave节点，RabbitMQ机制也会先将链接路由到master节点上。比如确认（Ack）一条消息，先在A节点上，即master节点上确认，之后再转向B节点，进而是C和D节点，最后再D返回Ack之后才真正将这条消息确认，进而标记为可删除。这个种复制原理和zookeeper的quorum原理不同，它可以保证更强的数据一致性。在这种一致性模型下，如果出现网络波动或者网络延迟等，那么整个复制链的性能就会下降。就以上图为例，如果C节点网络异常，那么整个A->B->C->D->A的循环复制过程就会大受影响，整个RabbitMQ服务性能将大打折扣，所以这里就需要引入网络分区来将异常的节点排离出整个分区之外，以确保整个RabbitMQ的性能。待网络情况转好之后再将此节点加入集群之中。

## 网络分区的判定
RabbitMQ中与网络分区的判定相关的是net_ticktime这个参数，默认为60s。在RabbitMQ集群中的每个broker节点会每隔 net_ticktime/4 (默认15s)计一次tick（如果有任何数据被写入节点中，此节点被认为被ticked），如果在连续四次某节点都没有被ticked到，则判定此节点处于down的状态，其余节点可以将此节点剥离出当前分区。将连续四次的tick时间即为T，那么T的取值范围为 0.75ticktime < T < 1.25ticktime。下图可以形象的描述出这个取值范围的原因。（每个节点代表一次tick判定的timestamp,在两个临界值的情况下会有4个tick的判定）

默认情况下，在45s<T<75s之间会判定出网络分区。

RabbitMQ会将queues，exchanges，bindings等信息存储在Erlang的分布式数据库——Mnesia中，许多围绕网络分区的一些细节都和这个Mnesia的行为有关。如果一个节点不能在T时间内连上另一个节点（这里的连上特指broker节点之间的内部通信），那么Mnesia通常

当一个节点起来的时候，RabbitMQ会记录是否发生了网络分区，你可以通过WebUI进行查看；或者可以通过rabbitmqctl cluster_status命令查看，如果查看到信息中的partitions那一项是空的，就想这样：
```sh
[{nodes,[{disc,['rabbit@node1', 'rabbit@node2']}]},
 {running_nodes,['rabbit@node2','rabbit@node1']},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,[]}]
```

然而当网络分区时，会变成这样：
```sh
[{nodes,  [{disc,  ['rabbit@node1','rabbit@node2']}]},
 {running_nodes,['rabbit@node1']},
 {cluster_name,<<"rabbit@node1">>},
 {partitions,  [{'rabbit@node1',['rabbit@node2']}]}]
```

当一个RabbitMQ集群发生网络分区时，这个集群会分成两个或者多个分区，它们各自为政，互相都认为对方分区的节点已经down,包括queues，bindings,exchanges这些信息的创建和销毁都处于自身分区内，与其它分区无关。如果原集群中配置了镜像队列，而这个镜像队列又牵涉到两个或者多个网络分区中的节点时，每一个网络分区中都会出现一个master节点，如果分区节点个数充足，也会出现新的slave节点，对于各个网络分区，彼此的队列都是相互独立的，当然也会有一些其他未知的、怪异的事情发生。当网络恢复时，网络分区的状态还是会保持，除非采取一些措施去解决他。

## 手动处理网络分区
为了从网络分区中恢复，首先需要挑选一个信任的分区，这个分区才有决定Mnesia内容的权限，发生在其他分区的改变将不被记录到Mnesia中而直接丢弃。手动恢复网络分区有两种思路：

停止其他分区中的节点，然后重新启动这些节点。最后重启信任分区中的节点，以去除告警。
关闭整个集群的节点，然后再启动每一个节点，这里需确保你启动的第一个节点在你所信任的分区之中。
停止/启动节点有两种操作方式：
```sh
rabbimqctl stop/ rabbitmq-server -detached
rabbitmqctl stop_app/ rabbitmqctl start_app
```

## 自动处理网络分区
RabbitMQ提供了4种处理网络分区的方式，在rabbitmq.config中配置cluster_partition_handling参数即可，分别为：
```sh
ignore
pause_minority
pause_if_all_down, [nodes], ignore|autoheal
autoheal
```

### ignore
```sh
[
        {
                rabbit, [
                        {cluster_partition_handling, ignore}
                ]

        }
 ].
```
默认是ignore，ignore的配置是当网络分区的时候，RabbitMQ不会自动做任何处理，即需要手动处理。

### pause_minority
```sh
[
        {
                rabbit, [
                        {cluster_partition_handling, pause_minority}
                ]
        }
 ].
```
当发生网络分区时，集群中的节点在观察到某些节点down掉时，会自动检测其自身是否处于少数派（小于或者等于集群中一般的节点数）。少数派中的节点在分区发生时会自动关闭，当分区结束时又会启动。这里的关闭是指RabbitMQ application关闭，而Erlang VM并不关闭，这个类似于执行了rabbitmqctl stop_app命令。处于关闭的节点会每秒检测一次是否可连通到剩余集群中，如果可以则启动自身的应用，相当于执行rabbitmqctl start_app命令。

需要注意的是RabbitMQ也会关闭不是严格意义上的大多数。比如在一个集群中只有两个节点的时候并不适合采用pause-minority模式，因为由于其中任何一个节点失败而发生网络分区时，两个节点都会被关闭。当网络恢复时，有可能两个节点会自动启动恢复网络分区，也有可能还是保持关闭状态。然而如果集群中的节点远大于两个时，pause_minority模式比ignore模式更加的可靠，特别是网络分区通常是由于单个节点网络故障而脱离原有分区引起的。不过也需要考虑2v2, 3v3这种情况，可能会引起所有集群节点的关闭。这种处理方式适合集群节点数大于2个且最好为奇数的情况。

### pause_if_all_down
```sh
[
        {
                rabbit, [
                        {cluster_partition_handling, {pause_if_all_down,  ['rabbit@node1'], ignore}}
                ]
        }
 ].
```
或者
```sh
[
        {
                rabbit, [
                        {cluster_partition_handling, {pause_if_all_down,  ['rabbit@node1'], autoheal}}
                ]
        }
 ].
```
在pause_if_all_down模式下，RabbitMQ会自动关闭不能和list中节点通信的节点。语法为{pause_if_all_down, [nodes], ignore|autoheal}，其中[nodes]即为前面所说的list。如果一个节点与list中的所有节点都无法通信时，自关闭其自身。如果list中的所有节点都down时，其余节点如果是ok的话，也会根据这个规则去关闭其自身，此时集群中所有的节点会关闭。如果某节点能够与list中的节点恢复通信，那么会启动其自身的RabbitMQ应用，慢慢的集群可以恢复。

为什么这里会有ignore和autoheal两种不同的配置，考虑这样一种情况：有两个节点node1和node2在机架A上，node3和node4在机架B上，此时机架A和机架B的通信出现异常，如果此时使用pause-minority的话会关闭所有的节点，如果此时采用pause-if-all-down，list中配置成[‘node1’, ‘node3’]的话，集群中的4个节点都不会关闭，但是会形成两个分区，此时就需要ignore或者autoheal来指引如何处理此种分区的情形。

### autoheal
```sh
[
        {
                rabbit, [
                        {cluster_partition_handling, autoheal}
                ]
        }
 ].
```
在autoheal模式下，当认为发生网络分区时，RabbitMQ会自动决定一个获胜的（winning）分区，然后重启不在这个分区中的节点以恢复网络分区。一个获胜的分区是指客户端连接最多的一个分区。如果产生一个平局，既有两个或者多个分区的客户端连接数一样多，那么节点数最多的一个分区就是获胜的分区。如果此时节点数也一样多，将会以一种特殊的方式来挑选获胜分区。