# rabbitmq技术内幕

Rabbitmq大体上可以分为两部分（Exchange和MQ），所有发送给RabbitMQ的消息都会先交给Exchange， Exchange的功能类似于路由器，它会根据自身类型（fanout、direct、topic）以及binding信息决定一个消息该被放到哪一个MQ， 而MQ的功能在于暂时存储消息，并将MQ中的消息以订阅或者poll的方式交给接收方。

## backing queue
MQ内部大致又可以分为两部分:`amqueue`和`backing queue`,  `amqqueue`负责实现amqp协议规定的mq的基本逻辑，`backing queue`则实现消息的存储，它会尽量为`durable=true`的消息做持久化的存储，而在内存不足时将一部分消息放入DISK换取更多的内存空间。`Backing queu`内部又细分为5各小Q，

消息在这些Q中传递的“一般”过程`q1->q2->delta->q3->q4`，这几个Q实现的是`RAM ->DISK->RAM`这一过程中对消息的分类管理。大多数情况下，一个消息并非需要走完每个小Q，通常大部分都可以略过。与这5各Q对应，在backing queue中消息的生命周期可以分为四个状态：
* Alpha：该消息的位置信息和消息本身都在RAM中，这类消息排列在Q1和Q4。
* Beta：消息的位置保存在RAM中，消息本身保存在DISK中，这类消息排列在Q2或Q3中。
* Gamma: 消息的位置保存RAM和DISK中，消息本身保存在DISK中，这类消息排列在Q2或Q3中。
* Delta：消息的位置和消息本身都保存在DISK中，这类消息排列在delta中。
  从`Q1->Q2->delta`这一个过程是将消息逐步从RAM移动到DISK的过程，而`delta->Q3->Q4`是从DISK逐步移动到RAM的过程。

通常在负载正常时，一个消息不会经历每种状态，如果消息被消费的速度不小于接收新消息的速度，对于不需要保证可靠不丢的消息极可能只会有Alpha状态。对于`durable=true`的消息，它一定会进入gamma，若开启publish confirm，只有到了这个阶段才会确认该消息已经被接收，若消息消费的速度足够快，内存也充足，这些消息也不会继续走到下一状态。

从上述`backing queue`对消息的处理过程可以看出，消息若能尽早被消费掉即在不要走完这5个队列，尽量在q1或q2中就被消费掉，就能减少系统的开销。若走的“太深”则会有内存的换入换出增加系统开销。这样就存在一个问题：
通常在系统负载较高时，已接收到的消息若不能很快的被消费掉，这些消息就会进入到很深的队列中去，增加处理每个消息的平均开销。因为要花更多的时间和资源处理“积压”的消息，所以用于处理新来的消息的能力就会降低，使得后来的消息又被积压进入很深的队列，继续加大处理每个消息的平均开销，这样情况就会越来越恶化，使得系统的处理能力大大降低。

根据官方博客，应对这一问题，有三个措施：
* 1. 进行流量控制。
* 2. 增加`prefetch`的值，即一次发送多个消息给接收者，加快消息被消费掉的速度。
* 3. 采用`mutli ack`，降低处理ack带来的开销。
    目前RabbitMQ已经有了很好的流量控制机制，MQ中堆积的消息数一直都很少（低于5个）。需要使用者做的就是2，3两点。

## mirror queue
mirror queue基本上就是一个特殊的backing queue， 它内部包裹了一个普通的backing queue做本地的消息持久化处理，在此基础上增加了将消息和ack复制到所有镜像的功能。所有对rabbit_mirror_queue_master的操作，会通过组播GM（Guarenteed Multicast）的方式同步到各slave节点。

### 新节点加入
允许新的slave节点中途加入到集群中，新加入的slave节点并不同步master节点的所有在该slave加入之前存在的消息，只对新来的消息保持同步，随着旧的消息被消费，经过一段时间后，slave节点就会与master节点完全同步。

