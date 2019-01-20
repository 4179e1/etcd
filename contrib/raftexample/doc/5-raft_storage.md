# Raft Storage

 `raft.Storage`是一个接口，用来存取raft 日志，raft自带了一个`MemoryStorage`的实现，应用也可以提供自己的实现。

 //TODO；虽然这么说，可我还是不明白这玩意干啥的.

 ## Raft Storage 接口

 该接口要求实现6个方法，`etcd raft`内部会调用这些方法.

```go
// Storage is an interface that may be implemented by the application
// to retrieve log entries from storage.
//
// If any Storage method returns an error, the raft instance will
// become inoperable and refuse to participate in elections; the
// application is responsible for cleanup and recovery in this case.
type Storage interface {
	// InitialState returns the saved HardState and ConfState information.
	InitialState() (pb.HardState, pb.ConfState, error)
	// Entries returns a slice of log entries in the range [lo,hi).
	// MaxSize limits the total size of the log entries returned, but
	// Entries returns at least one entry if any.
	Entries(lo, hi, maxSize uint64) ([]pb.Entry, error)
	// Term returns the term of entry i, which must be in the range
	// [FirstIndex()-1, LastIndex()]. The term of the entry before
	// FirstIndex is retained for matching purposes even though the
	// rest of that entry may not be available.
	Term(i uint64) (uint64, error)
	// LastIndex returns the index of the last entry in the log.
	LastIndex() (uint64, error)
	// FirstIndex returns the index of the first log entry that is
	// possibly available via Entries (older entries have been incorporated
	// into the latest Snapshot; if storage only contains the dummy entry the
	// first log entry is not available).
	FirstIndex() (uint64, error)
	// Snapshot returns the most recent snapshot.
	// If snapshot is temporarily unavailable, it should return ErrSnapshotTemporarilyUnavailable,
	// so raft state machine could know that Storage needs some time to prepare
	// snapshot and call Snapshot later.
	Snapshot() (pb.Snapshot, error)
}
```

## MemoryStorage

注意`NewMemoryStorage()`只初始化了一个元素：`ents: make([]pb.Entry, 1)`，这里创建了只有一个元素的切片，`ents[0]`这个位置实际上是不存Entry Data的，但是Index和Term还是会更新，作为哨兵来使用，表示上一次Compat的最后一个index。这里没有初始化，因此Term和Index都是0。

> raft的index从1开始

```go
type MemoryStorage struct {
	// Protects access to all fields. Most methods of MemoryStorage are
	// run on the raft goroutine, but Append() is run on an application
	// goroutine.
	sync.Mutex

	hardState pb.HardState
	snapshot  pb.Snapshot
	// ents[i] has raft log position i+snapshot.Metadata.Index
	ents []pb.Entry
}

// NewMemoryStorage creates an empty MemoryStorage.
func NewMemoryStorage() *MemoryStorage {
	return &MemoryStorage{
		// When starting from scratch populate the list with a dummy entry at term zero.
		ents: make([]pb.Entry, 1),
	}
}
```

回顾一下`pb.Entry`:

```go
type Entry struct {
	Term             uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`
	Index            uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
	Type             EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
	Data             []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
	XXX_unrecognized []byte    `json:"-"`
}
```

### raft Storage 要求的6个方法


`InittialState()`直接返回`hardState`和快照的`Metadata.ConfState`，对于后者不需要进行防御性编程，即使ms.snapshot为nil，`ms.snapshot.Metadata.ConfState`依然能取到一个合理的空值而不触发错误。

```go
// InitialState implements the Storage interface.
func (ms *MemoryStorage) InitialState() (pb.HardState, pb.ConfState, error) {
	return ms.hardState, ms.snapshot.Metadata.ConfState, nil
}
```

`FirstIndex()`是对`firstIndex()`加锁的封装，返回的是`ents[0].Index + 1`,前面说了`ents[0]`是个哨兵。


```go
// FirstIndex implements the Storage interface.
func (ms *MemoryStorage) FirstIndex() (uint64, error) {
	ms.Lock()
	defer ms.Unlock()
	return ms.firstIndex(), nil
}

