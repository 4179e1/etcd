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

// TODO：上文说的Message的input是指啥？ proposeC/confChangeC， 或者transport发送的消息？

## 初始化Node

> The primary object in raft is a Node. Either start a Node from scratch using raft.StartNode or start a Node from some initial state using raft.RestartNode.

应用根据首次启动还是重启的不同，初始化Node的流程稍有不同。

### 首次启动

1. 创建`raft.Storage`对象
1. 创建`raft.Config`，其中`Storage`指定为第1步创建的`raft.Storage`
1. 通过`raft.StartNode(c, []raft.Peer{...})`创建`raft.Node`，其中c是第2步创建的`raft.Config`，另一个参数为集群其他成员的id。

### 重启

1. 创建`raft.Storage`对象
1. 在`raft.Storage`对象中回放`snapshot`，`state`，`entries`
2. 创建`raft.Config`，其中`Storage`指定为第1步创建的`raft.Storage`
3. 通过`raft.RestartNode(c)`创建`raft.Node`，其中c是第2步创建的`raft.Config`，这里不再需要集群其他成员的id了，在`snapshot`或者`entries`中已经记录。

### 创建raft.Storage

`raft.Storage`是一个接口，

## Reference
[etcd raft library设计原理和使用](https://zhuanlan.zhihu.com/p/27767675)