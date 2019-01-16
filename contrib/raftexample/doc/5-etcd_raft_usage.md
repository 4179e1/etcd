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
- Messages: Messages []pb.Message
- log entries: Entries []pb.Entry
- Raft state chanes: pb.HardState。*SoftState算吗？