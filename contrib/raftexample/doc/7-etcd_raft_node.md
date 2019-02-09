# Etcd Raft Node 结构

## Ready

### Ready 数据结构

Ready是一个只读的结构，封装了那些可以用来读取、持久化、提交，或者发给别的节点的Entry或Message

```go
// Ready encapsulates the entries and messages that are ready to read,
// be saved to stable storage, committed or sent to other peers.
// All fields in Ready are read-only.
type Ready struct {
	// The current volatile state of a Node.
	// SoftState will be nil if there is no update.
	// It is not required to consume or store SoftState.
	*SoftState

	// The current state of a Node to be saved to stable storage BEFORE
	// Messages are sent.
	// HardState will be equal to empty state if there is no update.
	pb.HardState

	// ReadStates can be used for node to serve linearizable read requests locally
	// when its applied index is greater than the index in ReadState.
	// Note that the readState will be returned when raft receives msgReadIndex.
	// The returned is only valid for the request that requested to read.
	ReadStates []ReadState

	// Entries specifies entries to be saved to stable storage BEFORE
	// Messages are sent.
	Entries []pb.Entry

	// Snapshot specifies the snapshot to be saved to stable storage.
	Snapshot pb.Snapshot

	// CommittedEntries specifies entries to be committed to a
	// store/state-machine. These have previously been committed to stable
	// store.
	CommittedEntries []pb.Entry

	// Messages specifies outbound messages to be sent AFTER Entries are
	// committed to stable storage.
	// If it contains a MsgSnap message, the application MUST report back to raft
	// when the snapshot has been received or has failed by calling ReportSnapshot.
	Messages []pb.Message

	// MustSync indicates whether the HardState and Entries must be synchronously
	// written to disk or if an asynchronous write is permissible.
	MustSync bool
}
```

### Ready 初始化

- 从`raft`结构中初始化`Entries`，`CommittedEntries`，以及`Messages`三个字段。
- 如果`hardstate`或者`softstate`有更新，则更新这些字段
- 同样，根据`raft`结构的内容更新snapshot和readStates
- 最后更新MustSync，应用层需要检查这个值来判断是不是要持久化。

```go
func isHardStateEqual(a, b pb.HardState) bool {
	return a.Term == b.Term && a.Vote == b.Vote && a.Commit == b.Commit
}

// IsEmptyHardState returns true if the given HardState is empty.
func IsEmptyHardState(st pb.HardState) bool {
	return isHardStateEqual(st, emptyState)
}

// MustSync returns true if the hard state and count of Raft entries indicate
// that a synchronous write to persistent storage is required.
func MustSync(st, prevst pb.HardState, entsnum int) bool {
	// Persistent state on all servers:
	// (Updated on stable storage before responding to RPCs)
	// currentTerm
	// votedFor
	// log entries[]
	return entsnum != 0 || st.Vote != prevst.Vote || st.Term != prevst.Term
}

func newReady(r *raft, prevSoftSt *SoftState, prevHardSt pb.HardState) Ready {
	rd := Ready{
		Entries:          r.raftLog.unstableEntries(),
		CommittedEntries: r.raftLog.nextEnts(),
		Messages:         r.msgs,
	}
	if softSt := r.softState(); !softSt.equal(prevSoftSt) {
		rd.SoftState = softSt
	}
	if hardSt := r.hardState(); !isHardStateEqual(hardSt, prevHardSt) {
		rd.HardState = hardSt
	}
	if r.raftLog.unstable.snapshot != nil {
		rd.Snapshot = *r.raftLog.unstable.snapshot
	}
	if len(r.readStates) != 0 {
		rd.ReadStates = r.readStates
	}
	rd.MustSync = MustSync(r.hardState(), prevHardSt, len(rd.Entries))
	return rd
}
```

### Ready方法

- `containsUpdates()`检查Ready是不是携带有更新
- `appliedCursor()`返回已经applied的index（`node.Advance()`之后）

