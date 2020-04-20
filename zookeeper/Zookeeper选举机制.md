## 前言
ZooKeeper服务器会在本地处理只读请求（exists、getData和getChildren）。假如一个服务器接收到客户端的getData请求，服务器读取该状态信息，并将这些信息返回给客户端。因为服务器会在本地处理请求，所以ZooKeeper在处理以只读请求为主要负载时，性能会很高。我们还可以增加更多的服务器到ZooKeeper集群中，这样就可以处理更多的读请求，大幅提高整体处理能力。

那些会改变ZooKeeper状态的客户端请求（create、delete和setData）将会被转发给群首，集群在同一时刻只会存在一个群首，其他服务器追随群首被称为追随者（follower）。群首作为中心点处理所有对ZooKeeper系统变更的请求，它就像一个定序器，建立了所有对ZooKeeper状态的更新的顺序。Leader接收到客户端的请求后，会将请求构建成一个提议(Proposal)，同时会为该提议绑定一个zxid(zxid可以表示执行顺序)，然后将该提议广播到集群上的所有服务器，Leader等待Follwer反馈，当有过半数（>=N/2+1） 的Follower反馈信息后，Leader将再次向集群内Follower广播Commit信息，Commit为将之前的Proposal提交。

* zxid：事务请求唯一标记，由leader服务器负责分配对事务请求进行定序，是8字节的long类型，由两部分组成：前4字节代表epoch，后4字节代表counter，即zxid=epoch+counter。

* epoch可以认为是Leader编号，每一次重新选举出一个新Leader时，都会为该Leader分配一个epoch，该值也是一个递增的，可以防止旧Leader活过来后继续广播之前旧提议造成状态不一致问题，只有当前Leader的提议才会被Follower处理。Leader没有进行选举期间，epoch是一致不会变化的。

* counter：ZooKeeper状态的每一次改变, counter就会递增加1.

* zxid=epoch+counter，其中epoch不会改变，counter每次递增1，,这样zxid就具有递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生。

这就是ZAB协议在处理数据一致性大致的原理流程，由于请求间可能存在着依赖关系，ZAB协议保证Leader广播的变更序列被顺序的处理：一个状态被处理那么它所依赖的状态也已经提前被处理；ZAB协议支持的崩溃恢复可以保证在Leader进程崩溃的时候可以重新选出Leader并且保证数据的完整性。

## 一、选举初始化
Leader选举初始化入口：QuorumPeer.startLeaderElection()，代码如下：
```java
public synchronized void startLeaderElection() {
	try {
		if (getPeerState() == ServerState.LOOKING) {
			currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
		}
	} catch (IOException e) {
		RuntimeException re = new RuntimeException(e.getMessage());
		re.setStackTrace(e.getStackTrace());
		throw re;
	}

	this.electionAlg = createElectionAlgorithm(electionType);
}
```
从上面可看出，当前节点在启动的时候，初始状态都是LOOKING，都会先投自己一票。

然后我们在进入createElectionAlgorithm：
```java
protected Election createElectionAlgorithm(int electionAlgorithm) {
	Election le = null;

	//TODO: use a factory rather than a switch
	switch (electionAlgorithm) {
	case 1:
		throw new UnsupportedOperationException("Election Algorithm 1 is not supported.");
	case 2:
		throw new UnsupportedOperationException("Election Algorithm 2 is not supported.");
	case 3:
		QuorumCnxManager qcm = createCnxnManager();
		QuorumCnxManager oldQcm = qcmRef.getAndSet(qcm);
		if (oldQcm != null) {
			LOG.warn("Clobbering already-set QuorumCnxManager (restarting leader election?)");
			oldQcm.halt();
		}
		QuorumCnxManager.Listener listener = qcm.listener;
		if (listener != null) {
			listener.start();
			FastLeaderElection fle = new FastLeaderElection(this, qcm);
			fle.start();
			le = fle;
		} else {
			LOG.error("Null listener when initializing cnx manager");
		}
		break;
	default:
		assert false;
	}
	return le;
}
```
大致工作如下：
 1、创建一个QuorumCnxManager实例；
 2、启动QuorumCnxManager.Listener线程；
 3、构建一种选举算法FastLeaderElection，早期Zookeeper实现了四种选举算法，但是后面废弃了三种，最新版本只保留FastLeaderElection这一种选举算法；

