# Etcd Raft


## Raft Config

`Config contains the parameters to start a raft.`

在raftexample中，新建了这样的配置

```go
	c := &raft.Config{
		ID:                        uint64(rc.id),
		ElectionTick:              10,
		HeartbeatTick:             1,
		Storage:                   rc.raftStorage,
		MaxSizePerMsg:             1024 * 1024,
		MaxInflightMsgs:           256,
		MaxUncommittedEntriesSize: 1 << 30,
	}
```

上面设置的：

- `Applied`当前已经apply的值，只有在重启时可以设置。但是否设置要看应用的需求
- `MaxSizePerMsg`每条消息最大的长度，上面是1mb
- `MaxUncommittedEntriesSize`raft leader能保存的未提交log的总大小，上面是1GB（1 << 30)
- `MaxInflightMsgs`用来限制正在进行的log replication的总数

一些值得注意的：

- `MaxCommittedSizePerReady`: 一次ready最多能apply多大的大小？
- `CheckQuorum`， raft thesis 6.2 节，让leader定时检查quorum确定自己是否还合法
- `PreVote`, raft thesis 9.6节，用来避免节点出现网络分区后重新加入集群导致的干扰。
- `Logger`，（debug用的）日志类
- `DisableProposalForwarding`,是否允许follower把propesal请求转发给leader，在特定的场合有用。

另外一些私有成员
- `peers` 这东西是用来测试的，别用。只有在初次启动时设置（在`raft.StartNode()`中）,否则会panic
- `learners`可以理解为raft的not voting member

`validate()`用于校验Config的合法性，并尝试修复。


```go
// Config contains the parameters to start a raft.
type Config struct {
	// ID is the identity of the local raft. ID cannot be 0.
	ID uint64

	// peers contains the IDs of all nodes (including self) in the raft cluster. It
	// should only be set when starting a new raft cluster. Restarting raft from
	// previous configuration will panic if peers is set. peer is private and only
	// used for testing right now.
	peers []uint64

	// learners contains the IDs of all learner nodes (including self if the
	// local node is a learner) in the raft cluster. learners only receives
	// entries from the leader node. It does not vote or promote itself.
	learners []uint64

	// ElectionTick is the number of Node.Tick invocations that must pass between
	// elections. That is, if a follower does not receive any message from the
	// leader of current term before ElectionTick has elapsed, it will become
	// candidate and start an election. ElectionTick must be greater than
	// HeartbeatTick. We suggest ElectionTick = 10 * HeartbeatTick to avoid
	// unnecessary leader switching.
	ElectionTick int
	// HeartbeatTick is the number of Node.Tick invocations that must pass between
	// heartbeats. That is, a leader sends heartbeat messages to maintain its
	// leadership every HeartbeatTick ticks.
	HeartbeatTick int

	// Storage is the storage for raft. raft generates entries and states to be
	// stored in storage. raft reads the persisted entries and states out of
	// Storage when it needs. raft reads out the previous state and configuration
	// out of storage when restarting.
	Storage Storage
	// Applied is the last applied index. It should only be set when restarting
	// raft. raft will not return entries to the application smaller or equal to
	// Applied. If Applied is unset when restarting, raft might return previous
	// applied entries. This is a very application dependent configuration.
	Applied uint64

	// MaxSizePerMsg limits the max byte size of each append message. Smaller
	// value lowers the raft recovery cost(initial probing and message lost
	// during normal operation). On the other side, it might affect the
	// throughput during normal replication. Note: math.MaxUint64 for unlimited,
	// 0 for at most one entry per message.
	MaxSizePerMsg uint64
	// MaxCommittedSizePerReady limits the size of the committed entries which
	// can be applied.
	MaxCommittedSizePerReady uint64
	// MaxUncommittedEntriesSize limits the aggregate byte size of the
	// uncommitted entries that may be appended to a leader's log. Once this
	// limit is exceeded, proposals will begin to return ErrProposalDropped
	// errors. Note: 0 for no limit.
	MaxUncommittedEntriesSize uint64
	// MaxInflightMsgs limits the max number of in-flight append messages during
	// optimistic replication phase. The application transportation layer usually
	// has its own sending buffer over TCP/UDP. Setting MaxInflightMsgs to avoid
	// overflowing that sending buffer. TODO (xiangli): feedback to application to
	// limit the proposal rate?
	MaxInflightMsgs int

	// CheckQuorum specifies if the leader should check quorum activity. Leader
	// steps down when quorum is not active for an electionTimeout.
	CheckQuorum bool

	// PreVote enables the Pre-Vote algorithm described in raft thesis section
	// 9.6. This prevents disruption when a node that has been partitioned away
	// rejoins the cluster.
	PreVote bool

	// ReadOnlyOption specifies how the read only request is processed.
	//
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	//
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	// CheckQuorum MUST be enabled if ReadOnlyOption is ReadOnlyLeaseBased.
	ReadOnlyOption ReadOnlyOption

	// Logger is the logger used for raft log. For multinode which can host
	// multiple raft group, each raft group can have its own logger
	Logger Logger

	// DisableProposalForwarding set to true means that followers will drop
	// proposals, rather than forwarding them to the leader. One use case for
	// this feature would be in a situation where the Raft leader is used to
	// compute the data of a proposal, for example, adding a timestamp from a
	// hybrid logical clock to data in a monotonically increasing way. Forwarding
	// should be disabled to prevent a follower with an inaccurate hybrid
	// logical clock from assigning the timestamp and then forwarding the data
	// to the leader.
	DisableProposalForwarding bool
}
```

## Raft 数据结构

Raft数据结构中封装了非常多的成员，其中一部分来自配置信息;一部分比较复杂的成员单独放到下一节详细描述;另一些我在行内直接注释。

- pendingConfIndex
etcd raft对成员变更的变种实现
> To ensure there is no attempt to commit two membership changes at once by matching log positions (which would be unsafe since they should have different quorum requirements), any proposed membership change is simply disallowed while any uncommitted change appears in the leader's log.

