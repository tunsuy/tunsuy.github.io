## Raft协议集群选主投票算法详解
在之前的文章中，我们说过，raft节点会在各自的状态里面不断的轮询，监听RPC请求事件。
下面我看下这个方法.

hashicorp/raft.go

### Rpc请求处理
```go
// processRPC is called to handle an incoming RPC request. This must only be
// called from the main thread.
func (r *Raft) processRPC(rpc RPC) {
	if err := r.checkRPCHeader(rpc); err != nil {
		rpc.Respond(nil, err)
		return
	}

	switch cmd := rpc.Command.(type) {
	case *AppendEntriesRequest:
		r.appendEntries(rpc, cmd)
	case *RequestVoteRequest:
		r.requestVote(rpc, cmd)
	case *InstallSnapshotRequest:
		r.installSnapshot(rpc, cmd)
	case *TimeoutNowRequest:
		r.timeoutNow(rpc, cmd)
	default:
		r.logger.Error("got unexpected command",
			"command", hclog.Fmt("%#v", rpc.Command))
		rpc.Respond(nil, fmt.Errorf("unexpected command"))
	}
}
```
这个就是RPC请求处理的总流程，包括如下类型事件：
* 1、主节点发送过来的日志请求
* 2、其他节点发送过来的投票请求
* 3、主节点发送过来的快照同步请求
* 4、跟主节点的心跳超时

今天我们主要讲解投票请求流程

### 投票流程
根据上面看到的，我们进入requestVote方法：
```go
// requestVote is invoked when we get an request vote RPC call.
func (r *Raft) requestVote(rpc RPC, req *RequestVoteRequest) {
	defer metrics.MeasureSince([]string{"raft", "rpc", "requestVote"}, time.Now())
	r.observe(*req)

	// Setup a response
	resp := &RequestVoteResponse{
		RPCHeader: r.getRPCHeader(),
		Term:      r.getCurrentTerm(),
		Granted:   false,
	}
	var rpcErr error
	defer func() {
		rpc.Respond(resp, rpcErr)
	}()

	// Version 0 servers will panic unless the peers is present. It's only
	// used on them to produce a warning message.
	if r.protocolVersion < 2 {
		resp.Peers = encodePeers(r.configurations.latest, r.trans)
	}

	// Check if we have an existing leader [who's not the candidate] and also
	// check the LeadershipTransfer flag is set. Usually votes are rejected if
	// there is a known leader. But if the leader initiated a leadership transfer,
	// vote!
	candidate := r.trans.DecodePeer(req.Candidate)
	if leader := r.Leader(); leader != "" && leader != candidate && !req.LeadershipTransfer {
		r.logger.Warn("rejecting vote request since we have a leader",
			"from", candidate,
			"leader", leader)
		return
	}

	// Ignore an older term
	if req.Term < r.getCurrentTerm() {
		return
	}

	// Increase the term if we see a newer one
	if req.Term > r.getCurrentTerm() {
		// Ensure transition to follower
		r.logger.Debug("lost leadership because received a requestVote with a newer term")
		r.setState(Follower)
		r.setCurrentTerm(req.Term)
		resp.Term = req.Term
	}

	// Check if we have voted yet
	lastVoteTerm, err := r.stable.GetUint64(keyLastVoteTerm)
	if err != nil && err.Error() != "not found" {
		r.logger.Error("failed to get last vote term", "error", err)
		return
	}
	lastVoteCandBytes, err := r.stable.Get(keyLastVoteCand)
	if err != nil && err.Error() != "not found" {
		r.logger.Error("failed to get last vote candidate", "error", err)
		return
	}

	// Check if we've voted in this election before
	if lastVoteTerm == req.Term && lastVoteCandBytes != nil {
		r.logger.Info("duplicate requestVote for same term", "term", req.Term)
		if bytes.Compare(lastVoteCandBytes, req.Candidate) == 0 {
			r.logger.Warn("duplicate requestVote from", "candidate", req.Candidate)
			resp.Granted = true
		}
		return
	}

	// Reject if their term is older
	lastIdx, lastTerm := r.getLastEntry()
	if lastTerm > req.LastLogTerm {
		r.logger.Warn("rejecting vote request since our last term is greater",
			"candidate", candidate,
			"last-term", lastTerm,
			"last-candidate-term", req.LastLogTerm)
		return
	}

	if lastTerm == req.LastLogTerm && lastIdx > req.LastLogIndex {
		r.logger.Warn("rejecting vote request since our last index is greater",
			"candidate", candidate,
			"last-index", lastIdx,
			"last-candidate-index", req.LastLogIndex)
		return
	}

	// Persist a vote for safety
	if err := r.persistVote(req.Term, req.Candidate); err != nil {
		r.logger.Error("failed to persist vote", "error", err)
		return
	}

	resp.Granted = true
	r.setLastContact()
	return
}
```
这个方法的流程比较清晰，下面一一讲解如下：