```go
// IsEmptySnap returns true if the given Snapshot is empty.
func IsEmptySnap(sp pb.Snapshot) bool {
	return sp.Metadata.Index == 0
}

func (rd Ready) containsUpdates() bool {
	return rd.SoftState != nil || !IsEmptyHardState(rd.HardState) ||
		!IsEmptySnap(rd.Snapshot) || len(rd.Entries) > 0 ||
		len(rd.CommittedEntries) > 0 || len(rd.Messages) > 0 || len(rd.ReadStates) != 0
}

// appliedCursor extracts from the Ready the highest index the client has
// applied (once the Ready is confirmed via Advance). If no information is
// contained in the Ready, returns zero.
func (rd Ready) appliedCursor() uint64 {
	if n := len(rd.CommittedEntries); n > 0 {
		return rd.CommittedEntries[n-1].Index
	}
	if index := rd.Snapshot.Metadata.Index; index > 0 {
		return index
	}
	return 0
}
```

## Node 接口

`Node`代表raft 集群中的几个节点，它的接口如下

```go
// Node represents a node in a raft cluster.
type Node interface {
	// Tick increments the internal logical clock for the Node by a single tick. Election
	// timeouts and heartbeat timeouts are in units of ticks.
	Tick()
	// Campaign causes the Node to transition to candidate state and start campaigning to become leader.
	Campaign(ctx context.Context) error
	// Propose proposes that data be appended to the log. Note that proposals can be lost without
	// notice, therefore it is user's job to ensure proposal retries.
	Propose(ctx context.Context, data []byte) error
	// ProposeConfChange proposes config change.
	// At most one ConfChange can be in the process of going through consensus.
	// Application needs to call ApplyConfChange when applying EntryConfChange type entry.
	ProposeConfChange(ctx context.Context, cc pb.ConfChange) error
	// Step advances the state machine using the given message. ctx.Err() will be returned, if any.
	Step(ctx context.Context, msg pb.Message) error

	// Ready returns a channel that returns the current point-in-time state.
	// Users of the Node must call Advance after retrieving the state returned by Ready.
	//
	// NOTE: No committed entries from the next Ready may be applied until all committed entries
	// and snapshots from the previous one have finished.
	Ready() <-chan Ready

	// Advance notifies the Node that the application has saved progress up to the last Ready.
	// It prepares the node to return the next available Ready.
	//
	// The application should generally call Advance after it applies the entries in last Ready.
	//
	// However, as an optimization, the application may call Advance while it is applying the
	// commands. For example. when the last Ready contains a snapshot, the application might take
	// a long time to apply the snapshot data. To continue receiving Ready without blocking raft
	// progress, it can call Advance before finishing applying the last ready.
	Advance()
	// ApplyConfChange applies config change to the local node.
	// Returns an opaque ConfState protobuf which must be recorded
	// in snapshots. Will never return nil; it returns a pointer only
	// to match MemoryStorage.Compact.
	ApplyConfChange(cc pb.ConfChange) *pb.ConfState

	// TransferLeadership attempts to transfer leadership to the given transferee.
	TransferLeadership(ctx context.Context, lead, transferee uint64)

	// ReadIndex request a read state. The read state will be set in the ready.
	// Read state has a read index. Once the application advances further than the read
	// index, any linearizable read requests issued before the read request can be
	// processed safely. The read state will have the same rctx attached.
	ReadIndex(ctx context.Context, rctx []byte) error

	// Status returns the current status of the raft state machine.
	Status() Status
	// ReportUnreachable reports the given node is not reachable for the last send.
	ReportUnreachable(id uint64)
	// ReportSnapshot reports the status of the sent snapshot. The id is the raft ID of the follower
	// who is meant to receive the snapshot, and the status is SnapshotFinish or SnapshotFailure.
	// Calling ReportSnapshot with SnapshotFinish is a no-op. But, any failure in applying a
	// snapshot (for e.g., while streaming it from leader to follower), should be reported to the
	// leader with SnapshotFailure. When leader sends a snapshot to a follower, it pauses any raft
	// log probes until the follower can apply the snapshot and advance its state. If the follower
	// can't do that, for e.g., due to a crash, it could end up in a limbo, never getting any
	// updates from the leader. Therefore, it is crucial that the application ensures that any
	// failure in snapshot sending is caught and reported back to the leader; so it can resume raft
	// log probing in the follower.
	ReportSnapshot(id uint64, status SnapshotStatus)
	// Stop performs any necessary termination of the Node.
	Stop()
}
```