- tick/step
这是两个没有初始化的函数指针，他们是在状态转换，比如`becomeLeader()`的时候设置的，不同的角色需要执行的动作不同

- electionElapsed/heartbeatElapsed
这是两个计数器，从0开始递增，当它们的值大于等于	heartbeatTimeout/electionTimeout 时就会重置为0,并且触发特定的动作 -- Step 一条特定的消息.
注意`electionElapsed`对leader/candidate 以及 follower 的含义稍有不同

```go
// Possible values for StateType.
const (
	StateFollower StateType = iota
	StateCandidate
	StateLeader
	StatePreCandidate
	numStates
)

type raft struct {
	id uint64			// 由Config初始化

	Term uint64
	Vote uint64

	readStates []ReadState 	// 貌似是ReadOnly请求相关的数据结构

	// the log
	raftLog *raftLog		// Raft log Entry的数据结构

	maxMsgSize         uint64		// 由Config初始化
	maxUncommittedSize uint64		// 由Config初始化
	maxInflight        int			// 由Config初始化
	prs                map[uint64]*Progress  // 集群成员AE rpc的进度（match, next)，由leader维护
	learnerPrs         map[uint64]*Progress  // non voting member的进度
	matchBuf           uint64Slice			// 见section开始部分的说明

	state StateType  // 当前状态，leader，follower，candidate，这里还有一个额外的pre-candidate

	// isLearner is true if the local raft node is a learner.
	isLearner bool

	votes map[uint64]bool  // preVote 阶段用来收集voter，我要是去竞选leader，你会给我投票吗？

	msgs []pb.Message   // TODO

	// the leader id
	lead uint64 
	// leadTransferee is id of the leader transfer target when its value is not zero.
	// Follow the procedure defined in raft thesis 3.10.
	leadTransferee uint64  //被转移的对象
	// Only one conf change may be pending (in the log, but not yet
	// applied) at a time. This is enforced via pendingConfIndex, which
	// is set to a value >= the log index of the latest pending
	// configuration change (if any). Config changes are only allowed to
	// be proposed if the leader's applied index is greater than this
	// value.
	pendingConfIndex uint64   // 简单的说，如果applied index小于这个值，不允许提交变更
	// an estimate of the size of the uncommitted tail of the Raft log. Used to
	// prevent unbounded log growth. Only maintained by the leader. Reset on
	// term changes.
	uncommittedSize uint64

	readOnly *readOnly 	// readonly 结构

	// number of ticks since it reached last electionTimeout when it is leader
	// or candidate.
	// number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	electionElapsed int

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int 

	checkQuorum bool     // 由Config初始化
	preVote     bool     // 由Config初始化

	heartbeatTimeout int		// 由Config初始化
	electionTimeout  int		// 由Config初始化
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int  // 根据electiontimout算出一个随机的超时时间
	disableProposalForwarding bool   // 由Config初始化

	tick func()
	step stepFunc

	logger Logger			// 由Config初始化
}
```

## 新建Raft数据结构

- 首先通过`c.validate()`验证配置的有效性
- `raftlog := newLogWithSize(c.Storage, c.Logger, c.MaxCommittedSizePerReady)` 新建`raftlog`对象  //TODO
- `hs, cs, err := c.Storage.InitialState()`获取初始的hardstate和confstate，ConfState中包含`Nodes`和`Learners`
- peers 和 learners是后面用来初始化`prs`和`learnerPrs`的，他们的值
  - 要么是初次启动时从Confg中设置，这种方式貌似并没有使用，而是在`node.StartNode()`时直接调用`raft.addNode()`添加
  - 要么是从Storage中读出来（上面的cs，来自快照）
- 使用传入的Config初始化raft数据结构
- 根据上面的peers和learners初始化`prs`和`learnerPrs`，这仅在有快照的情况下发生。 //TODO
- 如果storage中的hardstate非空，加载进来
- 如果c.Applied > 0，直接`raftlog.appliedTo(c.Applied)`应用到这个log //TODO 什么场景会用这个功能？
- 不管三七二十一，先`r.becomeFollower(r.Term, None)`把自己变成follower再说
- 最后打印raft当前的信息并返回raft数据结构

```go
func (r *raft) loadState(state pb.HardState) {
	if state.Commit < r.raftLog.committed || state.Commit > r.raftLog.lastIndex() {
		r.logger.Panicf("%x state.commit %d is out of range [%d, %d]", r.id, state.Commit, r.raftLog.committed, r.raftLog.lastIndex())
	}
	r.raftLog.committed = state.Commit
	r.Term = state.Term
	r.Vote = state.Vote
}

func newRaft(c *Config) *raft {
	if err := c.validate(); err != nil {
		panic(err.Error())
	}
	raftlog := newLogWithSize(c.Storage, c.Logger, c.MaxCommittedSizePerReady)
	hs, cs, err := c.Storage.InitialState()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	peers := c.peers
	learners := c.learners
	if len(cs.Nodes) > 0 || len(cs.Learners) > 0 {
		if len(peers) > 0 || len(learners) > 0 {
			// TODO(bdarnell): the peers argument is always nil except in
			// tests; the argument should be removed and these tests should be
			// updated to specify their nodes through a snapshot.
			panic("cannot specify both newRaft(peers, learners) and ConfState.(Nodes, Learners)")
		}
		peers = cs.Nodes
		learners = cs.Learners
	}
	r := &raft{
		id:                        c.ID,
		lead:                      None,
		isLearner:                 false,
		raftLog:                   raftlog,
		maxMsgSize:                c.MaxSizePerMsg,
		maxInflight:               c.MaxInflightMsgs,
		maxUncommittedSize:        c.MaxUncommittedEntriesSize,
		prs:                       make(map[uint64]*Progress),
		learnerPrs:                make(map[uint64]*Progress),
		electionTimeout:           c.ElectionTick,
		heartbeatTimeout:          c.HeartbeatTick,
		logger:                    c.Logger,
		checkQuorum:               c.CheckQuorum,
		preVote:                   c.PreVote,
		readOnly:                  newReadOnly(c.ReadOnlyOption),
		disableProposalForwarding: c.DisableProposalForwarding,
	}
	for _, p := range peers {
		r.prs[p] = &Progress{Next: 1, ins: newInflights(r.maxInflight)}
	}
	for _, p := range learners {
		if _, ok := r.prs[p]; ok {
			panic(fmt.Sprintf("node %x is in both learner and peer list", p))
		}
		r.learnerPrs[p] = &Progress{Next: 1, ins: newInflights(r.maxInflight), IsLearner: true}
		if r.id == p {
			r.isLearner = true
		}
	}

	if !isHardStateEqual(hs, emptyState) {
		r.loadState(hs)
	}
	if c.Applied > 0 {
		raftlog.appliedTo(c.Applied)
	}
	r.becomeFollower(r.Term, None)

	var nodesStrs []string
	for _, n := range r.nodes() {
		nodesStrs = append(nodesStrs, fmt.Sprintf("%x", n))
	}

	r.logger.Infof("newRaft %x [peers: [%s], term: %d, commit: %d, applied: %d, lastindex: %d, lastterm: %d]",
		r.id, strings.Join(nodesStrs, ","), r.Term, r.raftLog.committed, r.raftLog.applied, r.raftLog.lastIndex(), r.raftLog.lastTerm())
	return r
}
```