Leader选举期间集群中各节点之间互相进行投票，就会涉及到网络IO通信，QuorumCnxManager就是用来管理维护选举期间网络IO通信的工具类。

Leader选举涉及到两个核心类：QuorumCnxManager和FastLeaderElection，下面分别详细介绍

## 二、网络IO
QuorumCnxManager维护选举期间的网络IO的大致流程：

### Listener 

1、QuorumCnxManager有一个内部类Listener，其继承了Thread，初始化阶段就会启动该线程，Listener的run方法实现也非常简单：初始化一个ServerSocket，然后在一个while循环中调用accept接收客户端(注意：这里的客户端指的是集群中其它服务器)连接；

```java
public void run() {
    while((!shutdown) && (numRetries < 3)){
        try {
            ss = new ServerSocket();
            ss.setReuseAddress(true);
            addr=根据配置信息获取地址
            setName(addr.toString());
            //监听选举端口
            ss.bind(addr);
            while (!shutdown) {
                try {
                    //接收客户端连接
                    client = ss.accept();
                    //设置连接参数
                    setSockOpts(client);
                    //开始处理
                    receiveConnection(client);
                } catch (SocketTimeoutException e) {
                }
            }
        } catch (IOException e) {
        }
    }
    }
}
```
 2、当有客户端连接进来后，会将该客户端Socket封装成RecvWorker和SendWorker，它们都是线程，分别负责和该Socket所代表的客户端进行读写；RecvWorker和SendWorker是成对出现的，每对负责维护和集群中的一台服务器进行网络IO通信；
```java
public void receiveConnection(final Socket sock) {
    DataInputStream din = null;
    try {
        din = new DataInputStream(new BufferedInputStream(sock.getInputStream()));
        handleConnection(sock, din);
    } catch (IOException e) {
    }
}

private void handleConnection(Socket sock, DataInputStream din) throws IOException {
	...
	
    //这里的思路是如果请求连接的节点的ServerId小于当前节点，则关闭连接，并由当前节点发起连接
    //隐含的意思就是ZK集群中节点的连接都是由ServerId大的连ServerId小的
    if (sid < self.getId()) {
        //如果连接已经建立则关闭
        SendWorker sw = senderWorkerMap.get(sid);
        if (sw != null) {
            sw.finish();
        }
        closeSocket(sock);
        //当前节点去连接对方节点
        if (electionAddr != null) {
            connectOne(sid, electionAddr);
        } else {
            connectOne(sid);
        }
    } else {
        //如果接受该连接，则创建对应的读写worker
        SendWorker sw = new SendWorker(sock, sid);
        RecvWorker rw = new RecvWorker(sock, din, sid, sw);
        sw.setRecv(rw);
        //如果已经创建则关闭旧的
        SendWorker vsw = senderWorkerMap.get(sid);
        if (vsw != null) {
            vsw.finish();
        }
        senderWorkerMap.put(sid, sw);
        queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
        //启动读写事件处理
        sw.start();
        rw.start();
    }
}
```
对于两个worker来说，它们本身的逻辑很简单，SendWorker就是不断的把queueSendMap中存放的对应serverId的数据发出去。RecvWorker就是把收到的数据加入recvQueue队列.

现在假设这个场景：集群中存在A、B两个节点：
* 当A节点连接B节点时，在B节点上会维护一对RecvWorker和SendWorker用于B节点和A节点进行通信；
* 同理，如果B节点连接A节点，在A节点上会维护一对RecvWorker和SendWorker用于A节点和B节点进行通信；
* A和B之间创建了两条通道，实际上A和B间通信只需要一条通道即可，为避免浪费资源，Zookeeper采用如下原则：myid小的一方作为服务端，否则连接无效会被关闭；
* 比如A的myid是1，B的myid是2，如果A去连接B，B收到连接请求后B发现对端myid小于自己，判定该连接无效，会关闭该连接；
  如果是B连接A，A收到连接请求后发现对端myid大于自己，则认为该连接有效，并会为该连接创建一对RecvWorker和SendWorker线程并启动

