## zookeeper的日志

### 一、日志类型
zookeeper中有两类日志,分别是：

1、事务日志log，对应代码类：org.apache.zookeeper.server.persistence.FileTxnLog
2、快照日志snapshot，对应代码类：org.apache.zookeeper.server.persistence.FileTxnSnapLog

事务日志 : 顾名思义，就是用于存放事务执行的相关信息，如zxid、cxid等。

快照日志：zookeeper数据节点数据是运行在内存中的，当然内存保存这些结点的数据不可能无限大，而且数据节点的内容是动态变化的，因此zookeeper提供将数据节点持久化的机制，每隔一段时间，zookeeper会将内存中的数据节点DataTree序列到磁盘中，因此就形成了我们的快照日志。

ZooKeeper集群中的每个服务器节点每次接收到写操作请求时，都会先将这次请求发送给leader，leader将这次写操作转换为带有状态的事务，然后leader会对这次写操作广播出去以便进行协调。当协调通过(大多数节点允许这次写)后，leader通知所有的服务器节点，让它们将这次写操作应用到内存数据库中，并将其记录到事务日志中。

当事务日志记录的次数达到一定数量后(默认10W次)，就会将内存数据库序列化一次，使其持久化保存到磁盘上，序列化后的文件称为"快照文件"。每次拍快照都会生成新的事务日志。
有了事务日志和快照，就可以让任意节点恢复到任意时间点(只要没有清理事务日志和快照)。

### 二、事务日志和快照相关的配置项
dataDir：
ZooKeeper的数据目录，主要目的是存储内存数据库序列化后的快照路径。如果没有配置事务日志(即dataLogDir配置项)的路径，那么ZooKeeper的事务日志也存放在数据目录中。

dataLogDir：
指定事务日志的存放目录。事务日志对ZooKeeper的影响非常大，强烈建议事务日志目录和数据目录分开，不要将事务日志记录在数据目录(主要用来存放内存数据库快照)下。

preAllocSize：
为事务日志预先开辟磁盘空间。默认是64M，意味着每个事务日志大小就是64M(可以去事务日志目录中看一下，每个事务日志只要被创建出来，就是64M)。如果ZooKeeper产生快照频率较大，可以考虑减小这个参数，因为每次快照后都会切换到新的事务日志，但前面的64M根本就没写完。(见snapCount配置项)

snapCount：
ZooKeeper使用事务日志和快照来持久化每个事务(注意是日志先写)。该配置项指定ZooKeeper在将内存数据库序列化为快照之前，需要先写多少次事务日志。也就是说，每写几次事务日志，就快照一次。默认值为100000。为了防止所有的ZooKeeper服务器节点同时生成快照(一般情况下，所有实例的配置文件是完全相同的)，当某节点的先写事务数量在(snapCount/2+1,snapCount)范围内时(挑选一个随机值)，这个值就是该节点拍快照的时机。

autopurge.snapRetainCount：
该配置项指定开启了ZooKeeper的自动清理功能后(见下一个配置项)，每次自动清理时要保留的版本数量。默认值为3，最小值也为3。它表示在自动清理时，会保留最近3个快照以及这3个快照对应的事务日志。其它的所有快照和日志都清理。

autopurge.purgeInterval：
指定触发自动清理功能的时间间隔，单位为小时，值为大于或等于1的整数，默认值为0，表示不开启自动清理功能。

### 三、事务日志和快照的命名规则
在ZooKeeper集群启动后，当第一个客户端连接到某个服务器节点时，会创建一个会话，这个会话也是事务，于是创建第一个事务日志，一般名为log.100000001，这里的100000001是这次会话的事务id(zxid)。之后的事务都将写入到这个文件中，直到拍下一个快照。
如果是事务ZXID5触发的拍快照，那么快照名就是snapshot.ZXID5，拍完后，下一个事务的ID就是ZXID6，于是新的事务日志名为log.ZXID6。

使用ZXID作为文件后缀，可以帮助我们迅速定位到某一个事务操作所在的事务日志。同时，使用ZXID作为事务日志后缀的另一个优势是：ZXID本身由两部分组成，高32位代表当前leader周期（epoch）,低32位则是真正的操作序列号，因此，将ZXID作为文件后缀，我们就可以清楚地看出当前运行时的zookeeper的leader周期。