### 节点失效
当master节点失效后，所有slave中消息队列最长者会成为新的master，因为这样的节点最有可能与原来的master节点完全同步。

### 节点重启
当一个节点无论master还是slave失效后重启，都会丢弃本地记录在disk中的所有消息，作为一个全新的slave节点加入到集群中去。

### GM
GM模块实现的一种可靠的组播通讯协议，该协议能够保证组播消息的原子性，即保证组中活着的节点要么都收到消息要么都收不到。它的实现大致如下：
一个组的所有成员组成一个ring，例如 A->B->C->D->A。假如A是master节点，A要发组播消息，A首先会将消息发送到A的后继节点B，B接收到消息后在传递给C然后是D，最后D再发给A。在此过程中若有节点失效，发送节点就会往失效的节点的后继节点发消息，若后继节点也失效就往后继的后继发消息。当A收到传回来的消息时，A就可以确认所有“活着的”节点都已收到该消息，但此时B、C、D并不能确认所有节点都收到了该消息，所以不能往上提交该消息。这时，A往B发Ack，当B收到ack后就能确认所有的节点都收到该消息，B将该ack继续传递给C，D最终又传回A，至此整个发送过程就完成了。若最终A没有收到ack，则说明此次发送失败。


## Rabbitmq进程
* `tcp_acceptor` 进程接收客户端连接，并创建 `rabbit_reader、rabbit_writer、rabbit_channel` 进程；
* `rabbit_reader` 负责接收客户端连接，解析 AMQP 帧；
* `rabbit_writer` 负责向客户端返回数据；
* `rabbit_channel` 负责解析 AMQP 方法，对消息进行路由，然后发给相应队列进程；
* `rabbit_amqqueue_process` 即队列进程，在 RabbitMQ 启动（恢复 durable 类型队列）或创建队列时创建；
* `rabbit_msg_store` 是负责消息持久化的进程；

无论何时我们向RabbitMQ发布AMQP消息，我们都有以下的erlang消息流：
```sh
reader process -> channel process -> amqqueue process -> message store
```

## Rabbitmq流控
在RabbitMQ中，消息可能被存储在多个不同的队列，消息越早被消费，那么消息经过的队列层次越少，则平均每个消息处理的开销就越小。但若接收消息的速率过快，MQ来不及处理，这些消息就可能进入很深层次的队列，大大增加平均每个消息的处理开销，进一步使得处理新消息和发送旧消息的能力减弱，更多的消息会进入很深的队列，循环往复，整个系统的性能就会极大的降低。另外若接收消息的速率过快还会实现某些进程的mailbox过大，可能会产生很严重的后果。为此，RabbitMQ设计了一套流控机制，本文从以下三个方面去阐述该流控机制是如何工作的。

Erlang 进程之间并不共享内存（binary 类型除外），而是通过消息传递来通信，每个进程都有自己的进程邮箱。Erlang 默认没有对进程邮箱大小设限制，所以当有大量消息持续发往某个进程时，会导致该进程邮箱过大，最终内存溢出并崩溃。

在 RabbitMQ 中，如果生产者持续高速发送，而消费者消费速度较低时，在没有流控的情况下，会导致内部进程邮箱大小迅速增大，进而达到 RabbitMQ 的整体内存阈值限制，阻塞生产者（得益于这种阻塞机制，RabbitMQ 本身并不会崩溃），与此同时，RabbitMQ 会进行 page 操作，将内存中的数据持久化到磁盘中。

为了解决该问题，RabbitMQ 使用了一种基于信用证（Credit）的流控机制。即每个消息处理进程都具有一个信用组 `{InitialCredit，MoreCreditAfter}`，默认值为 {200, 50} 。

当消息发送者进程 A 向接收者进程 B 发消息时：
* 对于发送者 A ，每发送一条消息，Credit 数量减 1，直到为 0 后被 block 住；
* 对于接收者 B ，每接收 MoreCreditAfter 条消息，会向 A 发送一条消息，给予 A MoreCreditAfter 个 Credit ；
* 当 A 的 Credit > 0 时，A 可以继续向 B 发送消息；

