# Gossip协议及Consul中的实现
gossip 协议（gossip protocol）又称 epidemic 协议（epidemic protocol），是基于流行病传播方式的节点或者进程之间信息交换的协议，在分布式系统中被广泛使用，比如我们可以使用 gossip 协议来确保网络中所有节点的数据一样。
gossip 协议利用一种随机的方式将信息传播到整个网络中，并在一定时间内使得系统内的所有节点数据一致。Gossip 其实是一种去中心化思路的分布式协议，解决状态在集群中的传播和状态一致性的保证两个问题。

## 特点
1、可扩展性（Scalable）
gossip 协议是可扩展的，一般需要 O(logN) 轮就可以将信息传播到所有的节点，其中 N 代表节点的个数。每个节点仅发送固定数量的消息。在数据传送的时候，节点并不会等待消息的 ack，所以消息传送失败也没有关系，因为可以通过其他节点将消息传递给之前传送失败的节点。系统可以轻松扩展到数百万个进程。

2、容错（Fault-tolerance）
网络中任何节点的重启或者宕机都不会影响 gossip 协议的运行。

3、健壮性（Robust）
gossip 协议是去中心化的协议，所以集群中的所有节点都是对等的，没有特殊的节点，所以任何节点出现问题都不会阻止其他节点继续发送消息。任何节点都可以随时加入或离开，而不会影响系统的整体服务质量（QOS）

4、最终一致性（Convergent consistency）
Gossip 协议实现信息指数级的快速传播，因此在有新信息需要传播时，消息可以快速地发送到全局节点，在有限的时间内能够做到所有节点都拥有最新的数据

## 缺点
1、消息的延迟
由于 Gossip 协议中，节点只会随机向少数几个节点发送消息，消息最终是通过多个轮次的散播而到达全网的，因此使用 Gossip 协议会造成不可避免的消息延迟。不适合用在对实时性要求较高的场景下。

2、消息冗余
Gossip协议规定，节点会定期随机选择周围节点发送消息，而收到消息的节点也会重复该步骤，因此就不可避免的存在消息重复发送给同一节点的情况，造成了消息的冗余，同时也增加了收到消息的节点的处理压力。而且，由于是定期发送，因此，即使收到了消息的节点还会反复收到重复消息，加重了消息的冗余。

## 执行过程
Gossip过程是由种子节点发起，当一个种子节点有状态需要更新到网络中的其他节点时，它会随机的选择周围几个节点散播消息，收到消息的节点也会重复该过程，直至最终网络中所有的节点都收到了消息。这个过程可能需要一定的时间，由于不能保证某个时刻所有节点都收到消息，但是理论上最终所有节点都会收到消息，因此它是一个最终一致性协议。


## 通信模式

在 Gossip 协议下，网络中两个节点之间有三种通信方式:
* Push: 节点 A 将数据 (key,value,version) 及对应的版本号推送给 B 节点，B 节点更新 A 中比自己新的数据
* Pull：A 仅将数据 key, version 推送给 B，B 将本地比 A 新的数据（Key, value, version）推送给 A，A 更新本地
* Push/Pull：与 Pull 类似，只是多了一步，A 再将本地比 B 新的数据推送给 B，B 则更新本地
  如果把两个节点数据同步一次定义为一个周期，则在一个周期内，Push 需通信 1 次，Pull 需 2 次，Push/Pull 则需 3 次。虽然消息数增加了，但从效果上来讲，Push/Pull 最好，理论上一个周期内可以使两个节点完全一致。直观上，Push/Pull 的收敛速度也是最快的。

## Consul中的Gossip
Consul用了两种不同的Gossip池。把这两种池分别叫做LAN池和WAN池。

### LAN池
Consul中的每个数据中心有一个LAN池，它包含了这个数据中心的所有成员，包括clients和servers。LAN池用于以下几个目的:
* 成员关系信息允许client自动发现server, 减少了所需要的配置量。
* 分布式失败检测机制使得由整个集群来做失败检测这件事， 而不是集中到几台机器上。
* gossip池使得类似领导人选举这样的事件变得可靠而且迅速。

