# Etcd Raft 相关的数据结构

在进入raft数据结构前，先了解一下它内部用到的其他一些对象：

## unstable

`unstable`用在`raftLog`中，保存了

### unstable 结构

`unstable`保存的是还没有commit的entry

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

### 获取index和term

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

### stable 系列函数

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

`stableSnapTo()`这个函数在index跟快照的index相当的时候，丢弃unstable的快照，标记为stable

```go
func (u *unstable) stableSnapTo(i uint64) {
	if u.snapshot != nil && u.snapshot.Metadata.Index == i {
		u.snapshot = nil
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
	return newLogWithSize(storage, logger, noLimit) 
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

### index和Term

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

- `term()`从unstable或者storage中返回某个index的term，如果找到的话
- `lastTerm()`返回最后一个entry的term

```go
func (l *raftLog) lastTerm() uint64 {
	t, err := l.term(l.lastIndex())
	if err != nil {
		l.logger.Panicf("unexpected error when getting the last term (%v)", err)
	}
	return t
}

func (l *raftLog) term(i uint64) (uint64, error) {
	// the valid term range is [index of dummy entry, last index]
	dummyIndex := l.firstIndex() - 1
	if i < dummyIndex || i > l.lastIndex() {
		// TODO: return an error instead?
		return 0, nil
	}

	if t, ok := l.unstable.maybeTerm(i); ok {
		return t, nil
	}

	t, err := l.storage.Term(i)
	if err == nil {
		return t, nil
	}
	if err == ErrCompacted || err == ErrUnavailable {
		return 0, err
	}
	panic(err) // TODO(bdarnell)
}
```

`matchTerm()`检查给定的index和term在unstable或者storage中是否匹配
`isUpToDate()`比较给定的index和term跟当前的log相比谁更uptodate，其实就是raft 论文5.4.1最后一段

```go
// isUpToDate determines if the given (lastIndex,term) log is more up-to-date
// by comparing the index and term of the last entries in the existing logs.
// If the logs have last entries with different terms, then the log with the
// later term is more up-to-date. If the logs end with the same term, then
// whichever log has the larger lastIndex is more up-to-date. If the logs are
// the same, the given log is up-to-date.
func (l *raftLog) isUpToDate(lasti, term uint64) bool {
	return term > l.lastTerm() || (term == l.lastTerm() && lasti >= l.lastIndex())
}

func (l *raftLog) matchTerm(i, term uint64) bool {
	t, err := l.term(i)
	if err != nil {
		return false
	}
	return t == term
}
```

### 获取entry

node的`newReady()`会调用下面两个函数，用来返回uncommit和要commit的log：

- `unstableEntries()`获取没有commit的log entry，直接返回`l.unstable.entries`，应用层收到后需要写入WAL，并加到Storage中
- `nextEnts()`返回已经commit可以apply的log entry，应用层收到后直接应用到状态机中
  - `off := max(l.applied+1, l.firstIndex())`
	- off其实是已经applied的index。注释说`If applied is smaller than the index of snapshot, it returns all committed entries after the index of snapshot.`
  - `if l.committed+1 > off`如果commit的index > off ，那么通过`l.slice()`找出applied和committed之间的log entry

FIXME：这两函数是不是重复传数据了?至少在`node.Ready()`这一层面


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

关于entry还有另外两个函数

`entries()`尝试返回storage和unstable中index i 之后的所有entries，前提是满足maxsize
`allEntreis()`调用`entries()`返回所有entries
- maxize被设置为`noLimit`，即math.MaxUint64
- 为了解决log compact可能导致的race condition，居然递归调用自己…… // TODO 这个race condition从哪来的？为什么前面俩个那个函数不用处理这个？

```go
func (l *raftLog) entries(i, maxsize uint64) ([]pb.Entry, error) {
	if i > l.lastIndex() {
		return nil, nil
	}
	return l.slice(i, l.lastIndex()+1, maxsize)
}

