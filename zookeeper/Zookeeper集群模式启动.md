## 启动入口
通过zkServer.sh启动ZooKeeper时，应用的统一入口为QuorumPeerMain。此处Quorum的含义是“保证数据冗余和最终一致的机制”，Peer表示集群中的一个平等地位节点。
```java
protected void initializeAndRun(String[] args) throws ConfigException, IOException, AdminServerException {
	QuorumPeerConfig config = new QuorumPeerConfig();
	if (args.length == 1) {
		config.parse(args[0]);
	}

	// Start and schedule the the purge task
	DatadirCleanupManager purgeMgr = new DatadirCleanupManager(
		config.getDataDir(),
		config.getDataLogDir(),
		config.getSnapRetainCount(),
		config.getPurgeInterval());
	purgeMgr.start();

	if (args.length == 1 && config.isDistributed()) {
		runFromConfig(config);
	} else {
		LOG.warn("Either no config or no quorum defined in config, running in standalone mode");
		// there is only server in the quorum -- run as standalone
		ZooKeeperServerMain.main(args);
	}
}
```
从代码中可以看出：
QuorumPeerMain会做一个判断，当使用配置文件(args.length == 1)且是集群配置的情况下，启动集群形式QuorumPeer，否则启动单机模式ZooKeeperServer。这里还启动了DatadirCleanupManager，用于清理早期的版本快照文件。

下面讲解集群模式启动

```java
#如果zoo.cfg中配置如下信息，代码中servers就会大于0，就会以集群环境运行；否则以standalone模式运行
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889
```

## 一、指标收集
```java
try {
	metricsProvider = MetricsProviderBootstrap.startMetricsProvider(
		config.getMetricsProviderClassName(),
		config.getMetricsProviderConfiguration());
} catch (MetricsProviderLifeCycleException error) {
	throw new IOException("Cannot boot MetricsProvider " + config.getMetricsProviderClassName(), error);
}
ServerMetrics.metricsProviderInitialized(metricsProvider);
```
MetricsProvider是一个可以收集指标、将当前值发送到外部设备的一个组件。数据在server端和client端可以共享

## 二、创建ServerCnxnFactory实例
```java
ServerCnxnFactory cnxnFactory = null;
ServerCnxnFactory secureCnxnFactory = null;

if (config.getClientPortAddress() != null) {
	cnxnFactory = ServerCnxnFactory.createFactory();
	cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
}

if (config.getSecureClientPortAddress() != null) {
	secureCnxnFactory = ServerCnxnFactory.createFactory();
	secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
}
```
ServerCnxnFactory从名字就可以看出其是一个工厂类，负责管理ServerCnxn，ServerCnxn这个类代表了一个客户端与一个server的连接，每个客户端连接过来都会被封装成一个ServerCnxn实例用来维护了服务器与客户端之间的Socket通道。
首先要有监听端口，客户端连接才能过来，ServerCnxnFactory.configure()方法的核心就是启动监听端口供客户端连接进来，端口号由配置文件中clientPort属性进行配置，默认是2181