## node 数据结构

除了`logger`以外，所有的成员都是channel，其中只有`tickc`是有buffer的。

```go


// node is the canonical implementation of the Node interface
type node struct {
	propc      chan msgWithResult
	recvc      chan pb.Message
	confc      chan pb.ConfChange
	confstatec chan pb.ConfState
	readyc     chan Ready
	advancec   chan struct{}
	tickc      chan struct{}
	done       chan struct{}
	stop       chan struct{}
	status     chan chan Status

	logger Logger
}

func newNode() node {
	return node{
		propc:      make(chan msgWithResult),
		recvc:      make(chan pb.Message),
		confc:      make(chan pb.ConfChange),
		confstatec: make(chan pb.ConfState),
		readyc:     make(chan Ready),
		advancec:   make(chan struct{}),
		// make tickc a buffered chan, so raft node can buffer some ticks when the node
		// is busy processing raft messages. Raft node will resume process buffered
		// ticks when it becomes idle.
		tickc:  make(chan struct{}, 128),
		done:   make(chan struct{}),
		stop:   make(chan struct{}),
		status: make(chan chan Status),
	}
}
```

## 启动Node

启动分为首次启动`StartNode()`和重启`RestartNode()`，两者都会：

1. 调用`r := newRaft(c)`创建raft数据结构
2. 调用`n := newNode()`创建node数据结构
3. `go n.run(r)`进入协议的处理

`StartNode()`要稍微复杂些，它在1和2之间：

1. `r.becomeFollower(1, None)`，设置自己为follower，term为1, leader为`None(unit64 0)`
2. 以`Term 1`把所有（TODO:这里包括自己？)peers作为配置变更追加到raftLog中的，其中`Context: peer.Context`在`raftexample`中是一个未初始化的空值。这里应该是有个假设的前提--所有节点都会以同样的顺序添加这些配置信息，也就是它们传入的--peers参数是相同的。
3. `r.raftLog.committed = r.raftLog.lastIndex()`把这些日志标记为`committed`，但是没有标记`applied`，这样后续应用层能从`Ready.CommittedEntries`中发现这些变更并应用
4. 循环`r.addNode(peer.ID)`添加所有节点。注释中说这些节点会添加两次，第二次是应用看到这些配置变更并应用时(`raftexample`中的`rc.confState = *rc.node.ApplyConfChange(cc)`)

这有点像raft thesis中提到的`we recommend that the very first time a cluster is created, one server is initialized with a configuration entry as the first entry in its log. This configuration lists only that one server; it alone forms a majority of its configuration, so it can consider this configuration committed.  Other servers from then on should be initialized with empty logs; they are added to the cluster and learn of the current configuration through the membership change mechanism.`

