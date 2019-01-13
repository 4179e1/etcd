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

WAL保存的数据是`Protocol Buffer`序列化后的`walpb.Record`组成，其结构为
```go
type Record struct {
	Type             int64  `protobuf:"varint,1,opt,name=type" json:"type"`
	Crc              uint32 `protobuf:"varint,2,opt,name=crc" json:"crc"`
	Data             []byte `protobuf:"bytes,3,opt,name=data" json:"data,omitempty"`
	XXX_unrecognized []byte `json:"-"`
}
```

其中Data是上层数据结构序列化后的数据，Crc是*所有`walpb.Record`记录*到目前为止的校验和，Type包括以下类型：

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

上文中`snapshotType`对应的数据结构如下，只记录了`Index`和`Term`

```go
type Snapshot struct {
	Index            uint64 `protobuf:"varint,1,opt,name=index" json:"index"`
	Term             uint64 `protobuf:"varint,2,opt,name=term" json:"term"`
	XXX_unrecognized []byte `json:"-"`
}
```

etcd在启动并加载快照后，会根据快照metadata中的index找到包含这条快照的WAL文件，这条snapshot后面的日志开始读取，忽略前面的WAL和日志。

正常来说，快照前面的WAL是不再需要的，etcd的[issue#1810](https://github.com/etcd-io/etcd/pull/1810)加入了自动清理WAL的功能，而我们的raftexample则完全没有这个功能。

### 创建WAL

WAL通过`Create()`创建，一个非常长的函数

- 先创建一个`<snapshot>.tmp`的目录，在该目录下面创建首个WAL文件`0000000000000001-0000000000000000.wal`（预先分配64MB），并给这个文件加锁
- 在WAL文件上创建`encoder`对象，用来写入WAL。后面还有一个`decoder`对象，用来读取WAL。后面再详细看这两个对象
- 通过`w.encoder`依次往这个WAL文件中写入上文提到的三条数据
  - `w.saveCrc(0)`保存累计至今的CRC，此时是0
  - `w.encoder.encode(&walpb.Record{Type: metadataType, Data: metadata});`保存metadata, 这是来自该函数的参数，raftexample 直接传了个nil
  - `w.SaveSnapshot(walpb.Snapshot{})`保存一个空的`walpb.Snapshot{}`，`Index`和`Term`都默认初始化为0
- 调用`renameWAL()`完成以下操作
  - 把`<snapshot>.tmp`目录重命名为`<snapshot>`，这样为了把以上流程当作一个原子操作 //TODO：这样做是要解决什么问题？
  - `w.fp = newFilePipeline(w.lg, w.dir, SegmentSizeBytes)`打开一个`filePipeLine`对象，用来提前创建后续的WAL文件，再议。
- 在父目录上调用`Fsync()`确保rename成功
  - 打开WAL目录的父目录
  - 在父目录上执行`Fsync()`
  - 关闭父目录

根据(https://lwn.net/Articles/457667/):
> The more subtle usages deal with newly created files, or overwriting existing files. A newly created file may require an fsync() of not just the file itself, but also of the directory in which it was created (since this is where the file system looks to find your file). This behavior is actually file system (and mount option) dependent. You can either code specifically for each file system and mount option combination, or just perform fsync() calls on the directories to ensure that your code is portable. 

```go
// Create creates a WAL ready for appending records. The given metadata is
// recorded at the head of each WAL file, and can be retrieved with ReadAll.
func Create(lg *zap.Logger, dirpath string, metadata []byte) (*WAL, error) {
	if Exist(dirpath) {
		return nil, os.ErrExist
	}

	// keep temporary wal directory so WAL initialization appears atomic
	tmpdirpath := filepath.Clean(dirpath) + ".tmp"
	if fileutil.Exist(tmpdirpath) {
		if err := os.RemoveAll(tmpdirpath); err != nil {
			return nil, err
		}
	}
	if err := fileutil.CreateDirAll(tmpdirpath); err != nil {
		if lg != nil {
			lg.Warn(
				"failed to create a temporary WAL directory",
				zap.String("tmp-dir-path", tmpdirpath),
				zap.String("dir-path", dirpath),
				zap.Error(err),
			)
		}
		return nil, err
	}

	p := filepath.Join(tmpdirpath, walName(0, 0))
	f, err := fileutil.LockFile(p, os.O_WRONLY|os.O_CREATE, fileutil.PrivateFileMode)
	if err != nil {
		if lg != nil {
			lg.Warn(
				"failed to flock an initial WAL file",
				zap.String("path", p),
				zap.Error(err),
			)
		}
		return nil, err
	}
	if _, err = f.Seek(0, io.SeekEnd); err != nil {
		if lg != nil {
			lg.Warn(
				"failed to seek an initial WAL file",
				zap.String("path", p),
				zap.Error(err),
			)
		}
		return nil, err
	}
	if err = fileutil.Preallocate(f.File, SegmentSizeBytes, true); err != nil {
		if lg != nil {
			lg.Warn(
				"failed to preallocate an initial WAL file",
				zap.String("path", p),
				zap.Int64("segment-bytes", SegmentSizeBytes),
				zap.Error(err),
			)
		}
		return nil, err
	}

	w := &WAL{
		lg:       lg,
		dir:      dirpath,
		metadata: metadata,
	}
	w.encoder, err = newFileEncoder(f.File, 0)
	if err != nil {
		return nil, err
	}
	w.locks = append(w.locks, f)
	if err = w.saveCrc(0); err != nil {
		return nil, err
	}
	if err = w.encoder.encode(&walpb.Record{Type: metadataType, Data: metadata}); err != nil {
		return nil, err
	}
	if err = w.SaveSnapshot(walpb.Snapshot{}); err != nil {
		return nil, err
	}

	if w, err = w.renameWAL(tmpdirpath); err != nil {
		if lg != nil {
			lg.Warn(
				"failed to rename the temporary WAL directory",
				zap.String("tmp-dir-path", tmpdirpath),
				zap.String("dir-path", w.dir),
				zap.Error(err),
			)
		}
		return nil, err
	}

	// directory was renamed; sync parent dir to persist rename
	pdir, perr := fileutil.OpenDir(filepath.Dir(w.dir))
	if perr != nil {
		if lg != nil {
			lg.Warn(
				"failed to open the parent data directory",
				zap.String("parent-dir-path", filepath.Dir(w.dir)),
				zap.String("dir-path", w.dir),
				zap.Error(perr),
			)
		}
		return nil, perr
	}
	if perr = fileutil.Fsync(pdir); perr != nil {
		if lg != nil {
			lg.Warn(
				"failed to fsync the parent data directory file",
				zap.String("parent-dir-path", filepath.Dir(w.dir)),
				zap.String("dir-path", w.dir),
				zap.Error(perr),
			)
		}
		return nil, perr
	}
	if perr = pdir.Close(); err != nil {
		if lg != nil {
			lg.Warn(
				"failed to close the parent data directory file",
				zap.String("parent-dir-path", filepath.Dir(w.dir)),
				zap.String("dir-path", w.dir),
				zap.Error(perr),
			)
		}
		return nil, perr
	}

	return w, nil
}
```

### 打开WAL

假如不是第一次启动，WAL已经存在的情况下，则不调用`Create()`，而是通过`Open()`打开，它是对`openAtIndex()`的封装。我们不需要所有的WAL数据，只需要快照`Index`之后的数据，所以它的最后一个参数是一个`walpb.Snapshot`，里面记录了`Index`。

`openAtIndex()`包含一个`write bool`参数，决定了WAL是只读打开还是以读写方式打开（要求读完最后一条才能接着写）。

- `names, err := readWALNames(lg, dirpath)`拿到所有WAL文件名 //TODO，我感觉这少做了一个排序
- `nameIndex, ok := searchIndex(lg, names, snap.Index)`从WAL文件列表中找到包含这个Index的序号
- 打开这个序号之后的所有WAL文件，用了三种不同接口类型的slice来处理后续的读取和关闭，它们本质上都是`*fileutil.LockedFile`
```go
	rcs := make([]io.ReadCloser, 0)
	rs := make([]io.Reader, 0)
	ls := make([]*fileutil.LockedFile, 0)
```
- 如果是读写方式，再次设置filePipeLine


```go
// Open opens the WAL at the given snap.
// The snap SHOULD have been previously saved to the WAL, or the following
// ReadAll will fail.
// The returned WAL is ready to read and the first record will be the one after
// the given snap. The WAL cannot be appended to before reading out all of its
// previous records.
func Open(lg *zap.Logger, dirpath string, snap walpb.Snapshot) (*WAL, error) {
	w, err := openAtIndex(lg, dirpath, snap, true)
	if err != nil {
		return nil, err
	}
	if w.dirFile, err = fileutil.OpenDir(w.dir); err != nil {
		return nil, err
	}
	return w, nil
}

// OpenForRead only opens the wal files for read.
// Write on a read only wal panics.
func OpenForRead(lg *zap.Logger, dirpath string, snap walpb.Snapshot) (*WAL, error) {
	return openAtIndex(lg, dirpath, snap, false)
}

func openAtIndex(lg *zap.Logger, dirpath string, snap walpb.Snapshot, write bool) (*WAL, error) {
	names, err := readWALNames(lg, dirpath)
	if err != nil {
		return nil, err
	}

	nameIndex, ok := searchIndex(lg, names, snap.Index)
	if !ok || !isValidSeq(lg, names[nameIndex:]) {
		return nil, ErrFileNotFound
	}

	// open the wal files
	rcs := make([]io.ReadCloser, 0)
	rs := make([]io.Reader, 0)
	ls := make([]*fileutil.LockedFile, 0)
	for _, name := range names[nameIndex:] {
		p := filepath.Join(dirpath, name)
		if write {
			l, err := fileutil.TryLockFile(p, os.O_RDWR, fileutil.PrivateFileMode)
			if err != nil {
				closeAll(rcs...)
				return nil, err
			}
			ls = append(ls, l)
			rcs = append(rcs, l)
		} else {
			rf, err := os.OpenFile(p, os.O_RDONLY, fileutil.PrivateFileMode)
			if err != nil {
				closeAll(rcs...)
				return nil, err
			}
			ls = append(ls, nil)
			rcs = append(rcs, rf)
		}
		rs = append(rs, rcs[len(rcs)-1])
	}

	closer := func() error { return closeAll(rcs...) }

	// create a WAL ready for reading
	w := &WAL{
		lg:        lg,
		dir:       dirpath,
		start:     snap,
		decoder:   newDecoder(rs...),
		readClose: closer,
		locks:     ls,
	}

	if write {
		// write reuses the file descriptors from read; don't close so
		// WAL can append without dropping the file lock
		w.readClose = nil
		if _, _, err := parseWALName(filepath.Base(w.tail().Name())); err != nil {
			closer()
			return nil, err
		}
		w.fp = newFilePipeline(w.lg, w.dir, SegmentSizeBytes)
	}

	return w, nil
}
```

### encoder - 用来写入WAL

#### 创建encoder

创建WAL时调用`newFileEncoder()`新建了一个`encoder`对象，它是对`newEncoder()`的封装，调用了`offset, err := f.Seek(0, io.SeekCurrent)`，按照APUE的说法`currpos = lseek(fd, 0, SEEK_CUR)`可以用来获取文件的当前offset.


```go
// walPageBytes is the alignment for flushing records to the backing Writer.
// It should be a multiple of the minimum sector size so that WAL can safely
// distinguish between torn writes and ordinary data corruption.
const walPageBytes = 8 * minSectorSize

type encoder struct {
	mu sync.Mutex
	bw *ioutil.PageWriter

	crc       hash.Hash32
	buf       []byte
	uint64buf []byte
}

func newEncoder(w io.Writer, prevCrc uint32, pageOffset int) *encoder {
	return &encoder{
		bw:  ioutil.NewPageWriter(w, walPageBytes, pageOffset),
		crc: crc.New(prevCrc, crcTable),
		// 1MB buffer
		buf:       make([]byte, 1024*1024),
		uint64buf: make([]byte, 8),
	}
}

// newFileEncoder creates a new encoder with current file offset for the page writer.
func newFileEncoder(f *os.File, prevCrc uint32) (*encoder, error) {
	offset, err := f.Seek(0, io.SeekCurrent)
	if err != nil {
		return nil, err
	}
	return newEncoder(f, prevCrc, int(offset)), nil
}
```

#### ioutil.PageWriter

encoder结构中有一个`bw *ioutil.PageWriter`的成员，这里才真正把数据写入磁盘，他的作用是按照page的大小对齐buffer的数据，当数据大小达到o

数据结构如下：
- w io.Writer 文件io.Writer对象
- pageOffset int 文件起始的偏移量
- pageBytes int 每个page的大小，在创建时传入，encoder创建时传入了一个常量`const walPageBytes = 8 * minSectorSize`，即4Kib
- bufferedBytes int 当前缓存的字节数
- buf []byte, 大小为`defaultBufferBytes+pageBytes`即128 + 4 = 132Kib
- bufWatermarkBytes int，flush的水位，大小为`defaultBufferBytes`即128Kib
 
接下来看看`Write()`函数

- 首先看看这个数据放到buf的话会不会超过水位，如果不会则放到buf就返回
- 否则看看填满一个page还需要多少字节`slack := pw.pageBytes - ((pw.pageOffset + pw.bufferedBytes) % pw.pageBytes)`
  - 先不考虑`partial := slack > len(p)`的情况，我们先把一页给填满，
  - partial的情况我没看明白会在什么条件触发，可能跟初始的pageOffset有关，不过如果一开始判断加上新写入的数据会超过水位的话，这里没道理`新数据的长度`会小于`写满一个page需要的长度`。
- Flush buf中的所有数据
- `if len(p) > pw.pageBytes`如果还有剩下没写入的数据，按page对其后直接写入io.Writer里面，不走buffer了。
- 经过上面的处理，可能还有有一小截没写入的数据，它的大小小于一个page，递归调用自己`pw.Write(p)`把这些数据放到buf中

```go
var defaultBufferBytes = 128 * 1024

// PageWriter implements the io.Writer interface so that writes will
// either be in page chunks or from flushing.
type PageWriter struct {
	w io.Writer
	// pageOffset tracks the page offset of the base of the buffer
	pageOffset int
	// pageBytes is the number of bytes per page
	pageBytes int
	// bufferedBytes counts the number of bytes pending for write in the buffer
	bufferedBytes int
	// buf holds the write buffer
	buf []byte
	// bufWatermarkBytes is the number of bytes the buffer can hold before it needs
	// to be flushed. It is less than len(buf) so there is space for slack writes
	// to bring the writer to page alignment.
	bufWatermarkBytes int
}

// NewPageWriter creates a new PageWriter. pageBytes is the number of bytes
// to write per page. pageOffset is the starting offset of io.Writer.
func NewPageWriter(w io.Writer, pageBytes, pageOffset int) *PageWriter {
	return &PageWriter{
		w:                 w,
		pageOffset:        pageOffset,
		pageBytes:         pageBytes,
		buf:               make([]byte, defaultBufferBytes+pageBytes),
		bufWatermarkBytes: defaultBufferBytes,
	}
}

func (pw *PageWriter) Write(p []byte) (n int, err error) {
	if len(p)+pw.bufferedBytes <= pw.bufWatermarkBytes {
		// no overflow
		copy(pw.buf[pw.bufferedBytes:], p)
		pw.bufferedBytes += len(p)
		return len(p), nil
	}
	// complete the slack page in the buffer if unaligned
	slack := pw.pageBytes - ((pw.pageOffset + pw.bufferedBytes) % pw.pageBytes)
	if slack != pw.pageBytes {
		partial := slack > len(p)
		if partial {
			// not enough data to complete the slack page
			slack = len(p)
		}
		// special case: writing to slack page in buffer
		copy(pw.buf[pw.bufferedBytes:], p[:slack])
		pw.bufferedBytes += slack
		n = slack
		p = p[slack:]
		if partial {
			// avoid forcing an unaligned flush
			return n, nil
		}
	}
	// buffer contents are now page-aligned; clear out
	if err = pw.Flush(); err != nil {
		return n, err
	}
	// directly write all complete pages without copying
	if len(p) > pw.pageBytes {
		pages := len(p) / pw.pageBytes
		c, werr := pw.w.Write(p[:pages*pw.pageBytes])
		n += c
		if werr != nil {
			return n, werr
		}
		p = p[pages*pw.pageBytes:]
	}
	// write remaining tail to buffer
	c, werr := pw.Write(p)
	n += c
	return n, werr
}

func (pw *PageWriter) Flush() error {
	if pw.bufferedBytes == 0 {
		return nil
	}
	_, err := pw.w.Write(pw.buf[:pw.bufferedBytes])
	pw.pageOffset = (pw.pageOffset + pw.bufferedBytes) % pw.pageBytes
	pw.bufferedBytes = 0
	return err
}
```

#### encoder写入数据

`walpb.Reacord`通过`encode()`方法提交给`pageWriter`进行写入，每写入一条`walpb.Record`，它总是先通过`writeUint64()`写入这个记录的长度，然后再通过`e.bw.Write(data)`写入实际的数据。

这里需要注意的一点是这里的`data`总是按照8字节对齐的，如果其长度不是8的倍数，则需要补0，`encodeFrameSize()`就是做这个的，它返回`data`的*长度*`lenField`以及需要补充的字节数`padBytes`。注意以下`lenField`的计算，它实际由4个部分组成:

- MSB第1位如果为1表示有填充
- MSB2-5位没有使用
- MSB6-8位表示填充字段的长度，即`padBytes`的二进制表示，最大可能的取值为7，即111
- 最后56位才是`data`的实际长度

好了，现在我们总算了解wal文件完整的数据结构了
![](https://github.com/4179e1/etcd/raw/master/contrib/raftexample/doc/pic/wal.png)

```go
func (e *encoder) encode(rec *walpb.Record) error {
	e.mu.Lock()
	defer e.mu.Unlock()

	e.crc.Write(rec.Data)
	rec.Crc = e.crc.Sum32()
	var (
		data []byte
		err  error
		n    int
	)

	if rec.Size() > len(e.buf) {
		data, err = rec.Marshal()
		if err != nil {
			return err
		}
	} else {
		n, err = rec.MarshalTo(e.buf)
		if err != nil {
			return err
		}
		data = e.buf[:n]
	}

	lenField, padBytes := encodeFrameSize(len(data))
	if err = writeUint64(e.bw, lenField, e.uint64buf); err != nil {
		return err
	}

	if padBytes != 0 {
		data = append(data, make([]byte, padBytes)...)
	}
	_, err = e.bw.Write(data)
	return err
}

func encodeFrameSize(dataBytes int) (lenField uint64, padBytes int) {
	lenField = uint64(dataBytes)
	// force 8 byte alignment so length never gets a torn write
	padBytes = (8 - (dataBytes % 8)) % 8
	if padBytes != 0 {
		lenField |= uint64(0x80|padBytes) << 56
	}
	return lenField, padBytes
}

func (e *encoder) flush() error {
	e.mu.Lock()
	defer e.mu.Unlock()
	return e.bw.Flush()
}

func writeUint64(w io.Writer, n uint64, buf []byte) error {
	// http://golang.org/src/encoding/binary/binary.go
	binary.LittleEndian.PutUint64(buf, n)
	_, err := w.Write(buf)
	return err
}

```

### 写入WAL

WAL中封装了几种常用类型的save方法，分别是
- saveCrc
- SaveSnapShot
- saveState
- saveEntry
- 以及封装了saveState和saveEntry的Save方法

```go
func (w *WAL) saveState(s *raftpb.HardState) error {
	if raft.IsEmptyHardState(*s) {
		return nil
	}
	w.state = *s
	b := pbutil.MustMarshal(s)
	rec := &walpb.Record{Type: stateType, Data: b}
	return w.encoder.encode(rec)
}

func (w *WAL) Save(st raftpb.HardState, ents []raftpb.Entry) error {
	w.mu.Lock()
	defer w.mu.Unlock()

	// short cut, do not call sync
	if raft.IsEmptyHardState(st) && len(ents) == 0 {
		return nil
	}

	mustSync := raft.MustSync(st, w.state, len(ents))

	// TODO(xiangli): no more reference operator
	for i := range ents {
		if err := w.saveEntry(&ents[i]); err != nil {
			return err
		}
	}
	if err := w.saveState(&st); err != nil {
		return err
	}

	curOff, err := w.tail().Seek(0, io.SeekCurrent)
	if err != nil {
		return err
	}
	if curOff < SegmentSizeBytes {
		if mustSync {
			return w.sync()
		}
		return nil
	}

	return w.cut()
}
```

其中`raft.MustSync`会检查以下三项来确定是否需要调用`w.sync()`进行持久化
- entry的数目非0
- Vote的状态发生变化（从未投票到投了票，投票的对象在同一个Term是不能改的）
- Term任期发生了变化

```go
// MustSync returns true if the hard state and count of Raft entries indicate
// that a synchronous write to persistent storage is required.
func MustSync(st, prevst pb.HardState, entsnum int) bool {
	// Persistent state on all servers:
	// (Updated on stable storage before responding to RPCs)
	// currentTerm
	// votedFor
	// log entries[]
	return entsnum != 0 || st.Vote != prevst.Vote || st.Term != prevst.Term
}
```

最后如果文件的长度超过了64MiB，调用`cut()`方法开始写下一个wal文件

```go
// cut closes current file written and creates a new one ready to append.
// cut first creates a temp wal file and writes necessary headers into it.
// Then cut atomically rename temp wal file to a wal file.
func (w *WAL) cut() error {
	// close old wal file; truncate to avoid wasting space if an early cut
	off, serr := w.tail().Seek(0, io.SeekCurrent)
	if serr != nil {
		return serr
	}

	if err := w.tail().Truncate(off); err != nil {
		return err
	}

	if err := w.sync(); err != nil {
		return err
	}

	fpath := filepath.Join(w.dir, walName(w.seq()+1, w.enti+1))

	// create a temp wal file with name sequence + 1, or truncate the existing one
	newTail, err := w.fp.Open()
	if err != nil {
		return err
	}

	// update writer and save the previous crc
	w.locks = append(w.locks, newTail)
	prevCrc := w.encoder.crc.Sum32()
	w.encoder, err = newFileEncoder(w.tail().File, prevCrc)
	if err != nil {
		return err
	}

	if err = w.saveCrc(prevCrc); err != nil {
		return err
	}

	if err = w.encoder.encode(&walpb.Record{Type: metadataType, Data: w.metadata}); err != nil {
		return err
	}

	if err = w.saveState(&w.state); err != nil {
		return err
	}

	// atomically move temp wal file to wal file
	if err = w.sync(); err != nil {
		return err
	}

	off, err = w.tail().Seek(0, io.SeekCurrent)
	if err != nil {
		return err
	}

	if err = os.Rename(newTail.Name(), fpath); err != nil {
		return err
	}
	if err = fileutil.Fsync(w.dirFile); err != nil {
		return err
	}

	// reopen newTail with its new path so calls to Name() match the wal filename format
	newTail.Close()

	if newTail, err = fileutil.LockFile(fpath, os.O_WRONLY, fileutil.PrivateFileMode); err != nil {
		return err
	}
	if _, err = newTail.Seek(off, io.SeekStart); err != nil {
		return err
	}

	w.locks[len(w.locks)-1] = newTail

	prevCrc = w.encoder.crc.Sum32()
	w.encoder, err = newFileEncoder(w.tail().File, prevCrc)
	if err != nil {
		return err
	}

	if w.lg != nil {
		w.lg.Info("created a new WAL segment", zap.String("path", fpath))
	} else {
		plog.Infof("segmented wal file %v is created", fpath)
	}
	return nil
}
```

### decoder - 用来读取WAL

#### decoder 结构

跟`encoder`对应的是`decoder`，`newEncoder`的参数就是`Create`时的`rs`这个切片，记录了所有需要打开的wal文件

```go
const minSectorSize = 512

// frameSizeBytes is frame size in bytes, including record size and padding size.
const frameSizeBytes = 8

type decoder struct {
	mu  sync.Mutex
	brs []*bufio.Reader

	// lastValidOff file offset following the last valid decoded record
	lastValidOff int64
	crc          hash.Hash32
}

func newDecoder(r ...io.Reader) *decoder {
	readers := make([]*bufio.Reader, len(r))
	for i := range r {
		readers[i] = bufio.NewReader(r[i])
	}
	return &decoder{
		brs: readers,
		crc: crc.New(0, crcTable),
	}
}
```

#### encoder读取数据

每次调用`decode`就会读取一段Record，直到一个文件返回EOF，递归读下一个，当所有文件都到了结尾，返回EOF

跟`decoder`相对的，
- 首先`l, err := readInt64(d.brs[0])`读取Record的长度
  - 如果读到io.EOF，则尝试读取下一个wal文件
  - 如果已经是最后一个，则返回io.EOF表示WAL已经读完了
- `recBytes, padBytes := decodeFrameSize(l)`计算出Record的实际长度和填充的长度
- 读取总长度的数据并反序列化为`walpb.Record`(通过参数的rec 参数传入
  - 如果反序列化失败，会检查是不是torn write, `isTornEntry()`会把按照`inSectorSize`的大小即512把`data`拆分成多个chunk，如果任何一个chunk的数据全为0，则认为是一个torn write
- 计算CRC是不是符合预期
- 更新文件的偏移量后返回

> so what is a torn write?
> http://www.joshodgers.com/tag/torn-write/

```go
func (d *decoder) decode(rec *walpb.Record) error {
	rec.Reset()
	d.mu.Lock()
	defer d.mu.Unlock()
	return d.decodeRecord(rec)
}

func (d *decoder) decodeRecord(rec *walpb.Record) error {
	if len(d.brs) == 0 {
		return io.EOF
	}

	l, err := readInt64(d.brs[0])
	if err == io.EOF || (err == nil && l == 0) {
		// hit end of file or preallocated space
		d.brs = d.brs[1:]
		if len(d.brs) == 0 {
			return io.EOF
		}
		d.lastValidOff = 0
		return d.decodeRecord(rec)
	}
	if err != nil {
		return err
	}

	recBytes, padBytes := decodeFrameSize(l)

	data := make([]byte, recBytes+padBytes)
	if _, err = io.ReadFull(d.brs[0], data); err != nil {
		// ReadFull returns io.EOF only if no bytes were read
		// the decoder should treat this as an ErrUnexpectedEOF instead.
		if err == io.EOF {
			err = io.ErrUnexpectedEOF
		}
		return err
	}
	if err := rec.Unmarshal(data[:recBytes]); err != nil {
		if d.isTornEntry(data) {
			return io.ErrUnexpectedEOF
		}
		return err
	}

	// skip crc checking if the record type is crcType
	if rec.Type != crcType {
		d.crc.Write(rec.Data)
		if err := rec.Validate(d.crc.Sum32()); err != nil {
			if d.isTornEntry(data) {
				return io.ErrUnexpectedEOF
			}
			return err
		}
	}
	// record decoded as valid; point last valid offset to end of record
	d.lastValidOff += frameSizeBytes + recBytes + padBytes
	return nil
}

func decodeFrameSize(lenField int64) (recBytes int64, padBytes int64) {
	// the record size is stored in the lower 56 bits of the 64-bit length
	recBytes = int64(uint64(lenField) & ^(uint64(0xff) << 56))
	// non-zero padding is indicated by set MSb / a negative length
	if lenField < 0 {
		// padding is stored in lower 3 bits of length MSB
		padBytes = int64((uint64(lenField) >> 56) & 0x7)
	}
	return recBytes, padBytes
}

// isTornEntry determines whether the last entry of the WAL was partially written
// and corrupted because of a torn write.
func (d *decoder) isTornEntry(data []byte) bool {
	if len(d.brs) != 1 {
		return false
	}

	fileOff := d.lastValidOff + frameSizeBytes
	curOff := 0
	chunks := [][]byte{}
	// split data on sector boundaries
	for curOff < len(data) {
		chunkLen := int(minSectorSize - (fileOff % minSectorSize))
		if chunkLen > len(data)-curOff {
			chunkLen = len(data) - curOff
		}
		chunks = append(chunks, data[curOff:curOff+chunkLen])
		fileOff += int64(chunkLen)
		curOff += chunkLen
	}

	// if any data for a sector chunk is all 0, it's a torn write
	for _, sect := range chunks {
		isZero := true
		for _, v := range sect {
			if v != 0 {
				isZero = false
				break
			}
		}
		if isZero {
			return true
		}
	}
	return false
}
```



## Reference

- [etcd raft模块分析--WAL日志](https://my.oschina.net/fileoptions/blog/1825531)
- [ETCD V3 中的 .wal 文件](https://blog.zhesih.com/2017/10/03/the-wal-files-in-etcd-v3/)
- [Everything You Always Wanted to Know About Fsync()](http://blog.httrack.com/blog/2013/11/15/everything-you-always-wanted-to-know-about-fsync/)
- [Ensuring data reaches disk](https://lwn.net/Articles/457667/)