## Progress相关 

获取peer的Progess，同时可以用来检查某个id是不是在voter或者learner中

```go
func (r *raft) getProgress(id uint64) *Progress {
	if pr, ok := r.prs[id]; ok {
		return pr
	}

	return r.learnerPrs[id]
}
```

这是新建progress？

- 如果是learner，给`r.learnerPrs`对应的id新建Progrss
- 如果不是，尝试把它从`r.learnerPrs`中删除，然后给`r.Prs`对应的id新建Progress

> learner 可以转换为 voter
> voter 不能转换为 learner
> 一个id要么是voter，要么是learner，要么都不是

```go
func (r *raft) setProgress(id, match, next uint64, isLearner bool) {
	if !isLearner {
		delete(r.learnerPrs, id)
		r.prs[id] = &Progress{Next: next, Match: match, ins: newInflights(r.maxInflight)}
		return
	}

	if _, ok := r.prs[id]; ok {
		panic(fmt.Sprintf("%x unexpected changing from voter to learner for %x", r.id, id))
	}
	r.learnerPrs[id] = &Progress{Next: next, Match: match, ins: newInflights(r.maxInflight), IsLearner: true}
}
```

直接从prs和learnerPrs删掉就是了

```go
func (r *raft) delProgress(id uint64) {
	delete(r.prs, id)
	delete(r.learnerPrs, id)
}
```

为每一个voter和learner执行函数f ，参数id和pr是当前的id和Progress

```go
func (r *raft) forEachProgress(f func(id uint64, pr *Progress)) {
	for id, pr := range r.prs {
		f(id, pr)
	}

	for id, pr := range r.learnerPrs {
		f(id, pr)
	}
}
```

## 添加节点

添加节点本身很简单
- 如果不存在，调用`setProgress()`新建一个就是,match设置为0, next设置为`r.raftLog.lastIndex()+1`
- 如果存在的话，就涉及到状态转换了，前面说过
  - 不能从voter转换为learner
  - 可以从learner转为voter，直接复用原来的Progress就行，不过得吧isLearner值为false
  - 如果状态相同`if isLearner == pr.IsLearner`,那就是初次启动的时候重复添加了，忽略就行
- 如果添加的id是自己，设置一下自己的isLearner状态
- 最后更新pr.RecentActive = true，防止`CheckQuorum()`失败——我们加了一个node耶

```go
func (r *raft) addNode(id uint64) {
	r.addNodeOrLearnerNode(id, false)
}

func (r *raft) addLearner(id uint64) {
	r.addNodeOrLearnerNode(id, true)
}

func (r *raft) addNodeOrLearnerNode(id uint64, isLearner bool) {
	pr := r.getProgress(id)
	if pr == nil {
		r.setProgress(id, 0, r.raftLog.lastIndex()+1, isLearner)
	} else {
		if isLearner && !pr.IsLearner {
			// can only change Learner to Voter
			r.logger.Infof("%x ignored addLearner: do not support changing %x from raft peer to learner.", r.id, id)
			return
		}

		if isLearner == pr.IsLearner {
			// Ignore any redundant addNode calls (which can happen because the
			// initial bootstrapping entries are applied twice).
			return
		}

		// change Learner to Voter, use origin Learner progress
		delete(r.learnerPrs, id)
		pr.IsLearner = false
		r.prs[id] = pr
	}

	if r.id == id {
		r.isLearner = isLearner
	}

	// When a node is first added, we should mark it as recently active.
	// Otherwise, CheckQuorum may cause us to step down if it is invoked
	// before the added node has a chance to communicate with us.
	pr = r.getProgress(id)
	pr.RecentActive = true
}
```

## 删除节点

删完节点之后要做的两个额外处理

TODO:下列函数的实现
- 现在quorum变小了，看看是不是能commit了`r.maybeCommit()`，是的话`r.bcastAppend()`
- 如果我是leader并且被删除的是leader transfer的目标，`r.abortLeaderTransfer()`

`r.abortLeaderTransfer()`的实现特别简单

```go
func (r *raft) removeNode(id uint64) {
	r.delProgress(id)

	// do not try to commit or abort transferring if there is no nodes in the cluster.
	if len(r.prs) == 0 && len(r.learnerPrs) == 0 {
		return
	}

	// The quorum size is now smaller, so see if any pending entries can
	// be committed.
	if r.maybeCommit() {
		r.bcastAppend()
	}
	// If the removed node is the leadTransferee, then abort the leadership transferring.
	if r.state == StateLeader && r.leadTransferee == id {
		r.abortLeaderTransfer()
	}
}

func (r *raft) abortLeaderTransfer() {
	r.leadTransferee = None
}
```

## 获取节点和状态信息

获取voter和learner