```go
type Peer struct {
	ID      uint64
	Context []byte
}

// StartNode returns a new Node given configuration and a list of raft peers.
// It appends a ConfChangeAddNode entry for each given peer to the initial log.
func StartNode(c *Config, peers []Peer) Node {
	r := newRaft(c)
	// become the follower at term 1 and apply initial configuration
	// entries of term 1
	r.becomeFollower(1, None)
	for _, peer := range peers {
		cc := pb.ConfChange{Type: pb.ConfChangeAddNode, NodeID: peer.ID, Context: peer.Context}
		d, err := cc.Marshal()
		if err != nil {
			panic("unexpected marshal error")
		}
		e := pb.Entry{Type: pb.EntryConfChange, Term: 1, Index: r.raftLog.lastIndex() + 1, Data: d}
		r.raftLog.append(e)
	}
	// Mark these initial entries as committed.
	// TODO(bdarnell): These entries are still unstable; do we need to preserve
	// the invariant that committed < unstable?
	r.raftLog.committed = r.raftLog.lastIndex()
	// Now apply them, mainly so that the application can call Campaign
	// immediately after StartNode in tests. Note that these nodes will
	// be added to raft twice: here and when the application's Ready
	// loop calls ApplyConfChange. The calls to addNode must come after
	// all calls to raftLog.append so progress.next is set after these
	// bootstrapping entries (it is an error if we try to append these
	// entries since they have already been committed).
	// We do not set raftLog.applied so the application will be able
	// to observe all conf changes via Ready.CommittedEntries.
	for _, peer := range peers {
		r.addNode(peer.ID)
	}

	n := newNode()
	n.logger = c.Logger
	go n.run(r)
	return &n
}

// RestartNode is similar to StartNode but does not take a list of peers.
// The current membership of the cluster will be restored from the Storage.
// If the caller has an existing state machine, pass in the last log index that
// has been applied to it; otherwise use zero.
func RestartNode(c *Config) Node {
	r := newRaft(c)

	n := newNode()
	n.logger = c.Logger
	go n.run(r)
	return &n
}
```

## node 主循环

主循环处理多个channel，其中`readyc` 进行发送，其他所有的channel都是负责接收的

主循环的开始部分有两个检查

一是设置互斥的`readyc`跟`advancec`，如果`advancec`非`nil`，则readyc必须置`nil`；反之，当`advancec`为`nil`时，如果ready包含更新（`rd.containsUpdates()`），则设置`readyc = n.readyc`。

来看看后文中这个两个分支：　
- `case readyc <- rd:`
  - 首先通过多个prev变量记录当前的信息（下一次循环在别的处理逻辑来看就是prev了）
    - `prevSoftSt = rd.SoftState`
    - `prevLastUnstablei = rd.Entries[len(rd.Entries)-1].Index`
    - `prevLastUnstablet = rd.Entries[len(rd.Entries)-1].Term`
    - `prevHardSt = rd.HardState`
    - `prevSnapi = rd.Snapshot.Metadata.Index`
    - `index := rd.appliedCursor()`表示应用层（如果）applied这批更新的话，它的index会是什么。
  - 置空raft结构中的msgs和readStates，并调用`r.reduceUncommittedSize(rd.CommittedEntries)`// 这个函数是干啥的？ raft数据结构的`uncommittedSize`字段的注释说：

  ```go
  	// an estimate of the size of the uncommitted tail of the Raft log. Used to
	// prevent unbounded log growth. Only maintained by the leader. Reset on
	// term changes
  ```

  - 最后设置`advancec = n.advancec`让主循环等待应用层处理完成（通过`node.Advance()`)
- `case <-advancec:`
  - `r.raftLog.appliedTo(applyingToI)`告诉raft协议这个index已经applied
  - `r.raftLog.stableTo(prevLastUnstablei, prevLastUnstablet)`告诉raft协议应用层已经持久化这个index和term，可以认为是satable的。
  - `r.raftLog.stableSnapTo(prevSnapi)`到目前为止的快照也可以认为是stable的
  - 最后`advance =nil`等待下一批ready

`readyc`和`advancec`的处理逻辑大概是这样的，`readyc`先发数据，然后在主循环中打开advancec（因此在循环开始会屏蔽自己）等待应用层处理并确认（`node.Advance()`)，确认完之后`advancec`会关闭，下一次循环开始时打开`readyc`，因此`readyc`可以接着发数据（如果有的话）。

二是检查leader的状态变更，`lead`变量记录了之前leader的信息，如果leader发生了变化并且现在有leader，`propc = n.propc`打开`propc`以允许提交，否则关闭`propc`，就是下面这个分支：
- `case pm := <-propc:`

