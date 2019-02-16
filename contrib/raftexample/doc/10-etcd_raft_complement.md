# 补充部分

## Raft 集群成员是怎么保存的

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


## readonly request

raft node 有一个 `ReadIndex()`，用来查询当前安全的读index，这个请求也是通过propc发到raft处理的

如果follower收到这个消息，转发给leader

leader的处理中

A 如果使用ReadOnlyLeaseBased

- 对于本地节点，直接append r.readStates, 其中Index 为但前的commit index，RequestCtx 为m.Entries[0].Data
- 对于非本地节点，组装一条MsgReadIndexResp回复，其中Index为当前的commit index, Entreies为m.Entries(照原样返回)
  - follower 根据回复组装个ReadState append到r.readStates中


B 如果使用ReadOnlySafe

处理逻辑只有两行
```go
				r.readOnly.addRequest(r.raftLog.committed, m)
                r.bcastHeartbeatWithCtx(m.Entries[0].Data)
```

1 前者组装一个readIndexStatus结构，记录
- index ： 当前的commit index。
- req: m 原始的信息
- acks 统计接受的回复

req中记录了原始发送者，当多数派ack之后，就可以返回当前记录的commit index

把这个结构放到一个map中，使用第一条Entry的Data字段作为索引(`m.Entries[0].Data`)

```go
// addRequest adds a read only reuqest into readonly struct.
// `index` is the commit index of the raft state machine when it received
// the read only request.
// `m` is the original read only request message from the local or remote node.
func (ro *readOnly) addRequest(index uint64, m pb.Message) {
	ctx := string(m.Entries[0].Data)
	if _, ok := ro.pendingReadIndex[ctx]; ok {
		return
	}
	ro.pendingReadIndex[ctx] = &readIndexStatus{index: index, req: m, acks: make(map[uint64]struct{})}
	ro.readIndexQueue = append(ro.readIndexQueue, ctx)
}
```

2 后者用同样的内容(`m.Entries[0].Data`)作为heartbeat的的Context广播出去
注意`commit := min(r.getProgress(to).Match, r.raftLog.committed)`

```go
func (r *raft) bcastHeartbeatWithCtx(ctx []byte) {
	r.forEachProgress(func(id uint64, _ *Progress) {
		if id == r.id {
			return
		}
		r.sendHeartbeat(id, ctx)
	})
}


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

3 follower 或 candidate收到heartbeat后调用r.handleHeartbeat(m)并回复leader
```go
func (r *raft) handleHeartbeat(m pb.Message) {
	r.raftLog.commitTo(m.Commit)
	r.send(pb.Message{To: m.From, Type: pb.MsgHeartbeatResp, Context: m.Context})
}
```

不过，如果leader term过期了，其他节点是不会回复的。(step中的`m.Term < r.Term`)

4 leader 收到resp后

```go
	case pb.MsgHeartbeatResp:
        ...
		ackCount := r.readOnly.recvAck(m)      // <=== 确认接受
		if ackCount < r.quorum() {             // <=== 确认多数派已经接受commit
			return nil
		}

		rss := r.readOnly.advance(m)
		for _, rs := range rss {
			req := rs.req
			if req.From == None || req.From == r.id { // from local member
				r.readStates = append(r.readStates, ReadState{Index: rs.index, RequestCtx: req.Entries[0].Data})
			} else {
				r.send(pb.Message{To: req.From, Type: pb.MsgReadIndexResp, Index: rs.index, Entries: req.Entries})
			}
		}
```

消息的索引就是m.Context，第2步发出去那个

```go
// recvAck notifies the readonly struct that the raft state machine received
// an acknowledgment of the heartbeat that attached with the read only request
// context.
func (ro *readOnly) recvAck(m pb.Message) int {
	rs, ok := ro.pendingReadIndex[string(m.Context)]
	if !ok {
		return 0
	}

	rs.acks[m.From] = struct{}{}
	// add one to include an ack from local node
	return len(rs.acks) + 1
}
```

满足quorum之后调用`r.readOnly.advance(m)`删除所有之前index的readonly request。
组装一个ReadState
- 如果是自己发起的，append到自己的r.readStates中
- 否则这个就请求是由follower转发过来的，重新返回给它（在req中记录了发送方）


5 最终由 node.ready取走readonly