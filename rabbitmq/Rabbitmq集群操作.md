# Rabbitmq集群操作

## 开启独立节点
通过将现有RabbitMQ节点重新配置为集群配置来建立集群。因此，第一步是以正常方式在所有节点上启动RabbitMQ：
```sh
# on rabbit1
rabbitmq-server -detached
# on rabbit2
rabbitmq-server -detached
# on rabbit3
rabbitmq-server -detached
```

这将创建三个独立的RabbitMQ代理，每个节点上一个，由cluster_status命令确认：
```sh
# on rabbit1
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1]}]},{running_nodes,[rabbit@rabbit1]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit2]}]},{running_nodes,[rabbit@rabbit2]}]
# => ...done.

# on rabbit3
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit3 ...
# => [{nodes,[{disc,[rabbit@rabbit3]}]},{running_nodes,[rabbit@rabbit3]}]
# => ...done.
```

## 创建集群
为了链接集群中的三个节点，我们告诉其中两个节点，例如`rabbit@rabbit2`和`rabbit@rabbit3`，加入第三个节点，即`rabbit@rabbit1`。在此之前，必须重新设置两个新加入的成员。

我们首先将`rabbit@rabbit2`与`rabbit@rabbit1`一起加入集群。为此，在`rabbit@rabbit2`上，我们停止RabbitMQ应用程序并加入`rabbit@rabbit1`集群，然后重新启动RabbitMQ应用程序。请注意，必须先重置节点才能加入现有集群。重置节点将删除该节点上先前存在的所有资源和数据。这意味着节点不能同时成为集群的成员并且保留其现有数据。
```sh
# on rabbit2
rabbitmqctl stop_app
# => Stopping node rabbit@rabbit2 ...done.

rabbitmqctl reset
# => Resetting node rabbit@rabbit2 ...

rabbitmqctl join_cluster rabbit@rabbit1
# => Clustering node rabbit@rabbit2 with [rabbit@rabbit1] ...done.

rabbitmqctl start_app
# => Starting node rabbit@rabbit2 ...done.
```

通过在任一节点上运行cluster_status命令，可以看到两个节点已加入集群：
```sh
# on rabbit1
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
# =>  {running_nodes,[rabbit@rabbit2,rabbit@rabbit1]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
# =>  {running_nodes,[rabbit@rabbit1,rabbit@rabbit2]}]
# => ...done.
```

现在，我们将`rabbit@rabbit3`加入同一集群。这些步骤与上面的步骤相同，除了这次我们将集群到rabbit2以证明选择集群的节点无关紧要-提供一个在线节点就足够了，并且该节点将集群到该集群。
```sh
# on rabbit3
rabbitmqctl stop_app
# => Stopping node rabbit@rabbit3 ...done.

# on rabbit3
rabbitmqctl reset
# => Resetting node rabbit@rabbit3 ...

rabbitmqctl join_cluster rabbit@rabbit2
# => Clustering node rabbit@rabbit3 with rabbit@rabbit2 ...done.

rabbitmqctl start_app
# => Starting node rabbit@rabbit3 ...done.
```

通过在任何节点上运行cluster_status命令，我们可以看到三个节点已加入集群：
```sh
# on rabbit1
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit3,rabbit@rabbit2,rabbit@rabbit1]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit3,rabbit@rabbit1,rabbit@rabbit2]}]
# => ...done.

# on rabbit3
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit3 ...
# => [{nodes,[{disc,[rabbit@rabbit3,rabbit@rabbit2,rabbit@rabbit1]}]},
# =>  {running_nodes,[rabbit@rabbit2,rabbit@rabbit1,rabbit@rabbit3]}]
# => ...done.
```

通过执行上述步骤，我们可以在集群运行时随时将新节点添加到集群中。

## 重启节点
可以随时停止已加入集群的节点。它们也可能失败或被操作系统终止。在所有情况下，集群的其余部分都可以继续运行，并且节点在再次启动时会自动与其他集群节点“同步”（同步）。请注意，某些分区处理策略可能会有所不同，并影响其他节点。
我们关闭节点`rabbit@rabbit1`和`rabbit@rabbit3`，并在每个步骤中检查集群状态：
```sh
# on rabbit1
rabbitmqctl stop
# => Stopping and halting node rabbit@rabbit1 ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit3,rabbit@rabbit2]}]
# => ...done.

# on rabbit3
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit3 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit2,rabbit@rabbit3]}]
# => ...done.

# on rabbit3
rabbitmqctl stop
# => Stopping and halting node rabbit@rabbit3 ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit2]}]
# => ...done.
```