pm是一个带有回执的消息
```go
type msgWithResult struct {
	m      pb.Message
	result chan error
}
```

- `m := pm.m；m.From = r.id`设置发送人
- `err := r.Step(m)`提交给raft协议处理 //TODO，具体做了啥？该函数的注释说`// Handle the message term, which may result in our stepping down to a follower.`
- 最后的if部分把消息处理结果发回去，官调pm的result channel通知发送者处理结果

- `case m := <-n.recvc:`

// filter out response message from unknown From
收到消息，如果不是响应类型（Msg*Resp)的消息，让`r.Step(m)`去处理

- `case cc := <-n.confc:`

接收到配置变更
- `if cc.NodeID == None`这一段貌似是没有节点状态的修改，只是把raft协议当前的Nodes和Learnder发送给`n.confstatec`，退出循环
- 否则根据`cc.Type`执行不同的动作
  - `pb.ConfChangeAddNode`调用`r.addNode(cc.NodeID)`添加节点
  - `pb.ConfChangeAddLearnerNode`调用`r.addLearner(cc.NodeID)`添加learner
  - `pb.ConfChangeRemoveNode`，如果移除的节点是自己，把propc置nil阻止提交，然后调用`r.removeNode(cc.NodeID)`移除节点。
  - `pb.ConfChangeUpdateNode`啥也不做
最后把更新过的配置发给`n.confstatec`处理


```go
type ConfChange struct {
	ID               uint64        q `protobuf:"varint,1,opt,name=ID" json:"ID"`
	Type             ConfChangeType `protobuf:"varint,2,opt,name=Type,enum=raftpb.ConfChangeType" json:"Type"`
	NodeID           uint64         `protobuf:"varint,3,opt,name=NodeID" json:"NodeID"`
	Context          []byte         `protobuf:"bytes,4,opt,name=Context" json:"Context,omitempty"`
	XXX_unrecognized []byte         `json:"-"`
}
```

- `case <-n.tickc:`

直接让`r.tick()`处理

- `case c := <-n.status:`

执行`c <- getStatus(r)`,
c是一个channel，类型为Status,`getStatus gets a copy of the current raft status.`

- `case <-n.stop:`
close(n.done)，用来退出循环


Note： 这里任何可能阻塞的地方都会等待`n.done`，确保需要的时候可以停止循环。