### WAN池
WAN池是全局唯一的，因为所有的server都应该加入到WAN池中，无论它位于哪个数据中心。由WAN池提供的成员关系信息允许server做一些跨数据中心的请求。一体化的失败检测机制允许Consul优雅地去处理：整个数据中心失去连接， 或者仅仅是别的数据中心的某一台失去了连接。

consul在gossip上的实现实际上是使用的memberlist库，也是自家公司提供的。
其实现了集群内节点发现、 节点失效探测、节点故障转移、节点状态同步等。
下面我们主要来看下周期性任务这一块

### 节点状态
节点的state有3种
* 1、alive： 节点是”活的”
* 2、suspect： 对于PingMsg没有应答或应答超时，这个节点的状态是”可疑的”
* 3、dead： 节点”已死亡”

### 周期性任务
```go
// Schedule is used to ensure the Tick is performed periodically. This
// function is safe to call multiple times. If the memberlist is already
// scheduled, then it won't do anything.
func (m *Memberlist) schedule() {
	m.tickerLock.Lock()
	defer m.tickerLock.Unlock()

	// If we already have tickers, then don't do anything, since we're
	// scheduled
	if len(m.tickers) > 0 {
		return
	}

	// Create the stop tick channel, a blocking channel. We close this
	// when we should stop the tickers.
	stopCh := make(chan struct{})

	// Create a new probeTicker
	if m.config.ProbeInterval > 0 {
		t := time.NewTicker(m.config.ProbeInterval)
		go m.triggerFunc(m.config.ProbeInterval, t.C, stopCh, m.probe)
		m.tickers = append(m.tickers, t)
	}

	// Create a push pull ticker if needed
	if m.config.PushPullInterval > 0 {
		go m.pushPullTrigger(stopCh)
	}

	// Create a gossip ticker if needed
	if m.config.GossipInterval > 0 && m.config.GossipNodes > 0 {
		t := time.NewTicker(m.config.GossipInterval)
		go m.triggerFunc(m.config.GossipInterval, t.C, stopCh, m.gossip)
		m.tickers = append(m.tickers, t)
	}

	// If we made any tickers, then record the stopTick channel for
	// later.
	if len(m.tickers) > 0 {
		m.stopTick = stopCh
	}
}
```
从上面的代码，可以看到，主要有以下几个周期性任务：
* 1、故障检测
* 2、状态合并（push/pull消息）
* 3、广播gossip消息

#### 1、故障检测
```go
func (m *Memberlist) probe() {
	// Track the number of indexes we've considered probing
	numCheck := 0
START:
	m.nodeLock.RLock()

	// Make sure we don't wrap around infinitely
	if numCheck >= len(m.nodes) {
		m.nodeLock.RUnlock()
		return
	}

	// Handle the wrap around case
	if m.probeIndex >= len(m.nodes) {
		m.nodeLock.RUnlock()
		m.resetNodes()
		m.probeIndex = 0
		numCheck++
		goto START
	}

	// Determine if we should probe this node
	skip := false
	var node nodeState

	node = *m.nodes[m.probeIndex]
	if node.Name == m.config.Name {
		skip = true
	} else if node.DeadOrLeft() {
		skip = true
	}

	// Potentially skip
	m.nodeLock.RUnlock()
	m.probeIndex++
	if skip {
		numCheck++
		goto START
	}

	// Probe the specific node
	m.probeNode(&node)
}
```
上面的代码主要做了几件事：
* 1、所有的节点是否都已经探测过，如果是，则直接返回
* 2、重置节点：在节点列表中移除死亡节点，并更新活跃节点，并随机打乱节点列表（目的不详）
* 3、检查节点是否是否探测：在配置文件中或者状态是dead和left的节点会跳过
* 4、开始探测节点

