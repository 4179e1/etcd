# Etcd Raft 相关的数据结构

## unstable

`unstable`用在`raftLog`中，保存了

### unstable 结构

`unstable`保存的是还没有持久化的entry

其中offset是保存的是index的offset，表示第一条unstable entry的index。

```go
// unstable.entries[i] has raft log position i+unstable.offset.
// Note that unstable.offset may be less than the highest log
// position in storage; this means that the next write to storage
// might need to truncate the log before persisting unstable.entries.
type unstable struct {
	// the incoming unstable snapshot, if any.
	snapshot *pb.Snapshot
	// all entries that have not yet been written to storage.
	entries []pb.Entry
	offset  uint64

	logger Logger
}
```

### maybe 系列函数 - 猜测可能的index和term

- `maybeFirstIndex()`返回可能是entry中第一个entry的index
  - 如果它有快照的话，就是返回快照index + 1
  - 否则返回0，说明这里还没存过东西
  - TODO 为什么不检查entries？
- `maybeLastIndex()`返回可能是entry中最后一个entry的index
  - 如果它至少有一个entry，返回最后一个entry的index
  - 否则如果它有快照，返回index + 1
  - 否则返回0,说明还没存过东西
- `maybeTerm()`返回index i对应的term
  - 如果i小于unstable的offset，那么之后当i等于snapshot index时能取到term
  - 否则只有i在[u.offset, last]这个范围内能取到term

```go
// maybeFirstIndex returns the index of the first possible entry in entries
// if it has a snapshot.
func (u *unstable) maybeFirstIndex() (uint64, bool) {
	if u.snapshot != nil {
		return u.snapshot.Metadata.Index + 1, true
	}
	return 0, false
}

// maybeLastIndex returns the last index if it has at least one
// unstable entry or snapshot.
func (u *unstable) maybeLastIndex() (uint64, bool) {
	if l := len(u.entries); l != 0 {
		return u.offset + uint64(l) - 1, true
	}
	if u.snapshot != nil {
		return u.snapshot.Metadata.Index, true
	}
	return 0, false
}

// maybeTerm returns the term of the entry at index i, if there
// is any.
func (u *unstable) maybeTerm(i uint64) (uint64, bool) {
	if i < u.offset {
		if u.snapshot == nil {
			return 0, false
		}
		if u.snapshot.Metadata.Index == i {
			return u.snapshot.Metadata.Term, true
		}
		return 0, false
	}

	last, ok := u.maybeLastIndex()
	if !ok {
		return 0, false
	}
	if i > last {
		return 0, false
	}
	return u.entries[i-u.offset].Term, true
}
```

### stableTo() - entry被持久化后

`stableTo()`把index i, term t 以前的entry标记为stable并丢弃这个stable的entry，当满足这个条件时
- `u.maybeTerm(i)`猜测的term == t
- i >= u.offset

当`u.entries`这个切片太长，并且大部分元素都不使用时，尝试用`shrinkEntriesArray()`复制一份在使用的元素，并释放老的
//TODO 这种情况怎么出现的？

```go
func (u *unstable) stableTo(i, t uint64) {
	gt, ok := u.maybeTerm(i)
	if !ok {
		return
	}
	// if i < offset, term is matched with the snapshot
	// only update the unstable entries if term is matched with
	// an unstable entry.
	if gt == t && i >= u.offset {
		u.entries = u.entries[i+1-u.offset:]
		u.offset = i + 1
		u.shrinkEntriesArray()
	}
}

// shrinkEntriesArray discards the underlying array used by the entries slice
// if most of it isn't being used. This avoids holding references to a bunch of
// potentially large entries that aren't needed anymore. Simply clearing the
// entries wouldn't be safe because clients might still be using them.
func (u *unstable) shrinkEntriesArray() {
	// We replace the array if we're using less than half of the space in
	// it. This number is fairly arbitrary, chosen as an attempt to balance
	// memory usage vs number of allocations. It could probably be improved
	// with some focused tuning.
	const lenMultiple = 2
	if len(u.entries) == 0 {
		u.entries = nil
	} else if len(u.entries)*lenMultiple < cap(u.entries) {
		newEntries := make([]pb.Entry, len(u.entries))
		copy(newEntries, u.entries)
		u.entries = newEntries
	}
}
```

### restore()

`restore()`在恢复快照时使用，这是唯一会设置`u.snapshot`的地方，影响`maybeFirstIndex()`

```go
func (u *unstable) restore(s pb.Snapshot) {
	u.offset = s.Metadata.Index + 1
	u.entries = nil
	u.snapshot = &s
}
```