```go
func (n *node) run(r *raft) {
	var propc chan msgWithResult
	var readyc chan Ready
	var advancec chan struct{}
	var prevLastUnstablei, prevLastUnstablet uint64
	var havePrevLastUnstablei bool
	var prevSnapi uint64
	var applyingToI uint64
	var rd Ready

	lead := None
	prevSoftSt := r.softState()
	prevHardSt := emptyState

	for {
		if advancec != nil {
			readyc = nil
		} else {
			rd = newReady(r, prevSoftSt, prevHardSt)
			if rd.containsUpdates() {
				readyc = n.readyc
			} else {
				readyc = nil
			}
		}

		if lead != r.lead {  // leader发生了变化
			if r.hasLeader() {  // raft协议现在有leader
				if lead == None { // 之前没有leader，现在有了，说明新的leader选出来了
					r.logger.Infof("raft.node: %x elected leader %x at term %d", r.id, r.lead, r.Term)
				} else { // 之前有leader，现在有一个不同的leader，说明leader发生了变化
					r.logger.Infof("raft.node: %x changed leader from %x to %x at term %d", r.id, lead, r.lead, r.Term)
				}
				propc = n.propc
			} else { // 集群现在没leader
				r.logger.Infof("raft.node: %x lost leader %x at term %d", r.id, lead, r.Term)
				propc = nil
			}
			lead = r.lead
		}

		select {
		// TODO: maybe buffer the config propose if there exists one (the way
		// described in raft dissertation)
		// Currently it is dropped in Step silently.
		case pm := <-propc:
			m := pm.m
			m.From = r.id
			err := r.Step(m)
			if pm.result != nil {
				pm.result <- err
				close(pm.result)
			}
		case m := <-n.recvc:
			// filter out response message from unknown From.
			if pr := r.getProgress(m.From); pr != nil || !IsResponseMsg(m.Type) {
				r.Step(m)
			}
		case cc := <-n.confc:
			if cc.NodeID == None { 
				select {
				case n.confstatec <- pb.ConfState{
					Nodes:    r.nodes(),
					Learners: r.learnerNodes()}:
				case <-n.done:
				}
				break
			}
			switch cc.Type {
			case pb.ConfChangeAddNode:
				r.addNode(cc.NodeID)
			case pb.ConfChangeAddLearnerNode:
				r.addLearner(cc.NodeID)
			case pb.ConfChangeRemoveNode:
				// block incoming proposal when local node is
				// removed
				if cc.NodeID == r.id {
					propc = nil
				}
				r.removeNode(cc.NodeID)
			case pb.ConfChangeUpdateNode:
			default:
				panic("unexpected conf type")
			}
			select {
			case n.confstatec <- pb.ConfState{
				Nodes:    r.nodes(),
				Learners: r.learnerNodes()}:
			case <-n.done:
			}
		case <-n.tickc:
			r.tick()
		case readyc <- rd:
			if rd.SoftState != nil {
				prevSoftSt = rd.SoftState
			}
			if len(rd.Entries) > 0 {
				prevLastUnstablei = rd.Entries[len(rd.Entries)-1].Index
				prevLastUnstablet = rd.Entries[len(rd.Entries)-1].Term
				havePrevLastUnstablei = true
			}
			if !IsEmptyHardState(rd.HardState) {
				prevHardSt = rd.HardState
			}
			if !IsEmptySnap(rd.Snapshot) {
				prevSnapi = rd.Snapshot.Metadata.Index
			}
			if index := rd.appliedCursor(); index != 0 {
				applyingToI = index
			}

			r.msgs = nil
			r.readStates = nil
			r.reduceUncommittedSize(rd.CommittedEntries)
			advancec = n.advancec
		case <-advancec:
			if applyingToI != 0 {
				r.raftLog.appliedTo(applyingToI)
				applyingToI = 0
			}
			if havePrevLastUnstablei {
				r.raftLog.stableTo(prevLastUnstablei, prevLastUnstablet)
				havePrevLastUnstablei = false
			}
			r.raftLog.stableSnapTo(prevSnapi)
			advancec = nil
		case c := <-n.status:
			c <- getStatus(r)
		case <-n.stop:
			close(n.done)
			return
		}
	}
}
```

## Node 对外接口

分成两类 
- `Propose()`，`ProposeConfChange()`，`ApplyConfChange()`用来提交和应用变更
- `Ready()`,`Advance()`,`Tick()`用在raft协议的处理

### Propose() 

`Propose()`用于提交请求，[](./2-raftexample.md)的`Raft 协议处理`描述了应用层的调用链。
这个函数本质上是对`stepWithWaitOption()`的封装，直接看这个函数

- 如果消息类型不是`pb.MsgProp`，则认为是收到一条消息
  - 直接发给`n.recv`。这条消息在在`run()`循环的`case m := <-n.recvc:`中交给raft去处理
- 否则认为对于`pb.MsgProp`，发送给`n.proc`，在`run()`循环的`case pm := <-propc`中处理
  - 如果不需要消息的处理结果，把messgae封装为`msgWithResult{m: m}`,`result`留空发给`n.proc`后直接返回nil。
  - 如果要结果的话设置`result`字段，发给`n.proc`之后等待返回值。