##### 1.1 直接探测节点
```go
func (m *Memberlist) probeNode(node *nodeState) {
	defer metrics.MeasureSince([]string{"memberlist", "probeNode"}, time.Now())

	// We use our health awareness to scale the overall probe interval, so we
	// slow down if we detect problems. The ticker that calls us can handle
	// us running over the base interval, and will skip missed ticks.
	probeInterval := m.awareness.ScaleTimeout(m.config.ProbeInterval)
	if probeInterval > m.config.ProbeInterval {
		metrics.IncrCounter([]string{"memberlist", "degraded", "probe"}, 1)
	}

	// Prepare a ping message and setup an ack handler.
	selfAddr, selfPort := m.getAdvertise()
	ping := ping{
		SeqNo:      m.nextSeqNo(),
		Node:       node.Name,
		SourceAddr: selfAddr,
		SourcePort: selfPort,
		SourceNode: m.config.Name,
	}
	ackCh := make(chan ackMessage, m.config.IndirectChecks+1)
	nackCh := make(chan struct{}, m.config.IndirectChecks+1)
	m.setProbeChannels(ping.SeqNo, ackCh, nackCh, probeInterval)

	// Mark the sent time here, which should be after any pre-processing but
	// before system calls to do the actual send. This probably over-reports
	// a bit, but it's the best we can do. We had originally put this right
	// after the I/O, but that would sometimes give negative RTT measurements
	// which was not desirable.
	sent := time.Now()

	// Send a ping to the node. If this node looks like it's suspect or dead,
	// also tack on a suspect message so that it has a chance to refute as
	// soon as possible.
	deadline := sent.Add(probeInterval)
	addr := node.Address()

	// Arrange for our self-awareness to get updated.
	var awarenessDelta int
	defer func() {
		m.awareness.ApplyDelta(awarenessDelta)
	}()
	if node.State == StateAlive {
		if err := m.encodeAndSendMsg(node.FullAddress(), pingMsg, &ping); err != nil {
			m.logger.Printf("[ERR] memberlist: Failed to send ping: %s", err)
			if failedRemote(err) {
				goto HANDLE_REMOTE_FAILURE
			} else {
				return
			}
		}
	} else {
		var msgs [][]byte
		if buf, err := encode(pingMsg, &ping); err != nil {
			m.logger.Printf("[ERR] memberlist: Failed to encode ping message: %s", err)
			return
		} else {
			msgs = append(msgs, buf.Bytes())
		}
		s := suspect{Incarnation: node.Incarnation, Node: node.Name, From: m.config.Name}
		if buf, err := encode(suspectMsg, &s); err != nil {
			m.logger.Printf("[ERR] memberlist: Failed to encode suspect message: %s", err)
			return
		} else {
			msgs = append(msgs, buf.Bytes())
		}

		compound := makeCompoundMessage(msgs)
		if err := m.rawSendMsgPacket(node.FullAddress(), &node.Node, compound.Bytes()); err != nil {
			m.logger.Printf("[ERR] memberlist: Failed to send compound ping and suspect message to %s: %s", addr, err)
			if failedRemote(err) {
				goto HANDLE_REMOTE_FAILURE
			} else {
				return
			}
		}
	}

	// Arrange for our self-awareness to get updated. At this point we've
	// sent the ping, so any return statement means the probe succeeded
	// which will improve our health until we get to the failure scenarios
	// at the end of this function, which will alter this delta variable
	// accordingly.
	awarenessDelta = -1

	// Wait for response or round-trip-time.
	select {
	case v := <-ackCh:
		if v.Complete == true {
			if m.config.Ping != nil {
				rtt := v.Timestamp.Sub(sent)
				m.config.Ping.NotifyPingComplete(&node.Node, rtt, v.Payload)
			}
			return
		}

		// As an edge case, if we get a timeout, we need to re-enqueue it
		// here to break out of the select below.
		if v.Complete == false {
			ackCh <- v
		}
	case <-time.After(m.config.ProbeTimeout):
		// Note that we don't scale this timeout based on awareness and
		// the health score. That's because we don't really expect waiting
		// longer to help get UDP through. Since health does extend the
		// probe interval it will give the TCP fallback more time, which
		// is more active in dealing with lost packets, and it gives more
		// time to wait for indirect acks/nacks.
		m.logger.Printf("[DEBUG] memberlist: Failed ping: %s (timeout reached)", node.Name)
	}
...
}
```
该部分代码逻辑如下：
* 1、如果该节点alive状态的，则直接发送ping消息。如果该节点是其他状态，则会先发送ping消息，然后再发送suspect（可疑）消息，以便该节点有机会尽快进行响应（表明自己不是不正常状态）。
* 2、发送之后，开始计时，如果在超时时间之内返回，则表示正常，直接返回；否则，进入间接探测流程