```go
func (r *raft) nodes() []uint64 {
	nodes := make([]uint64, 0, len(r.prs))
	for id := range r.prs {
		nodes = append(nodes, id)
	}
	sort.Sort(uint64Slice(nodes))
	return nodes
}

func (r *raft) learnerNodes() []uint64 {
	nodes := make([]uint64, 0, len(r.learnerPrs))
	for id := range r.learnerPrs {
		nodes = append(nodes, id)
	}
	sort.Sort(uint64Slice(nodes))
	return nodes
}

```

上面的unit64Slice是干嘛的？它实现了Sort接口

```go
// uint64Slice implements sort interface
type uint64Slice []uint64

func (p uint64Slice) Len() int           { return len(p) }
func (p uint64Slice) Less(i, j int) bool { return p[i] < p[j] }
func (p uint64Slice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] 
```

softstate 和 hardstate

```go
func (r *raft) softState() *SoftState { return &SoftState{Lead: r.lead, RaftState: r.state} }

func (r *raft) hardState() pb.HardState {
	return pb.HardState{
		Term:   r.Term,
		Vote:   r.Vote,
		Commit: r.raftLog.committed,
	}
}
```

## 发送消息

### 发送Append

`sendAppend()`其实是对`maybeSendAppend()`的封装，这个函数的参数是`to`，发给谁;`sendIfEmpty`是否发空的消息
- 如果peer已经pause，则暂停发送
- `term, errt := r.raftLog.term(pr.Next - 1)`获取上一条Log的term，用于log match
- `ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)`获取最大为`r.maxMsgSize`的entries
- 如果取不到term 或者 ents，说明peer的状态是空的，它大概一条entry都没有，组装一个snapshot 算了(`m.Type = pb.MsgSnap`) // TODO
  - 如果` r.raftLog.snapshot()`报错说`ErrSnapshotTemporarilyUnavailable`，就返回好了，晚点再试；如果是其他错误，我也不知道是啥，好慌 ，还是panic吧
  - 调用`pr.becomeSnapshot(sindex)`把peer状态变为snapshot, （`IsPaused()`会返回True）
- 如果peer有状态，那就发正常的AppenEntries RPC (`m.Type = pb.MsgApp`)
  - `m.Index = pr.Next - 1` 和 `m.LogTerm = term`用于Log Match
  - `m.Commit = r.raftLog.committed`告诉follower commit index
  - 接着根据peer的状态采不同的行为
    - 如果是正常的`ProgressStateReplicate`，
	  - `last := m.Entries[n-1].Index`先获取最后一条entry的index
	  - `pr.optimisticUpdate(last)`把peer的Next index更新为要发送的最后一条
	  - `pr.ins.add(last)`把这个index加到progress的inflight （滑动窗口）中
	- 如果是`ProgressStateProbe`，发完这一条就得先暂停一下了，//TODO，谁再把probe打开？
- 最后调用`m.send`把组装好的消息发出去

```go
// sendAppend sends an append RPC with new entries (if any) and the
// current commit index to the given peer.
func (r *raft) sendAppend(to uint64) {
	r.maybeSendAppend(to, true)
}

// maybeSendAppend sends an append RPC with new entries to the given peer,
// if necessary. Returns true if a message was sent. The sendIfEmpty
// argument controls whether messages with no entries will be sent
// ("empty" messages are useful to convey updated Commit indexes, but
// are undesirable when we're sending multiple messages in a batch).
func (r *raft) maybeSendAppend(to uint64, sendIfEmpty bool) bool {
	pr := r.getProgress(to)
	if pr.IsPaused() {
		return false
	}
	m := pb.Message{}
	m.To = to

	term, errt := r.raftLog.term(pr.Next - 1)
	ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)
	if len(ents) == 0 && !sendIfEmpty {
		return false
	}

	if errt != nil || erre != nil { // send snapshot if we failed to get term or entries
		if !pr.RecentActive {
			r.logger.Debugf("ignore sending snapshot to %x since it is not recently active", to)
			return false
		}

		m.Type = pb.MsgSnap
		snapshot, err := r.raftLog.snapshot()
		if err != nil {
			if err == ErrSnapshotTemporarilyUnavailable {
				r.logger.Debugf("%x failed to send snapshot to %x because snapshot is temporarily unavailable", r.id, to)
				return false
			}
			panic(err) // TODO(bdarnell)
		}
		if IsEmptySnap(snapshot) {
			panic("need non-empty snapshot")
		}
		m.Snapshot = snapshot
		sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
		r.logger.Debugf("%x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
			r.id, r.raftLog.firstIndex(), r.raftLog.committed, sindex, sterm, to, pr)
		pr.becomeSnapshot(sindex)
		r.logger.Debugf("%x paused sending replication messages to %x [%s]", r.id, to, pr)
	} else {
		m.Type = pb.MsgApp
		m.Index = pr.Next - 1
		m.LogTerm = term
		m.Entries = ents
		m.Commit = r.raftLog.committed
		if n := len(m.Entries); n != 0 {
			switch pr.State {
			// optimistically increase the next when in ProgressStateReplicate
			case ProgressStateReplicate:
				last := m.Entries[n-1].Index
				pr.optimisticUpdate(last)
				pr.ins.add(last)
			case ProgressStateProbe:
				pr.pause()
			default:
				r.logger.Panicf("%x is sending append in unhandled state %s", r.id, pr.State)
			}
		}
	}
	r.send(m)
	return true
}
```

### send - 实际发送

`send()`需要检查Term的合法性，
- Leader election相关的消息需要，//TODO Resp中Term的含义，如果我接受选举，Leader的Term至少应该是
- 其他类型的消息不需要 //TODO
  - 但是pb.MsgProp 和 pb.MsgReadIndex 会自动把自己的`r.Term`带上，注释没看太明白，不过这两种消息是需要forward给leader的

最后把这些消息添加到r.msgs中，node.Ready()会取走这些msgs，应用层需要自己把这些消息发出去

