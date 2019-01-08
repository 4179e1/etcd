# Snapshot & WAL

本文介绍ETCD的Snapshot和WAL实现，etcd中这两个组件是一定程度上耦合的，WAL依赖于snapshot。

## Snapshot

当我们说snapshot的时候，etcd里面其实有两个snapshot类型，它们都使用了`Protocol Buffers`:

- 一个是raft 协议中使用的Snapshot，位于`raft/raftpb/raft.pb.go`中，下文简称为`raft snapshot`
- 另一个是etcd实际写入到磁盘的snapshot，位于`etcdserver/api/snap/snappb/snap.pb.go`中，它实际上对`raft snapshot`做了一层封装,下文简称为`etcd snapshoto`

下文直接基于编译后的`Protocol Buffers`代码进行描述，我觉得比.proto文件好读一些

### raft snapshot

`raft snapshot`本身只有两个字段

- Data： snapshot本身的内容
- Metadata： raft协议的metadata, 包含：
  - Index:  保存snapshot时，Raft协议的最后一条日志Index
  - Term:   保存snapshot是，Raft协议的Term
  - ConfState:  Raft集群的配置信息，
    - Nodes
    - Learners
    - 以上两个成员都是unit64的数组，大概表示成员的ID，怎么根据这个id找到实际的ip：port? 大概是根据启动时的`cluster`参数?

```go
type Snapshot struct {
	Data             []byte           `protobuf:"bytes,1,opt,name=data" json:"data,omitempty"`
	Metadata         SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata"`
	XXX_unrecognized []byte           `json:"-"`
}

type SnapshotMetadata struct {
	ConfState        ConfState `protobuf:"bytes,1,opt,name=conf_state,json=confState" json:"conf_state"`
	Index            uint64    `protobuf:"varint,2,opt,name=index" json:"index"`
	Term             uint64    `protobuf:"varint,3,opt,name=term" json:"term"`
    XXX_unrecognized []byte    `json:"-"`
}

type ConfState struct {
	Nodes            []uint64 `protobuf:"varint,1,rep,name=nodes" json:"nodes,omitempty"`
	Learners         []uint64 `protobuf:"varint,2,rep,name=learners" json:"learners,omitempty"`
	XXX_unrecognized []byte   `json:"-"`
}
```