## 三、初始化QuorumPeer
```java
quorumPeer = getQuorumPeer();
quorumPeer.setTxnFactory(new FileTxnSnapLog(config.getDataLogDir(), config.getDataDir()));
quorumPeer.enableLocalSessions(config.areLocalSessionsEnabled());
quorumPeer.enableLocalSessionsUpgrading(config.isLocalSessionsUpgradingEnabled());
//quorumPeer.setQuorumPeers(config.getAllMembers());
quorumPeer.setElectionType(config.getElectionAlg()); //选举类型，用于确定选举算法
quorumPeer.setMyid(config.getServerId()); //myid用于区分不同端
quorumPeer.setTickTime(config.getTickTime());
quorumPeer.setMinSessionTimeout(config.getMinSessionTimeout());
quorumPeer.setMaxSessionTimeout(config.getMaxSessionTimeout());
quorumPeer.setInitLimit(config.getInitLimit());
quorumPeer.setSyncLimit(config.getSyncLimit());
quorumPeer.setConnectToLearnerMasterLimit(config.getConnectToLearnerMasterLimit());
quorumPeer.setObserverMasterPort(config.getObserverMasterPort());
quorumPeer.setConfigFileName(config.getConfigFilename());
quorumPeer.setClientPortListenBacklog(config.getClientPortListenBacklog());
quorumPeer.setZKDatabase(new ZKDatabase(quorumPeer.getTxnFactory())); //ZKDatabase维护ZK在内存中的数据结构
quorumPeer.setQuorumVerifier(config.getQuorumVerifier(), false);
if (config.getLastSeenQuorumVerifier() != null) {
	quorumPeer.setLastSeenQuorumVerifier(config.getLastSeenQuorumVerifier(), false);
}
quorumPeer.initConfigInZKDatabase();
quorumPeer.setCnxnFactory(cnxnFactory); //ServerCnxnFactory客户端请求管理工厂类
quorumPeer.setSecureCnxnFactory(secureCnxnFactory);
quorumPeer.setSslQuorum(config.isSslQuorum());
quorumPeer.setUsePortUnification(config.shouldUsePortUnification());
quorumPeer.setLearnerType(config.getPeerType());
quorumPeer.setSyncEnabled(config.getSyncEnabled());
quorumPeer.setQuorumListenOnAllIPs(config.getQuorumListenOnAllIPs());
if (config.sslQuorumReloadCertFiles) {
	quorumPeer.getX509Util().enableCertFileReloading();
}

// sets quorum sasl authentication configurations
quorumPeer.setQuorumSaslEnabled(config.quorumEnableSasl);
if (quorumPeer.isQuorumSaslAuthEnabled()) {
	quorumPeer.setQuorumServerSaslRequired(config.quorumServerRequireSasl);
	quorumPeer.setQuorumLearnerSaslRequired(config.quorumLearnerRequireSasl);
	quorumPeer.setQuorumServicePrincipal(config.quorumServicePrincipal);
	quorumPeer.setQuorumServerLoginContext(config.quorumServerLoginContext);
	quorumPeer.setQuorumLearnerLoginContext(config.quorumLearnerLoginContext);
}
quorumPeer.setQuorumCnxnThreadsSize(config.quorumCnxnThreadsSize);
quorumPeer.initialize();

if (config.jvmPauseMonitorToRun) {
	quorumPeer.setJvmPauseMonitor(new JvmPauseMonitor(config));
}

quorumPeer.start();
ZKAuditProvider.addZKStartStopAuditLog();
quorumPeer.join();
```
Quorum在Zookeeper中代表集群中大多数节点的意思，即一半以上节点，Peer是端、节点的意思，Zookeeper集群中一半以上的节点其实就可以代表整个集群的状态，QuorumPeer就是管理维护的整个集群的一个核心类.
这一步主要是创建一个QuorumPeer实例，并进行各种初始化工作，

QuorumPeer.start()是Zookeeper中非常重要的一个方法入口，其代码如下：
```java
public synchronized void start() {
	if (!getView().containsKey(myid)) {
		throw new RuntimeException("My id " + myid + " not in the peer list");
	}
	loadDataBase();
	startServerCnxnFactory();
	try {
		adminServer.start();
	} catch (AdminServerException e) {
		LOG.warn("Problem starting AdminServer", e);
		System.out.println(e);
	}
	startLeaderElection();
	startJvmPauseMonitor();
	super.start();
}
```
start方法实现的业务主要包含四个方面：
1、loadDataBase：涉及到的核心类是ZKDatabase，并借助于FileTxnSnapLog工具类将snap和transaction log反序列化到内存中，最终构建出内存数据结构DataTree

2、startServerCnxnFactory：之前介绍过ServerCnxnFactory作用，ServerCnxnFactory本身也可以作为一个线程，其run方法实现的大致逻辑是：构建reactor模型的EventLoop，Selector每隔1秒执行一次select方法来处理IO请求，并分发到对应的代表该客户端的ServerCnxn中并利用doIO进行处理。

3、startLeaderElection()：这个主要是初始化一些Leader选举工作，所有节点启动的初始状态都是LOOKING，因此这里都会是创建一张投自己为Leader的票
这部分的关键代码在QuorumPeer.createElectionAlgorithm，大致如下：
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
			listener.start(); //Listener是一个线程，这里启动Listener线程，主要启动选举监听端口并处理连接进来的Socket
			FastLeaderElection fle = new FastLeaderElection(this, qcm); //选举算法使用FastLeaderElection，存在好几种算法实现，但是其它集中算法实现都已经慢慢废弃
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
Leader选举涉及到节点间的网络IO，QuorumCnxManager就是负责集群中各节点的网络IO，QuorumCnxManager包含一个内部类Listener，Listener是一个线程，这里启动Listener线程，主要启动选举监听端口并处理连接进来的Socket；FastLeaderElection就是封装了具体选举算法的实现。

具体的选举逻辑放在下一篇文章进展讲解