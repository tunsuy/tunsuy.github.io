# zookeeper运维

## 常用参数
tickTime：CS通信心跳数
Zookeeper 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。tickTime以毫秒为单位。

initLimit：LF初始通信时限
集群中的follower服务器(F)与leader服务器(L)之间初始连接时能容忍的最多心跳数（tickTime的数量）。

syncLimit：LF同步通信时限
集群中的follower服务器与leader服务器之间请求和应答之间能容忍的最多心跳数（tickTime的数量）。
	
preAllocSize
用于配置ZooKeeper事务日志文件预分配的磁盘空间大小。默认的块大小是64M。改变块大小的其中一个原因是当数据快照文件生成比较频繁时可以适当减少块大小。比如 1000次事务会新产生一个快照（参数为snapCount)，新产生快照后会用新的事务日志文件，假设一个事务信息大小100b，那么事务日志预分配的磁盘空间大小为100kb会比较好。
	
autopurge.snapRetainCount	  
启用后，ZooKeeper自动清除功能会在dataDir和dataLogDir中分别保留autopurge.snapRetainCount最新快照和相应的事务日志，并删除其余快照。

autopurge.purgeInterval
必须触发清除任务的时间间隔（以小时为单位）。设置为正整数（1或更大）以启用自动清除。

maxClientCnxns
ZooKeeper服务器允许的最大客户端连接数。为避免耗尽允许的连接，请将其设置为0（无限制）。

globalOutstandingLimit
客户端提交请求的速度可能比ZooKeeper处理的速度快得多，特别是当客户端的数量非常多的时候。为了防止ZooKeeper因为排队的请求而耗尽内存，ZooKeeper将会对客户端进行限流，即限制系统中未处理的请求数量不超过globalOutstandingLimit设置的值。默认的限制是 1000。

server.N=YYY:A:B
每一个机器是ZooKeeper集群的一部分。他们应该知道集群中的其它机器。你使用server.id=host:port:port这样的形式来达到这样的目的，
其中id为server的唯一标识，第一个port为LF通信端口，第二个port为选举端口。参数host和port是直观的。
需要创建一个名字为myid的文件来区分每一台机器的服务id，每一服务端一个，它放在服务端的配置文件指定的dataDir的参数的数据目录下。
myid文件包含一个单行文本，它的值是机器的id。所以服务端1的myid将包含文本"1"而没有其它。这个id必须在集群中是唯一的并且是一个1到255之间的值。
当一个ZK服务端实例启动的时候，它从myid文件中读它的id，然后使用这个id,从配置文件读取找出它应该监视的端口。

group.x=x:x:x
”x” 是一个组的标识，等号右边的数字对应于服务器的标识。赋值操作右边是冒号分隔的服务器标识。注意：组必须是不相交的，并且所有组联合后必须是 ZooKeeper 集群。
例如，我们有1、2、3、4、5、6、7七个节点。我们做如下配置：
```sh
group.1=1:2:3
group.2=4:5:6
group.2=7
```
将七台机器分为三个组，这时，只要三个组中的两个是稳定的，整个集群的状态就是稳定的。即有2n+1个组，只要有n+1个组是稳定状态，整个集群就是稳定的。也就是过半原则。

weight.x=x
和“group”一起使用，当形成集群时它给每个服务器赋权重值。这个值对应于投票时服务器的权重。ZooKeeper中只有少数部分需要投票，比如Leader选举以及原子的广播协议。服务器权重的默认值是1。如果配置文件中定义了组，但是没有权重，那么所有服务器的权重将会赋值为1。
经过实际测试和翻阅zk的投票源码，Weight等于0的节点不参与投票，没有被选举权。而不影响投票的权重. 源码片段如下:
```java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch,long curId, long curZxid, long curEpoch){
    if (self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }
    
    return ((newEpoch > curEpoch) || 
             (newEpoch == curEpoch && newZxid > curZxid) || 
             (newZxid == curZxid && newId > curId));
}
```
Weight可以调节一个组内单个节点的权重，默认每个节点的权重是1（如果配置是0不参与leader的选举）.每个组有一个法定数的概念，法定人数等于组内所有节点的权重之和.此时判断一个组是否稳定，是要判断存活的节点权重之和是否大于该组法定数的权重.

接上面的例子，我们做如下配置:
```sh
weight.1=3
weight.2=1
weight.3=1
weight.4=1
weight.5=1
weight.6=1
weight.7=1
```
此时Group1的法定数是：3+1+1=5，只要节点权重之和过半该组就是稳定的。也就是说，该组中，节点1（权重是3）只要存活,该组就是稳定的. 经过以上配置,停掉节点2，3，4，5，6整个集群仍然是稳定的. 此时Group1和Group3是稳定状态.