事务日志的写入是采用了磁盘预分配的策略。因为事务日志的写入性能直接决定看Zookeeper服务器对事务请求的响应，也就是说事务写入可被看做是一个磁盘IO过程，所以为了提高性能，避免磁盘寻址seek所带来的性能下降，所以zk在创建事务日志的时候就会进行文件空间“预分配”，即：在文件创建之初就想操作系统预分配一个很大的磁盘块，默认是64M，而一旦已分配的文件空间不足4KB时，那么将会再次进行预分配，再申请64M空间。

### 四、事务日志可视化
zookeeper日志都是以二进制存储的，那么怎么可以查看里面的内容呢，下面以linux系统为例：
`java -cp /usr/local/bin/zookeeper/zookeeper/lib/slf4j-api-1.7.25.jar:/usr/local/bin/zookeeper/zookeeper/lib/zookeeper-3.4.14.jar org.apache.zookeeper.server.LogFormatter ./version-2/log.10000000c > test.log`

其中-cp表示将LogFormatter类所属的jar包及其依赖的jar包放入环境变量中，jar的路径根据自身安装的zookeeper路径为准
LogFormatter 后面跟需要解析的二进制日志文件

test.log表示将解析之后的文本日志文件写入test.log中，便于查看

然后我们可以打开test.log看到日志如下：
```sh
ZooKeeper Transactional Log File with dbid 0 txnlog format version 2
1/20/20 4:29:59 AM UTC session 0x30014da58050000 cxid 0x0 zxid 0x10000000c createSession 30000

1/20/20 4:30:00 AM UTC session 0x30014da58050000 cxid 0x4 zxid 0x10000000d create '/test_zookeeper/test/item,,v{s{31,s{'djdigest,'CgcA1GMivoBYyZZWuQDgeLuz5L45jmuVDyLKi2J0swQ=:MEonRpvlUHyT9yHsCnPddPJ0QVCMGVM5ylIV0Zv/VaY=}}},F,2

1/20/20 4:30:00 AM UTC session 0x30014da58050000 cxid 0x5 zxid 0x10000000e delete '/test_zookeeper/test/item

1/20/20 4:30:00 AM UTC session 0x30014da58050000 cxid 0x6 zxid 0x10000000f setACL '/,v{s{31,s{'djdigest,'T9ihPbFmp0odTgrtigbbYJgBkC5Pe6XkWO543Hl1+jc=:dk8WmGqk2QsbWdyxv98BRJWaiW3xEGxpvbzVVx8z8ig=}}},2

1/20/20 4:30:00 AM UTC session 0x30014da58050000 cxid 0x8 zxid 0x100000010 setACL '/zookeeper,v{s{31,s{'djdigest,'V4AoltP7EqnmD6tlXT9D+yzozXnf2aN/FTmYOVekewQ=:vvEn1n8041x6LDHUgnGC/+tAAwKpFePtXZAdWP4hY3Y=}}},2

1/20/20 4:30:00 AM UTC session 0x30014da58050000 cxid 0xa zxid 0x100000011 setACL '/zookeeper/quota,v{s{31,s{'djdigest,'mqf1V7QQvkgzN7+Z4pp8PZvYh5wyVDYpuPzMLWKlYYI=:kRWUuqEYdWFbrCfxUCTMkDhl+nd8aOWA4txN6LjtiX8=}}},2

1/20/20 4:30:01 AM UTC session 0x30014da58050000 cxid 0xc zxid 0x100000012 setACL '/test_zookeeper,v{s{31,s{'djdigest,'IlpdjxvqVztCEvkwfaMb6msDuq4KAuRDNsCHvRmxE3M=:QQsEq6dyA2sl3TC0NxK2OEtgLLFpvn1gBYb4k/7nlpA=}}},2

1/20/20 4:30:01 AM UTC session 0x30014da58050000 cxid 0xe zxid 0x100000013 setACL '/test_zookeeper/test,v{s{31,s{'djdigest,'QAyFllAzzz5GEeYw5oqQNlfY8JctWi/QQOuAcHSDfG4=:UVSf8vxMBsf1wiL9kgeiEVPt1anF/lBZ0iOyItQBHus=}}},2

1/20/20 4:30:01 AM UTC session 0x30014da58050000 cxid 0x10 zxid 0x100000014 closeSession null
1/20/20 4:30:07 AM UTC session 0x20014dbe5970001 cxid 0x0 zxid 0x100000015 createSession 30000

1/20/20 4:30:08 AM UTC session 0x20014dbe5970002 cxid 0x0 zxid 0x100000016 createSession 30000

1/20/20 4:30:08 AM UTC session 0x20014dbe5970002 cxid 0x4 zxid 0x100000017 create '/test_zookeeper/test/item,,v{s{31,s{'djdigest,'C2bB1tFWzcVCJtJ/oEHT4Il4ZjfI0EH0OyfOb0HvWjQ=:EpOdCuJXIo9sETNyI967xYfltzIs6wzB9I+mudSB7VU=}}},F,3
```
日志分析：

