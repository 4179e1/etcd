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

`Write Ahead Logging`预写式日志通常是在应用在响应一个请求结果前，先把请求的结果写到磁盘中。etcd的WAL记录了集群所有的操作，包括但不限于：

- 集群成员变更
- commited log
- 创建一个snapshot
- ……

WAL文件按照`<sequence>-<index>.wal`的方式命名，例如`0000000000000001-00000000deadbeef.wal`，其中`<sequence>`是WAL文件的序号，`<index>`是Raft日志的序号。
这些文件预分配64MB，写入超过这个值后会切割。


### WAL的格式

WAL可以想象为一个支持不同数据类型的数组，数组元素的类型包括：

```go
const (
	metadataType int64 = iota + 1  // WAL的metadata，不知道干啥的
	entryType      // Raft的提交日志raftpb.Entry
	stateType      // raftpb.HardState
	crcType        // WAL文件中，截止至今的crc校验和 
	snapshotType   // walpb.Snapshot ,这里只记录了snapshot的Index和Term
)
```

首个WAL文件的开头是这样的
![](https://github.com/4179e1/etcd/raw/master/contrib/raftexample/doc/pic/initial_wal.jpg)

后续WAL文件的开头
![](https://github.com/4179e1/etcd/raw/master/contrib/raftexample/doc/pic/cut_wal.jpeg)

首个WAL和后续不同的在于那个snapshot，里面的Index和Term都是0，所以当snapshot不存在的情况下，会从它的后一条记录开始读取WAL。

后续写入WAL的时候，每次会包含数目不定的entryType，以及一个stateType，如下面魔改`wal.ReadAll()`产生的输出所示

```
2019-01-12 17:38:54.860629 I | crcType
2019-01-12 17:38:54.860659 I | metadataType
2019-01-12 17:38:54.860666 I | snapshotType
2019-01-12 17:38:54.860677 I | entryType index 1
2019-01-12 17:38:54.860684 I | entryType index 2
2019-01-12 17:38:54.860690 I | entryType index 3
2019-01-12 17:38:54.860695 I | stateType
2019-01-12 17:38:54.860704 I | stateType
2019-01-12 17:38:54.860711 I | entryType index 4
2019-01-12 17:38:54.860716 I | stateType
2019-01-12 17:38:54.860722 I | entryType index 5
2019-01-12 17:38:54.860727 I | stateType
2019-01-12 17:38:54.860735 I | entryType index 6
2019-01-12 17:38:54.860740 I | stateType
2019-01-12 17:38:54.860745 I | entryType index 7
2019-01-12 17:38:54.860750 I | stateType
2019-01-12 17:38:54.860756 I | entryType index 8
2019-01-12 17:38:54.860761 I | stateType
2019-01-12 17:38:54.860767 I | entryType index 9
2019-01-12 17:38:54.860772 I | stateType
2019-01-12 17:38:54.860778 I | entryType index 10
2019-01-12 17:38:54.860782 I | stateType
2019-01-12 17:38:54.860801 I | entryType index 11
2019-01-12 17:38:54.860805 I | stateType
2019-01-12 17:38:54.860810 I | snapshotType
2019-01-12 17:38:54.860821 I | entryType index 12
2019-01-12 17:38:54.860828 I | entryType index 13
2019-01-12 17:38:54.860833 I | stateType
2019-01-12 17:38:54.860838 I | stateType
2019-01-12 17:38:54.860844 I | entryType index 14
2019-01-12 17:38:54.860849 I | stateType
2019-01-12 17:38:54.860854 I | entryType index 15
2019-01-12 17:38:54.860860 I | stateType
2019-01-12 17:38:54.860865 I | entryType index 16
```


### WAL内容
上一节讲述的WAL的格式，但raft协议关心的只有entryType，其中包括提交日志和配置变更等。

etcd提供了一个[etcd-dump-logs](https://github.com/etcd-io/etcd/tree/master/tools/etcd-dump-logs)来查看WAL的内容，以下内容取自一个三节点的测试环境环境：
```
W
Snapshot:
empty
Start dupmping log entries from snapshot.
WAL metadata:
nodeID=da37c2e9715e7728 clusterID=cbe50ed33c7d3758 term=407449 commitIndex=14825 vote=da37c2e9715e7728
WAL entries:
lastIndex=14826
term         index      type    data
   1             1      conf    method=ConfChangeAddNode id=880db10f35870d26
   1             2      conf    method=ConfChangeAddNode id=da37c2e9715e7728
   1             3      conf    method=ConfChangeAddNode id=e1aabd4a7f715232
  54             4      norm
  54             5      norm    method=PUT path="/0/members/da37c2e9715e7728/attributes" val="{\"name\":\"s1\",\"clientURLs\":[\"http://192.168.1.160:2379\"]}"
  54             6      norm    method=PUT path="/0/members/880db10f35870d26/attributes" val="{\"name\":\"s2\",\"clientURLs\":[\"http://192.168.1.224:2379\"]}"
  54             7      norm    method=PUT path="/0/version" val="3.0.0"
  54             8      norm    method=PUT path="/0/members/e1aabd4a7f715232/attributes" val="{\"name\":\"s3\",\"clientURLs\":[\"http://192.168.1.30:2379\"]}"
  54             9      norm    method=PUT path="/0/version" val="3.3.0"
  54            10      norm    header:<ID:8586224779106083855 > put:<key:"mykey" value:"this is awesome" >
 ...
```

我们可以看到，前面三条都是配置更改，这个集群的初始成员为空， 每增加一个成员都有一条对应的记录。


> 这里列出了一些预期的输出格式(https://github.com/etcd-io/etcd/tree/master/tools/etcd-dump-logs/expectedoutput)

> 这个工具同样可以dump raftexample的日志，但是很多内容它认不出来。


### WAL和snapshot的关系

etcd在启动并加载快照后，会根据快照metadata中的index找到包含这条index的WAL文件，从WAL的这条index后面的日志开始读取，忽略前面的WAL和日志。

正常来说，快照前面的WAL是不再需要的，etcd的[issue#1810](https://github.com/etcd-io/etcd/pull/1810)加入了自动清理WAL的功能，而我们的raftexample则完全没有这个功能。



## Reference

- [etcd raft模块分析--WAL日志](https://my.oschina.net/fileoptions/blog/1825531)
- [ETCD V3 中的 .wal 文件](https://blog.zhesih.com/2017/10/03/the-wal-files-in-etcd-v3/)