// allEntries returns all entries in the log.
func (l *raftLog) allEntries() []pb.Entry {
	ents, err := l.entries(l.firstIndex(), noLimit)
	if err == nil {
		return ents
	}
	if err == ErrCompacted { // try again if there was a racing compaction
		return l.allEntries()
	}
	// TODO (xiangli): handle error?
	panic(err)
}
```

### to 系列函数，更新log的状态

`commitTo()`更新当前committed的index `l.committed`, log的maybe系列函数会调用这个
`appliedTo()`更新当前applied的index `l.applied`

node的`run()`循环中处理advancec

- 如果有commit的entry，会调用`appliedTo()`这个表示这些entries已经apply
- 每次都会调用`l.stableSnapTo()`标记snapshot已经stable

```go
func (l *raftLog) commitTo(tocommit uint64) {
	// never decrease commit
	if l.committed < tocommit {
		if l.lastIndex() < tocommit {
			l.logger.Panicf("tocommit(%d) is out of range [lastIndex(%d)]. Was the raft log corrupted, truncated, or lost?", tocommit, l.lastIndex())
		}
		l.committed = tocommit
	}
}

func (l *raftLog) appliedTo(i uint64) {
	if i == 0 {
		return
	}
	if l.committed < i || i < l.applied {
		l.logger.Panicf("applied(%d) is out of range [prevApplied(%d), committed(%d)]", i, l.applied, l.committed)
	}
	l.applied = i
}
```

这两函数就是对unstable的简单封装，注意node每次处理advanc都会调用`stableSnapTo()`

```go
func (l *raftLog) stableTo(i, t uint64) { l.unstable.stableTo(i, t) }

func (l *raftLog) stableSnapTo(i uint64) { l.unstable.stableSnapTo(i) }
```

### maybeAppend()


参数
- `commited` 当前已确认commited的index？
- `ents .. pb.Entry`其实是个可变参数，突然想起来这个地方为啥不直接传一个slice?

首先`l.matchTerm(index, logTerm)`是否符合，如果不符合就直接返回(0, false)，说明log没法append 
> 这是raft在处理appendEntry时调用的
> 我猜测index和logTerm是AppendEntries RPC中用于log match的`prebLogIndex`和`prevLogterm`，表示要append的entry在这个index和logTerm之后

- lastnewi 是最后一条append的entry
- `ci := l.findConflict(ents)`找出第一条冲突的entry，这个函数循环用ents的index和term调用l.matchTerm(ne.Index, ne.Term)，一旦不符合就返回，有三种情况
  - 没有冲突，要append的ents在raftLog里面都有了，返回0
  - 没有冲突，但是ents包含了新的log entry，返回这条新的
  - 有冲突，满足`if ne.Index <= l.lastIndex()`这个条件
    - 打印一下冲突的位置，`zeroTermOnErrCompacted()`用来查询现有index对应的term，并且在entry已经在被compact的情况下返回term 0
    - 返回冲突的index
- 后续的三种情况
  - `ci == 0`: 说明要append的ents在raftLog里面都有了，啥都不用干
  - `ci <= l.committed` 冲突log index小于commited，这肯定是append有问题
  - `defualt`：其实有两种情况，1.真的有冲突，返回冲突log的index；2,并没有冲突，ents里面的都是新的log；不管是那一种情况，调用`append()`用冲突的log`ents[ci - offset:]`覆盖现有的
    - `append()`做一些边界检查后，调用`l.unstable.truncateAndAppend(ents)`追加到unstable中（可能会覆盖一部分）
  - 最后调用`l.commitTo()`把commited更新为`min(committed, lastnewi)`，min的目的大概是有可能lastnewi 当前小于 committed，也就是log entry太多一次没发完

```go
// maybeAppend returns (0, false) if the entries cannot be appended. Otherwise,
// it returns (last index of new entries, true).
func (l *raftLog) maybeAppend(index, logTerm, committed uint64, ents ...pb.Entry) (lastnewi uint64, ok bool) {
	if l.matchTerm(index, logTerm) {
		lastnewi = index + uint64(len(ents))
		ci := l.findConflict(ents)
		switch {
		case ci == 0:
		case ci <= l.committed:
			l.logger.Panicf("entry %d conflict with committed entry [committed(%d)]", ci, l.committed)
		default:
			offset := index + 1
			l.append(ents[ci-offset:]...)
		}
		l.commitTo(min(committed, lastnewi))
		return lastnewi, true
	}
	return 0, false
}

func (l *raftLog) append(ents ...pb.Entry) uint64 {
	if len(ents) == 0 {
		return l.lastIndex()
	}
	if after := ents[0].Index - 1; after < l.committed {
		l.logger.Panicf("after(%d) is out of range [committed(%d)]", after, l.committed)
	}
	l.unstable.truncateAndAppend(ents)
	return l.lastIndex()
}

