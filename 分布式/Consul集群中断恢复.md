## Consul集群中断恢复

这篇文章概述了由于集群中的大多数服务器节点丢失而从Consul中断中恢复的过程。中断类型有几种，具体取决于服务器节点的数量和发生故障的服务器节点的数量。我们将概述如何从以下方法恢复：
* 单个服务器集群发生故障。这是当您只有一台Consul服务器并且失败时。
* 多服务器集群中少数服务器发生故障。
* 多服务器群集中的多台服务器出现故障。

### 单台集群服务器故障
如果只有一台服务器并且发生故障，只需重新启动它即可。单个服务器配置需要`-bootstrap`或`-bootstrap-expect=1`标志。
```sh
consul agent -bootstrap-expect=1
```
如果无法恢复服务器，则需要使用部署指南启动新服务器。

在单个服务器集群中出现不可恢复的服务器故障并且没有备份过程的情况下，由于没有将数据复制到任何其他服务器，因此数据丢失是不可避免的。这就是为什么从不建议部署单个服务器的原因。

当新服务器上线时，由于代理执行反熵，将重新填充在代理中注册的所有服务。


### 少数服务器故障
如果发生故障的服务器是可恢复的，最好的选择是使其恢复联机状态，并使其重新加入具有相同IP地址的几区。这将使群集恢复到完全健康的状态。同样，如果您需要重建新的Consul服务器以替换发生故障的节点，则可能希望立即执行此操作。注意，重建的服务器需要与故障服务器具有相同的IP地址。同样，一旦该服务器联机并重新加入，集群将返回到完全正常的状态。
```sh
consul agent -bootstrap-expect=3 -bind=192.172.2.4 -auto-rejoin=192.172.2.3
```

这两种策略都可能需要很长时间才能重新引导或重建故障服务器。如果这不切实际，或者无法选择使用相同IP构建新服务器，则需要删除发生故障的服务器。通常，如果故障服务器仍然是集群的成员，则可以发出consul force-leave命令来删除发生故障的服务器。
```sh
consul force-leave <node.name.consul>
```
如果consul force-leave无法删除服务器，则有两种方法可以删除它，具体取决于Consul的版本：
* 在Consul 0.7及更高版本中，如果群集中有领导者，则可以使用consul operator命令动态删除陈旧的对等服务器，而不会造成停机。
  在Consul 0.7和更高版本中，可以使用consul operator命令检查Raft配置：
```sh
consul operator raft list-peers

Node     ID              Address         State     Voter RaftProtocol
alice    10.0.1.8:8300   10.0.1.8:8300   follower  true  3
bob      10.0.1.6:8300   10.0.1.6:8300   leader    true  3
carol    10.0.1.7:8300   10.0.1.7:8300   follower  true  3
```
* 在0.7之前的Consul版本中，您可以使用所有其余服务器上的`raft/peers.json`恢复文件手动删除过时的对等服务器，请参见以下有关使用peers.json进行手动恢复的部分。

### 多数服务器故障
如果有多台服务器丢失，导致仲裁丢失和完全中断，则可以使用群集中其余服务器上的数据进行部分恢复。在这种情况下，可能会出现如下两种数据问题：
* 因为剩余节点所提交内容的信息可能不完整，因此导致数据丢失。
* 因为恢复过程会隐式提交所有未完成的Raft日志记录，也就是说会提交在故障之前未提交的数据，因此会造成业务和consul数据不一致。

有关恢复过程的详细信息，请参见以下有关使用peers.json进行手动恢复的部分。您只需在`raft/peers.json`恢复文件中仅包含其余服务器即可。一旦所有其余服务器都使用相同的`raft/peers.json`配置重新启动，集群就应该能够选举领导者

如果需要引入新服务器，使用Consul的join命令将其加入。
```sh
consul agent -join=192.172.2.3
```

在极端情况下，应该可以通过在`raft/peers.json`恢复文件中将自身作为唯一对等方启动该单个服务器来仅使用剩余的单个服务器进行恢复。

在Consul0.7之前，使用`raft/peers.json`并不总是能够从某些类型的中断中恢复，因为在回放任何Raft日志条目之前已将其提取。
在Consul0.7及更高版本中，`raft/peers.json`恢复文件是最终文件，并且在摄取快照后会对其进行快照，因此可以确保从恢复的配置开始。这确实隐式提交了所有Raft日志条目，因此仅应用于从中断中恢复，但应允许从存在某些集群数据的任何情况下进行恢复。