​	1、第一行：ZooKeeper Transactional Log File with dbid 0 txnlog format version 2
​	表示日志头，这里magic没有输出，只输出了dbid还有version；

	2、第二行：1/20/20 4:29:59 AM UTC session 0x30014da58050000 cxid 0x0 zxid 0x10000000c createSession 30000
	这也就是具体的事务日志内容了，这里是说xxx时间有一个sessionid为0x30014da58050000、cxid为0x0、zxid为0x10000000c、类型为createSession、超时时间为4000毫秒
	
	3、第三行：1/20/20 4:30:00 AM UTC session 0x30014da58050000 cxid 0x4 zxid 0x10000000d create '/test_zookeeper/test/item,,v{s{31,s{'djdigest,'CgcA1GMivoBYyZZWuQDgeLuz5L45jmuVDyLKi2J0swQ=:MEonRpvlUHyT9yHsCnPddPJ0QVCMGVM5ylIV0Zv/VaY=}}},F,2
	sessionID为0x30014da58050000，cxid：0x4、zxid：0x10000000d 创建了一个节点路径为：/test_zookeeper/test/item、节点内容为空，acl为'djdigest,'CgcA1GMivoBYyZZWuQDgeLuz5L45jmuVDyLKi2J0swQ=:MEonRpvlUHyT9yHsCnPddPJ0QVCMGVM5ylIV0Zv/VaY=，父节点子版本：2

### 五、数据快照
数据快照是Zookeeper数据存储中非常核心的运行机制，数据快照用来记录Zookeeper服务器上某一时刻的全量内存数据内容，并将其写入指定的磁盘文件中。也是使用ZXID来作为文件后缀名，并没有采用磁盘预分配的策略，因此数据快照文件在一定程度上反映了当前zookeeper的全量数据大小。

与事务文件类似，Zookeeper快照文件也可以指定特定磁盘目录，通过dataDir属性来配置。在运行过程中会在该目录下创建version-2的目录，该目录确定了当前Zookeeper使用的快照数据格式版本号。在Zookeeper运行时，会生成一系列文件。

针对客户端的每一次事务操作，Zookeeper都会将他们记录到事务日志中，同时也会将数据变更应用到内存数据库中，Zookeeper在进行若干次（snapCount）事务日志记录后，将内存数据库的全量数据Dump到本地文件中，这就是数据快照。

过半随机策略：每进行一次事务记录后，Zookeeper都会检测当前是否需要进行数据快照。理论上进行snapCount次事务操作就会开始数据快照，但是考虑到数据快照对于Zookeeper所在机器的整体性能的影响，需要避免Zookeeper集群中所有机器在同一时刻进行数据快照。因此zk采用`过半随机`的策略，来判断是否需要进行数据快照。即：符合如下条件就可进行数据快照：
logCount > (snapCount / 2 + randRoll)   randRoll位1~snapCount / 2之间的随机数。这种策略避免了zk集群的所有机器在同一时刻都进行数据快照，影响整体性能。

开始快照时，首先关闭当前日志文件（已经到了该快照的数了），重新创建一个新的日志文件，创建单独的异步线程来进行数据快照以避免影响Zookeeper主流程，从内存中获取zookeeper的全量数据和校验信息，并序列化写入到本地磁盘文件中，以本次写入的第一个事务ZXID作为后缀。