```go
// send persists state to stable storage and then sends to its mailbox.
func (r *raft) send(m pb.Message) {
	m.From = r.id
	if m.Type == pb.MsgVote || m.Type == pb.MsgVoteResp || m.Type == pb.MsgPreVote || m.Type == pb.MsgPreVoteResp {
		if m.Term == 0 {
			// All {pre-,}campaign messages need to have the term set when
			// sending.
			// - MsgVote: m.Term is the term the node is campaigning for,
			//   non-zero as we increment the term when campaigning.
			// - MsgVoteResp: m.Term is the new r.Term if the MsgVote was
			//   granted, non-zero for the same reason MsgVote is
			// - MsgPreVote: m.Term is the term the node will campaign,
			//   non-zero as we use m.Term to indicate the next term we'll be
			//   campaigning for
			// - MsgPreVoteResp: m.Term is the term received in the original
			//   MsgPreVote if the pre-vote was granted, non-zero for the
			//   same reasons MsgPreVote is
			panic(fmt.Sprintf("term should be set when sending %s", m.Type))
		}
	} else {
		if m.Term != 0 {
			panic(fmt.Sprintf("term should not be set when sending %s (was %d)", m.Type, m.Term))
		}
		// do not attach term to MsgProp, MsgReadIndex
		// proposals are a way to forward to the leader and
		// should be treated as local message.
		// MsgReadIndex is also forwarded to leader.
		if m.Type != pb.MsgProp && m.Type != pb.MsgReadIndex {
			m.Term = r.Term
		}
	}
	r.msgs = append(r.msgs, m)
}
```

### 发送心跳

心跳同样是对`send()`的封装，最重要计算commit index `commit := min(r.getProgress(to).Match, r.raftLog.committed)`

```go
// sendHeartbeat sends a heartbeat RPC to the given peer.
func (r *raft) sendHeartbeat(to uint64, ctx []byte) {
	// Attach the commit as min(to.matched, r.committed).
	// When the leader sends out heartbeat message,
	// the receiver(follower) might not be matched with the leader
	// or it might not have all the committed entries.
	// The leader MUST NOT forward the follower's commit to
	// an unmatched index.
	commit := min(r.getProgress(to).Match, r.raftLog.committed)
	m := pb.Message{
		To:      to,
		Type:    pb.MsgHeartbeat,
		Commit:  commit,
		Context: ctx,
	}

	r.send(m)
}
```

## 广播

### bcastAppend

`brcastAppend()` 最终是对`maybeSendAppend()`的封装，给所有follower发一次

```go
// bcastAppend sends RPC, with entries to all peers that are not up-to-date
// according to the progress recorded in r.prs.
func (r *raft) bcastAppend() {
	r.forEachProgress(func(id uint64, _ *Progress) {
		if id == r.id {
			return
		}

		r.sendAppend(id)
	})
}

```
### bcastHeartBeat

`bcastHeartBeat()`最终是对`sendHeartbeat()`的封装，给所有follower发一次
如果raft有正在等待的readonly request，把最后一个request context 带上 // TODO

```go
// bcastHeartbeat sends RPC, without entries to all the peers.
func (r *raft) bcastHeartbeat() {
	lastCtx := r.readOnly.lastPendingRequestCtx()
	if len(lastCtx) == 0 {
		r.bcastHeartbeatWithCtx(nil)
	} else {
		r.bcastHeartbeatWithCtx([]byte(lastCtx))
	}
}

func (r *raft) bcastHeartbeatWithCtx(ctx []byte) {
	r.forEachProgress(func(id uint64, _ *Progress) {
		if id == r.id {
			return
		}
		r.sendHeartbeat(id, ctx)
	})
}
```

## 随机超时器

`resetRandomizedElectionTimeout()`，状态转换reset的时候设置r.randomizedElectionTimeout
`pastElectionTimeout()检查election timeout是否到了

```go
// pastElectionTimeout returns true iff r.electionElapsed is greater
// than or equal to the randomized election timeout in
// [electiontimeout, 2 * electiontimeout - 1].
func (r *raft) pastElectionTimeout() bool {
	return r.electionElapsed >= r.randomizedElectionTimeout
}

func (r *raft) resetRandomizedElectionTimeout() {
	r.randomizedElectionTimeout = r.electionTimeout + globalRand.Intn(r.electionTimeout)
}

// lockedRand is a small wrapper around rand.Rand to provide
// synchronization among multiple raft groups. Only the methods needed
// by the code are exposed (e.g. Intn).
type lockedRand struct {
	mu   sync.Mutex
	rand *rand.Rand
}

func (r *lockedRand) Intn(n int) int {
	r.mu.Lock()
	v := r.rand.Intn(n)
	r.mu.Unlock()
	return v
}

var globalRand = &lockedRand{
	rand: rand.New(rand.NewSource(time.Now().UnixNano())),
}
```

## 状态转换

### reset

`reset()`需要一个term参数，重置现在的状态
- 如果term发生了变化，重置Vote，否则保留
- 重置各种计数器
- 取消leadertransfer
- 重置每一个voter/learner的Progress状态

```go
func (r *raft) reset(term uint64) {
	if r.Term != term {
		r.Term = term
		r.Vote = None
	}
	r.lead = None

	r.electionElapsed = 0
	r.heartbeatElapsed = 0
	r.resetRandomizedElectionTimeout()

	r.abortLeaderTransfer()

	r.votes = make(map[uint64]bool)
	r.forEachProgress(func(id uint64, pr *Progress) {
		*pr = Progress{Next: r.raftLog.lastIndex() + 1, ins: newInflights(r.maxInflight), IsLearner: pr.IsLearner}
		if id == r.id {
			pr.Match = r.raftLog.lastIndex()
		}
	})

	r.pendingConfIndex = 0
	r.uncommittedSize = 0
	r.readOnly = newReadOnly(r.readOnly.option)
}
```

### follower

`becomeFollower()`比较简单，`reset()`并且设置
- `step`,`tick`两个函数指针
- `state` 为`StateFollower`
- lead，新的leader，可能为None

```go
func (r *raft) becomeFollower(term uint64, lead uint64) {
	r.step = stepFollower
	r.reset(term)
	r.tick = r.tickElection
	r.lead = lead
	r.state = StateFollower
	r.logger.Infof("%x became follower at term %d", r.id, r.Term)
}