// findConflict finds the index of the conflict.
// It returns the first pair of conflicting entries between the existing
// entries and the given entries, if there are any.
// If there is no conflicting entries, and the existing entries contains
// all the given entries, zero will be returned.
// If there is no conflicting entries, but the given entries contains new
// entries, the index of the first new entry will be returned.
// An entry is considered to be conflicting if it has the same index but
// a different term.
// The first entry MUST have an index equal to the argument 'from'.
// The index of the given entries MUST be continuously increasing.
func (l *raftLog) findConflict(ents []pb.Entry) uint64 {
	for _, ne := range ents {
		if !l.matchTerm(ne.Index, ne.Term) {
			if ne.Index <= l.lastIndex() {
				l.logger.Infof("found conflict at index %d [existing term: %d, conflicting term: %d]",
					ne.Index, l.zeroTermOnErrCompacted(l.term(ne.Index)), ne.Term)
			}
			return ne.Index
		}
	}
	return 0
}

func (l *raftLog) zeroTermOnErrCompacted(t uint64, err error) uint64 {
	if err == nil {
		return t
	}
	if err == ErrCompacted {
		return 0
	}
	l.logger.Panicf("unexpected error (%v)", err)
	return 0
}
```

### maybeCommit

这个函数尝试commit （maxIndex，term）对应的log

```go
func (l *raftLog) maybeCommit(maxIndex, term uint64) bool {
	if maxIndex > l.committed && l.zeroTermOnErrCompacted(l.term(maxIndex)) == term {
		l.commitTo(maxIndex)
		return true
	}
	return false
}
```

### snapshot 和 restore

类似的，如果unstable没有snapshot，就从storage里面去找

```go
func (l *raftLog) snapshot() (pb.Snapshot, error) {
	if l.unstable.snapshot != nil {
		return *l.unstable.snapshot, nil
	}
	return l.storage.Snapshot()
}

func (l *raftLog) restore(s pb.Snapshot) {
	l.logger.Infof("log [%s] starts to restore snapshot [index: %d, term: %d]", l, s.Metadata.Index, s.Metadata.Term)
	l.committed = s.Metadata.Index
	l.unstable.restore(s)
}
```

## Progress

Progress表示每一个peer或者learner当前的进度(从leader的角度),设计请参考[design.md](https://github.com/etcd-io/etcd/blob/master/raft/design.md)


### Progress 数据结构


- Match/Next: 对应raft 论文Figure2 中`Volatile state on leaders`中的`nextIndex[]`和`matchIndex[]`
> 为什么要维护match 和 next 两个变量呢？ 为了batch append，只维护一个的话就要发一个等一个

- State 是下面三者之一
```go
const (
	ProgressStateProbe ProgressStateType = iota
	ProgressStateReplicate
	ProgressStateSnapshot
)
```

- ins *inflights: 见下一节

```go
// Progress represents a follower’s progress in the view of the leader. Leader maintains
// progresses of all followers, and sends entries to the follower based on its progress.
type Progress struct {
	Match, Next uint64
	// State defines how the leader should interact with the follower.
	//
	// When in ProgressStateProbe, leader sends at most one replication message
	// per heartbeat interval. It also probes actual progress of the follower.
	//
	// When in ProgressStateReplicate, leader optimistically increases next
	// to the latest entry sent after sending replication message. This is
	// an optimized state for fast replicating log entries to the follower.
	//
	// When in ProgressStateSnapshot, leader should have sent out snapshot
	// before and stops sending any replication message.
	State ProgressStateType

	// Paused is used in ProgressStateProbe.
	// When Paused is true, raft should pause sending replication message to this peer.
	Paused bool
	// PendingSnapshot is used in ProgressStateSnapshot.
	// If there is a pending snapshot, the pendingSnapshot will be set to the
	// index of the snapshot. If pendingSnapshot is set, the replication process of
	// this Progress will be paused. raft will not resend snapshot until the pending one
	// is reported to be failed.
	PendingSnapshot uint64

	// RecentActive is true if the progress is recently active. Receiving any messages
	// from the corresponding follower indicates the progress is active.
	// RecentActive can be reset to false after an election timeout.
	RecentActive bool

	// inflights is a sliding window for the inflight messages.
	// Each inflight message contains one or more log entries.
	// The max number of entries per message is defined in raft config as MaxSizePerMsg.
	// Thus inflight effectively limits both the number of inflight messages
	// and the bandwidth each Progress can use.
	// When inflights is full, no more message should be sent.
	// When a leader sends out a message, the index of the last
	// entry should be added to inflights. The index MUST be added
	// into inflights in order.
	// When a leader receives a reply, the previous inflights should
	// be freed by calling inflights.freeTo with the index of the last
	// received entry.
	ins *inflights

	// IsLearner is true if this progress is tracked for a learner.
	IsLearner bool
}
```

### inflights

inflights 在Progress的注释中有很详细的说明，它是一个滑动窗口，两个限制

- 个数限制：配置中的`MaxInflightMsgs`,最多同时发多少消息
- 大小限制：配置中的`MaxSizePerMsg`，一条消息最大能有多大

所以窗口中的消息最多是 `MaxInflightMsgs` * `MaxSizePerMsg`

inflights本质是一个数组实现的Queue (FIFO)，它的复杂的地方在于`buffer`是动态分配的，按照指数形式增长，但是最终不会超过`in.size` (n年前我好像也写过类似的玩意，处理起来头大）

- size: 就是消息个数的限制，`raft.go`中创建`Progress`时`newInflights()`创建新的对象，size参数传入`r.maxInflight`
- start: 队列头，初始化为0
- count: 当前queue里面放了几个元素（有几个inflight的消息），根据start 和 count可以算出 队列尾的位置(next)
- buffer: 动态分配的queue，每个元素存的是发送的一批entries里面最后一条的index


```go
type inflights struct {
	// the starting index in the buffer
	start int
	// number of inflights in the buffer
	count int

	// the size of the buffer
	size int

	// buffer contains the index of the last entry
	// inside one message.
	buffer []uint64
}