### 追加entry

after是第一条要追加的entry的index，这里的处理分三种情况
- `case after == u.offset+uint64(len(u.entries)):` 刚后跟原来的最后一条接上了，直接append就ok
- `case after <= u.offset:` after在unstable的offset之前，或者相等，直接用append的entry取代u.entries，并且把u.offset也改为after这个index。 出现这种情况可能是老的leader复制了一条entry到本节点之后crash了，新的leader复制了一条不同的entry。
- `default` 要append的log在offset和最后一条unstable中间，要truncate一下再拼接起来。第一个append的`[]pb.Entry{}`新建了一个空的slice

```go
func (u *unstable) truncateAndAppend(ents []pb.Entry) {
	after := ents[0].Index
	switch {
	case after == u.offset+uint64(len(u.entries)):
		// after is the next index in the u.entries
		// directly append
		u.entries = append(u.entries, ents...)
	case after <= u.offset:
		u.logger.Infof("replace the unstable entries from index %d", after)
		// The log is being truncated to before our current offset
		// portion, so set the offset and replace the entries
		u.offset = after
		u.entries = ents
	default:
		// truncate to after and copy to u.entries
		// then append
		u.logger.Infof("truncate the unstable entries before index %d", after)
		u.entries = append([]pb.Entry{}, u.slice(u.offset, after)...)
		u.entries = append(u.entries, ents...)
	}
}

func (u *unstable) slice(lo uint64, hi uint64) []pb.Entry {
	u.mustCheckOutOfBounds(lo, hi)
	return u.entries[lo-u.offset : hi-u.offset]
}

// u.offset <= lo <= hi <= u.offset+len(u.entries)
func (u *unstable) mustCheckOutOfBounds(lo, hi uint64) {
	if lo > hi {
		u.logger.Panicf("invalid unstable.slice %d > %d", lo, hi)
	}
	upper := u.offset + uint64(len(u.entries))
	if lo < u.offset || hi > upper {
		u.logger.Panicf("unstable.slice[%d,%d) out of bound [%d,%d]", lo, hi, u.offset, upper)
	}
}
```

## raftLog

### raftLog 结构

- storage： MemoryStorage
- unstable： 保存还没持久化的log entry
- committed： server已知的committed index，不需要持久化
- applied: server已知的applied index，不需要持久化
- maxNextEntsSize: 一次最多返回的大小

```go
type raftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage

	// unstable contains all unstable entries and snapshot.
	// they will be saved into storage.
	unstable unstable

	// committed is the highest log position that is known to be in
	// stable storage on a quorum of nodes.
	committed uint64
	// applied is the highest log position that the application has
	// been instructed to apply to its state machine.
	// Invariant: applied <= committed
	applied uint64

	logger Logger

	// maxNextEntsSize is the maximum number aggregate byte size of the messages
	// returned from calls to nextEnts.
	maxNextEntsSize uint64
}
```

### 新建raftLog

注意这两个传入的参数

- storage: raftlog需要从storage里面取log entry
- maxNextEntsSize，一次最多返回多大的log（大小，非个数）

`firstIndex`和`lastIndex`都取自stoarge
- log.unstable.offset 初始化为 `lastIndex + 1`，也就是第一条unstable entry的index
- log.committed 和 log.applied 都初始化为`firstIndex - 1`，这个好理解，storage之前的数据肯定已经在snapshot中了，snapshot中的数据必然是已经applied了。从WAL回放的数据需要重新commit和apply一次。


```go
// newLog returns log using the given storage and default options. It
// recovers the log to the state that it just commits and applies the
// latest snapshot.
func newLog(storage Storage, logger Logger) *raftLog {
	return newLogWithSize(storage, logger, noLTODO 
}

// newLogWithSize returns a log using the given storage and max
// message size.
func newLogWithSize(storage Storage, logger Logger, maxNextEntsSize uint64) *raftLog {
	if storage == nil {
		log.Panic("storage must not be nil")
	}
	log := &raftLog{
		storage:         storage,
		logger:          logger,
		maxNextEntsSize: maxNextEntsSize,
	}
	firstIndex, err := storage.FirstIndex()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	lastIndex, err := storage.LastIndex()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	log.unstable.offset = lastIndex + 1
	log.unstable.logger = logger
	// Initialize our committed and applied pointers to the time of the last compaction.
	log.committed = firstIndex - 1
	log.applied = firstIndex - 1

	return log
}
```


### 获取索引

`firstIndex()`和`lastIndex()`都先查看l.unstable是否有元素，有的话从这里返回，否则从storage返回。
相当于把unstable和stable的log entry联合起来查询.