### 使用peers.json手动恢复
首先，停止所有剩余的服务器。您可以尝试使用leave命令，但在大多数情况下不起作用。如果leave错误退出，请不要担心。因此此时群集处于不正常状态。

在Consul 0.7及更高版本中，`peers.json`文件默认不再存在，仅在执行恢复时使用。在Consul启动并提取此文件后，该文件将被删除。 Consul0.7还使用了一个新的自动创建的`raft/peers.info`文件，以避免在升级后的首次启动时提取`raft/peers.json`文件。确保将`raft/peers.info`留在适当的位置以进行正确的操作。

注：使用`raft/peers.json`进行恢复可能会导致未提交的Raft日志被隐式提交，因此，仅在出现中断（无法使用其他选项来恢复丢失的服务器）后才可以使用此命令。因此确保不要在其他场景下存在该文件。

具体操作：
进入每个Consul服务器的`-data-dir`。在该目录中，将存在一个`raft/`子目录。我们需要创建一个`raft/peers.json`文件。该文件的格式取决于服务器为其Raft协议版本配置的内容。
1、对于Raft协议版本2和更早版本，应将其格式化为JSON数组，其中包含集群中每个Consul服务器的地址和端口，如下所示：
```sh
["10.1.0.1:8300", "10.1.0.2:8300", "10.1.0.3:8300"]
```
2、对于Raft协议版本3和更高版本，应将其格式化为JSON数组，其中包含集群中每个Consul服务器的`节点ID，地址：端口和投票信息`，如下所示：
```sh
[
  {
    "id": "adf4238a-882b-9ddc-4a9d-5b6758e4159e",
    "address": "10.1.0.1:8300",
    "non_voter": false
  },
  {
    "id": "8b6dda82-3103-11e7-93ae-92361f002671",
    "address": "10.1.0.2:8300",
    "non_voter": false
  },
  {
    "id": "97e17742-3103-11e7-93ae-92361f002671",
    "address": "10.1.0.3:8300",
    "non_voter": false
  }
]

```
* id (string: <required>): 指定服务器的节点ID。如果服务器是自动生成的，则可以在服务器启动时在日志中找到它，也可以在服务器的数据目录中的node-id文件中找到它。
* address (string: <required>): 指定服务器的IP和端口。该端口是用于集群通信的服务器的RPC端口。
* non_voter (bool: <false>)：这可以控制服务器是否为非投票者，这在某些高级自动驾驶仪配置中使用。如果省略，它将默认为false，这对于大多数群集来说都是典型的。

只需为所有服务器创建条目。您必须确认未包含在此处的服务器确实发生了故障，并且以后将不会重新加入群集。确保所有其余服务器节点上的此文件均相同。

此时，您可以重新启动所有剩余的服务器。在Consul 0.7及更高版本中，您将看到它们接收恢复文件：
```sh
...
[INFO] consul: found peers.json file, recovering Raft configuration
...
[INFO] consul.fsm: snapshot created in 12.484µs
[INFO] snapshot: Creating new snapshot at /tmp/peers/raft/snapshots/2-5-1471383560779.tmp
[INFO] consul: deleted peers.json file after successful recovery
[INFO] raft: Restored from snapshot 2-5-1471383560779
[INFO] raft: Initial configuration (index=1): [{Suffrage:Voter ID:10.212.15.121:8300 Address:10.212.15.121:8300}]
...

```

如果有任何服务器设法执行正常离开，则可能需要使用join命令使它们重新加入集群：
```sh
consul join <Node Address>

Successfully joined cluster by contacting 1 nodes.
```
应当注意，任何可用成员都可以用来重新加入集群，因为流言协议（gossip）将负责发现服务器节点。

此时，集群应再次处于可操作状态。节点之一应要求领导，并发出类似以下的日志：
```sh
[INFO] consul: cluster leadership acquired

```

在Consul 0.7和更高版本中，可以使用consul operator命令检查Raft配置：
```sh
consul operator raft list-peers

Node     ID              Address         State     Voter  RaftProtocol
alice    10.0.1.8:8300   10.0.1.8:8300   follower  true   3
bob      10.0.1.6:8300   10.0.1.6:8300   leader    true   3
carol    10.0.1.7:8300   10.0.1.7:8300   follower  true   3
```