### FastLeaderElection
1、FastLeaderElection负责Leader选举核心规则算法实现，注意FastLeaderElection类中也包含了两个内部类WorkerSender和WorkerReceiver，类似于QuorumCnxManager中的SendWorker和RecvWorker，也是用于发送和接收线程；
```java
public void start() {
    this.messenger.start();
}

void start(){
    //对应WorkerSender类
    this.wsThread.start();
    //对应WorkerReceiver类
    this.wrThread.start();
}
```
这里可以看到FastLeaderElection内部也是开启了两个线程负责读写，这里需要跟前面Listener的逻辑结合考虑。Listener开启的线程一个负责读取数据放入队列，一个负责把队列中的数据发出去，但读取的数据给谁用呢？发送的数据是哪来的呢？FastLeaderElection里的两线程就是跟它们交互的。

2、WorkSender线程代码如下：
```java
public void run() {
    while (!stop) {
        try {
            ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
            if(m == null) continue;
            //处理发送消息
            process(m);
        } catch (InterruptedException e) {
            break;
        }
    }
}

void process(ToSend m) {
    //序列化消息
    ByteBuffer requestBuffer = buildMsg(m.state.ordinal(),
                                        m.leader,
                                        m.zxid,
                                        m.electionEpoch,
                                        m.peerEpoch,
                                        m.configData);
    //发送数据
    manager.toSend(m.sid, requestBuffer);

}
public void toSend(Long sid, ByteBuffer b) {
    //如果数据时发送给自己的那么绕过IO直接加入到recv队列
    if (this.mySid == sid) {
         b.position(0);
         addToRecvQueue(new Message(b.duplicate(), sid));
    } else {
         //否则把数据加入到指定ServerId的待发送队列
         ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
         ArrayBlockingQueue<ByteBuffer> oldq = queueSendMap.putIfAbsent(sid, bq);
         if (oldq != null) {
             addToSendQueue(oldq, b);
         } else {
             addToSendQueue(bq, b);
         }
         //连接指定ServerId，该方法内部如果连接已经建立则会返回，否则创建连接
         connectOne(sid);
            
    }
}
```
从上面可看出，FastLeaderElection中进行选举广播投票信息时，将投票信息写入到对端服务器大致流程如下：
* 将数据封装成ToSend格式放入到sendqueue；
* WorkerSender线程会一直轮询提取sendqueue中的数据，当提取到ToSend数据后，会获取到集群中所有参与Leader选举节点(除Observer节点外的节点)的sid，如果sid即为本机节点，则转成Notification直接放入到recvqueue中，因为本机不再需要走网络IO；否则放入到queueSendMap中，key是要发送给哪个服务器节点的sid，ByteBuffer即为ToSend的内容，queueSendMap维护的着当前节点要发送的网络数据信息，由于发送到同一个sid服务器可能存在多条数据，所以queueSendMap的value是一个queue类型；
* QuorumCnxManager中的SendWorkder线程不停轮询queueSendMap中是否存在自己要发送的数据，每个SendWorkder线程都会绑定一个sid用于标记该SendWorkder线程和哪个对端服务器进行通信，因此，queueSendMap.get(sid)即可获取该线程要发送数据的queue，然后通过queue.poll()即可提取该线程要发送的数据内容；
* 然后通过调用SendWorkder内部维护的socket输出流即可将数据写入到对端服务器

