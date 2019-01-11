# Snapshot & WAL

本文介绍ETCD的Snapshot和WAL实现，etcd中这两个组件是一定程度上耦合的，WAL依赖于snapshot。

## Snapshot

当我们说snapshot的时候，etcd里面其实有两个snapshot类型，它们都使用了`Protocol Buffers`:

- 一个是raft 协议中使用的Snapshot，位于`raft/raftpb/raft.pb.go`中，下文简称为`raft snapshot`
- 另一个是etcd实际写入到磁盘的snapshot，位于`etcdserver/api/snap/snappb/snap.pb.go`中，它实际上对`raft snapshot`做了一层封装,下文简称为`etcd snapshot`

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

### etcd snapshot

`etcd snapshot`实际上只是对`raft snapshot`的封装
- Data 是`Protocol Buffer`序列化后的二进制数据
- Crc 是Data的校验码

```go
type Snapshot struct {
	Crc              uint32 `protobuf:"varint,1,opt,name=crc" json:"crc"`
	Data             []byte `protobuf:"bytes,2,opt,name=data" json:"data,omitempty"`
	XXX_unrecognized []byte `json:"-"`
}
```

### snapshotter 对象

主要记录一个快照目录

```go
type Snapshotter struct {
	lg  *zap.Logger
	dir string
}

func New(lg *zap.Logger, dir string) *Snapshotter {
	return &Snapshotter{
		lg:  lg,
		dir: dir,
	}
}
```

### 保持快照

- `SaveSnap()`的类型参数为`raftpb.Snapshot`，即上文说的`raft snapshot`，它在检查快照目录存在后调用`save()`.
- `save()`直接把`raft snapshot`通过`Protocol Buffer`序列化为二进制数据`data`，然后计算出这些数据的crc校验值`crc`，`data`和`crc`组装为一个`etcd snapshot`
- 然后保存到一个新的快照文件中，名为`<Term>-<Index>.snap`，如`0000000000000002-000000000000000a.snap`
- 文件如果保存失败，则会尝试删除，但是删除也有可能失败

TL'DR: 把`raft snapshot`序列化为`Protocol Buffer`数据，计算出crc后写入文件

```go
func (s *Snapshotter) SaveSnap(snapshot raftpb.Snapshot) error {
	if raft.IsEmptySnap(snapshot) {
		return nil
	}
	return s.save(&snapshot)
}

func (s *Snapshotter) save(snapshot *raftpb.Snapshot) error {
	start := time.Now()

	fname := fmt.Sprintf("%016x-%016x%s", snapshot.Metadata.Term, snapshot.Metadata.Index, snapSuffix)
	b := pbutil.MustMarshal(snapshot)
	crc := crc32.Update(0, crcTable, b)
	snap := snappb.Snapshot{Crc: crc, Data: b}
	d, err := snap.Marshal()
	if err != nil {
		return err
	}
	snapMarshallingSec.Observe(time.Since(start).Seconds())

	spath := filepath.Join(s.dir, fname)

	fsyncStart := time.Now()
	err = pioutil.WriteAndSyncFile(spath, d, 0666)
	snapFsyncSec.Observe(time.Since(fsyncStart).Seconds())

	if err != nil {
		if s.lg != nil {
			s.lg.Warn("failed to write a snap file", zap.String("path", spath), zap.Error(err))
		}
		rerr := os.Remove(spath)
		if rerr != nil {
			if s.lg != nil {
				s.lg.Warn("failed to remove a broken snap file", zap.String("path", spath), zap.Error(err))
			} else {
				plog.Errorf("failed to remove broken snapshot file %s", spath)
			}
		}
		return err
	}

	snapSaveSec.Observe(time.Since(start).Seconds())
	return nil
}
```

### 读取快照

- `Load()`首先调用`s.snapNames()`获取快照目录下的所有快照文件名，并反向排序，即Term, Index序号大的文件排在前面
- 然后通过一个循环调用`loadSnap()`尝试去读取这些快照文件，任何一个读取成功则退出循环，这是因为有些快照文件可能是有问题的，数据不能用
- `loadSnap()`只是对`Read()`的封装，如果读取失败或者读到了无效的数据则尝试修改快照文件名，在后面加一个`.broken`的后缀。
- `Read()`从去读快照文件后进行两次反序列化
  - 把文件内容反序列化为`etcd snapshot`， 并计算crc是否符合预期
  - crc校验通过则继续把`etcd snapshot`的`data`字段反序列化为`raft snapshot`

```go
func (s *Snapshotter) Load() (*raftpb.Snapshot, error) {
	names, err := s.snapNames()
	if err != nil {
		return nil, err
	}
	var snap *raftpb.Snapshot
	for _, name := range names {
		if snap, err = loadSnap(s.lg, s.dir, name); err == nil {
			break
		}
	}
	if err != nil {
		return nil, ErrNoSnapshot
	}
	return snap, nil
}

func loadSnap(lg *zap.Logger, dir, name string) (*raftpb.Snapshot, error) {
	fpath := filepath.Join(dir, name)
	snap, err := Read(lg, fpath)
	if err != nil {
		brokenPath := fpath + ".broken"
		if lg != nil {
			lg.Warn("failed to read a snap file", zap.String("path", fpath), zap.Error(err))
		}
		if rerr := os.Rename(fpath, brokenPath); rerr != nil {
			if lg != nil {
				lg.Warn("failed to rename a broken snap file", zap.String("path", fpath), zap.String("broken-path", brokenPath), zap.Error(rerr))
			} else {
				plog.Warningf("cannot rename broken snapshot file %v to %v: %v", fpath, brokenPath, rerr)
			}
		} else {
			if lg != nil {
				lg.Warn("renamed to a broken snap file", zap.String("path", fpath), zap.String("broken-path", brokenPath))
			}
		}
	}
	return snap, err
}
```

## WAL
