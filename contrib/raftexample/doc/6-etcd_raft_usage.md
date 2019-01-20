# Etcd Raft

Etcd Raft的使用请参考[这里](https://github.com/etcd-io/etcd/tree/master/raft)，本文将结合`raftexample`来理解其用法。

## Overview

> Most Raft implementations have a monolithic design, including storage handling, messaging serialization, and network transport. This library instead follows a minimalistic design philosophy by only implementing the core raft algorithm. This minimalism buys flexibility, determinism, and performance.
> 
> To keep the codebase small as well as provide flexibility, the library only implements the Raft algorithm; both network and disk IO are left to the user. Library users must implement their own transportation layer for message passing between Raft peers over the wire. Similarly, users must implement their own storage layer to persist the Raft log and state.
> 
> In order to easily test the Raft library, its behavior should be deterministic. To achieve this determinism, the library models Raft as a state machine. The state machine takes a `Message` as input. A message can either be a local timer update or a network message sent from a remote peer. The state machine's output is a 3-tuple `{[]Messages, []LogEntries, NextState}` consisting of an array of `Messages`, `log entries`, and `Raft state changes`. For state machines with the same state, the same state machine input should always generate the same state machine output.

- Etcd Raft 只实现了Raft协议的核心，网络传输层和存储层需要用户自己实现。
- 应用的行为应该是确定的，对于处于同样状态的状态机，同样的输入总是产生同样的输出。
- 应用把Raft当作一个状态机，他的输入是`Message`，输出是`{[]Messages, []LogEntries, NextState}`这样的三元组。
  
在`raft/node.go`中
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

Output的三元组对应：
1. Messages: Messages []pb.Message

```go
type Message struct {
	Type             MessageType `protobuf:"varint,1,opt,name=type,enum=raftpb.MessageType" json:"type"`
	To               uint64      `protobuf:"varint,2,opt,name=to" json:"to"`
	From             uint64      `protobuf:"varint,3,opt,name=from" json:"from"`
	Term             uint64      `protobuf:"varint,4,opt,name=term" json:"term"`
	LogTerm          uint64      `protobuf:"varint,5,opt,name=logTerm" json:"logTerm"`
	Index            uint64      `protobuf:"varint,6,opt,name=index" json:"index"`
	Entries          []Entry     `protobuf:"bytes,7,rep,name=entries" json:"entries"`
	Commit           uint64      `protobuf:"varint,8,opt,name=commit" json:"commit"`
	Snapshot         Snapshot    `protobuf:"bytes,9,opt,name=snapshot" json:"snapshot"`
	Reject           bool        `protobuf:"varint,10,opt,name=reject" json:"reject"`
	RejectHint       uint64      `protobuf:"varint,11,opt,name=rejectHint" json:"rejectHint"`
	Context          []byte      `protobuf:"bytes,12,opt,name=context" json:"context,omitempty"`
	XXX_unrecognized []byte      `json:"-"`
}
```

2. log entries: Entries []pb.Entry

```go
type Entry struct {
	Term             uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`
	Index            uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
	Type             EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
	Data             []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
	XXX_unrecognized []byte    `json:"-"`
}
```

3. Raft state chanes: pb.HardState。*SoftState算吗？

```go
type HardState struct {
	Term             uint64 `protobuf:"varint,1,opt,name=term" json:"term"`
	Vote             uint64 `protobuf:"varint,2,opt,name=vote" json:"vote"`
	Commit           uint64 `protobuf:"varint,3,opt,name=commit" json:"commit"`
	XXX_unrecognized []byte `json:"-"`
}
```

// TODO：上文说的Message的input是指啥？ proposeC/confChangeC， 或者transport发送的消息？

## 初始化Node

> The primary object in raft is a Node. Either start a Node from scratch using raft.StartNode or start a Node from some initial state using raft.RestartNode.

应用根据首次启动还是重启的不同（通过`oldwal := wal.Exist(rc.waldir)`是否为nil来判断），初始化Node的流程稍有不同。

### 首次启动

> startRaft(): 
> |- replayWal():
> |  |- 创建`raft.Storage`对象 (在`replyWAL()`中)
> |  |- 在`raft.Storage`对象中回放`snapshot`(如果有的话)，`state`，`entries`()
> |- 创建`raft.Config`，其中`Storage`指定为`replayWAL()`中创建的`raft.Storage`
> |- . 通过`raft.StartNode(c, []raft.Peer{...})`创建`raft.Node`，其中c是第2步创建的`raft.Config`，另一个参数为集群其他成员的id。 ```

 ### 重启