现在，我们再次启动节点，并检查集群状态：
```sh
# on rabbit1
rabbitmq-server -detached
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit2,rabbit@rabbit1]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit1,rabbit@rabbit2]}]
# => ...done.

# on rabbit3
rabbitmq-server -detached

# on rabbit1
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit2,rabbit@rabbit1,rabbit@rabbit3]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]
# => ...done.

# on rabbit3
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit3 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2,rabbit@rabbit3]}]},
# =>  {running_nodes,[rabbit@rabbit2,rabbit@rabbit1,rabbit@rabbit3]}]
# => ...done.
```

重要的是要了解流程节点在停止和重新启动时经历的过程。

停止节点选择一个集群成员（将仅考虑磁盘节点）以在重新启动后与之同步。重新启动后，默认情况下，该节点将尝试与集群对方联系10次，每次响应超时为30秒。如果对端在该时间间隔内可用，则该节点将成功启动，同步对端的信息并继续运行。如果对等方不可用，则重新启动的节点将放弃并自愿停止。

当节点在关闭过程中没有集群节点在线时，它将在不尝试与任何已知对端同步的情况下启动。它将等待对端重新加入它。

因此，当整个群集关闭时，最后一个关闭的节点是在关闭时唯一没有任何正在运行的对等节点的节点。该节点可以启动而无需先联系任何对等节点。由于节点将尝试与已知对等方联系最多5分钟（默认情况下），因此可以在该时间段内以任何顺序重新启动节点。在这种情况下，他们将成功地彼此重新加入。可以使用两种配置设置来调整此时间窗口：
```sh
# wait for 60 seconds instead of 30
mnesia_table_loading_retry_timeout = 60000

# retry 15 times instead of 10
mnesia_table_loading_retry_limit = 15
```
通过调整这些设置并调整必须返回已知对等方的时间窗口，可以解决可能需要超过5分钟才能完成的群集范围内的重新部署方案。
升级期间，有时最后一个要停止的节点必须是升级后要启动的第一个节点。该节点将被指定执行集群范围的架构迁移，其他节点可以在它们重新加入时从中进行同步并应用。

在某些情况下，无法恢复最后一个脱机节点。您可以使用`forget_cluster_node`

另外，可以在节点上使用force_boot rabbitmqctl命令使其引导，而无需尝试与任何对等节点同步（就像它们最后一次关闭一样）。仅当最后一个要关闭的节点或一组节点永远不会重新联机时，才通常需要这样做。

## 分解集群
有时有必要从集群中删除节点。操作员必须使用rabbitmqctl命令明确地执行此操作。
一些对等发现机制支持节点运行状况检查和强制删除发现后端未知的节点。该功能已启用（默认情况下处于禁用状态）

我们首先从集群中移除`rabbit@rabbit3`，使其成为独立的节点。步骤如下：
```sh
# on rabbit3
rabbitmqctl stop_app
# => Stopping node rabbit@rabbit3 ...done.

rabbitmqctl reset
# => Resetting node rabbit@rabbit3 ...done.

rabbitmqctl start_app
# => Starting node rabbit@rabbit3 ...done.
```

在节点上运行`cluster_status`命令可确认`rabbit@rabbit3`现在不再是集群的一部分，并且可以独立运行：
```sh
# on rabbit1
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
# => {running_nodes,[rabbit@rabbit2,rabbit@rabbit1]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit1,rabbit@rabbit2]}]},
# =>  {running_nodes,[rabbit@rabbit1,rabbit@rabbit2]}]
# => ...done.

# on rabbit3
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit3 ...
# => [{nodes,[{disc,[rabbit@rabbit3]}]},{running_nodes,[rabbit@rabbit3]}]
# => ...done.
```

们还可以远程删除节点。例如，在必须处理无响应的节点时，这很有用。例如，我们可以从`rabbit@rabbit2`删除`rabbit@rabbi1`。
```sh
# on rabbit1
rabbitmqctl stop_app
# => Stopping node rabbit@rabbit1 ...done.

# on rabbit2
rabbitmqctl forget_cluster_node rabbit@rabbit1
# => Removing node rabbit@rabbit1 from cluster ...
# => ...done.
```

请注意，rabbit1仍然认为其与Rabbit2群集在一起，尝试启动它会导致错误。我们将需要重置它以便能够再次启动它。
```sh
# on rabbit1
rabbitmqctl start_app
# => Starting node rabbit@rabbit1 ...
# => Error: inconsistent_cluster: Node rabbit@rabbit1 thinks it's clustered with node rabbit@rabbit2, but rabbit@rabbit2 disagrees

rabbitmqctl reset
# => Resetting node rabbit@rabbit1 ...done.

rabbitmqctl start_app
# => Starting node rabbit@rabbit1 ...
# => ...done.
```