可以看出：基于信用证的流控，消息发送进程的发送速度会被限制在消息处理进程的处理速度内；
```sh
reader process <–[grant]– channel process <–[grant]– amqqueue process <–[grant]– message store
```

### 1. 如何开关闸门
RabbitMQ使用TCP长连接进行通讯，接收数据的起点进程为rabbit_reader。
rabbit_reader每接收到一个包，就设置套接字属性为{active, onece}，若当前连接被blocked时则不设置{active,once}，这个接收进程就阻塞在receive方法上。通过这种方式来实现闸门的开关。

### 2. 何时关闭闸门
RabbitMQ是用erlang/OTP开发的，一个消息从被接收到被发送给订阅者，必然要在多个进程间的转发，从接收到被消费，一个消息所走过的所有进程自然形成一条消息链，RabbitMQ通过监控这条链上每个节点“mailbox”中未被接收的消息数量，决定何时关闭闸门。实现机制如下所述：
```sh
{{credit_from,B}, value}      {{credit_from,C}, value}     {{credit_from,pid}, value}
{{credit_to, pid}, value}	  {{credit_to,A}, value}       {{credit_to,B}, value}
A ==========================》B =========================》C ========================》
```
	
如图所示，进程A、B、C连成一条消息链，每个进程字典中有一对关于收发消息的credit值，以进程B为例，｛｛credit_from, C｝, Value｝，表示能发多少条消息给C，每发一条消息该值减1，当为0时，本进程阻塞住不再往下游进程发消息也不再接收上游的消息；｛｛credit_to, A｝, Value｝表示再接收多少个消息就向上游进程发增加credit值的消息｛bump_credit, { self(), Quantity}｝,在上游进程接收到该消息后，就增加｛credit_from, pid｝值，这样上游进程就能持续发消息。但当上游发送速率高于下游接收速率，credit值会逐渐被耗光这时进程就会被阻塞，阻塞的情况会一直传递到最上游
Rabbit_reader，这时rabbit_reader就关闭闸门。

### 3. 何时开启闸门
当上游进程收到来自下游进程的bump_credit消息时，若此时上游进程处于block状态则解除block状态，开始接收更上游进程的消息，一个个的传导最终能够解除rabbit_reader的block状态。

### 状态机
Rabbitmq的中处理队列收发逻辑的是一个有穷状态机进程。

* 1. 当MQ既有生产者也有消费者时，该状态机的处理流程为 ：接收消息->持久化->发送消息->接收消息 –> … ->。在流控机制的控制下，收发速率能够保持基本一致，队列中堆积的消息数会非常低。
* 2. 当没有消息者时，MQ会持续接收消息并持久化直到磁盘被写满，因为没有发送逻辑，这时可以达到更高的生产速率。
* 3. 当MQ中有消息堆积时，MQ会持续从队列中取出堆积的消息将其发送出去，直到没有了堆积消息，或者消费者的qos被用光，或者没有消费者，或者消费者的channel被阻塞。如果一直没有满足上述4个条件之一，MQ就会持续的发送堆积消息，不去处理新来的消息，在流控机制的作用下，发送端就被阻塞了。

总结：从上述描述可以看出，消息堆积后，发送速率降低是MQ的处理流程使然。这样的流程设计基于以下两个原因：
* 1. 让堆积的消息更快的被消费掉，降低消息的时延。
* 2. MQ中堆积的消息越少，每个消息处理的平均开销就越少，可以提高整体性能，所以需要尽快将堆积消息发送出去。

## Paging
Paging 就是在内存紧张时触发的，paging 将大量 alpha 状态的消息转换为 beta 和 gamma ；如果内存依然紧张，继续将 beta 和 gamma 状态转换为 delta 状态。Paging 是一个持续过程，涉及到大量消息的多种状态转换，所以 Paging 的开销较大，严重影响系统性能。