特别注意,`firstIndex()`要么返回unstable.snapshot 的index，要么返回storage的第一条index

```go
func (l *raftLog) firstIndex() uint64 {
	if i, ok := l.unstable.maybeFirstIndex(); ok {
		return i
	}
	index, err := l.storage.FirstIndex()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	return index
}

func (l *raftLog) lastIndex() uint64 {
	if i, ok := l.unstable.maybeLastIndex(); ok {
		return i
	}
	i, err := l.storage.LastIndex()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}
	return i
}
```

### 获取entry

两个方法

- `unstableEntries()`获取没有持久化的log entry，直接返回`l.unstable.entries`，应用层收到后需要写入WAL，并加到Storage中
- `nextEnts()`返回可以apply的log entry，应用层收到后直接应用到状态机中
  - `off := max(l.applied+1, l.firstIndex())`
    -  // TODO, why? what's that? 初始状态下l.applied跟l.firstIndex()是相同的。
	- l.applied + 1大概是为了取下一个要apply的entry，不过如果unstable里面有snapshot，并且snapshot的值比applied + 1要大，就取snapshot后面的内容吧。
	- 所以off其实是已经applied的index？
  - `if l.committed+1 > off`如果commit的index > off ，那么通过`l.slice()`找出applied和committed之间的log entry

FIXME：这两函数是不是重复传数据了？


```go
func (l *raftLog) unstableEntries() []pb.Entry {
	if len(l.unstable.entries) == 0 {
		return nil
	}
	return l.unstable.entries
}

// nextEnts returns all the available entries for execution.
// If applied is smaller than the index of snapshot, it returns all committed
// entries after the index of snapshot.
func (l *raftLog) nextEnts() (ents []pb.Entry) {
	off := max(l.applied+1, l.firstIndex())
	if l.committed+1 > off {
		ents, err := l.slice(off, l.committed+1, l.maxNextEntsSize)
		if err != nil {
			l.logger.Panicf("unexpected error when getting unapplied entries (%v)", err)
		}
		return ents
	}
	return nil
}
```

`slice()`从storage或者unstable中找出符合范围的log entry，并且保证返回的entry大小不大于`maxSize`
- `lo < l.unstable.offset`尝试从storage中找
  - 如果找到的内容大小已经满足了，直接返回，不再找unstable
- 如果大小限制还没满足，检查可能看是否`hi > l.unstable.offset`
  - 是的话把unstable的所有entry追加上去，//这种情况下entry的大小可能超了
- 最后调用`limitSize(ents, maxSize)`限制返回log的大小


```go
// slice returns a slice of log entries from lo through hi-1, inclusive.
func (l *raftLog) slice(lo, hi, maxSize uint64) ([]pb.Entry, error) {
	err := l.mustCheckOutOfBounds(lo, hi)
	if err != nil {
		return nil, err
	}
	if lo == hi {
		return nil, nil
	}
	var ents []pb.Entry
	if lo < l.unstable.offset {
		storedEnts, err := l.storage.Entries(lo, min(hi, l.unstable.offset), maxSize)
		if err == ErrCompacted {
			return nil, err
		} else if err == ErrUnavailable {
			l.logger.Panicf("entries[%d:%d) is unavailable from storage", lo, min(hi, l.unstable.offset))
		} else if err != nil {
			panic(err) // TODO(bdarnell)
		}

		// check if ents has reached the size limitation
		if uint64(len(storedEnts)) < min(hi, l.unstable.offset)-lo {
			return storedEnts, nil
		}

		ents = storedEnts
	}
	if hi > l.unstable.offset {
		unstable := l.unstable.slice(max(lo, l.unstable.offset), hi)
		if len(ents) > 0 {
			ents = append([]pb.Entry{}, ents...)
			ents = append(ents, unstable...)
		} else {
			ents = unstable
		}
	}
	return limitSize(ents, maxSize), nil
}

// l.firstIndex <= lo <= hi <= l.firstIndex + len(l.entries)
func (l *raftLog) mustCheckOutOfBounds(lo, hi uint64) error {
	if lo > hi {
		l.logger.Panicf("invalid slice %d > %d", lo, hi)
	}
	fi := l.firstIndex()
	if lo < fi {
		return ErrCompacted
	}

	length := l.lastIndex() + 1 - fi
	if lo < fi || hi > fi+length {
		l.logger.Panicf("slice[%d,%d) out of bound [%d,%d]", lo, hi, fi, l.lastIndex())
	}
	return nil
}
```

## Progress

## readOnly