#### 1、请求体
我们先来看下请求的body体：
```go
type RequestVoteRequest struct {
	RPCHeader

	// Provide the term and our id
	Term      uint64
	Candidate []byte

	// Used to ensure safety
	LastLogIndex uint64
	LastLogTerm  uint64

	// Used to indicate to peers if this vote was triggered by a leadership
	// transfer. It is required for leadership transfer to work, because servers
	// wouldn't vote otherwise if they are aware of an existing leader.
	LeadershipTransfer bool
}
```
可以看到一个节点发起投票时，会传递这些信息过来：
* term: 自己节点当前的0选举轮次；
* Candidate：自己选举某个节点为leader的serverid
* LastLogIndex：在集群状态正常情况下，最后一次的事务日志id号
* LastLogTerm：在集群状态正常情况下，最后一次的选举轮次

#### 2、返回体
我们再来看下投票请求的返回结构：
```go
resp := &RequestVoteResponse{
	RPCHeader: r.getRPCHeader(),
	Term:      r.getCurrentTerm(),
	Granted:   false,
}
```
可以看到返回给对端的主要是两个值：
* Term：当前节点的选举轮次；
* Granted：是否同意请求节点的选票，也就是说，发送这个请求过来的节点的选票是期望自己是主节点，如果该节点同意，则该字段置为True

#### 3、拒绝选票场景
好了，下面我们来看下哪些情况下会拒绝对端的投票
* 当前节点已经有一个leader
```go
candidate := r.trans.DecodePeer(req.Candidate)
if leader := r.Leader(); leader != "" && leader != candidate && !req.LeadershipTransfer {
	r.logger.Warn("rejecting vote request since we have a leader",
		"from", candidate,
		"leader", leader)
	return
}
```
注：这里有一个场景除外，就是如果对端节点是从一个正常的集群状态初始化转换到候选者的场景

* 对端节点的选举轮次小于本节点
```go
if req.Term < r.getCurrentTerm() {
	return
}
```

* 没有找到本节点上一次的选举轮次：
```go
lastVoteTerm, err := r.stable.GetUint64(keyLastVoteTerm)
if err != nil && err.Error() != "not found" {
	r.logger.Error("failed to get last vote term", "error", err)
	return
}
```

* 没有找到本节点上一次的投票信息：
```go
lastVoteCandBytes, err := r.stable.Get(keyLastVoteCand)
if err != nil && err.Error() != "not found" {
	r.logger.Error("failed to get last vote candidate", "error", err)
	return
}
```

* 同一轮选举收到重复的节点投票请求：
```go
// Check if we've voted in this election before
if lastVoteTerm == req.Term && lastVoteCandBytes != nil {
	r.logger.Info("duplicate requestVote for same term", "term", req.Term)
	if bytes.Compare(lastVoteCandBytes, req.Candidate) == 0 {
		r.logger.Warn("duplicate requestVote from", "candidate", req.Candidate)
		resp.Granted = true
	}
	return
}
```
注：一个节点在每一轮选举中只能投一次票

* 本节点最后一次集群稳定状态下的选举轮次大于对端节点的选举轮次：
```go
// Reject if their term is older
lastIdx, lastTerm := r.getLastEntry()
if lastTerm > req.LastLogTerm {
	r.logger.Warn("rejecting vote request since our last term is greater",
		"candidate", candidate,
		"last-term", lastTerm,
		"last-candidate-term", req.LastLogTerm)
	return
}
```

* 本节点最后一次集群稳定状态下的选举轮次等于对端节点的选举轮次的情况下，最后一次的事务日志id大于对端的事务日志id：
```go
if lastTerm == req.LastLogTerm && lastIdx > req.LastLogIndex {
	r.logger.Warn("rejecting vote request since our last index is greater",
		"candidate", candidate,
		"last-index", lastIdx,
		"last-candidate-index", req.LastLogIndex)
	return
}
```

* 持久化本次选举信息失败
```go
// Persist a vote for safety
if err := r.persistVote(req.Term, req.Candidate); err != nil {
	r.logger.Error("failed to persist vote", "error", err)
	return
}
```

除了上述的场景之外，其他情况都会同意对端的请求，并且在对端节点的选举轮次大于本节点的情况下，更新自己为follower和自己的选举轮次
```go
// Increase the term if we see a newer one
if req.Term > r.getCurrentTerm() {
	// Ensure transition to follower
	r.logger.Debug("lost leadership because received a requestVote with a newer term")
	r.setState(Follower)
	r.setCurrentTerm(req.Term)
	resp.Term = req.Term
}
```