func newInflights(size int) *inflights {
	return &inflights{
		size: size,
	}
}
```

#### 检查和重置

`full()`检查窗口是不是满了

```go
// full returns true if the inflights is full.
func (in *inflights) full() bool {
	return in.count == in.size
}

// resets frees all inflights.
func (in *inflights) reset() {
	in.count = 0
	in.start = 0
}
```

#### 添加

TODO 为啥满了之后添加是直接panic呢？除非caller有相应的防御机制，看到有一些，但是似乎不是所有rpc都做了这个检查

添加复杂的地方在于
```go
// add adds an inflight into inflights
func (in *inflights) add(inflight uint64) {
	if in.full() {
		panic("cannot add into a full inflights")
	}
	next := in.start + in.count
	size := in.size
	if next >= size {
		next -= size
	}
	if next >= len(in.buffer) {
		in.growBuf()
	}
	in.buffer[next] = inflight
	in.count++
}

// grow the inflight buffer by doubling up to inflights.size. We grow on demand
// instead of preallocating to inflights.size to handle systems which have
// thousands of Raft groups per process.
func (in *inflights) growBuf() {
	newSize := len(in.buffer) * 2
	if newSize == 0 {
		newSize = 1
	} else if newSize > in.size {
		newSize = in.size
	}
	newBuffer := make([]uint64, newSize)
	copy(newBuffer, in.buffer)
	in.buffer = newBuffer
}
```

#### 删除

这函数细节我没看太懂（rotate那块），不过大意是从队列头部释放一批slot，直到to为止

```go
// freeTo frees the inflights smaller or equal to the given `to` flight.
func (in *inflights) freeTo(to uint64) {
	if in.count == 0 || to < in.buffer[in.start] {
		// out of the left side of the window
		return
	}

	idx := in.start
	var i int
	for i = 0; i < in.count; i++ {
		if to < in.buffer[idx] { // found the first large inflight
			break
		}

		// increase index and maybe rotate
		size := in.size
		if idx++; idx >= size {
			idx -= size
		}
	}
	// free i inflights and set new start index
	in.count -= i
	in.start = idx
	if in.count == 0 {
		// inflights is empty, reset the start index so that we don't grow the
		// buffer unnecessarily.
		in.start = 0
	}
}

func (in *inflights) freeFirstOne() { in.freeTo(in.buffer[in.start]) }
```

### 状态转换

design.md 中有一段，没看懂太懂

> A progress changes to replicate when the follower replies with a non-rejection msgAppResp, which implies that it has matched the index sent. At this point, leader starts to stream log entries to the follower fast. The progress will fall back to probe when the follower replies a rejection msgAppResp or the link layer reports the follower is unreachable. We aggressively reset next to match+1 since if we receive any msgAppResp soon, both match and next will increase directly to the index in msgAppResp. (We might end up with sending some duplicate entries when aggressively reset next too low. see open question)

```go
func (pr *Progress) resetState(state ProgressStateType) {
	pr.Paused = false
	pr.PendingSnapshot = 0
	pr.State = state
	pr.ins.reset()
}