### candidate

`becomeCandidate()`的几个特点：

- 不能从Leader转换过来
- Term 需要 + 1，这是唯一会增加term的地方
- 给自己投一票 `r.Vote = r.id`

func (r *raft) becomeCandidate() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateLeader {
		panic("invalid transition [leader -> candidate]")
	}
	r.step = stepCandidate
	r.reset(r.Term + 1)
	r.tick = r.tickElection
	r.Vote = r.id
	r.state = StateCandidate
	r.logger.Infof("%x became candidate at term %d", r.id, r.Term)
}

### precandidate

`becomePreCandidate()`只有启用PreVote的时候才会调用，跟`becomeCandiate()`类似，但是：
- 不增加term
- 不给自己投票
- 需要征求voter的意见(r.votes)

func (r *raft) becomePreCandidate() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateLeader {
		panic("invalid transition [leader -> pre-candidate]")
	}
	// Becoming a pre-candidate changes our step functions and state,
	// but doesn't change anything else. In particular it does not increase
	// r.Term or change r.Vote.
	r.step = stepCandidate
	r.votes = make(map[uint64]bool)
	r.tick = r.tickElection
	r.state = StatePreCandidate
	r.logger.Infof("%x became pre-candidate at term %d", r.id, r.Term)
}

### leader

`becomeLeader()`:

- 不能从Follower切过来
- `r.lead = r.id`把自己设置为leader
- `r.prs[r.id].becomeReplicate()`让自己的Progress进入replicate状态，我自己就是leader耶
- `r.pendingConfIndex = r.raftLog.lastIndex()`保守起见先不允许配置变更
- `r.appendEntry(emptyEnt)`提交一个no op的entry，来保证之前的log都可以commit （raft thesis 3.6.2)
- 上一步`appendEntry()`的时候会调用`increaseUncommittedSize()`把这个no op entry的大小算进去，这里是个special case不希望算进去，因此继续调用`reduceUncommittedSize()`减去这个大小

