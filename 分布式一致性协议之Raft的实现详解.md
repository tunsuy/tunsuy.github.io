# 分布式一致性协议之Raft的实现详解

到目前为止，不管是哪门语言，应该都已经有一些raft协议的实现了。但是大家的实现也都是根据raft协议论文来的，根据自己的服务形态在细节上有一些差异而已，大体上是一样的，因此今天这里就以golang语言中的raft实现库为例，进行剖析，让大家实际感受下raft的流程。

hashicorp/raft 是常用的 Golang 版 Raft 算法的实现，被众多流行软件使用，如 Consul、InfluxDB、IPFS 等
raft.go 是 Hashicorp Raft 的核心代码文件，大部分的核心功能都是在这个文件中实现。
在 Hashicorp Raft 中，支持两种节点间通讯机制，内存型和 TCP 协议型，其中，内存型通讯机制，主要用于测试，2 种通讯机制的代码实现，分别在文件 inmem_transport.go 和 tcp_transport.go 中。


### 概念与术语
* leader：领导者，提供服务(生成写日志)的节点，任何时候raft系统中只能有一个leader。
* follower：跟随者，被动接受请求的节点，不会发送任何请求，只会响应来自leader或者candidate的请求。如果接受到客户请求，会转发给leader。
* candidate：候选人，选举过程中产生，follower在超时时间内没有收到leader的心跳或者日志，则切换到candidate状态，进入选举流程。
* termId：任期号，时间被划分成一个个任期，每次选举后都会产生一个新的termId，一个任期内只有一个leader。
* RequestVote：请求投票，candidate在选举过程中发起，收到quorum(多数派）响应后，成为leader。
* AppendEntries：附加日志，leader发送日志和心跳的机制


### 多数派计算:
```go
func (r *Raft) quorumSize() int {
	voters := 0
	for _, server := range r.configurations.latest.Servers {
		if server.Suffrage == Voter {
			voters++
		}
	}
	return voters/2 + 1
}
```

### Raft节点运行
```go
func (r *Raft) run() {
	for {
		// Check if we are doing a shutdown
		select {
		case <-r.shutdownCh:
			// Clear the leader to prevent forwarding
			r.setLeader("")
			return
		default:
		}

		// Enter into a sub-FSM
		switch r.getState() {
		case Follower:
			r.runFollower()
		case Candidate:
			r.runCandidate()
		case Leader:
			r.runLeader()
		}
	}
}
```
跟随者、候选人、领导者 3 种节点状态都有分别对应的功能函数

### 创建Raft节点
每次服务起来的时候，都是创建一个raft节点：
```go
func NewRaft(conf *Config, fsm FSM, logs LogStore, stable StableStore, snaps SnapshotStore, trans Transport) (*Raft, error) {
	// Validate the configuration.
	if err := ValidateConfig(conf); err != nil {
		return nil, err
	}

	// Ensure we have a LogOutput.
	var logger hclog.Logger
	if conf.Logger != nil {
		logger = conf.Logger
	} else {
		if conf.LogOutput == nil {
			conf.LogOutput = os.Stderr
		}

		logger = hclog.New(&hclog.LoggerOptions{
			Name:   "raft",
			Level:  hclog.LevelFromString(conf.LogLevel),
			Output: conf.LogOutput,
		})
	}

	// Try to restore the current term.
	currentTerm, err := stable.GetUint64(keyCurrentTerm)
	if err != nil && err.Error() != "not found" {
		return nil, fmt.Errorf("failed to load current term: %v", err)
	}

	// Read the index of the last log entry.
	lastIndex, err := logs.LastIndex()
	if err != nil {
		return nil, fmt.Errorf("failed to find last log: %v", err)
	}

	// Get the last log entry.
	var lastLog Log
	if lastIndex > 0 {
		if err = logs.GetLog(lastIndex, &lastLog); err != nil {
			return nil, fmt.Errorf("failed to get last log at index %d: %v", lastIndex, err)
		}
	}

	// Make sure we have a valid server address and ID.
	protocolVersion := conf.ProtocolVersion
	localAddr := ServerAddress(trans.LocalAddr())
	localID := conf.LocalID

	// TODO (slackpad) - When we deprecate protocol version 2, remove this
	// along with the AddPeer() and RemovePeer() APIs.
	if protocolVersion < 3 && string(localID) != string(localAddr) {
		return nil, fmt.Errorf("when running with ProtocolVersion < 3, LocalID must be set to the network address")
	}

	// Create Raft struct.
	r := &Raft{
		protocolVersion:       protocolVersion,
		applyCh:               make(chan *logFuture),
		conf:                  *conf,
		fsm:                   fsm,
		fsmMutateCh:           make(chan interface{}, 128),
		fsmSnapshotCh:         make(chan *reqSnapshotFuture),
		leaderCh:              make(chan bool),
		localID:               localID,
		localAddr:             localAddr,
		logger:                logger,
		logs:                  logs,
		configurationChangeCh: make(chan *configurationChangeFuture),
		configurations:        configurations{},
		rpcCh:                 trans.Consumer(),
		snapshots:             snaps,
		userSnapshotCh:        make(chan *userSnapshotFuture),
		userRestoreCh:         make(chan *userRestoreFuture),
		shutdownCh:            make(chan struct{}),
		stable:                stable,
		trans:                 trans,
		verifyCh:              make(chan *verifyFuture, 64),
		configurationsCh:      make(chan *configurationsFuture, 8),
		bootstrapCh:           make(chan *bootstrapFuture),
		observers:             make(map[uint64]*Observer),
		leadershipTransferCh:  make(chan *leadershipTransferFuture, 1),
	}

	// Initialize as a follower.
	r.setState(Follower)

	// Start as leader if specified. This should only be used
	// for testing purposes.
	if conf.StartAsLeader {
		r.setState(Leader)
		r.setLeader(r.localAddr)
	}

	// Restore the current term and the last log.
	r.setCurrentTerm(currentTerm)
	r.setLastLog(lastLog.Index, lastLog.Term)

	// Attempt to restore a snapshot if there are any.
	if err := r.restoreSnapshot(); err != nil {
		return nil, err
	}

	// Scan through the log for any configuration change entries.
	snapshotIndex, _ := r.getLastSnapshot()
	for index := snapshotIndex + 1; index <= lastLog.Index; index++ {
		var entry Log
		if err := r.logs.GetLog(index, &entry); err != nil {
			r.logger.Error("failed to get log", "index", index, "error", err)
			panic(err)
		}
		r.processConfigurationLogEntry(&entry)
	}
	r.logger.Info("initial configuration",
		"index", r.configurations.latestIndex,
		"servers", hclog.Fmt("%+v", r.configurations.latest.Servers))

	// Setup a heartbeat fast-path to avoid head-of-line
	// blocking where possible. It MUST be safe for this
	// to be called concurrently with a blocking RPC.
	trans.SetHeartbeatHandler(r.processHeartbeat)

	if conf.skipStartup {
		return r, nil
	}
	// Start the background work.
	r.goFunc(r.run)
	r.goFunc(r.runFSM)
	r.goFunc(r.runSnapshots)
	return r, nil
}
```
上面的代码流程如下：
* 恢复选举轮次term
* 读取最后一个事务日志的索引
* 获取所有的事务日志
* 创建一个raft数据结构
* 设置该raft节点为follower状态
* 更新该raft节点的term和事务日志
* 恢复快照，如果有的话
* 加载从最后一个快照索引之后的所有事务日志
* 开始心跳
* 启动一个raft后台运行协程
* 启动一个状态机后台运行协程
* 启动一个快照处理后台运行协程


### 恢复快照
```go
func (r *Raft) restoreSnapshot() error {
	snapshots, err := r.snapshots.List()
	if err != nil {
		r.logger.Error("failed to list snapshots", "error", err)
		return err
	}

	// Try to load in order of newest to oldest
	for _, snapshot := range snapshots {
		if !r.conf.NoSnapshotRestoreOnStart {
			_, source, err := r.snapshots.Open(snapshot.ID)
			if err != nil {
				r.logger.Error("failed to open snapshot", "id", snapshot.ID, "error", err)
				continue
			}

			err = r.fsm.Restore(source)
			// Close the source after the restore has completed
			source.Close()
			if err != nil {
				r.logger.Error("failed to restore snapshot", "id", snapshot.ID, "error", err)
				continue
			}

			r.logger.Info("restored from snapshot", "id", snapshot.ID)
		}
		// Update the lastApplied so we don't replay old logs
		r.setLastApplied(snapshot.Index)

		// Update the last stable snapshot info
		r.setLastSnapshot(snapshot.Index, snapshot.Term)

		// Update the configuration
		var conf Configuration
		var index uint64
		if snapshot.Version > 0 {
			conf = snapshot.Configuration
			index = snapshot.ConfigurationIndex
		} else {
			conf = decodePeers(snapshot.Peers, r.trans)
			index = snapshot.Index
		}
		r.setCommittedConfiguration(conf, index)
		r.setLatestConfiguration(conf, index)

		// Success!
		return nil
	}

	// If we had snapshots and failed to load them, its an error
	if len(snapshots) > 0 {
		return fmt.Errorf("failed to load any existing snapshots")
	}
	return nil
}
```
上面的代码流程如下：
* 遍历所有的快照（从新到旧）
* 恢复每个快照到内存中
* 更新raft快照信息
* 更新raft配置实例Configuration信息

```sh
    2020-11-30T11:45:37.185-0500 [INFO]  agent.server.raft: restored from snapshot: id=53-81931-1606661162244
    2020-11-30T11:45:37.238-0500 [INFO]  agent.server.raft: initial configuration: index=15382 servers="[{Suffrage:Voter ID:3d09d521-77b7-2b48-41fd-6b7d19453c0e Address:51.6.196.201:8300} {Suffrage:Voter ID:2ef84c5d-dea0-8af7-fadd-b7effc0dbdd4 Address:51.6.196.211:8300}]"
```

### follower流程
上面我们说了，raft在启动的时候会初始化自己为follower状态，然后最后会执行raft.run方法：
```go
func (r *Raft) run() {
	for {
		// Check if we are doing a shutdown
		select {
		case <-r.shutdownCh:
			// Clear the leader to prevent forwarding
			r.setLeader("")
			return
		default:
		}

		// Enter into a sub-FSM
		switch r.getState() {
		case Follower:
			r.runFollower()
		case Candidate:
			r.runCandidate()
		case Leader:
			r.runLeader()
		}
	}
}
```
可以看到，这是一个轮询方法，会一直获取raft的状态，然后进入对应的状态处理流程中，这里我们先来看下follower流程。
```go
func (r *Raft) runFollower() {
	didWarn := false
	r.logger.Info("entering follower state", "follower", r, "leader", r.Leader())
	metrics.IncrCounter([]string{"raft", "state", "follower"}, 1)
	heartbeatTimer := randomTimeout(r.conf.HeartbeatTimeout)

	for r.getState() == Follower {
		select {
		case rpc := <-r.rpcCh:
			r.processRPC(rpc)

		case c := <-r.configurationChangeCh:
			// Reject any operations since we are not the leader
			c.respond(ErrNotLeader)

		case a := <-r.applyCh:
			// Reject any operations since we are not the leader
			a.respond(ErrNotLeader)

		case v := <-r.verifyCh:
			// Reject any operations since we are not the leader
			v.respond(ErrNotLeader)

		case r := <-r.userRestoreCh:
			// Reject any restores since we are not the leader
			r.respond(ErrNotLeader)

		case r := <-r.leadershipTransferCh:
			// Reject any operations since we are not the leader
			r.respond(ErrNotLeader)

		case c := <-r.configurationsCh:
			c.configurations = r.configurations.Clone()
			c.respond(nil)

		case b := <-r.bootstrapCh:
			b.respond(r.liveBootstrap(b.configuration))

		case <-heartbeatTimer:
			// Restart the heartbeat timer
			heartbeatTimer = randomTimeout(r.conf.HeartbeatTimeout)

			// Check if we have had a successful contact
			lastContact := r.LastContact()
			if time.Now().Sub(lastContact) < r.conf.HeartbeatTimeout {
				continue
			}

			// Heartbeat failed! Transition to the candidate state
			lastLeader := r.Leader()
			r.setLeader("")

			if r.configurations.latestIndex == 0 {
				if !didWarn {
					r.logger.Warn("no known peers, aborting election")
					didWarn = true
				}
			} else if r.configurations.latestIndex == r.configurations.committedIndex &&
				!hasVote(r.configurations.latest, r.localID) {
				if !didWarn {
					r.logger.Warn("not part of stable configuration, aborting election")
					didWarn = true
				}
			} else {
				r.logger.Warn("heartbeat timeout reached, starting election", "last-leader", lastLeader)
				metrics.IncrCounter([]string{"raft", "transition", "heartbeat_timeout"}, 1)
				r.setState(Candidate)
				return
			}

		case <-r.shutdownCh:
			return
		}
	}
}
```
在状态维护过程中，会根据收到的不同的标识进入不同的处理流程，这里只看下心跳这一块：
如果与leader最后一次的联系时间间隔大于了心跳超时时间，则将自己的状态置为candidate，也就是观察者，然后就会进入观察者流程中。

```sh
    2020-11-30T11:45:37.238-0500 [INFO]  agent.server.raft: entering follower state: follower="Node at 51.6.196.201:8300 [Follower]" leader=
    2020-11-30T11:45:37.240-0500 [WARN]  agent.server.memberlist.wan: memberlist: Binding to public address without encryption!
    2020-11-30T11:45:37.240-0500 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: node2_201.dc1 51.6.196.201
    2020-11-30T11:45:37.240-0500 [INFO]  agent.server.serf.wan: serf: Attempting re-join to previously known node: node3_211.dc1: 51.6.196.211:8302
    2020-11-30T11:45:37.240-0500 [WARN]  agent.server.memberlist.lan: memberlist: Binding to public address without encryption!
    2020-11-30T11:45:37.240-0500 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: node2_201 51.6.196.201
    2020-11-30T11:45:37.240-0500 [INFO]  agent.server.serf.lan: serf: Attempting re-join to previously known node: node3_211: 51.6.196.211:8301
    2020-11-30T11:45:37.241-0500 [INFO]  agent.server: Handled event for server in area: event=member-join server=node2_201.dc1 area=wan
    2020-11-30T11:45:37.241-0500 [INFO]  agent.server: Adding LAN server: server="node2_201 (Addr: tcp/51.6.196.201:8300) (DC: dc1)"
    2020-11-30T11:45:37.241-0500 [WARN]  agent: Service name will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.: service=java_register
    2020-11-30T11:45:37.242-0500 [INFO]  agent.server.serf.lan: serf: EventMemberJoin: node3_211 51.6.196.211
    2020-11-30T11:45:37.242-0500 [INFO]  agent.server: Adding LAN server: server="node3_211 (Addr: tcp/51.6.196.211:8300) (DC: dc1)"
    2020-11-30T11:45:37.242-0500 [INFO]  agent.server.serf.lan: serf: Re-joined to previously known node: node3_211: 51.6.196.211:8301
    2020-11-30T11:45:37.242-0500 [INFO]  agent.server.serf.wan: serf: EventMemberJoin: node3_211.dc1 51.6.196.211
    2020-11-30T11:45:37.243-0500 [INFO]  agent.server: Handled event for server in area: event=member-join server=node3_211.dc1 area=wan
    2020-11-30T11:45:37.243-0500 [INFO]  agent: Started DNS server: address=51.6.196.201:8600 network=udp
    2020-11-30T11:45:37.243-0500 [INFO]  agent: Started DNS server: address=51.6.196.201:8600 network=tcp
    2020-11-30T11:45:37.243-0500 [INFO]  agent.server.serf.wan: serf: Re-joined to previously known node: node3_211.dc1: 51.6.196.211:8302
    2020-11-30T11:45:37.244-0500 [INFO]  agent: Started HTTP server: address=51.6.196.201:8500 network=tcp
    2020-11-30T11:45:37.244-0500 [INFO]  agent: started state syncer
==> Consul agent running!
    2020-11-30T11:45:38.954-0500 [INFO]  agent.server: New leader elected: payload=node3_211
```

### candidate流程
```go
func (r *Raft) runCandidate() {
	r.logger.Info("entering candidate state", "node", r, "term", r.getCurrentTerm()+1)
	metrics.IncrCounter([]string{"raft", "state", "candidate"}, 1)

	// Start vote for us, and set a timeout
	voteCh := r.electSelf()

	// Make sure the leadership transfer flag is reset after each run. Having this
	// flag will set the field LeadershipTransfer in a RequestVoteRequst to true,
	// which will make other servers vote even though they have a leader already.
	// It is important to reset that flag, because this priviledge could be abused
	// otherwise.
	defer func() { r.candidateFromLeadershipTransfer = false }()

	electionTimer := randomTimeout(r.conf.ElectionTimeout)

	// Tally the votes, need a simple majority
	grantedVotes := 0
	votesNeeded := r.quorumSize()
	r.logger.Debug("votes", "needed", votesNeeded)

	for r.getState() == Candidate {
		select {
		case rpc := <-r.rpcCh:
			r.processRPC(rpc)

		case vote := <-voteCh:
			// Check if the term is greater than ours, bail
			if vote.Term > r.getCurrentTerm() {
				r.logger.Debug("newer term discovered, fallback to follower")
				r.setState(Follower)
				r.setCurrentTerm(vote.Term)
				return
			}

			// Check if the vote is granted
			if vote.Granted {
				grantedVotes++
				r.logger.Debug("vote granted", "from", vote.voterID, "term", vote.Term, "tally", grantedVotes)
			}

			// Check if we've become the leader
			if grantedVotes >= votesNeeded {
				r.logger.Info("election won", "tally", grantedVotes)
				r.setState(Leader)
				r.setLeader(r.localAddr)
				return
			}

		case c := <-r.configurationChangeCh:
			// Reject any operations since we are not the leader
			c.respond(ErrNotLeader)

		case a := <-r.applyCh:
			// Reject any operations since we are not the leader
			a.respond(ErrNotLeader)

		case v := <-r.verifyCh:
			// Reject any operations since we are not the leader
			v.respond(ErrNotLeader)

		case r := <-r.userRestoreCh:
			// Reject any restores since we are not the leader
			r.respond(ErrNotLeader)

		case c := <-r.configurationsCh:
			c.configurations = r.configurations.Clone()
			c.respond(nil)

		case b := <-r.bootstrapCh:
			b.respond(ErrCantBootstrap)

		case <-electionTimer:
			// Election failed! Restart the election. We simply return,
			// which will kick us back into runCandidate
			r.logger.Warn("Election timeout reached, restarting election")
			return

		case <-r.shutdownCh:
			return
		}
	}
}
```
上面的流程如下：
* 首先发送一个选票消息给所有的节点：推选自己为leader，等候其他所有的节点的回复
* 进入candidate状态轮询流程：
* 如果收到的其他节点的vote选票信息中，选举轮次term大于自己的选举轮次term，表明自己的选举落后了，自己没有资格在竞争leader，于是设置自己的状态为follower，同时更新自己的选举轮询为最新的。最后退出候选者流程。
* 如果收到的其他节点的vote选票，同意自己的提案，也就是同意自己成为leader，则自己的选票加一。如果自己的选票数大于了半数节点数，则表明自己成功当选为leader，更新自己的状态为leader。最后退出候选者流程。
* 如果选举请求超时，则直接退出候选者流程，等待下一次再次进入候选流程发起选票。

### leader流程
在上面的candidate流程中，我们说到了，在自己当选为leader之后，就会设置自己的状态为leader，此时就会进入到leader流程中：
```go
func (r *Raft) runLeader() {
	r.logger.Info("entering leader state", "leader", r)
	metrics.IncrCounter([]string{"raft", "state", "leader"}, 1)

	// Notify that we are the leader
	asyncNotifyBool(r.leaderCh, true)

	// Push to the notify channel if given
	if notify := r.conf.NotifyCh; notify != nil {
		select {
		case notify <- true:
		case <-r.shutdownCh:
		}
	}

	// setup leader state. This is only supposed to be accessed within the
	// leaderloop.
	r.setupLeaderState()

	// Cleanup state on step down
	defer func() {
		// Since we were the leader previously, we update our
		// last contact time when we step down, so that we are not
		// reporting a last contact time from before we were the
		// leader. Otherwise, to a client it would seem our data
		// is extremely stale.
		r.setLastContact()

		// Stop replication
		for _, p := range r.leaderState.replState {
			close(p.stopCh)
		}

		// Respond to all inflight operations
		for e := r.leaderState.inflight.Front(); e != nil; e = e.Next() {
			e.Value.(*logFuture).respond(ErrLeadershipLost)
		}

		// Respond to any pending verify requests
		for future := range r.leaderState.notify {
			future.respond(ErrLeadershipLost)
		}

		// Clear all the state
		r.leaderState.commitCh = nil
		r.leaderState.commitment = nil
		r.leaderState.inflight = nil
		r.leaderState.replState = nil
		r.leaderState.notify = nil
		r.leaderState.stepDown = nil

		// If we are stepping down for some reason, no known leader.
		// We may have stepped down due to an RPC call, which would
		// provide the leader, so we cannot always blank this out.
		r.leaderLock.Lock()
		if r.leader == r.localAddr {
			r.leader = ""
		}
		r.leaderLock.Unlock()

		// Notify that we are not the leader
		asyncNotifyBool(r.leaderCh, false)

		// Push to the notify channel if given
		if notify := r.conf.NotifyCh; notify != nil {
			select {
			case notify <- false:
			case <-r.shutdownCh:
				// On shutdown, make a best effort but do not block
				select {
				case notify <- false:
				default:
				}
			}
		}
	}()

	// Start a replication routine for each peer
	r.startStopReplication()

	// Dispatch a no-op log entry first. This gets this leader up to the latest
	// possible commit index, even in the absence of client commands. This used
	// to append a configuration entry instead of a noop. However, that permits
	// an unbounded number of uncommitted configurations in the log. We now
	// maintain that there exists at most one uncommitted configuration entry in
	// any log, so we have to do proper no-ops here.
	noop := &logFuture{
		log: Log{
			Type: LogNoop,
		},
	}
	r.dispatchLogs([]*logFuture{noop})

	// Sit in the leader loop until we step down
	r.leaderLoop()
}
```
上面的代码流程如下：
* 向外通知自己为leader
* 设置领导者状态
* 设置上一次联系时间：由于我们以前是领导者，当我们下线时更新了上次联系时间，因此在我们从下线到再次成为领导者之间是没有更新最后联系时间的。因此，对于follower而言，他们就会认为我们的数据非常陈旧。

```sh
    2020-11-30T11:45:39.011-0500 [WARN]  agent.server.raft: Election timeout reached, restarting election
    2020-11-30T11:45:39.011-0500 [INFO]  agent.server.raft: entering candidate state: node="Node at 51.6.196.211:8300 [Candidate]" term=272
    2020-11-30T11:45:39.145-0500 [INFO]  agent.server.raft: election won: tally=2
    2020-11-30T11:45:39.145-0500 [INFO]  agent.server.raft: entering leader state: leader="Node at 51.6.196.211:8300 [Leader]"
    2020-11-30T11:45:39.146-0500 [INFO]  agent.server.raft: added peer, starting replication: peer=3d09d521-77b7-2b48-41fd-6b7d19453c0e
    2020-11-30T11:45:39.146-0500 [INFO]  agent.server: cluster leadership acquired
    2020-11-30T11:45:39.146-0500 [INFO]  agent.server: New leader elected: payload=node3_211
```

### 复制数据
```go
func (r *Raft) startStopReplication() {
	inConfig := make(map[ServerID]bool, len(r.configurations.latest.Servers))
	lastIdx := r.getLastIndex()

	// Start replication goroutines that need starting
	for _, server := range r.configurations.latest.Servers {
		if server.ID == r.localID {
			continue
		}
		inConfig[server.ID] = true
		if _, ok := r.leaderState.replState[server.ID]; !ok {
			r.logger.Info("added peer, starting replication", "peer", server.ID)
			s := &followerReplication{
				peer:                server,
				commitment:          r.leaderState.commitment,
				stopCh:              make(chan uint64, 1),
				triggerCh:           make(chan struct{}, 1),
				triggerDeferErrorCh: make(chan *deferError, 1),
				currentTerm:         r.getCurrentTerm(),
				nextIndex:           lastIdx + 1,
				lastContact:         time.Now(),
				notify:              make(map[*verifyFuture]struct{}),
				notifyCh:            make(chan struct{}, 1),
				stepDown:            r.leaderState.stepDown,
			}
			r.leaderState.replState[server.ID] = s
			r.goFunc(func() { r.replicate(s) })
			asyncNotifyCh(s.triggerCh)
			r.observe(PeerObservation{Peer: server, Removed: false})
		}
	}

	// Stop replication goroutines that need stopping
	for serverID, repl := range r.leaderState.replState {
		if inConfig[serverID] {
			continue
		}
		// Replicate up to lastIdx and stop
		r.logger.Info("removed peer, stopping replication", "peer", serverID, "last-index", lastIdx)
		repl.stopCh <- lastIdx
		close(repl.stopCh)
		delete(r.leaderState.replState, serverID)
		r.observe(PeerObservation{Peer: repl.peer, Removed: true})
	}
}

func (r *Raft) replicate(s *followerReplication) {
	// Start an async heartbeating routing
	stopHeartbeat := make(chan struct{})
	defer close(stopHeartbeat)
	r.goFunc(func() { r.heartbeat(s, stopHeartbeat) })

RPC:
	shouldStop := false
	for !shouldStop {
		select {
		case maxIndex := <-s.stopCh:
			// Make a best effort to replicate up to this index
			if maxIndex > 0 {
				r.replicateTo(s, maxIndex)
			}
			return
		case deferErr := <-s.triggerDeferErrorCh:
			lastLogIdx, _ := r.getLastLog()
			shouldStop = r.replicateTo(s, lastLogIdx)
			if !shouldStop {
				deferErr.respond(nil)
			} else {
				deferErr.respond(fmt.Errorf("replication failed"))
			}
		case <-s.triggerCh:
			lastLogIdx, _ := r.getLastLog()
			shouldStop = r.replicateTo(s, lastLogIdx)
		// This is _not_ our heartbeat mechanism but is to ensure
		// followers quickly learn the leader's commit index when
		// raft commits stop flowing naturally. The actual heartbeats
		// can't do this to keep them unblocked by disk IO on the
		// follower. See https://github.com/hashicorp/raft/issues/282.
		case <-randomTimeout(r.conf.CommitTimeout):
			lastLogIdx, _ := r.getLastLog()
			shouldStop = r.replicateTo(s, lastLogIdx)
		}

		// If things looks healthy, switch to pipeline mode
		if !shouldStop && s.allowPipeline {
			goto PIPELINE
		}
	}
	return

PIPELINE:
	// Disable until re-enabled
	s.allowPipeline = false

	// Replicates using a pipeline for high performance. This method
	// is not able to gracefully recover from errors, and so we fall back
	// to standard mode on failure.
	if err := r.pipelineReplicate(s); err != nil {
		if err != ErrPipelineReplicationNotSupported {
			r.logger.Error("failed to start pipeline replication to", "peer", s.peer, "error", err)
		}
	}
	goto RPC
}
```
leader向follower发送日志时，会顺带邻近的前一条日志，follwer接收日志时，会在相同任期号和索引位置找前一条日志，如果存在且匹配，则接收日志；否则拒绝，leader会减少日志索引位置并进行重试，直到某个位置与follower达成一致。然后follower删除索引后的所有日志，并追加leader发送的日志，一旦日志追加成功，则follower和leader的所有日志就保持一致。只有在多数派的follower都响应接受到日志后，表示事务可以提交，才能返回客户端提交成功。

follower接收快照流程
* 1.如果leaderTermId<currentTerm, 则返回
* 2.如果是第一个块，创建快照
* 3.在指定的偏移，将数据写入快照
* 4.如果不是最后一块，等待更多的块
* 5.接收完毕后，丢掉以前旧的快照
* 6.删除掉不需要的日志

```sh
2020-11-30T11:45:39.213-0500 [INFO]  agent.server.raft: pipelining replication: peer="{Voter 3d09d521-77b7-2b48-41fd-6b7d19453c0e 51.6.196.201:8300}"
```

### 创建快照
在上面我们提到会启动一个快照处理协程，那么这里我们就来看看它是怎么处理的
```go
func (r *Raft) runSnapshots() {
	for {
		select {
		case <-randomTimeout(r.conf.SnapshotInterval):
			// Check if we should snapshot
			if !r.shouldSnapshot() {
				continue
			}

			// Trigger a snapshot
			if _, err := r.takeSnapshot(); err != nil {
				r.logger.Error("failed to take snapshot", "error", err)
			}

		case future := <-r.userSnapshotCh:
			// User-triggered, run immediately
			id, err := r.takeSnapshot()
			if err != nil {
				r.logger.Error("failed to take snapshot", "error", err)
			} else {
				future.opener = func() (*SnapshotMeta, io.ReadCloser, error) {
					return r.snapshots.Open(id)
				}
			}
			future.respond(err)

		case <-r.shutdownCh:
			return
		}
	}
}
```
上面代码流程如下：
* 检查是否需要生成快照：比较上一次生成快照的日志id跟当前最新的日志id，如果他们之间的差值达到了设置的快照阈值，则表示需要生成快照。
* 向FSM发起创建快照的请求
* 本地持久化快照，同时将本地所有的旧的快照文件删除掉
* 更新最新的生成快照的日志id
* 截断最新快照日志id之前的事务日志，减少事务日志文件的大小。

注：如果leader发现follower日志落后太远(超过阀值)，则触发发送快照流程
备注：快照不能太频繁，否则会导致磁盘IO压力较大；但也需要定期做，清理非必要的日志，缓解日志的空间压力，另外可以提高follower追赶的速度。


```sh
    2020-12-01T09:42:50.581-0500 [INFO]  agent.server.fsm: snapshot created: duration=76.493µs
    2020-12-01T09:42:50.581-0500 [INFO]  agent.server.raft: starting snapshot up to: index=98317
    2020-12-01T09:42:50.581-0500 [INFO]  snapshot: creating new snapshot: path=/root/consul/data/raft/snapshots/272-98317-1606833770581.tmp
    2020-12-01T09:42:50.659-0500 [INFO]  snapshot: reaping snapshot: path=/root/consul/data/raft/snapshots/53-65542-1606491438513
    2020-12-01T09:42:50.660-0500 [INFO]  agent.server.raft: compacting logs: from=71691 to=88077
    2020-12-01T09:42:50.695-0500 [INFO]  agent.server.raft: snapshot complete up to: index=98317

```

### 一问一答
##### 1.Raft协议中是否存在“活锁”，如何解决？
活锁是相对死锁而言，所谓死锁，就是两个或多个线程相互锁等待，导致都无法推进的情况，而活锁则是多个工作线程(节点)都在运转，但是整体系统的状态无法推进，比如basic-paxos中某些情况下投票总是没有办法达成多数派。在Raft中，由于只要一阶段提交(只有leader提议)，在日志复制的过程中不存在活锁问题。但是选主过程中，可能出现多个节点同时发起选主的情况，这样导致选票瓜分，无法选出主，在下一轮选举中依旧如此，导致系统状态无法往前推进。Raft通过随机超时解决这个“活锁”问题。

##### 2.Raft系统对于各个节点的物理时钟强一致有要求吗？
     Raft协议对物理时钟一致性没有要求，不需要通过原子钟NTP来校准时间，但是对于超时时间的设置有要求，具体规则如下：
broadcastTime ≪ electionTimeout ≪ MTBF(Mean Time Between Failure)
首先，广播时间要远小于选举超时时间，leader通过广播不间断给follower发送心跳，如果这个时间比超时时间短，就会导致follower误以为leader挂了，触发选主；然后是超时时间要远小于机器的平均故障时间，如果MTBF比超时时间还短，则永远会发生选主问题，而在选主完成之前，无法对外正常提供服务，因此需要保证。一般broadcastTime可以认为是一个网络RTT，同城1ms以内，异地100ms以内，如果是跨国家，可能需要几百ms；而机器平均故障时间至少是以月为单位，因此选举超时时间需要设置1s到5s左右即可。 

##### 3.如何保证leader上拥有了所有日志？
一方面，对于leader不变场景，日志只能从leader流向follower，并且发生冲突时以leader的日志为准；另一方面，对于leader一直有变换的场景，通过选举机制来保证，选举时采用(LogTerm,LogIndex)谁更新的比对方式，并且要得到多数派的认可，说明新leader的日志至少是多数派中最新的，另一方面，提交的日志一定也是达成了多数派，所以推断出leader有所有已提交的日志，不会漏。

##### 4.Raft协议为什么需要日志连续性，日志连续性有有什么优势和劣势？
由Raft协议的选主过程可知，(termId,logId)一定在多数派中最新才可能成为leader，也就是说leader中一定已经包含了所有已经提交的日志。所以leader不需要从其它follower节点获取日志，保证了日志永远只从leader流向follower，简化了逻辑。但缺陷在于，任何一个follower在接受日志前，都需要接受之前的所有日志，并且只有追赶上了，才能有投票权利【否则，复制日志时，不考虑它们是大多数】，如果日志差的比较多，就会导致follower需要较长的时间追赶。任何一个没有追上最新日志的follower，没有投票权利，导致网络比较差的情况下，不容易达成多数派。

##### 5.Raft如何保证日志连续性？
leader向follower发送日志时，会顺带邻近的前一条日志，follwer接受日志时，会在相同任期号和索引位置找前一条日志，如果存在且匹配，则接受日志，否则拒绝接受，leader会减少日志索引位置并进行重试，直到某个位置与follower达成一致。然后follower删除索引后的所有日志，并追加leader发送的日志，一旦日志追加成功，则follower和leader的所有日志就保持一致。

##### 6.如果TermId小的先达成多数派，TermId大的怎么办？可能吗？
如果TermId小的达成了多数派，则说明TermId大的节点以前是leader，拥有最多的日志，但是没有达成多数派，因此它的日志可以被覆盖。但该节点会尝试继续投票，新leader发送日志给该节点，如果leader发现返回的termT>currentTerm，且还没有达成多数派，则重新变为follower，促使TermId更大的节点成为leader。但并不保证拥有较大termId的节点一定会成为leader，因为leader是优先判断是否达成多数派，如果已经达成多数派了，则继续为leader。

##### 7.达成多数派的日志就一定认为是提交的？
不一定，一定是在current_term内产生的日志，并且达成多数派才能认为是提交的，持久化的，不会变的。Raft中，leader保持原始日志的termId不变，任何一条日志，都有termId和logIndex属性。在leader频繁变更的情况下，有可能出现某条日志在某种状态下达成了多数派，但日志最终可能被覆盖掉

#### 参考文档
https://raft.github.io/raft.pdf
https://ramcloud.stanford.edu/~ongaro/thesis.pdf
https://ramcloud.stanford.edu/~ongaro/userstudy/paxos.pdf