func (pr *Progress) becomeProbe() {
	// If the original state is ProgressStateSnapshot, progress knows that
	// the pending snapshot has been sent to this peer successfully, then
	// probes from pendingSnapshot + 1.
	if pr.State == ProgressStateSnapshot {
		pendingSnapshot := pr.PendingSnapshot
		pr.resetState(ProgressStateProbe)
		pr.Next = max(pr.Match+1, pendingSnapshot+1)
	} else {
		pr.resetState(ProgressStateProbe)
		pr.Next = pr.Match + 1
	}
}

func (pr *Progress) becomeReplicate() {
	pr.resetState(ProgressStateReplicate)
	pr.Next = pr.Match + 1
}

func (pr *Progress) becomeSnapshot(snapshoti uint64) {
	pr.resetState(ProgressStateSnapshot)
	pr.PendingSnapshot = snapshoti
}
```


### replicate state

```go
func (pr *Progress) pause()  { pr.Paused = true }
func (pr *Progress) resume() { pr.Paused = false }

// IsPaused returns whether sending log entries to this node has been
// paused. A node may be paused because it has rejected recent
// MsgApps, is currently waiting for a snapshot, or has reached the
// MaxInflightMsgs limit.
func (pr *Progress) IsPaused() bool {
	switch pr.State {
	case ProgressStateProbe:
		return pr.Paused
	case ProgressStateReplicate:
		return pr.ins.full()
	case ProgressStateSnapshot:
		return true
	default:
		panic("unexpected state")
	}
}
```

### 更新Match和Next

`maybeUpdated()`在AE rpc成功之后更新Match和Next
`optimisticUpdzte()`啥都不检查，直接更新Next

> 0 <= match < next <= lastindex

```go
// maybeUpdate returns false if the given n index comes from an outdated message.
// Otherwise it updates the progress and returns true.
func (pr *Progress) maybeUpdate(n uint64) bool {
	var updated bool
	if pr.Match < n {
		pr.Match = n
		updated = true
		pr.resume()
	}
	if pr.Next < n+1 {
		pr.Next = n + 1
	}
	return updated
}

func (pr *Progress) optimisticUpdate(n uint64) { pr.Next = n + 1 }
```

`maybeDecrTo()`在AE rpc失败之后，更新Next
- 有几处返回false的地方是因为这个Reject是过时的，reject的index在match之前了(比如重传)
- raft 调用这个函数时last传入的参数是`m.RejectHint`，follower知道的第一条confilt的log index

```go
// maybeDecrTo returns false if the given to index comes from an out of order message.
// Otherwise it decreases the progress next index to min(rejected, last) and returns true.
func (pr *Progress) maybeDecrTo(rejected, last uint64) bool {
	if pr.State == ProgressStateReplicate {
		// the rejection must be stale if the progress has matched and "rejected"
		// is smaller than "match".
		if rejected <= pr.Match {
			return false
		}
		// directly decrease next to match + 1
		pr.Next = pr.Match + 1
		return true
	}

	// the rejection must be stale if "rejected" does not match next - 1
	if pr.Next-1 != rejected {
		return false
	}

	if pr.Next = min(rejected, last+1); pr.Next < 1 {
		pr.Next = 1
	}
	pr.resume()
	return true
}
```

###  snapshot 相关

```go
func (pr *Progress) snapshotFailure() { pr.PendingSnapshot = 0 }

// needSnapshotAbort returns true if snapshot progress's Match
// is equal or higher than the pendingSnapshot.
func (pr *Progress) needSnapshotAbort() bool {
	return pr.State == ProgressStateSnapshot && pr.Match >= pr.PendingSnapshot
}
```

### RecentActive 

RecentActive 是checkquorum启用时leader用这个状态检查自己是不是合法的leader
- 当leader收到`pb.MsgAppResp`,`pb.MsgHeartBeatResp`时置true
- 添加节点时置true
- leader调用`checkQuorumActive()`检查时重置为false

## readOnly

raft thesis 6.4 讨论的只读查询，不需要append log。

为什么readonly 要单独拎出来说？ 因为写请求总是要写log征求多数派的同意，因此写请求在成功之后总是最新的。
而只读请求本质上不需要写log，但是如果不征求多数派的意见，有可能leader返回的数据是旧的，比如leader跟多数派peer存在网络分区，已经有个新的leader选出来了。

> 为了提高效率，可以积累若干readonly 请求再回复，但是考虑这个场景：
index 100: client1 read x
index 101: client2 write x+=1 
		   client1 read x
		   
		   
apply index到了100的时候就要回复client1,
apply index到了101的时候要再次回复client1,并且是不同的值

### readOnly数据结构

readonly使用string类型来识别它的元素-同时使用了map和slice来索引，
- readIndexQueue: slice确保key的顺序
- pendingReadIndex: map用来索引和实际存储元素

readIndexStatus包含三个成员
- req  : 原始的请求消息
- index：收到readonly请求时raft的commit index
- acks ：记录从peers受到的回复

```go
type readIndexStatus struct {
	req   pb.Message
	index uint64
	acks  map[uint64]struct{}
}