3、WorkerReceiver线程代码如下：
```java
public void run() {

    Message response;
    while (!stop) {
        try {
            //这里本质上是从recvQueue里取出数据
            response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
            //没有数据则继续等待
            if(response == null) continue;
            
			...
			
            int rstate = response.buffer.getInt();
            long rleader = response.buffer.getLong();
            long rzxid = response.buffer.getLong();
            long relectionEpoch = response.buffer.getLong();
            long rpeerepoch;
            QuorumVerifier rqv = null;
            //如果不是一个有投票权的节点，例如Observer节点
            if(!validVoter(response.sid)) {
                //直接把自己的投票信息返回
                Vote current = self.getCurrentVote();
                QuorumVerifier qv = self.getQuorumVerifier();
                ToSend notmsg = new ToSend(ToSend.mType.notification,
                        current.getId(),
                        current.getZxid(),
                        logicalclock.get(),
                        self.getPeerState(),
                        response.sid,
                        current.getPeerEpoch(),
                        qv.toString().getBytes());
                sendqueue.offer(notmsg);
            } else {
                //获取发消息的节点的状态
                QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                switch (rstate) {
                case 0:
                    ackstate = QuorumPeer.ServerState.LOOKING;
                    break;
                case 1:
                    ackstate = QuorumPeer.ServerState.FOLLOWING;
                    break;
                case 2:
                    ackstate = QuorumPeer.ServerState.LEADING;
                    break;
                case 3:
                    ackstate = QuorumPeer.ServerState.OBSERVING;
                    break;
                default:
                    continue;
                }

                //赋值Notification
                n.leader = rleader;
                n.zxid = rzxid;
                n.electionEpoch = relectionEpoch;
                n.state = ackstate;
                n.sid = response.sid;
                n.peerEpoch = rpeerepoch;
                n.version = version;
                n.qv = rqv;
                //如果当前节点正在寻找Leader
                if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){
                    //把收到的消息加入队列
                    recvqueue.offer(n);
                    //如果对方节点也是LOOKING状态，且周期小于自己，则把自己投票信息发回去
                    if((ackstate == QuorumPeer.ServerState.LOOKING) && (n.electionEpoch < logicalclock.get())){
                        Vote v = getVote();
                        QuorumVerifier qv = self.getQuorumVerifier();
                        ToSend notmsg = new ToSend(ToSend.mType.notification,
                                v.getId(),
                                v.getZxid(),
                                logicalclock.get(),
                                self.getPeerState(),
                                response.sid,
                                v.getPeerEpoch(),
                                qv.toString().getBytes());
                        sendqueue.offer(notmsg);
                    }
                } else {
                    //如果当前节点不是LOOKING状态，那么它已经知道谁是Leader了
                    Vote current = self.getCurrentVote();
                    //如果对方是LOOKING状态，那么就把自己认为的Leader信息返给对方
                    if(ackstate == QuorumPeer.ServerState.LOOKING){
                        QuorumVerifier qv = self.getQuorumVerifier();
                        ToSend notmsg = new ToSend(
                                ToSend.mType.notification,
                                current.getId(),
                                current.getZxid(),
                                current.getElectionEpoch(),
                                self.getPeerState(),
                                response.sid,
                                current.getPeerEpoch(),
                                qv.toString().getBytes());
                        sendqueue.offer(notmsg);
                    }
                }
            }
        } catch (InterruptedException e) {
        }
    }
}
```
从上面代码可看出，FastLeaderElection中进行选举广播投票信息时，从对端服务器读取投票信息的大致流程如下：
* QuorumCnxManager中的RecvWorker线程会一直从Socket的输入流中读取数据，当读取到对端发送过来的数据时，转成Message格式并放入到recvQueue中；
* FastLeaderElection.WorkerReceiver线程会轮询方式从recvQueue提取数据并转成Notification格式放入到recvqueue中；
* FastLeaderElection从recvqueu提取所有的投票信息进行比较 最终选出一个Leader

## 三、leader选举
之前的文章已介绍过Zookeeper集群启动的大致流程，QuorumPeer线程中会有一个Loop循环，获取serverState状态后进入不同分支，当分支退出后继续下次循环，FastLeaderElection选举策略调用就是发生在检测到serverState状态为LOOKING时进入到LOOKING分支中调用的。分支代码如下：
```java
case LOOKING:
	LOG.info("LOOKING");
	ServerMetrics.getMetrics().LOOKING_COUNT.add(1);

	if (Boolean.getBoolean("readonlymode.enabled")) {
		LOG.info("Attempting to start ReadOnlyZooKeeperServer");

		// Create read-only server but don't start it immediately
		final ReadOnlyZooKeeperServer roZk = new ReadOnlyZooKeeperServer(logFactory, this, this.zkDb);

		// Instead of starting roZk immediately, wait some grace
		// period before we decide we're partitioned.
		//
		// Thread is used here because otherwise it would require
		// changes in each of election strategy classes which is
		// unnecessary code coupling.
		Thread roZkMgr = new Thread() {
			public void run() {
				try {
					// lower-bound grace period to 2 secs
					sleep(Math.max(2000, tickTime));
					if (ServerState.LOOKING.equals(getPeerState())) {
						roZk.startup();
					}
				} catch (InterruptedException e) {
					LOG.info("Interrupted while attempting to start ReadOnlyZooKeeperServer, not started");
				} catch (Exception e) {
					LOG.error("FAILED to start ReadOnlyZooKeeperServer", e);
				}
			}
		};
		try {
			roZkMgr.start();
			reconfigFlagClear();
			if (shuttingDownLE) {
				shuttingDownLE = false;
				startLeaderElection();
			}
			setCurrentVote(makeLEStrategy().lookForLeader());
		} catch (Exception e) {
			LOG.warn("Unexpected exception", e);
			setPeerState(ServerState.LOOKING);
		} finally {
			// If the thread is in the the grace period, interrupt
			// to come out of waiting.
			roZkMgr.interrupt();
			roZk.shutdown();
		}
	} else {
		try {
			reconfigFlagClear();
			if (shuttingDownLE) {
				shuttingDownLE = false;
				startLeaderElection();
			}
			setCurrentVote(makeLEStrategy().lookForLeader());
		} catch (Exception e) {
			LOG.warn("Unexpected exception", e);
			setPeerState(ServerState.LOOKING);
		}
	}
	break;
```
从上面代码可以看出，Leader选举策略入口方法为：FastLeaderElection.lookForLeader()方法。当QuorumPeer.serverState变成LOOKING时，该方法会被调用，表示执行新一轮Leader选举。下面来看下lookForLeader方法的大致实现逻辑：

