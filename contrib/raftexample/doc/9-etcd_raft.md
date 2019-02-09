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

- matchedbuf
是这个吗？
> raft thesis 10.2.1 提出一种优化，在发送AppendEntry的同时进行持久化：To handle this simply, the leader uses its own match index to indicate the latest entry to have been durably written to its disk. Once an entry in the leader’s current term is covered by a majority of match indexes, the leader can advance its commit index. 

- pendingConfIndex
etcd raft对成员变更的变种实现
> To ensure there is no attempt to commit two membership changes at once by matching log positions (which would be unsafe since they should have different quorum requirements), any proposed membership change is simply disallowed while any uncommitted change appears in the leader's log.

- tick/step
这是两个没有初始化的函数指针，他们是在状态转换，比如`becomeLeader()`的时候设置的，不同的角色需要执行的动作不同

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
	electionElapsed int //TODO，对于leader/candidate 跟 follower的意义不太一样，没看懂

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int  // 同样没看懂

	checkQuorum bool     // 由Config初始化
	preVote     bool     // 由Config初始化

	heartbeatTimeout int		// 由Config初始化
	electionTimeout  int		// 由Config初始化
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int  // 根据electiontimout算出一个随机的超时时间
	disableProposalForwarding bool   // 由Config初始化

	tick func()				//
	step stepFunc			// 

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
```

## 随机超时器

入口是`resetRandomizedElectionTimeout()`，状态转换reset的时候设置

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

`becomeLeader()`:

- 不能从Follower切过来
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


## UncommittedSize

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

当ents被commit之后，他们占用的ota就可以被释放了

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