##### 1.2 间接探测流程
该流程同样在上面那个函数中，代码如下：
```go
func (m *Memberlist) probeNode(node *nodeState) {
...
HANDLE_REMOTE_FAILURE:
	// Get some random live nodes.
	m.nodeLock.RLock()
	kNodes := kRandomNodes(m.config.IndirectChecks, m.nodes, func(n *nodeState) bool {
		return n.Name == m.config.Name ||
			n.Name == node.Name ||
			n.State != StateAlive
	})
	m.nodeLock.RUnlock()

	// Attempt an indirect ping.
	expectedNacks := 0
	selfAddr, selfPort = m.getAdvertise()
	ind := indirectPingReq{
		SeqNo:      ping.SeqNo,
		Target:     node.Addr,
		Port:       node.Port,
		Node:       node.Name,
		SourceAddr: selfAddr,
		SourcePort: selfPort,
		SourceNode: m.config.Name,
	}
	for _, peer := range kNodes {
		// We only expect nack to be sent from peers who understand
		// version 4 of the protocol.
		if ind.Nack = peer.PMax >= 4; ind.Nack {
			expectedNacks++
		}

		if err := m.encodeAndSendMsg(peer.FullAddress(), indirectPingMsg, &ind); err != nil {
			m.logger.Printf("[ERR] memberlist: Failed to send indirect ping: %s", err)
		}
	}

	// Also make an attempt to contact the node directly over TCP. This
	// helps prevent confused clients who get isolated from UDP traffic
	// but can still speak TCP (which also means they can possibly report
	// misinformation to other nodes via anti-entropy), avoiding flapping in
	// the cluster.
	//
	// This is a little unusual because we will attempt a TCP ping to any
	// member who understands version 3 of the protocol, regardless of
	// which protocol version we are speaking. That's why we've included a
	// config option to turn this off if desired.
	fallbackCh := make(chan bool, 1)

	disableTcpPings := m.config.DisableTcpPings ||
		(m.config.DisableTcpPingsForNode != nil && m.config.DisableTcpPingsForNode(node.Name))
	if (!disableTcpPings) && (node.PMax >= 3) {
		go func() {
			defer close(fallbackCh)
			didContact, err := m.sendPingAndWaitForAck(node.FullAddress(), ping, deadline)
			if err != nil {
				m.logger.Printf("[ERR] memberlist: Failed fallback ping: %s", err)
			} else {
				fallbackCh <- didContact
			}
		}()
	} else {
		close(fallbackCh)
	}

	// Wait for the acks or timeout. Note that we don't check the fallback
	// channel here because we want to issue a warning below if that's the
	// *only* way we hear back from the peer, so we have to let this time
	// out first to allow the normal UDP-based acks to come in.
	select {
	case v := <-ackCh:
		if v.Complete == true {
			return
		}
	}

	// Finally, poll the fallback channel. The timeouts are set such that
	// the channel will have something or be closed without having to wait
	// any additional time here.
	for didContact := range fallbackCh {
		if didContact {
			m.logger.Printf("[WARN] memberlist: Was able to connect to %s but other probes failed, network may be misconfigured", node.Name)
			return
		}
	}

	// Update our self-awareness based on the results of this failed probe.
	// If we don't have peers who will send nacks then we penalize for any
	// failed probe as a simple health metric. If we do have peers to nack
	// verify, then we can use that as a more sophisticated measure of self-
	// health because we assume them to be working, and they can help us
	// decide if the probed node was really dead or if it was something wrong
	// with ourselves.
	awarenessDelta = 0
	if expectedNacks > 0 {
		if nackCount := len(nackCh); nackCount < expectedNacks {
			awarenessDelta += (expectedNacks - nackCount)
		}
	} else {
		awarenessDelta += 1
	}

	// No acks received from target, suspect it as failed.
	m.logger.Printf("[INFO] memberlist: Suspect %s has failed, no acks received", node.Name)
	s := suspect{Incarnation: node.Incarnation, Node: node.Name, From: m.config.Name}
	m.suspectNode(&s)
}
```
在直接探测一个节点失败之后，会进入该流程，逻辑如下：
* 1、从其它alive节点中随机选择几个（可配置），向其发送ping消息：序列号，目标节点ip和端口（也就是刚刚直接探测失败的节点），源ip和端口
* 2、检查目标节点是否有应答，如果没有期望数量的应答，则将该节点置为suspect状态