### 六、快照日志可视化
zookeeper日志都是以二进制存储的，那么怎么可以查看里面的内容呢，下面以linux系统为例：
`java -cp /usr/local/bin/zookeeper/zookeeper/lib/slf4j-api-1.7.25.jar:/usr/local/bin/zookeeper/zookeeper/lib/zookeeper-3.4.14.jar org.apache.zookeeper.server.SnapshotFormatter ./version-2/snapshot.10000000b > test.log`
其中-cp表示将LogFormatter类所属的jar包及其依赖的jar包放入环境变量中，jar的路径根据自身安装的zookeeper路径为准
SnapshotFormatter 后面跟需要解析的二进制快照文件

test.log表示将解析之后的文本日志文件写入test.log中，便于查看

然后我们可以打开test.log看到日志如下：
```sh
ZNode Details (count=6):
----
/
  cZxid = 0x00000000000000
  ctime = Thu Jan 01 00:00:00 UTC 1970
  mZxid = 0x00000000000000
  mtime = Thu Jan 01 00:00:00 UTC 1970
  pZxid = 0x00000100000002
  cversion = 1
  dataVersion = 0
  aclVersion = 1
  ephemeralOwner = 0x00000000000000
  dataLength = 0
----
/zookeeper
  cZxid = 0x00000000000000
  ctime = Thu Jan 01 00:00:00 UTC 1970
  mZxid = 0x00000000000000
  mtime = Thu Jan 01 00:00:00 UTC 1970
  pZxid = 0x00000000000000
  cversion = 0
  dataVersion = 0
  aclVersion = 1
  ephemeralOwner = 0x00000000000000
  dataLength = 0
----
```
快照分析
　快照文件就很容易看得懂了，这就是Zookeeper整个节点数据的输出；

　第一行：ZNode Details (count=6):
　ZNode节点数总共有6个

/
  cZxid = 0x00000000000000
  ctime = Thu Jan 01 00:00:00 UTC 1970
  mZxid = 0x00000000000000
  mtime = Thu Jan 01 00:00:00 UTC 1970
  pZxid = 0x00000100000002
  cversion = 1
  dataVersion = 0
  aclVersion = 1
  ephemeralOwner = 0x00000000000000
  dataLength = 0

这么一段数据是说，根节点/：
　　cZxid：创建节点时的ZXID
　　ctime：创建节点的时间
　　mZxid：节点最新一次更新发生时的zxid
　　mtime：最近一次节点更新的时间
　　pZxid：父节点的zxid
　　cversion：子节点更新次数
　　dataVersion：节点数据更新次数
　　aclVersion：节点acl更新次数
　　ephemeralOwner：如果节点为ephemeral节点则该值为sessionid，否则为0
　　dataLength：该节点数据的长度

快照文件的末尾：
　　Session Details (sid, timeout, ephemeralCount): 
　　这里是说当前抓取快照文件的时间Zookeeper中Session的详情

### 七、数据恢复
在Zookeeper服务器启动期间，首先会进行数据初始化工作，用于将存储在磁盘上的数据文件加载到Zookeeper服务器内存中。

数据恢复时，会加载最近100个快照文件（如果没有100个，就加载全部的快照文件）。之所以要加载100个，是因为防止最近的那个快照文件不能通过校验。在逐个解析过程中，如果正确性校验通过之后，那么通常就只会解析最新的那个快照文件，但是如果校验发现最先的那个快照文件不可用，那么就会逐个进行解析，直到将这100个文件全部解析完。如果将所有的快照文件都解析后还是无法恢复出一个完整的`DataTree`和`sessionWithTimeouts`，则认为无法从磁盘中加载数据，服务器启动失败。

当基于快照文件构建了一个完整的DataTree实例和sessionWithTimeouts集合后，此时根据这个快照文件的文件名就可以解析出最新的ZXID，该ZXID代表了zookeeper开始进行数据快照的时刻，然后利用此ZXID定位到具体事务文件从哪一个开始，然后执行事务日志对应的事务，恢复到最新的状态，并得到最新的ZXID。

### 八、日志截断
在Zookeeper运行过程中，可能出现非Leader记录的事务ID比Leader上大，这是非法运行状态。此时，需要保证所有机器必须与该Leader的数据保持同步，即Leader会发送TRUNC命令给该机器，要求进行日志截断，Learner收到该命令后，就会删除所有包含或大于该事务ID的事务日志文件。