## AdminServer
AdminServer是一个内置的Jettry服务，它提供了一个HTTP接口为四字母单词命令。默认的，服务被启动在8080端口，并且命令被发起通过URL "/commands/[command name]",例如，http://localhost:8080/commands/stat。命令响应以JSON的格式返回。不像原来的协议，命令不是限制为四字母的名字，并且命令可以有多个名字。例如"stmk"可以被指定为"set_trace_mask"。为了查看所有可用命令的列表，指向一个浏览器的URL /commands (例如， http://localhost:8080/commands)。
AdminServer默认开启，但是可以被关闭通过下面的方法：
设置java系统属性`zookeeper.admin.enableServer为false`.
* 注意TCP四字母单词接口是仍然可用的如果AdminServer被关闭。
### 相关参数
admin.enableServer(Java system property: zookeeper.admin.enableServer)
设置为false来关闭AdminServer。默认是开启的。
admin.serverPort(Java system property: zookeeper.admin.serverPort)
内置的Jetty服务器监听的端口。默认是8080。
admin.commandURL(Java system property: zookeeper.admin.commandURL)
用于列举和发起的和root URL相关的命令的URL。默认是/commands。

### ZK四字命令
ZK响应一组命令。每一个命令由四个字母组成。可以通过telnet或nc发送命令给ZK。
比如：mntr 3.3.0新增 输入可以被用来监视群集健康状态的变量列表。
```sh
$ echo mntr | nc localhost 2185
              zk_version  3.4.0
              zk_avg_latency  0
              zk_max_latency  0
              zk_min_latency  0
              zk_packets_received 70
              zk_packets_sent 69
              zk_outstanding_requests 0
              zk_server_state leader
              zk_znode_count   4
              zk_watch_count  0
              zk_ephemerals_count 0
              zk_approximate_data_size    27
              zk_followers    4                   - only exposed by the Leader
              zk_synced_followers 4               - only exposed by the Leader
              zk_pending_syncs    0               - only exposed by the Leader
              zk_open_file_descriptor_count 23    - only available on Unix platforms
              zk_max_file_descriptor_count 1024   - only available on Unix platforms

```

#### 参数解析如下
zk_server_state
说明：集群中有且只能有一个leader，没有leader，则集群无法正常工作；两个或以上的leader，则视为脑裂，会导致数据不一致问题

zk_followers /zk_synced_followers
说明：如果上述两个值不相等，就表示部分follower异常了需要立即处理，很多低级事故，都是因为单个集群故障了太多的follower未及时处理导致

zk_outstanding_requests
说明：常态下该值应该持续为0，不应该有未处理请求

zk_pending_syncs
说明：常态下该值应该持续为0，不应该有未同步的数据

zk_znode_count
说明：节点数越多，集群的压力越大，性能会随之急剧下降
经验值：不要超过100万
建议：当节点数过多时，需要考虑以机房/地域/业务等维度进行拆分

zk_approximate_data_size
说明：当快照体积过大时，ZK的节点重启后，会因为在initLimit的时间内同步不完整个快照而无法加入集群
经验值：不要超过1GB体积
建议：不要把ZK当做文件存储系统来使用

zk_open_file_descriptor_count/zk_max_file_descriptor_count
说明：当上述两个值相等时，集群无法接收并处理新的请求
建议：修改/etc/security/limits.conf，将线上账号的文件句柄数调整到100万

zk_watch_count
说明：对于watch的数量较多，那么变更后ZK的通知压力也会较大

zk_packets_received/zk_packert_sent
说明：ZK节点接收/发送的packet的数量，每个节点的具体值均不同，通过求和的方式来获取集群的整体值
建议：通过两次命令执行间隔1s来获取差值

zk_num_alive_connections
说明：ZK节点的客户端连接数量，每个节点的具体值均不同，通过求和的方式来获取集群的整体值
建议：通过两次命令执行间隔1s来获取差值

zk_avg_latency/zk_max_latency/zk_min_latency
说明：需要关注平均延时的剧烈变化，业务上对延时有明确要求的，则可以针对具体阈值进行设置	  

## 监控
### 尝试操作

#### 创建/删除/读取节点
说明：在/zookeeper_monitor节点下，定期创建/删除节点，确保该功能可用
建议：创建/zookeeper_monitor节点，不要使用业务节点，避免互相影响
经验值：模拟用户请求的节点至少3个，从而确保覆盖ZK所有节点

#### 读取/更新内容
说明：在/zookeeper_monitor节点下，定期对内容读取和更新
建议：可以将时间戳写入，从而便于判断写入延时