func (ms *MemoryStorage) firstIndex() uint64 {
	return ms.ents[0].Index + 1
}
```

`LastIndex()`是对`lastIndex()`的封装，加了个锁。`lastIndex()`为啥不用`ms.ents[len (ms.ents) -1 ].Index`呢？

```go
// LastIndex implements the Storage interface.
func (ms *MemoryStorage) LastIndex() (uint64, error) {
	ms.Lock()
	defer ms.Unlock()
	return ms.lastIndex(), nil
}

func (ms *MemoryStorage) lastIndex() uint64 {
	return ms.ents[0].Index + uint64(len(ms.ents)) - 1
}
```

`Entires()`最多会返回index为`lo`和`hi`之间的entry，要求它们大小的和（字节数，不是长度）小于`maxSize`，`limitSize()`就是用来计算和限制长度的。

```go
// Entries implements the Storage interface.
func (ms *MemoryStorage) Entries(lo, hi, maxSize uint64) ([]pb.Entry, error) {
	ms.Lock()
	defer ms.Unlock()
	offset := ms.ents[0].Index
	if lo <= offset {
		return nil, ErrCompacted
	}
	if hi > ms.lastIndex()+1 {
		raftLogger.Panicf("entries' hi(%d) is out of bound lastindex(%d)", hi, ms.lastIndex())
	}
	// only contains dummy entries.
	if len(ms.ents) == 1 {
		return nil, ErrUnavailable
	}

	ents := ms.ents[lo-offset : hi-offset]
	return limitSize(ents, maxSize), nil
}