> startRaft(): 
> |- replayWal():
> |  |- 创建`raft.Storage`对象 (在`replyWAL()`中)
> |  |- 在`raft.Storage`对象中回放`snapshot`(如果有的话)，`state`，`entries`()
> |- 创建`raft.Config`，其中`Storage`指定为`replayWAL()`中创建的`raft.Storage`
> |- 通过`raft.RestartNode(c)`创建`raft.Node`，其中c是第2步创建的`raft.Config`，这里不再需要集群其他成员的id了，在`snapshot`或者`entries`中已经记录。

### 创建raft.Storage

`raft.Storage`详情请看上一篇文章。相关的操作在`replayWAL()`中:

- 如果存在快照，则通过`rc.raftStorage.ApplySnapshot(*snapshot)`加载到storage中
- `rc.raftStorage.(st)`恢复Hardstate
- `rc.raftStorage.Append(ents)`加载entry，这里包含了提交的日志和配置变更。


```go
	rc.raftStorage = raft.NewMemoryStorage()
	if snapshot != nil {
		rc.raftStorage.ApplySnapshot(*snapshot)
	}
	rc.raftStorage.(st)
	// append to storage so raft starts at the right place in log
	rc.raftStorage.Append(ents)
```

## Raft 协议处理

在初始化Node之后，进入Raft的协议处理部分，即`serveChannel()`，有4件事情要处理

```go
	// event loop on raft state machine updates
	for {
		select {
		case <-ticker.C:
			rc.node.Tick()

		// store raft entries to wal, then publish over commit channel
		case rd := <-rc.node.Ready():
			rc.wal.Save(rd.HardState, rd.Entries)
			if !raft.IsEmptySnap(rd.Snapshot) {
				rc.saveSnap(rd.Snapshot)
				rc.raftStorage.ApplySnapshot(rd.Snapshot)
				rc.publishSnapshot(rd.Snapshot)
			}
			rc.raftStorage.Append(rd.Entries)
			rc.transport.Send(rd.Messages)
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
			}
			rc.maybeTriggerSnapshot()
			rc.node.Advance()

		case err := <-rc.transport.ErrorC:
			log.Printf("serveChannels() rc.transport.ErrorC recv")
			rc.writeError(err)
			return

		case <-rc.stopc:
			log.Printf("serveChannels() rc.stop recv")
			rc.stop()
			return
		}
	}
```

### 处理Node.Ready()的更新

> First, read from the Node.Ready() channel and process the updates it contains. These steps may be performed in parallel, except as noted in step 2.

先看看`Node.Ready()`是干啥的：

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

它返回了一个`Ready`结构的channle，并要求：
- 在收到Ready的state之后调用`node.Advance()`
- 在Apply完上一批的entry和snapshot之前不能apply下一批的

`Ready`结构已经在前文提到。 


> 1.  Write Entries, HardState and Snapshot to persistent storage in order, i.e. Entries first, then HardState and Snapshot if they are not empty. If persistent storage supports atomic writes then all of them can be written together. Note that when writing an Entry with Index i, any previously-persisted entries with Index >= i must be discarded.


- `rc.wal.Save(rd.HardState, rd.Entries)`先保存entris，然后才是hardstate。
- `if !raft.IsEmptySnap(rd.Snapshot)`如果有快照，
  - `rc.saveSnap(rd.Snapshot)`保存到snap文件，释放不再使用的wal文件
  - `rc.raftStorage.ApplySnapshot(rd.Snapshot)`用快照更新storage的数据
  - `rc.publishSnapshot(rd.Snapshot)`通知应用加载快照
- `rc.raftStorage.Append(rd.Entries)`更新storage的数据.