type readOnly struct {
	option           ReadOnlyOption
	pendingReadIndex map[string]*readIndexStatus
	readIndexQueue   []string
}

func newReadOnly(option ReadOnlyOption) *readOnly {
	return &readOnly{
		option:           option,
		pendingReadIndex: make(map[string]*readIndexStatus),
	}
}
```

其中`ReadOnlyOption`在raft.go中定义 -- 读请求是否要先征求多数派，可选项包括

- `ReadOnlySafe` 通过征求多数派满足线性一致性
- `ReadOnlyLeaseBased` 通过lease机制返回数据，我猜满足顺序一致性，但不满足线性一致性。

它们对应etcd raft feature所说的
```
    Efficient linearizable read-only queries served by both the leader and followers
        leader checks with quorum and bypasses Raft log before processing read-only queries
        followers asks leader to get a safe read index before processing read-only queries
    More efficient lease-based linearizable read-only queries served by both the leader and followers
        leader bypasses Raft log and processing read-only queries locally
        followers asks leader to get a safe read index before processing read-only queries
        this approach relies on the clock of the all the machines in raft group
```

```go
type ReadOnlyOption int

const (
	// ReadOnlySafe guarantees the linearizability of the read only request by
	// communicating with the quorum. It is the default and suggested option.
	ReadOnlySafe ReadOnlyOption = iota
	// ReadOnlyLeaseBased ensures linearizability of the read only request by
	// relying on the leader lease. It can be affected by clock drift.
	// If the clock drift is unbounded, leader might keep the lease longer than it
	// should (clock can move backward/pause without any bound). ReadIndex is not safe
	// in that case.
	ReadOnlyLeaseBased
)
```

### ReadOnly方法

添加readonly，message第一条entry的data被当作index来使用

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

在收到心跳的回复之后，recvAck在收到回复之后往acks对应的peer里面插入一个空的struct，并返回新的长度
// TODO这里的key 怎么变成m.Context了？

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

Advance 让readonly推进到匹配的context，

- rss保存匹配的（以及之前的）readIndexStatus
- 当匹配的时候，把rss中的内容从readonly中删除

```go
// advance advances the read only request queue kept by the readonly struct.
// It dequeues the requests until it finds the read only request that has
// the same context as the given `m`.
func (ro *readOnly) advance(m pb.Message) []*readIndexStatus {
	var (
		i     int
		found bool
	)

	ctx := string(m.Context)
	rss := []*readIndexStatus{}

	for _, okctx := range ro.readIndexQueue {
		i++
		rs, ok := ro.pendingReadIndex[okctx]
		if !ok {
			panic("cannot find corresponding read state from pending map")
		}
		rss = append(rss, rs)
		if okctx == ctx {
			found = true
			break
		}
	}

	if found {
		ro.readIndexQueue = ro.readIndexQueue[i:]
		for _, rs := range rss {
			delete(ro.pendingReadIndex, string(rs.req.Entries[0].Data))
		}
		return rss
	}

	return nil
}
```

返回最后一条readonly请求的key，如果没有readonly Request，len == 0

```go
// lastPendingRequestCtx returns the context of the last pending read only
// request in readonly struct.
func (ro *readOnly) lastPendingRequestCtx() string {
	if len(ro.readIndexQueue) == 0 {
		return ""
	}
	return ro.readIndexQueue[len(ro.readIndexQueue)-1]
}
```

### readState

TODO 感觉是readonly的context

```go
// ReadState provides state for read only query.
// It's caller's responsibility to call ReadIndex first before getting
// this state from ready, it's also caller's duty to differentiate if this
// state is what it requests through RequestCtx, eg. given a unique id as
// RequestCtx
type ReadState struct {
	Index      uint64
	RequestCtx []byte
}
```