#### 2、状态合并（push/pull消息）
周期性的从已知的alive的集群节点中选1个节点进行push/pull交换信息
交换的信息包含2种
* a) 集群信息
* b) 用户自定义的状态信息，需要1个实现Delegate接口的struct

具体实现如下：
```go
func (m *Memberlist) pushPull() {
	// Get a random live node
	m.nodeLock.RLock()
	nodes := kRandomNodes(1, m.nodes, func(n *nodeState) bool {
		return n.Name == m.config.Name ||
			n.State != StateAlive
	})
	m.nodeLock.RUnlock()

	// If no nodes, bail
	if len(nodes) == 0 {
		return
	}
	node := nodes[0]

	// Attempt a push pull
	if err := m.pushPullNode(node.FullAddress(), false); err != nil {
		m.logger.Printf("[ERR] memberlist: Push/Pull with %s failed: %s", node.Name, err)
	}
}

// pushPullNode does a complete state exchange with a specific node.
func (m *Memberlist) pushPullNode(a Address, join bool) error {
	defer metrics.MeasureSince([]string{"memberlist", "pushPullNode"}, time.Now())

	// Attempt to send and receive with the node
	remote, userState, err := m.sendAndReceiveState(a, join)
	if err != nil {
		return err
	}

	if err := m.mergeRemoteState(join, remote, userState); err != nil {
		return err
	}
	return nil
}
```
从alive状态的节点中随机选择一个，用当前节点与该节点建立连接，然后做如下操作：

##### 2.1 发送当前节点的状态信息
部分代码如下：
```go
localNodes := make([]pushNodeState, len(m.nodes))
for idx, n := range m.nodes {
	localNodes[idx].Name = n.Name
	localNodes[idx].Addr = n.Addr
	localNodes[idx].Port = n.Port
	localNodes[idx].Incarnation = n.Incarnation
	localNodes[idx].State = n.State
	localNodes[idx].Meta = n.Meta
	localNodes[idx].Vsn = []uint8{
		n.PMin, n.PMax, n.PCur,
		n.DMin, n.DMax, n.DCur,
	}
}
m.nodeLock.RUnlock()

// Get the delegate state
var userData []byte
if m.config.Delegate != nil {
	userData = m.config.Delegate.LocalState(join)
}

// Create a bytes buffer writer
bufConn := bytes.NewBuffer(nil)

// Send our node state
header := pushPullHeader{Nodes: len(localNodes), UserStateLen: len(userData), Join: join}
hd := codec.MsgpackHandle{}
enc := codec.NewEncoder(bufConn, &hd)
```
可以看出，是将当前节点内存中保存的所有节点的相关信息发送给远端节点，具体信息封装在localNodes对象中。

##### 2.2 获取远端节点发送过来的状态信息
代码如下：
```go
func (m *Memberlist) readRemoteState(bufConn io.Reader, dec *codec.Decoder) (bool, []pushNodeState, []byte, error) {
	// Read the push/pull header
	var header pushPullHeader
	if err := dec.Decode(&header); err != nil {
		return false, nil, nil, err
	}

	// Allocate space for the transfer
	remoteNodes := make([]pushNodeState, header.Nodes)

	// Try to decode all the states
	for i := 0; i < header.Nodes; i++ {
		if err := dec.Decode(&remoteNodes[i]); err != nil {
			return false, nil, nil, err
		}
	}

	// Read the remote user state into a buffer
	var userBuf []byte
	if header.UserStateLen > 0 {
		userBuf = make([]byte, header.UserStateLen)
		bytes, err := io.ReadAtLeast(bufConn, userBuf, header.UserStateLen)
		if err == nil && bytes != header.UserStateLen {
			err = fmt.Errorf(
				"Failed to read full user state (%d / %d)",
				bytes, header.UserStateLen)
		}
		if err != nil {
			return false, nil, nil, err
		}
	}

	// For proto versions < 2, there is no port provided. Mask old
	// behavior by using the configured port
	for idx := range remoteNodes {
		if m.ProtocolVersion() < 2 || remoteNodes[idx].Port == 0 {
			remoteNodes[idx].Port = uint16(m.config.BindPort)
		}
	}

	return header.Join, remoteNodes, userBuf, nil
}
```
从代码可以看到，返回的消息跟当前节点发送过去的信息差不多

