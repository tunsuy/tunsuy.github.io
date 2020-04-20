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

下面讲解单机模式启动

启动类：ZooKeeperServerMain
initializeAndRun方法

## 一、加载配置
```java
protected void initializeAndRun(String[] args)throws ConfigException, IOException{
	ServerConfig config = new ServerConfig();
	if (args.length == 1) {
		config.parse(args[0]);
	} else {
		config.parse(conf);
	}
	runFromConfig(config);
}
```
命令输入的参数会有两种情况：
* 输入了配置文件名称
* 直接输入了配置参数，比如按顺序输入了clientPortAddress，dataDir，dataLogDir，tickTime，maxClientCnxns
  所以要根据这两种情况进行不同的解析
  注：必须按照顺序，因为代码中是按照固定顺序解析的。

下面介绍几个比较常用的配置项：
* clientPort：对外服务端口，一般2181
* dataDir：存储快照文件的目录，默认情况下，事务日志文件也会放在这
* tickTime：ZK中的一个时间单元。ZK中所有时间都是以这个时间单元为基础，进行整数倍配置
* preAllocSize：预先开辟磁盘空间，用于后续写入事务日志，默认64M
* snapCount：每进行snapCount次事务日志输出后，触发一次快照，默认是100,000
* maxClientCnxns：最大并发客户端数，默认是60

## 二、指标收集
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

## 三、事务日志和快照数据文件
```java
txnLog = new FileTxnSnapLog(config.dataLogDir, config.dataDir);
JvmPauseMonitor jvmPauseMonitor = null;
if (config.jvmPauseMonitorToRun) {
	jvmPauseMonitor = new JvmPauseMonitor(config);
}
final ZooKeeperServer zkServer = new ZooKeeperServer(jvmPauseMonitor, txnLog, config.tickTime, config.minSessionTimeout, config.maxSessionTimeout, config.listenBacklog, null, config.initialConfig);
txnLog.setServerStats(zkServer.serverStats());
```
FileTxnSnapLog是ZooKeeper上层服务于底层数据存储之间的对接层，提供了一系列操作数据文件的接口。包括事务日志文件和快照数据文件。根据dataDir和snapDir来创建FileTxnSnapLog。
```java
this.dataDir = new File(dataDir, version + VERSION);
this.snapDir = new File(snapDir, version + VERSION);
```

主要的方法有如下一些，在其他文章作为专题来讲
```java
// 读取快照文件和事务日志之后恢复服务端数据
restore(DataTree, Map, Integer>, PlayBackListener):long

// 把最新的事务日志快速更新到server database，与restore不同的是只处理事务日志
fastForwardFromEdits(DataTree, ap, Integer>,PlayBackListener)：long

// 把dataTree和sessions放入快照文件中
save(DataTree,ConcurrentHashMap, Integer>, boolean):void

// 在dataTree上处理事务
processTransaction(TxnHeader,DataTree ,Map, Integer> , Record ):void

// 读取快照文件和事务日志之后恢复服务端数据
restore(DataTree, Map, Integer>, PlayBackListener):long

// 把最新的事务日志快速更新到server database，与restore不同的是只处理事务日志
fastForwardFromEdits(DataTree, Map, Integer>,PlayBackListener)：long

// 把dataTree和sessions放入快照文件中
save(DataTree,ConcurrentHashMap, Integer>, boolean):void

// 在dataTree上处理事务
processTransaction(TxnHeader,DataTree ,Map, Integer> , Record ):void
```

## 四、服务下线回调
```java
// Registers shutdown handler which will be used to know the
// server error or shutdown state changes.
final CountDownLatch shutdownLatch = new CountDownLatch(1);
zkServer.registerServerShutdownHandler(new ZooKeeperServerShutdownHandler(shutdownLatch));
```
创建CountDownLatch，用来watch zk的状态，当zk关闭或者出现内部错误的时候优雅的关闭服务
ZooKeeperServerShutdownHandler是zk用于处理异常的组件。当系统发生错误时，会使用CountDownLatch通知其他线程停止工作

## 五、启动adminServer
```java 
// Start Admin server
adminServer = AdminServerFactory.createAdminServer();
adminServer.setZooKeeperServer(zkServer);
adminServer.start();
```
AdminServer用来管理ZooKeeperServer。有两种实现方式JettyAdminServer和DummyAdminServer。
当zookeeper.admin.enableServer为true时才启动AdminServer，通过反射的方式创建实例