func limitSize(ents []pb.Entry, maxSize uint64) []pb.Entry {
	if len(ents) == 0 {
		return ents
	}
	size := ents[0].Size()
	var limit int
	for limit = 1; limit < len(ents); limit++ {
		size += ents[limit].Size()
		if uint64(size) > maxSize {
			break
		}
	}
	return ents[:limit]
}
```

`Term()`根据指定的Index返回其对应的Term

```go
// Term implements the Storage interface.
func (ms *MemoryStorage) Term(i uint64) (uint64, error) {
	ms.Lock()
	defer ms.Unlock()
	offset := ms.ents[0].Index
	if i < offset {
		return 0, ErrCompacted
	}
	if int(i-offset) >= len(ms.ents) {
		return 0, ErrUnavailable
	}
	return ms.ents[i-offset].Term, nil
}
```

`Snapshot()`的实现简单粗暴。

``` go
// Snapshot implements the Storage interface.
func (ms *MemoryStorage) Snapshot() (pb.Snapshot, error) {
	ms.Lock()
	defer ms.Unlock()
	return ms.snapshot, nil
}
```


### 追加entry

- `if len(enries) == 0`啥时候会出现这种状态？反正初始化时不会。
- `if last < first`只有`len(ents) == 1`时会出现这种情况
- `if first > entries[0].Index`有些Entry已经compact了，直接丢弃。
- `offset := entries[0].Index - ms.ents[0].Index`记录现有entry和要append的entry首元素index的偏移，`len(ms.ents)`和`offset`的差可以用来判断`ms.ents`和要追加的`entries`之间是否重叠，或者是否有空缺。
  - `int64(len(ms.ents)) > offset`说明两者有重叠的一部分
    - `ms.ents = append([]pb.Entry{}, ms.ents[:offset]...)`这里新建了一个切片，并且第一个元素还是一个空的`pb.Entry()`作为哨兵，然后追加两者不重叠的部分`ms.ents[:offset]`
    - `ms.ents = append(ms.ents, entries...)`重叠的部分会被新追加的Entry覆盖，不管它们的值是否一致，反正以新append的为准。
  - `uint64(len(ms.ents)) == offset`说明两者没有重叠也没有gap，正好连上了，直接拼一起就好
  - `default`的情况是中间出现了空洞，除了panic还能怎么样呢。

> In Raft, the leader handles inconsistencies by forcing the followers’ logs to duplicate its own. This means that conflicting entries in follower logs will be overwritten with entries from the leader’s log. Section 5.4 will show that this is safe when coupled with one more restriction.

```go
// Append the new entries to storage.
// TODO (xiangli): ensure the entries are continuous and
// entries[0].Index > ms.entries[0].Index
func (ms *MemoryStorage) Append(entries []pb.Entry) error {
	if len(entries) == 0 {
		return nil
	}

	ms.Lock()
	defer ms.Unlock()

	first := ms.firstIndex()
	last := entries[0].Index + uint64(len(entries)) - 1

	// shortcut if there is no new entry.
	if last < first {
		return nil
	}
	// truncate compacted entries
	if first > entries[0].Index {
		entries = entries[first-entries[0].Index:]
	}

	offset := entries[0].Index - ms.ents[0].Index
	switch {
	case uint64(len(ms.ents)) > offset:
		ms.ents = append([]pb.Entry{}, ms.ents[:offset]...)
		ms.ents = append(ms.ents, entries...)
	case uint64(len(ms.ents)) == offset:
		ms.ents = append(ms.ents, entries...)
	default:
		raftLogger.Panicf("missing log entry [last: %d, append at: %d]",
			ms.lastIndex(), entries[0].Index)
	}
	return nil
}
```

### 压缩Entry

创建snapshot之后，部分的Entry就不再需要保存在Storage中了，`Compact()`就是做这个的，丢弃的Entry包含这个`compactIndex`。
注意`ents := make([]pb.Entry, 1, 1+uint64(len(ms.ents))-i)`第二个参数是1，里面还是插入了一个哨兵元素。

```go
// Compact discards all log entries prior to compactIndex.
// It is the application's responsibility to not attempt to compact an index
// greater than raftLog.applied.
func (ms *MemoryStorage) Compact(compactIndex uint64) error {
	ms.Lock()
	defer ms.Unlock()
	offset := ms.ents[0].Index
	if compactIndex <= offset {
		return ErrCompacted
	}
	if compactIndex > ms.lastIndex() {
		raftLogger.Panicf("compact %d is out of bound lastindex(%d)", compactIndex, ms.lastIndex())
	}

	i := compactIndex - offset
	ents := make([]pb.Entry, 1, 1+uint64(len(ms.ents))-i)
	ents[0].Index = ms.ents[i].Index
	ents[0].Term = ms.ents[i].Term
	ents = append(ents, ms.ents[i+1:]...)
	ms.ents = ents
	return nil
}
```

### 加载快照

`ms.snapshot = snap`直接给snapshot赋值
`ms.ents = []pb.Entry{{Term: snap.Metadata.Term, Index: snap.Metadata.Index}}`丢弃已有的entry，并且把哨兵元素的Term和Index更新为快照的（快照可以理解为上一次compact过）。

两种情况会调用这个函数
- 初始化时加载快照
- 处理`rc.node.Ready()` 时master发了个快照过来。

```go
// ApplySnapshot overwrites the contents of this Storage object with
// those of the given snapshot.
func (ms *MemoryStorage) ApplySnapshot(snap pb.Snapshot) error {
	ms.Lock()
	defer ms.Unlock()

	//handle check for old snapshot being applied
	msIndex := ms.snapshot.Metadata.Index
	snapIndex := snap.Metadata.Index
	if msIndex >= snapIndex {
		return ErrSnapOutOfDate
	}

	ms.snapshot = snap
	ms.ents = []pb.Entry{{Term: snap.Metadata.Term, Index: snap.Metadata.Index}}
	return nil
}
```

### 创建（和更新）快照

`CreateSnapshot()`使用特定的`index`及对应该index的快照数据`data`更新自己的`snapshot`字段，并返回这个新的snapshot。

```go
// CreateSnapshot makes a snapshot which can be retrieved with Snapshot() and
// can be used to reconstruct the state at that point.
// If any configuration changes have been made since the last compaction,
// the result of the last ApplyConfChange must be passed in.
func (ms *MemoryStorage) CreateSnapshot(i uint64, cs *pb.ConfState, data []byte) (pb.Snapshot, error) {
	ms.Lock()
	defer ms.Unlock()
	if i <= ms.snapshot.Metadata.Index {
		return pb.Snapshot{}, ErrSnapOutOfDate
	}

	offset := ms.ents[0].Index
	if i > ms.lastIndex() {
		raftLogger.Panicf("snapshot %d is out of bound lastindex(%d)", i, ms.lastIndex())
	}

	ms.snapshot.Metadata.Index = i
	ms.snapshot.Metadata.Term = ms.ents[i-offset].Term
	if cs != nil {
		ms.snapshot.Metadata.ConfState = *cs
	}
	ms.snapshot.Data = data
	return ms.snapshot, nil
}
```