所谓的`persistent storage`这里其实由两部分组成
- WAL: 持久化所有数据，按照顺序保存所有收到的entry
- MemoryStorage：在内存中保存当前的entry，负责处理重复和缺失的条目,`Note that when writing an Entry with Index i, any previously-persisted entries with Index >= i must be discarded.`就是在这里处理的。


> 2. Send all Messages to the nodes named in the To field. It is important that no messages be sent until the latest HardState has been persisted to disk, and all Entries written by any previous Ready batch (Messages may be sent while entries from the same batch are being persisted). To reduce the I/O latency, an optimization can be applied to make leader write to disk in parallel with its followers (as explained at section 10.2.1 in Raft thesis). If any Message has type MsgSnap, call Node.ReportSnapshot() after it has been sent (these messages may be large). Note: Marshalling messages is not thread-safe; it is important to make sure that no new entries are persisted while marshalling. The easiest way to achieve this is to serialise the messages directly inside the main raft loop.

就是`rc.transport.Send(rd.Messages)`，它跟前面的持久化时在同一个协程中的。

> If any Message has type MsgSnap, call Node.ReportSnapshot() after it has been sent (these messages may be large)
`transport.Send()`会调用peers的`send()`方法，其中有这么一段:

```go
	writec, name := p.pick(m)
	select {
	case writec <- m:
	default:
		p.r.ReportUnreachable(m.To)
		if isMsgSnap(m) {
			p.r.ReportSnapshot(m.To, raft.SnapshotFailure)
		}
```

这里只有失败的时候会调用`ReportUnreachable`，其实这是符合要求的，看看`Node.ReportSnapshot()`的注释。

> 3. Apply Snapshot (if any) and CommittedEntries to the state machine. If any committed Entry has Type EntryConfChange, call Node.ApplyConfChange() to apply it to the node. The configuration change may be cancelled at this point by setting the NodeID field to zero before calling ApplyConfChange (but ApplyConfChange must be called one way or the other, and the decision to cancel must be based solely on the state machine and not external information such as the observed health of the node).

Snapshot 和 Committedentries分别通过这两函数提交到应用的状态及（KV Store）
- `rc.publishSnapshot(rd.Snapshot)`
- `rc.publishEntries(rc.entriesToApply(rd.CommittedEntries))`

> 4. Call Node.Advance() to signal readiness for the next batch of updates. This may be done at any time after step 1, although all updates must be processed in the order they were returned by Ready.

处理完后告诉raft可以发下一个更新了`rc.node.Advance()`


### 更新Raft Storage Interface

> Second, all persisted log entries must be made available via an implementation of the Storage interface. The provided MemoryStorage type can be used for this (if repopulating its state upon a restart), or a custom disk-backed implementation can be supplied.

这个在1.1里面`wal.Save()`里面已经通过`rc.raftStorage.ApplySnapshot(rd.Snapshot)`和`rc.raftStorage.Append(rd.Entries)`更新Storage了。

###	处理来自其他节点的消息

> Third, after receiving a message from another node, pass it to Node.Step:

`raftexample`中定义了一个

```go
func (rc *raftNode) Process(ctx context.Context, m raftpb.Message) error {
	return rc.node.Step(ctx, m)
}
```

`transport`通过 `peers`(etcd/etcdserver/api/rafthttp/peer.go)会在接受消息时调用这个，在`startPeer()`中:

```go
	go func() {
		for {
			select {
			case mm := <-p.recvc:
				if err := r.Process(ctx, mm); err != nil {
					if t.Logger != nil {
						t.Logger.Warn("failed to process Raft message", zap.Error(err))
					} else {
						plog.Warningf("failed to process raft message (%v)", err)
					}
				}
			case <-p.stopc:
				return
			}
		}
	}()
```


### 触发心跳

> Finally, call Node.Tick() at regular intervals (probably via a time.Ticker). Raft has two important timeouts: heartbeat and the election timeout. However, internally to the raft package time is represented by an abstract "tick".

`ticker := time.NewTicker(100 * time.Millisecond)`，然后在for select循环中处理超时`case <-rc.stopc:`

## Reference
[etcd raft library设计原理和使用](https://zhuanlan.zhihu.com/p/27767675)