1、更新自己期望投票信息，即自己期望选哪个服务器作为Leader(用sid代替期望服务器节点)以及该服务器zxid、epoch等信息，第一次投票默认都是投自己当选Leader，然后调用sendNotifications方法广播该投票到集群中所有可以参与投票服务器，代码如下：
```java
synchronized (this) {
	logicalclock.incrementAndGet();
	updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
}

LOG.info(
	"New election. My id = {}, proposed zxid=0x{}",
	self.getId(),
	Long.toHexString(proposedZxid));
sendNotifications();
```
logicalclock维护electionEpoch，即选举轮次，在进行投票结果赛选的时候需要保证大家在一个投票轮次
updateProposal()方法有三个参数：a.期望投票给哪个服务器(sid)、b.该服务器的zxid、c.该服务器的epoch，在后面会看到这三个参数是选举Leader时的核心指标:
* getInitId()用于获取当前myid
* getInitLastLoggedZxid()提取lastProcessedZxid值，lastProcessedZxid是最后一次commit的事务请求的zxid
* getPeerEpoch()：获取epoch值，每个leader任期内都要有一个epoch代表该Leader轮次，同时把该epoch同步到集群送的所有其它节点，并会被保存到本地硬盘dataLogDir目录下currentEpoch文件中，这里的getPeerEpoch()就是获取最近一次Leader的epoch，如果是第一次部署启动则默认从0开始

发送给集群中所有可参与投票节点，注意也包括自身节点:
* 将proposedLeader、proposedZxid、electionEpoch、peerEpoch、sid(要发送给哪个节点的sid)等信息封装为一个ToSend对象，并放入到LinkedBlockingQueue<ToSend> sendqueue队列中，注意遍历集群中所有参与投票节点的sid，为每个sid封装成一个ToSend
* WorkerSender线程将会从sendqueue队列中获取要发送消息根据sid发送给集群中指定的节点

2、然后就开始等待其它服务器发送给自己的投票信息

3、将接收到投票的state进行判断确定执行哪个分支逻辑：
* 如果是FOLLOWING或LEADING，则说明对端已选举出Leader，这时只需要验证下这个Leader是否有效即可，有效则代表选举结束，否则继续接收投票信息
* OBSERVING：忽略该投票信息，因为Observer不能参与投票
* LOOKING：则表示对端也还处于Leader选举状态