### 采集数据
方式1：zookeeper四字命令mntr（上面已经说了）

方式2：JMX接口
开启JMX监控：
修改zookeeper的启动脚本vim zkServer.sh。找到启动参数ZOOMAIN，修改为下面值。
其中local.only=false，设为false才能在远程建立连接。
```sh
ZOOMAIN="-Dcom.sun.management.jmxremote  -Dcom.sun.management.jmxremote.local.only=false
 -Djava.rmi.server.hostname=127.0.0.1
 -Dcom.sun.management.jmxremote.port=9991
 -Dcom.sun.management.jmxremote.ssl=true
 -Dcom.sun.management.jmxremote.authenticate=true
 -Dcom.sun.management.jmxremote.access.file=/data/zookeeper/conf/jmxremote.access
 -Dcom.sun.management.jmxremote.password.file=/data/zookeeper/conf/jmxremote.password
 -Dzookeeper.jmx.log4j.disable=true
 org.apache.zookeeper.server.quorum.QuorumPeerMain"
```
在/data/zookeeper/conf目录下建立2个访问授权文件, 修改文件权限chmod 600 jmxremote.*
```sh
-rw-------  1 deploy deploy  149 Aug  6 13:44 jmxremote.access
-rw-------  1 deploy deploy   40 Aug  6 13:46 jmxremote.password
[deploy@liutp conf]$ pwd
/data/zookeeper/conf
[deploy@liutp conf]$ cat jmxremote.access
monitorRole   readonly
controlRole   readwrite \
              create javax.management.monitor.*,javax.management.timer.* \
              unregister
[deploy@liutp conf]$ cat jmxremote.password
monitorRole  1234567
controlRole  1234567
[deploy@liutp conf]$
```
重启zookeeper

使用Java自带的JConsole
在命令行输入JConsole，再回车。
在弹出的界面选择“远程进程”，输入“服务器IP:9991”(zookeeper服务器的IP和端口)

## 实践经验
### 分Group
要确保Zookeeper整个集群可靠运行，就是要确保投票集群可靠。可以将一个Zookeeper集群划分为多个小的Group，Leader+Follower为核心Group，核心Group一般是不向外提供服务的，然后根据不同的业务再加一些Observer，比如一个Zookeeper集群为服务发现，分布式锁，配置管理三个不同的组件提供服务，那么可以建立三个Observer Group，分别给这三个组件使用，而Client只会连接分配给它的Observer Group，不去连接核心Group。这样核心Group就不会给Client提供长连接服务，也不负责长连接的心跳，这大大的减轻了核心Group的压力，因为在实际环境中，一个Zookeeper集群要为上万台机器提供服务，维持长连接和心跳还是要消耗一定的资源的。因为Observer是不参与投票的所以加Observer并不会降低整体的吞吐量，而且Observer挂掉不会影响整个集群的健康。
但是这里要注意的是，分Observer Group只能解决部分问题，因为毕竟所有的写入还是要交给核心Group来处理的，所以对于写入量特别大的应用来说，还是需要进行集群上的隔离。

### 内存
因为Zookeeper将所有数据都放在内存里，所以对JVM以及机器的内存也要预先计划，如果出现Swap那将严重的影响Zookeeper集群的性能，在启动的时候可以修改java系统参数进行控制。
但是还是建议不要使用zk做大数据量存储。

### 日志清理
因为Zookeeper要频繁的写txlog(Zookeeper写的一种顺序日志)以及定期dump内存snapshot到磁盘，这样磁盘占用就越来越大，所以Zookeeper提供了清理这些文件的机制，但是这种机制不能根据业务进行动态的控制。比如有可能碰到高峰期间清理，所以可以根据业务场景决定是否关闭:autopurge.purgeInterval=0。然后使用crontab等机制，在业务低谷的时候清理。

### jvm配置
从官网直接下载的包如果直接启动运行是很糟糕的，这个包默认的配置日志是不会轮转的，而且是直接输出到终端。默认配置还没有设置任何jvm相关的参数(所以堆大小是个默认值)，这也是不可取的。
有几种方式可以配置：
1、直接修改Zookeeper的启动脚本zkServer.sh（一般不这样做）。
2、zkServer.sh会加载conf文件夹下一个名为zookeeper-env.sh的脚本，所以可以将一些定制化的配置写在这里。
3、zookeeper-env.sh会加载一个叫java.env的脚本，所以也可以将配置写在这里。
```sh
JAVA_HOME= #java home
ZOO_LOG_DIR= #日志文件放置的路径
ZOO_LOG4J_PROP="INFO,ROLLINGFILE" #设置日志轮转
JVMFLAGS="jvm的一些设置，比如堆大小，开gc log等"
```