func (r *raft) becomeLeader() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateFollower {
		panic("invalid transition [follower -> leader]")
	}
	r.step = stepLeader
	r.reset(r.Term)
	r.tick = r.tickHeartbeat
	r.lead = r.id
	r.state = StateLeader
	// Followers enter replicate mode when they've been successfully probed
	// (perhaps after having received a snapshot as a result). The leader is
	// trivially in this state. Note that r.reset() has initialized this
	// progress with the last index already.
	r.prs[r.id].becomeReplicate()

	// Conservatively set the pendingConfIndex to the last index in the
	// log. There may or may not be a pending config change, but it's
	// safe to delay any future proposals until we commit all our
	// pending log entries, and scanning the entire tail of the log
	// could be expensive.
	r.pendingConfIndex = r.raftLog.lastIndex()

	emptyEnt := pb.Entry{Data: nil}
	if !r.appendEntry(emptyEnt) {
		// This won't happen because we just called reset() above.
		r.logger.Panic("empty entry was dropped")
	}
	// As a special case, don't count the initial empty entry towards the
	// uncommitted log quota. This is because we want to preserve the
	// behavior of allowing one entry larger than quota if the current
	// usage is zero.
	r.reduceUncommittedSize([]pb.Entry{emptyEnt})
	r.logger.Infof("%x became leader at term %d", r.id, r.Term)
}
```

检查自己是不是能够变成leader，learner是不行的

```go
// promotable indicates whether state machine can be promoted to leader,
// which is true when its own id is in progress list.
func (r *raft) promotable() bool {
	_, ok := r.prs[r.id]
	return ok
}
```

检查cluster是不是有leader

```go
func (r *raft) hasLeader() bool { return r.lead != None }
```

## tick

node的主循环中，每当ticker超时的时候，都会调用一次`r.tick()`，这是一个函数指针，根据角色的不同指向两个不同的对象：

follower : tickElection
candidate: tickElection
preCandidate: tickElection
leader: tickHeartBeat


### tickElection

用于非leader角色
每次调用`tickElection()`累加r.electionElapsed
当自己可以被提升了leader，并且election timeout超时的时候，发一条`pb.MsgHup`, //TODO 这是选举还是啥？

肯定还有个别的地方会重置r.electionElapsed，估计在Step里面

```go
// tickElection is run by followers and candidates after r.electionTimeout.
func (r *raft) tickElection() {
	r.electionElapsed++

	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```

### tickHeartBeat

leader专属，除了累加electionElapsed以外，还会累加heartbeatElapsed

- 当election  timeout时
  - 重置r.electionElapsed
  - 如果启用了checkQuorum， step一条`pb.MsgCheckQuorum`的消息，检查自己是否还是合法的leader //TODO，这个什么时候会返回？
  - 如果一个election timeout之内还没法完成leader transfer,直接取消
- 如果自己不再是leader了，直接返回吧
- 最后是heartbeat的处理，每当heartbeat timeout，step 一条`pb.MsgBeat`

```go
// tickHeartbeat is run by leaders to send a MsgBeat after r.heartbeatTimeout.
func (r *raft) tickHeartbeat() {
	r.heartbeatElapsed++
	r.electionElapsed++

	if r.electionElapsed >= r.electionTimeout {
		r.electionElapsed = 0
		if r.checkQuorum {
			r.Step(pb.Message{From: r.id, Type: pb.MsgCheckQuorum})
		}
		// If current leader cannot transfer leadership in electionTimeout, it becomes leader again.
		if r.state == StateLeader && r.leadTransferee != None {
			r.abortLeaderTransfer()
		}
	}

	if r.state != StateLeader {
		return
	}

	if r.heartbeatElapsed >= r.heartbeatTimeout {
		r.heartbeatElapsed = 0
		r.Step(pb.Message{From: r.id, Type: pb.MsgBeat})
	}
}
```

## Append Entry 和 Commit

`appendEntry()`尝试增加一些entry
- 首先给es带上当前的term，标上index
- 然后调用`increaseUncommittedSize`增加quota - 见后文
- 使用`li = r.raftLog.append(es...)`直接追加到raftLog中，这里不需要`maybeAppend()`，因为编号都是按照现有的来，不存在冲突
- `r.getProgress(r.id).maybeUpdate(li)`把自己的Progress更新到最后一条
- 最后调用`r.maybeCommit()`尝试commit

```go
func (r *raft) appendEntry(es ...pb.Entry) (accepted bool) {
	li := r.raftLog.lastIndex()
	for i := range es {
		es[i].Term = r.Term
		es[i].Index = li + 1 + uint64(i)
	}
	// Track the size of this uncommitted proposal.
	if !r.increaseUncommittedSize(es) {
		r.logger.Debugf(
			"%x appending new entries to log would exceed uncommitted entry size limit; dropping proposal",
			r.id,
		)
		// Drop the proposal.
		return false
	}
	// use latest "last" index after truncate/append
	li = r.raftLog.append(es...)
	r.getProgress(r.id).maybeUpdate(li)
	// Regardless of maybeCommit's return, our caller will call bcastAppend.
	r.maybeCommit()
	return true
}
```

先看行内注释吧

tricky的地方在于`mci := mis[len(mis)-r.quorum()]`，mis是所有voter的match index，从小到大排列。

假设一个5节点的cluster， len(mis) = 5, r.quorum = 3，所以mci := mis [5 - 3] = mis[2]
mis的下标为 0, 1, 2, 3, 4, 也就是中间元素的下标，注意它们是从小到大排列的。
这也就意味着，多数派的match index已经大于等于mis[2] (mis[2,3,4])，那么mis[2]对应的log index已经复制到多数派了，这个index可以commit了

> raft thesis 10.2.1 提出一种优化，在发送AppendEntry的同时进行持久化：To handle this simply, the leader uses its own match index to indicate the latest entry to have been durably written to its disk. Once an entry in the leader’s current term is covered by a majority of match indexes, the leader can advance its commit index. 

```go
// maybeCommit attempts to advance the commit index. Returns true if
// the commit index changed (in which case the caller should call
// r.bcastAppend).
func (r *raft) maybeCommit() bool {
	// Preserving matchBuf across calls is an optimization
	// used to avoid allocating a new slice on each call.
	if cap(r.matchBuf) < len(r.prs) {
		r.matchBuf = make(uint64Slice, len(r.prs))  // 这一段貌似在分配空间（raft一开始没初始化），或者集群成员大小变化的时候重新分配
	}
	mis := r.matchBuf[:len(r.prs)]  // 按照prs的长度截断
	idx := 0
	for _, p := range r.prs {
		mis[idx] = p.Match  //获取每一个voter的 Match 进度
		idx++
	}
	sort.Sort(mis)   // 从小到大排列match index, matchBuf类型是上面提到过的uint64Slice
	mci := mis[len(mis)-r.quorum()]  
	return r.raftLog.maybeCommit(mci, r.Term)
}

func (r *raft) quorum() int { return len(r.prs)/2 + 1 }
```

## Uncommit Quota

新建raft的时候Config中有一项是用来限制leader中未commit log总大小的:

> 	// MaxUncommittedEntriesSize limits the aggregate byte size of the
> 	// uncommitted entries that may be appended to a leader's log. Once this
> 	// limit is exceeded, proposals will begin to return ErrProposalDropped
> 	// errors. Note: 0 for no limit.
> 	MaxUncommittedEntriesSize uint6


`increaseUncommittedSize()`检测和增加uncommit entry占用的quota，如果加上新的ents要超过限制就不允许增加了，返回false

```go
// increaseUncommittedSize computes the size of the proposed entries and
// determines whether they would push leader over its maxUncommittedSize limit.
// If the new entries would exceed the limit, the method returns false. If not,
// the increase in uncommitted entry size is recorded and the method returns
// true.
func (r *raft) increaseUncommittedSize(ents []pb.Entry) bool {
	var s uint64
	for _, e := range ents {
		s += uint64(PayloadSize(e))
	}

	if r.uncommittedSize > 0 && r.uncommittedSize+s > r.maxUncommittedSize {
		// If the uncommitted tail of the Raft log is empty, allow any size
		// proposal. Otherwise, limit the size of the uncommitted tail of the
		// log and drop any proposal that would push the size over the limit.
		return false
	}
	r.uncommittedSize += s
	return true
}

// PayloadSize is the size of the payload of this Entry. Notably, it does not
// depend on its Index or Term.
func PayloadSize(e pb.Entry) int {
	return len(e.Data)
}

```

当ents被commit之后，他们占用的quota就可以被释放了

```go
// reduceUncommittedSize accounts for the newly committed entries by decreasing
// the uncommitted entry size limit.
func (r *raft) reduceUncommittedSize(ents []pb.Entry) {
	if r.uncommittedSize == 0 {
		// Fast-path for followers, who do not track or enforce the limit.
		return
	}

	var s uint64
	for _, e := range ents {
		s += uint64(PayloadSize(e))
	}
	if s > r.uncommittedSize {
		// uncommittedSize may underestimate the size of the uncommitted Raft
		// log tail but will never overestimate it. Saturate at 0 instead of
		// allowing overflow.
		r.uncommittedSize = 0
	} else {
		r.uncommittedSize -= s
	}
}
```



## Step

`Step()`是raft处理消息的入口，当node处理这些channel的时候会调用Step
- propc 收到客户端请求
- recvc 收到voter/learner的

```go
func (r *raft) Step(m pb.Message) error {
	// Handle the message term, which may result in our stepping down to a follower.
	switch {
	case m.Term == 0:
		// local message
	case m.Term > r.Term:
		if m.Type == pb.MsgVote || m.Type == pb.MsgPreVote {
			force := bytes.Equal(m.Context, []byte(campaignTransfer))
			inLease := r.checkQuorum && r.lead != None && r.electionElapsed < r.electionTimeout
			if !force && inLease {
				// If a server receives a RequestVote request within the minimum election timeout
				// of hearing from a current leader, it does not update its term or grant its vote
				r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d, index: %d] at term %d: lease is not expired (remaining ticks: %d)",
					r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term, r.electionTimeout-r.electionElapsed)
				return nil
			}
		}
		switch {
		case m.Type == pb.MsgPreVote:
			// Never change our term in response to a PreVote
		case m.Type == pb.MsgPreVoteResp && !m.Reject:
			// We send pre-vote requests with a term in our future. If the
			// pre-vote is granted, we will increment our term when we get a
			// quorum. If it is not, the term comes from the node that
			// rejected our vote so we should become a follower at the new
			// term.
		default:
			r.logger.Infof("%x [term: %d] received a %s message with higher term from %x [term: %d]",
				r.id, r.Term, m.Type, m.From, m.Term)
			if m.Type == pb.MsgApp || m.Type == pb.MsgHeartbeat || m.Type == pb.MsgSnap {
				r.becomeFollower(m.Term, m.From)
			} else {
				r.becomeFollower(m.Term, None)
			}
		}

	case m.Term < r.Term:
		if (r.checkQuorum || r.preVote) && (m.Type == pb.MsgHeartbeat || m.Type == pb.MsgApp) {
			// We have received messages from a leader at a lower term. It is possible
			// that these messages were simply delayed in the network, but this could
			// also mean that this node has advanced its term number during a network
			// partition, and it is now unable to either win an election or to rejoin
			// the majority on the old term. If checkQuorum is false, this will be
			// handled by incrementing term numbers in response to MsgVote with a
			// higher term, but if checkQuorum is true we may not advance the term on
			// MsgVote and must generate other messages to advance the term. The net
			// result of these two features is to minimize the disruption caused by
			// nodes that have been removed from the cluster's configuration: a
			// removed node will send MsgVotes (or MsgPreVotes) which will be ignored,
			// but it will not receive MsgApp or MsgHeartbeat, so it will not create
			// disruptive term increases, by notifying leader of this node's activeness.
			// The above comments also true for Pre-Vote
			//
			// When follower gets isolated, it soon starts an election ending
			// up with a higher term than leader, although it won't receive enough
			// votes to win the election. When it regains connectivity, this response
			// with "pb.MsgAppResp" of higher term would force leader to step down.
			// However, this disruption is inevitable to free this stuck node with
			// fresh election. This can be prevented with Pre-Vote phase.
			r.send(pb.Message{To: m.From, Type: pb.MsgAppResp})
		} else if m.Type == pb.MsgPreVote {
			// Before Pre-Vote enable, there may have candidate with higher term,
			// but less log. After update to Pre-Vote, the cluster may deadlock if
			// we drop messages with a lower term.
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected %s from %x [logterm: %d, index: %d] at term %d",
				r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			r.send(pb.Message{To: m.From, Term: r.Term, Type: pb.MsgPreVoteResp, Reject: true})
		} else {
			// ignore other cases
			r.logger.Infof("%x [term: %d] ignored a %s message with lower term from %x [term: %d]",
				r.id, r.Term, m.Type, m.From, m.Term)
		}
		return nil
	}

	switch m.Type {
	case pb.MsgHup:
		if r.state != StateLeader {
			ents, err := r.raftLog.slice(r.raftLog.applied+1, r.raftLog.committed+1, noLimit)
			if err != nil {
				r.logger.Panicf("unexpected error getting unapplied entries (%v)", err)
			}
			if n := numOfPendingConf(ents); n != 0 && r.raftLog.committed > r.raftLog.applied {
				r.logger.Warningf("%x cannot campaign at term %d since there are still %d pending configuration changes to apply", r.id, r.Term, n)
				return nil
			}

			r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
			if r.preVote {
				r.campaign(campaignPreElection)
			} else {
				r.campaign(campaignElection)
			}
		} else {
			r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
		}

	case pb.MsgVote, pb.MsgPreVote:
		if r.isLearner {
			// TODO: learner may need to vote, in case of node down when confchange.
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] ignored %s from %x [logterm: %d, index: %d] at term %d: learner can not vote",
				r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			return nil
		}
		// We can vote if this is a repeat of a vote we've already cast...
		canVote := r.Vote == m.From ||
			// ...we haven't voted and we don't think there's a leader yet in this term...
			(r.Vote == None && r.lead == None) ||
			// ...or this is a PreVote for a future term...
			(m.Type == pb.MsgPreVote && m.Term > r.Term)
		// ...and we believe the candidate is up to date.
		if canVote && r.raftLog.isUpToDate(m.Index, m.LogTerm) {
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] cast %s for %x [logterm: %d, index: %d] at term %d",
				r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			// When responding to Msg{Pre,}Vote messages we include the term
			// from the message, not the local term. To see why consider the
			// case where a single node was previously partitioned away and
			// it's local term is now of date. If we include the local term
			// (recall that for pre-votes we don't update the local term), the
			// (pre-)campaigning node on the other end will proceed to ignore
			// the message (it ignores all out of date messages).
			// The term in the original message and current local term are the
			// same in the case of regular votes, but different for pre-votes.
			r.send(pb.Message{To: m.From, Term: m.Term, Type: voteRespMsgType(m.Type)})
			if m.Type == pb.MsgVote {
				// Only record real votes.
				r.electionElapsed = 0
				r.Vote = m.From
			}
		} else {
			r.logger.Infof("%x [logterm: %d, index: %d, vote: %x] rejected %s from %x [logterm: %d, index: %d] at term %d",
				r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), r.Vote, m.Type, m.From, m.LogTerm, m.Index, r.Term)
			r.send(pb.Message{To: m.From, Term: r.Term, Type: voteRespMsgType(m.Type), Reject: true})
		}

	default:
		err := r.step(r, m)
		if err != nil {
			return err
		}
	}
	return nil
}
```