AdminServer是3.5.0版本中新增特性，是一个内置的Jettry服务，它提供了一个HTTP接口为四字母单词命令。默认的，服务被启动在8080端口，并且命令被发起通过URL "/commands/[command name]",例如，http://localhost:8080/commands/stat。命令响应以JSON的格式返回。不像原来的协议，命令不是限制为四字母的名字，并且命令可以有多个名字。例如"stmk"可以被指定为"set_trace_mask"。为了查看所有可用命令的列表，指向一个浏览器的URL/commands (例如， http://localhost:8080/commands)。

AdminServer默认开启，但是可以被关闭通过下面的方法：
* 设置系统属性zookeeper.admin.enableServer为false.
* 从类路径中移除Jetty.(这个选项是有用的如果你想覆盖ZooKeeper的jetty依赖)。
  注意TCP四字母单词接口是仍然可用的如果AdminServer被关闭。

## 六、启动zkserver
```java
boolean needStartZKServer = true;
if (config.getClientPortAddress() != null) {
	cnxnFactory = ServerCnxnFactory.createFactory();
	cnxnFactory.configure(config.getClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), false);
	cnxnFactory.startup(zkServer);
	// zkServer has been started. So we don't need to start it again in secureCnxnFactory.
	needStartZKServer = false;
}
if (config.getSecureClientPortAddress() != null) {
	secureCnxnFactory = ServerCnxnFactory.createFactory();
	secureCnxnFactory.configure(config.getSecureClientPortAddress(), config.getMaxClientCnxns(), config.getClientPortListenBacklog(), true);
	secureCnxnFactory.startup(zkServer, needStartZKServer);
}
```
早期版本，都是自己实现NIO框架，从3.4.0版本引入了Netty，可以通过zookeeper.serverCnxnFactory来指定使用NIO还是Netty作为Zookeeper服务端网络连接工厂。
注：还会根据是否配置了安全，决定是否安全的启动zk服务器

cnxnFactory.startup(zkServer)，启动zk服务器
### 1、恢复本地数据
```java
public void startdata() throws IOException, InterruptedException {
	//check to see if zkDb is not null
	if (zkDb == null) {
		zkDb = new ZKDatabase(this.txnLogFactory);
	}
	if (!zkDb.isInitialized()) {
		loadData();
	}
}
```
从本地快照和事务日志文件中进行数据恢复

### 2、创建并启动会话管理器
```java
if (sessionTracker == null) {
	createSessionTracker();
}
startSessionTracker();
```
创建会话管理器SessionTracker，SessionTracker主要负责ZooKeeper服务端的会话管理，创建SessionTracker时，会设置expireInterval、NextExpirationTime和SessionWithTimeout，还会计算出一个初始化的SessionID

### 3、初始化ZooKeeper的请求处理链
```java
protected void setupRequestProcessors() {
	RequestProcessor finalProcessor = new FinalRequestProcessor(this);
	RequestProcessor syncProcessor = new SyncRequestProcessor(this, finalProcessor);
	((SyncRequestProcessor) syncProcessor).start();
	firstProcessor = new PrepRequestProcessor(this, syncProcessor);
	((PrepRequestProcessor) firstProcessor).start();
}
```
典型的责任链方式实现，在ZooKeeper服务器上，会有多个请求处理器一次来处理一个客户端请求，在服务器启动的时候，会将这些请求处理器串联起来形成一个请求处理链。单机版服务器的请求处理链主要包括PrepRequestProcessor、SyncRequestProcessor、FinalRequestProcessor

### 4、注册JMX服务
```java
protected void registerJMX() {
	// register with JMX
	try {
		jmxServerBean = new ZooKeeperServerBean(this);
		MBeanRegistry.getInstance().register(jmxServerBean, null);

		try {
			jmxDataTreeBean = new DataTreeBean(zkDb.getDataTree());
			MBeanRegistry.getInstance().register(jmxDataTreeBean, jmxServerBean);
		} catch (Exception e) {
			LOG.warn("Failed to register with JMX", e);
			jmxDataTreeBean = null;
		}
	} catch (Exception e) {
		LOG.warn("Failed to register with JMX", e);
		jmxServerBean = null;
	}
}
```
ZooKeeper会将服务器运行时的一些信息以JMS的方式暴露给外部

## 七、定时清理
```java
containerManager = new ContainerManager(
	zkServer.getZKDatabase(),
	zkServer.firstProcessor,
	Integer.getInteger("znode.container.checkIntervalMs", (int) TimeUnit.MINUTES.toMillis(1)),
	Integer.getInteger("znode.container.maxPerMinute", 10000),
	Long.getLong("znode.container.maxNeverUsedIntervalMs", 0)
);
containerManager.start();
```
创建定时清除容器节点管理器，用于处理容器节点下不存在子节点的清理容器节点工作等