现在，cluster_status命令显示所有三个作为独立RabbitMQ代理运行的节点：
```sh
# on rabbit1
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit1 ...
# => [{nodes,[{disc,[rabbit@rabbit1]}]},{running_nodes,[rabbit@rabbit1]}]
# => ...done.

# on rabbit2
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit2 ...
# => [{nodes,[{disc,[rabbit@rabbit2]}]},{running_nodes,[rabbit@rabbit2]}]
# => ...done.

# on rabbit3
rabbitmqctl cluster_status
# => Cluster status of node rabbit@rabbit3 ...
# => [{nodes,[{disc,[rabbit@rabbit3]}]},{running_nodes,[rabbit@rabbit3]}]
# => ...done.
```

注意，`rabbit@rabbit2`保留了集群的剩余状态，而`rabbit@rabbit1`和`rabbit@rabbit3`是刚初始化的RabbitMQ代理。如果要重新初始化`rabbit@rabbit2`，请按照与其他节点相同的步骤进行操作：
```sh
# on rabbit2
rabbitmqctl stop_app
# => Stopping node rabbit@rabbit2 ...done.
rabbitmqctl reset
# => Resetting node rabbit@rabbit2 ...done.
rabbitmqctl start_app
# => Starting node rabbit@rabbit2 ...done.
```


## 重置节点
有时可能需要重置节点（擦除其所有数据），然后使其重新加入群集。一般来说，有两种可能的情况：节点正在运行时以及节点无法启动或无法响应CLI工具命令时，例如由于诸如ERL-430之类的问题。
重置节点将删除其所有数据，群集成员信息，已配置的运行时参数，用户，虚拟主机以及任何其他节点数据。它还将从该群集中永久删除该节点。

要重置运行和响应的节点，请先使用rabbitmqctl stop_app在其上停止RabbitMQ，然后使用rabbitmqctl reset对其进行重置：
```sh
# on rabbit1
rabbitmqctl stop_app
# => Stopping node rabbit@rabbit1 ...done.
rabbitmqctl reset
# => Resetting node rabbit@rabbit1 ...done.
```

对于无响应的节点，必须首先使用任何必要的方法将其停止。对于无法启动的节点，情况已经如此。然后覆盖节点的数据目录位置或[删除]现有的数据存储。这将使该节点开始为空白节点。必须指示它重新加入其原始群集（如果有）。
已重置并重新加入其原始群集的节点将同步所有虚拟主机，用户，权限和拓扑（队列，交换，绑定），运行时参数和策略。如果选择托管副本，它可能会同步镜像队列的内容。节点上的非镜像队列内容将丢失。

不保证在已从对等方同步其架构的重置节点上还原队列数据目录，以确保该数据可用于客户端，因为受影响的队列的队列主控位置可能已更改。

## 单机集群
在某些情况下，在一台机器上运行RabbitMQ节点集群可能会很有用。这对于在台式机或笔记本电脑上进行集群试验是很有用的，而无需为集群启动多个虚拟机。
为了在一台机器上运行多个RabbitMQ节点，必须确保这些节点具有不同的节点名称，数据存储位置，日志文件位置，并绑定到不同的端口，包括插件所使用的端口。请参阅《配置指南》中的RABBITMQ_NODENAME，RABBITMQ_NODE_PORT和RABBITMQ_DIST_PORT，以及《文件和目录位置》指南中的RABBITMQ_MNESIA_DIR，RABBITMQ_CONFIG_FILE和RABBITMQ_LOG_BASE。

您可以通过重复调用rabbitmq-server（在Windows上为rabbitmq-server.bat）来手动在同一主机上启动多个节点。例如
```sh
RABBITMQ_NODE_PORT=5672 RABBITMQ_NODENAME=rabbit rabbitmq-server -detached
RABBITMQ_NODE_PORT=5673 RABBITMQ_NODENAME=hare rabbitmq-server -detached
rabbitmqctl -n hare stop_app
rabbitmqctl -n hare join_cluster rabbit@`hostname -s`
rabbitmqctl -n hare start_app
```
将建立一个两个节点的群集，两个节点都作为磁盘节点。请注意，如果节点侦听AMQP 0-9-1和AMQP 1.0以外的任何端口，则还必须配置这些端口以避免冲突。这可以通过命令行完成：
```sh
RABBITMQ_NODE_PORT=5672 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15672}]" RABBITMQ_NODENAME=rabbit rabbitmq-server -detached
RABBITMQ_NODE_PORT=5673 RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]" RABBITMQ_NODENAME=hare rabbitmq-server -detached
```