4、LOOKING状态
```java
case LOOKING:
	if (getInitLastLoggedZxid() == -1) {
		LOG.debug("Ignoring notification as our zxid is -1");
		break;
	}
	if (n.zxid == -1) {
		LOG.debug("Ignoring notification from member with -1 zxid {}", n.sid);
		break;
	}
	// If notification > current, replace and send messages out
	if (n.electionEpoch > logicalclock.get()) {
		logicalclock.set(n.electionEpoch);
		recvset.clear();
		if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
			updateProposal(n.leader, n.zxid, n.peerEpoch);
		} else {
			updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
		}
		sendNotifications();
	} else if (n.electionEpoch < logicalclock.get()) {
			LOG.debug(
				"Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x{}, logicalclock=0x{}",
				Long.toHexString(n.electionEpoch),
				Long.toHexString(logicalclock.get()));
		break;
	} else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
		updateProposal(n.leader, n.zxid, n.peerEpoch);
		sendNotifications();
	}

	LOG.debug(
		"Adding vote: from={}, proposed leader={}, proposed zxid=0x{}, proposed election epoch=0x{}",
		n.sid,
		n.leader,
		Long.toHexString(n.zxid),
		Long.toHexString(n.electionEpoch));

	// don't care about the version if it's in LOOKING state
	recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));

	voteSet = getVoteTracker(recvset, new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch));

	if (voteSet.hasAllQuorums()) {

		// Verify if there is any change in the proposed leader
		while ((n = recvqueue.poll(finalizeWait, TimeUnit.MILLISECONDS)) != null) {
			if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch, proposedLeader, proposedZxid, proposedEpoch)) {
				recvqueue.put(n);
				break;
			}
		}

		/*
		 * This predicate is true once we don't read any new
		 * relevant message from the reception queue
		 */
		if (n == null) {
			setPeerState(proposedLeader, voteSet);
			Vote endVote = new Vote(proposedLeader, proposedZxid, logicalclock.get(), proposedEpoch);
			leaveInstance(endVote);
			return endVote;
		}
	}
	break;
```
首先对之前提到的选举轮次electionEpoch进行判断，这里分为三种情况：
* 只有对方发过来的投票的electionEpoch和当前节点相等表示是同一轮投票，即投票有效，然后调用totalOrderPredicate()对投票进行PK，返回true代表对端胜出，则表示第一次投票是错误的(第一次都是投给自己)，更新自己投票期望对端为Leader，然后调用sendNotifications()将自己最新的投票广播出去。返回false则代表自己胜出，第一次投票没有问题，就不用管
* 如果对端发过来的electionEpoch大于自己，则表明重置自己的electionEpoch，然后清空之前获取到的所有投票recvset，因为之前获取的投票轮次落后于当前则代表之前的投票已经无效了，然后调用totalOrderPredicate()将当前期望的投票和对端投票进行PK，用胜出者更新当前期望投票，然后调用sendNotifications()将自己期望头破广播出去。注意：这里不管哪一方胜出，都需要广播出去，而不是步骤a中己方胜出不需要广播，这是因为由于electionEpoch落后导致之前发出的所有投票都是无效的，所以这里需要重新发送
* 如果对端发过来的electionEpoch小于自己，则表示对方投票无效，直接忽略不进行处理

## 四、选举PK机制
下面来看下`totalOrderPredicate`方法
```java
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
	LOG.debug(
		"id: {}, proposed id: {}, zxid: 0x{}, proposed zxid: 0x{}",
		newId,
		curId,
		Long.toHexString(newZxid),
		Long.toHexString(curZxid));

	if (self.getQuorumVerifier().getWeight(newId) == 0) {
		return false;
	}

	/*
	 * We return true if one of the following three cases hold:
	 * 1- New epoch is higher
	 * 2- New epoch is the same as current epoch, but new zxid is higher
	 * 3- New epoch is the same as current epoch, new zxid is the same
	 *  as current zxid, but server id is higher.
	 */

	return ((newEpoch > curEpoch)
			|| ((newEpoch == curEpoch)
				&& ((newZxid > curZxid)
					|| ((newZxid == curZxid)
						&& (newId > curId)))));
}
```
这个PK逻辑原理(胜出一方代表更有希望成为Leader)如下：
1、首先比较epoch，哪个epoch大哪个胜出，前面介绍过epoch代表了Leader的轮次，是一个递增的，epoch越大就意味着数据越新，Leader数据越新则可以减少后续数据同步的效率，当然应该优先选为Leader；
 2、然后才是比较zxid，由于zxid=epoch+counter，第一步已经把epoch比较过了，其实这步骤只是相当于比较counter大小，counter越大则代表数据越新，优先选为Leader。注：其实第1和第2可以合并到一起，直接比较zxid即可，因为zxid=epoch+counter，第1比较显的有些多余

3、如果前两个指标都没法比较出来，只能通过sid来确定，zxid相等说明两个服务器的数据是一致的，所以选哪个当Leader其实没有区别，这里就随机选择一个sid大的当Leader