##### 2.3 将本地节点信息跟远端节点信息合并
```go
func (m *Memberlist) mergeRemoteState(join bool, remoteNodes []pushNodeState, userBuf []byte) error {
	if err := m.verifyProtocol(remoteNodes); err != nil {
		return err
	}

	// Invoke the merge delegate if any
	if join && m.config.Merge != nil {
		nodes := make([]*Node, len(remoteNodes))
		for idx, n := range remoteNodes {
			nodes[idx] = &Node{
				Name:  n.Name,
				Addr:  n.Addr,
				Port:  n.Port,
				Meta:  n.Meta,
				State: n.State,
				PMin:  n.Vsn[0],
				PMax:  n.Vsn[1],
				PCur:  n.Vsn[2],
				DMin:  n.Vsn[3],
				DMax:  n.Vsn[4],
				DCur:  n.Vsn[5],
			}
		}
		if err := m.config.Merge.NotifyMerge(nodes); err != nil {
			return err
		}
	}

	// Merge the membership state
	m.mergeState(remoteNodes)

	// Invoke the delegate for user state
	if userBuf != nil && m.config.Delegate != nil {
		m.config.Delegate.MergeRemoteState(userBuf, join)
	}
	return nil
}
```

#### 3、广播gossip消息
```go
func (m *Memberlist) gossip() {
	defer metrics.MeasureSince([]string{"memberlist", "gossip"}, time.Now())

	// Get some random live, suspect, or recently dead nodes
	m.nodeLock.RLock()
	kNodes := kRandomNodes(m.config.GossipNodes, m.nodes, func(n *nodeState) bool {
		if n.Name == m.config.Name {
			return true
		}

		switch n.State {
		case StateAlive, StateSuspect:
			return false

		case StateDead:
			return time.Since(n.StateChange) > m.config.GossipToTheDeadTime

		default:
			return true
		}
	})
	m.nodeLock.RUnlock()

	// Compute the bytes available
	bytesAvail := m.config.UDPBufferSize - compoundHeaderOverhead
	if m.config.EncryptionEnabled() {
		bytesAvail -= encryptOverhead(m.encryptionVersion())
	}

	for _, node := range kNodes {
		// Get any pending broadcasts
		msgs := m.getBroadcasts(compoundOverhead, bytesAvail)
		if len(msgs) == 0 {
			return
		}

		addr := node.Address()
		if len(msgs) == 1 {
			// Send single message as is
			if err := m.rawSendMsgPacket(node.FullAddress(), &node.Node, msgs[0]); err != nil {
				m.logger.Printf("[ERR] memberlist: Failed to send gossip to %s: %s", addr, err)
			}
		} else {
			// Otherwise create and send a compound message
			compound := makeCompoundMessage(msgs)
			if err := m.rawSendMsgPacket(node.FullAddress(), &node.Node, compound.Bytes()); err != nil {
				m.logger.Printf("[ERR] memberlist: Failed to send gossip to %s: %s", addr, err)
			}
		}
	}
}
```
上面的代码主要做了几件事：
* 1、从满足条件（死亡状态）的节点中选择几个节点（可配置）
* 2、构造广播消息
* 3、依次向那些节点发送广播消息


##### 最后说明
* 1、广播其实也是一种Gossip，发布者并不把消息发给集群中的每一个节点，而是随机挑选n个(默认是3个)，将消息发送出去
* 2、处于dead状态的节点，仍然会被保留在集群信息中一段时间，以便Push/Pull的时候，这个状态能够被扩散出去