```go
func (n *node) Propose(ctx context.Context, data []byte) error {
	return n.stepWait(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Data: data}}})
}

func (n *node) stepWait(ctx context.Context, m pb.Message) error {
	return n.stepWithWaitOption(ctx, m, true)
}

// Step advances the state machine using msgs. The ctx.Err() will be returned,
// if any.
func (n *node) stepWithWaitOption(ctx context.Context, m pb.Message, wait bool) error {
	if m.Type != pb.MsgProp {
		select {
		case n.recvc <- m:
			return nil
		case <-ctx.Done():
			return ctx.Err()
		case <-n.done:
			return ErrStopped
		}
	}
	ch := n.propc
	pm := msgWithResult{m: m}
	if wait {
		pm.result = make(chan error, 1)
	}
	select {
	case ch <- pm:
		if !wait {
			return nil
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	select {
	case rsp := <-pm.result:
		if rsp != nil {
			return rsp
		}
	case <-ctx.Done():
		return ctx.Err()
	case <-n.done:
		return ErrStopped
	}
	return nil
}
```

### ProposeConfChange()

`ProposeConfChange()`同样是对`stepWithWaitOption()`的封装，最终还是会发给`n.propc`，两者殊途同归了。
不同之处在于:

- 忽略发给本地的消息
- 不等待返回结果

```go
func (n *node) ProposeConfChange(ctx context.Context, cc pb.ConfChange) error {
	data, err := cc.Marshal()
	if err != nil {
		return err
	}
	return n.Step(ctx, pb.Message{Type: pb.MsgProp, Entries: []pb.Entry{{Type: pb.EntryConfChange, Data: data}}})
}

func (n *node) Step(ctx context.Context, m pb.Message) error {
	// ignore unexpected local messages receiving over network
	if IsLocalMsg(m.Type) {
		// TODO: return an error?
		return nil
	}
	return n.step(ctx, m)
}

func IsLocalMsg(msgt pb.MessageType) bool {
	return msgt == pb.MsgHup || msgt == pb.MsgBeat || msgt == pb.MsgUnreachable ||
		msgt == pb.MsgSnapStatus || msgt == pb.MsgCheckQuorum
}

func (n *node) step(ctx context.Context, m pb.Message) error {
	return n.stepWithWaitOption(ctx, m, false)
}
```

### ReadIndex()

最终会调用stepWithWaitOption，发给recvc, 调用 raft.step()来处理。 //TODO，这干啥的？项目中似乎没用到

```go
func (n *node) ReadIndex(ctx context.Context, rctx []byte) error {
	return n.step(ctx, pb.Message{Type: pb.MsgReadIndex, Entries: []pb.Entry{{Data: rctx}}})
}
```

### ApplyConfChange()

这个函数时在应用层提交日志时执行的`publishEntries()`
把配置变更的请求发给`n.confc`，在`run()`的`case cc := <-n.confc:`中处理完之后，把结果返回给应用层。

```go
func (n *node) ApplyConfChange(cc pb.ConfChange) *pb.ConfState {
	var cs pb.ConfState
	select {
	case n.confc <- cc:
	case <-n.done:
	}
	select {
	case cs = <-n.confstatec:
	case <-n.done:
	}
	return &cs
}
```

### Ready()

直接返回`n.readyc`让应用层去读取，见[](./6-etcd_raft_usage.md)

```go
func (n *node) Ready() <-chan Ready { return n.readyc }
```

### Advance()

处理完毕可以读下一批数据

```go
func (n *node) Advance() {
	select {
	case n.advancec <- struct{}{}:
	case <-n.done:
	}
}
```

### Tick()

给raft协议的心跳

```go
// Tick increments the internal logical clock for this Node. Election timeouts
// and heartbeat timeouts are in units of ticks.
func (n *node) Tick() {
	select {
	case n.tickc <- struct{}{}:
	case <-n.done:
	default:
		n.logger.Warningf("A tick missed to fire. Node blocks too long!")
	}
}
```

## TODO raft结构需要关注的内容

- Ready()
  - r.softState()
  - r.hardState()
  - r.raftLog...
  - r.msg...
  - r.readStates
- run()
  - r.Step()
  - r.getProgress()
  - r.nodes()
  - r.learnNodes()
  - r.addNode()
  - r.addLearner()
  - r.reduceUncommittedSize()