## 消息的确认机制
开启confirm后，生产者与RabbitMQ之间通过发送确认序号来对消息进行确认，该序号是per channel的。对消息进行确认就是简单的将该消息对应的序号发回给生产者，但RabbitMQ收到消息后并不是立即回ack，在不同配置下，回ack的时机是不同的。Confirm的过程伴随着消息在MQ中整个处理流程，为此接下来我们从整个消息的生命周期来分析confirm的处理机制。

### 1. Rabbit_channel->
Rabbit_channel接收到新消息后，根据路由规则确定该消息应该被投递给哪些queue, 为每个消息生成一个全局唯一标识msgid,当然每个消息也有per channel的确认序号，后面还会有别的序号，为了避免冲突我们将该确认序号命名为ch_seq_no。在将该消息以及对应的msgid，ch_seq_no投递给相应queue进程。

### 2. Rabbit_amqqueue_process->
Rabbit_amqqueue_process收到信息以后，记录为未确认的消息，然后交给backing_queue做持久化。

### 3. backing_queue->
backing_queue收到新消息后，记录为未确认的消息，然后将消息投递给rabbit_msg_store。

### 4. rabbit_msg_store->
rabbit_msg_store收到新消息后，记录为未确认的消息，然后定期或者在切换消息存储文件时，对消息进行确认。
将所有需要被confirm的消息按所属的queue进行分组，然后分别调用对应queue发送callback函数MsgOnDiskFun以及需要confirm的MsgIds。

### 5. rabbit_amqqueue_process<-
rabbit_amqqueue_process收到确认消息后，执行传过来的callback函数MsgOnDiskFun，
在每次reply和norely将需要确认的消息发送给对应的Channel进程

### 6. channel<-
当channel收到confirm消息后，记录需要confirm的消息，然后合并可以合并的confirm消息，并发送给发送者


## Rabbitmq的HA
rabbitmq可以为Consumers做负载均衡，但rabbimq自身并没有负载均衡。用户连接到rabbitmq集群的任意节点都可以访问集群中的任意消息队列，但一个消息队列只存储在一个物理节点上，其它节点只存储该队列的元数据，这使得当队列里只有一个队列时，系统性能受限于单个节点的网络带宽和主机性能。若使用多个队列来提升性能，也会有新的问题，即如何在队列之间做负载均衡，同时网络连接也会影响系统性能，比如当一个用户往某个消息队列发消息时，而该用户当前连接的节点不是该队列真实所在的物理节点，这必然会产生rabbitmq节点间通讯，从而消耗的一部分网络带宽。

因此对于集群rabbitmq，有以下几种方案：

### 方案一  简单负载均衡方案
在发送到到rabbitmq之前加一层负载均衡，比如Haproxy+Keepalive。
发送端只需要发送到rabbitmq的浮动ip即可，不用关注具体的节点。
通过ha的能力和均衡策略将请求发送到具体的节点，并实时监控节点的健康，自动拉起异常节点。

### 方案二  镜像队列
Rabbitmq现提供队列mirror功能，通过这一功能可以提高Rabbitmq的可靠性，当某个Rabbitmq节点故障时，只要其它节点里存在该故障节点的队列镜像，该队列就能继续正常工作不会丢失数据。但使用该功能也会有些副作用，它这种通过冗余数据保障可靠性的方式会降低系统的性能，因为往一个队列发数据也就会往这个队列的所有镜像队列发数据，这必然产生大量Rabbitmq节点间数据的交互，降低吞吐率，镜像越多性能必然下降越多。与此同时，为充分利用集群的的资源，需要创建多个队列，若在所有节点上都有每个队列的镜像来实现可靠性，则队列镜像数会太多，过多的RabbitMq集群内部网络通讯会吃掉大量网络带宽。

### 方案三 三方库
采用业界的三方库，比如python的oslo_messaging等等
三方库一般会要求传入所有的rabbitmq节点，具体的请求分发由三方库处理。

