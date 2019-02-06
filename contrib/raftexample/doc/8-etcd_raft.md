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
- `MaxSizePerMsg`消息最大的长度，上面是1mb
- `MaxUncommittedEntriesSize`raft leader能保存的未提交log的总大小，上面是1GB（1 << 30)
- `MaxInflightMsgs`貌似是etcd的优化，用来限制正在进行的log replication的总数

一些值得注意的：

- `CheckQuorum`， TODO， 似乎是etcd的扩展，让leader定时检查quorum确定自己是否还合法
- `PreVOte`, raft thesis的PreVote扩展，用来处理节点出现网络分区后重新加入集群导致的干扰。
- `Logger`，（debug用的）日志类
- `DisableProposalForwarding`,是否允许follower把propesal请求转发给leader，在特定的场合有用。
- `ReadOnlyOption`，读请求是否要先征求多数派，可选项包括
  - `ReadOnlySafe` 通过征求多数派满足线性一致性
  - `ReadOnlyLeaseBased` 通过lease机制返回数据，我猜满足顺序一致性，但不满足线性一致性。

它们对应etcd raft feature所说的
```
    Efficient linearizable read-only queries served by both the leader and followers
        leader checks with quorum and bypasses Raft log before processing read-only queries
        followers asks leader to get a safe read index before processing read-only queries
    More efficient lease-based linearizable read-only queries served by both the leader and followers
        leader bypasses Raft log and processing read-only queries locally
        followers asks leader to get a safe read index before processing read-only queries
        this approach relies on the clock of the all the machines in raft group
```

```go
type ReadOnlyOption int

const (
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	ReadOnlySafe ReadOnlyOption = iota
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	ReadOnlyLeaseBased
)
```

另外一些私有成员
- `peers`只有在初次启动时设置（在`raft.StartNode()`中）,否则会panic
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


- ReadState 
- state 
- 

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
	prs                map[uint64]*Progress  // ？？？
	learnerPrs         map[uint64]*Progress  // ？？？
	matchBuf           uint64Slice			// 记录每一个node match index

	state StateType  // 当前状态，leader，follower，candidate，这里还有一个额外的pre-candidate

	// isLearner is true if the local raft node is a learner.
	isLearner bool

	votes map[uint64]bool  // TODO

	msgs []pb.Message   // TODO

	// the leader id
	lead uint64 
	// leadTransferee is id of the leader transfer target when its value is not zero.
	// Follow the procedure defined in raft thesis 3.10.
	leadTransferee uint64
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

	readOnly *readOnly 	// TODO

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

	tick func()				// TODO
	step stepFunc			// TODO

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

## 番外 - Raft 集群成员是怎么保存的

终于，我们能理清Raft node在初始化和重启的过程中是怎么保存的了，这部分是对[](./2-raftexample.md)中raftserver初始化的补充，包括三种情形

### 初始化

`node.StartNode()`的第二个参数中包含了所有peers，这个函数会
1. 为每一个peer创建一个`pb.ConfChangeAddNode`的log entry添加到raft log中，并标记为applied
1. 直接调用`raft.addNode()`把每一个peer添加进去。

等这些raft log被持久化到WAL中，在`node.Ready()`中通过`publishEntries()`发给应用层后，应用层会通过`node.ApplConfChange()`再调用一次`r.addNode()`，`r.addNode()`会忽略这种重复添加

### 重启，但是没有快照

没有快照的情况下，初始化后的peer的信息作为`pb.ConfChangeAddNode`保存在WAL中，（猜测，重启回放WAL时会把这些数据写到Storage暴露给raft log），在`node.Ready()`中通过`publishEntries()`发给应用层后，应用层会通过`node.ApplConfChange()`调用`r.addNode()`


### 重启，有快照

这种情况直接在`raft.newRaft()`中处理，WAL的快照记录中保存了peers，直接在这个函数完成初始化。如果后续还有节点的变更，在回放WAL的过程中通过`r.addNode()`完成，参照上一节