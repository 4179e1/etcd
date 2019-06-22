# Etcd raftexample概览

## Etcd raftexample 简介

Etcd raft是目前使用最广泛的raft实现，在etcd项目中包含了一个[raftexample](https://github.com/etcd-io/etcd/tree/master/contrib/raftexample)，来展示怎么使用etcd raft。

这个示例项目基于etcd raft实现了一个分布式KV存储。如上文所言，etcd raft只实现了核心的raft协议，其余的快照、WAL、消息序列化、网络传输等部分都直接使用了etcd 的实现，导致这个example特别不好读。raftexample 包含了三个组件：基于Raft的key value store，一个REST API Server，以及基于etcd raft实现的Raft Consensus Server。后文将逐一介绍这三个组件。

## Bug Fix

首先需要说明的是，官网当前（as of 2018/12/30）的实现，在处理对快照的处理方面是有严重bug的，具体有两点：

- 如果server启动时存在快照的话，server会读取这个快照，然后卡在一个循环里面不返回，阻塞了后续REST API Server的启动，导致无法响应客户端请求。
- 假使修复上面的bug，该实现还存在一个逻辑错误：它先回放WAL日志，然后再加载快照，跟正确的行为刚好相反。

Github上有人针对这些问题提过issue：

- [raftexample may never serve http request after loading snapshots](https://github.com/etcd-io/etcd/issues/10118)
- [raftexample: restore or overwrite kvstore after replaying has done?](https://github.com/etcd-io/etcd/issues/9263)

我原想着fix这几个问题，但是苦于很多细节自己都没弄明白，然后发现有人提了一个[patch contrib/raftexample: fix backend storage loading from a snapshot](https://github.com/etcd-io/etcd/pull/9918) ，一看作者是Yandex的毛子，感觉靠谱，把补丁拿下来打上再运行，符合预期，perfect。因此后文的分析都基于这个打过patch的版本，地址在[这里](https://github.com/4179e1/etcd/tree/19e567a1362d1c97749ea2b568040ac57c39fc29/contrib/raftexample])。

> 冷知识 – 如何在git上下载一个patch文件？在Pull Request的链接上加上.patch，如上文的 [https://github.com/etcd-io/etcd/pull/9918.patch]

因此raftexample的细节上未必经得起推敲，所以遇到疑惑的地方，不妨大胆的怀疑吧。看看这个怪异的实现，同样函数同样的参数执行两次，一次直接返回，另一次作为goroutine一直在后台接受请求。

```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *string, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	// replay log into key-value map
	s.readCommits(commitC, errorC)
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
}
```

打完补丁后就顺眼很多了
```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *string, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
}
```

## Raftexample 功能

- Key Value的增加/修改（HTTP PUT），查找（HTTP GET），对了，没有删除
- 支持单节点和多节点
- 支持节点的动态配置，增加一个节点（HTTP ADD），删除一个节点（HTTP DELETE）
- 集群容错，在对多故障N/2 -1 个节点的情况下能正常提供对外服务

具体使用请参考[官方手册](https://github.com/etcd-io/etcd/tree/master/contrib/raftexample)

下面来逐一介绍Raftexample的三个组件

## Key Value Store

这个Key Value Store 本质上是一个Key Value的map，其中保存了所有已经commit的KV，它衔接了REST Server 和 raft server。(从REST server来的）请求Key Value update请求会经过这个store，然后发送到raft server。当raft server报告一个KV已经提交的时候，它更新自己的map。

数据结构很简单, 核心就是kvstore，kv用于序列化/反序列化

```go
// a key-value store backed by raft
type kvstore struct {
	proposeC    chan<- string // channel for proposing updates
	mu          sync.RWMutex
	kvStore     map[string]string // current committed key-value pairs
	snapshotter *snap.Snapshotter // 快照相关的对象
}

type kv struct {
	Key string
	Val string
}
```

newKVStore 返回一个新的kvsotre，需要传入四个参数

- snapshotter复杂保存和加载快照， 后续文章会详细介绍这个对象
- procposeC 用来提交请求，后续文章会详细描述其数据流向
- commitC 用来接受（来自raft server的)committed的请求，后续文章会详细描述其数据流向
- errorC 当收到错误时退出

```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *string, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
	// read commits from raft into kvStore map until error
	go s.readCommits(commitC, errorC)
	return s
}
```

kvstore的核心循环时readCommits，它一直读取commitC的内容并处理，可能读到的值有两种

- 一种是nil，表示WAL已经读取完（还没回放），指示加载快照
- 二是实际的KV键值对，用于更新map

这个函数是这样调用的

- 启动时当WAL读取完之后，raft server会写往commitC 写入nil，kvstore尝试寻找有无可用的快照，有的话从快照恢复map 数据。如果我没有理解错的话，raft Server往commitC写入的第一条数据就应该是nil。
- raft server继续王commitC写入WAL中需要回放的数据，kvstore按照同样的顺序读取和更新map
- 后续来自客户端的请求会经由REST API发到kvstore proposeC，当raft server达成共识提交后继续写入commitC ,当然raft Server在回复共识协议之前会先写WAL

```go
func (s *kvstore) readCommits(commitC <-chan *string, errorC <-chan error) {
	for data := range commitC {
		if data == nil {
			// done replaying log; new data incoming
			// OR signaled to load snapshot
			snapshot, err := s.snapshotter.Load() // 这里会找到*最后*一个快照，如果有的话
			if err == snap.ErrNoSnapshot {
				continue
			}
			if err != nil {
				log.Panic(err)
			}
            log.Printf("loading snapshot at term %d and index %d", snapshot.Metadata.Term, snapshot.Metadata.Index)
            // 从快照恢复
			if err := s.recoverFromSnapshot(snapshot.Data); err != nil {
				log.Panic(err)
			}
			continue
		}

        // 如果不是nil，则收到一个KV键值对的更新，反序列化后直接更新到map中
		var dataKv kv
		dec := gob.NewDecoder(bytes.NewBufferString(*data))
		if err := dec.Decode(&dataKv); err != nil {
		}
		s.mu.Lock()
		s.kvStore[dataKv.Key] = dataKv.Val
		s.mu.Unlock()
	}
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```

Key Value Store快照数据的创建和恢复非常简单，直接使用JSON序列化/反序列化
```go
// 创建快照数据， TODO; 如何持久化
func (s *kvstore) getSnapshot() ([]byte, error) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return json.Marshal(s.kvStore)
}

func (s *kvstore) recoverFromSnapshot(snapshot []byte) error {
	var store map[string]string
	if err := json.Unmarshal(snapshot, &store); err != nil {
		return err
	}
	s.mu.Lock()
	s.kvStore = store
	s.mu.Unlock()
	return nil
}
```

Key Value的查找和更新同样简单。
> TODO 对于Propse()函数，注意它没有返回值，所以如果客户端如何确定提交是成功还是失败？

```go
func (s *kvstore) Lookup(key string) (string, bool) {
	s.mu.RLock()
	v, ok := s.kvStore[key]
	s.mu.RUnlock()
	return v, ok
}

// gob 序列化后提交到proposeC
func (s *kvstore) Propose(k string, v string) {
	var buf bytes.Buffer
	if err := gob.NewEncoder(&buf).Encode(kv{k, v}); err != nil {
		log.Fatal(err)
	}
	s.proposeC <- buf.String()
}
```


## REST Server

REST Server是对外的接口：

 - GET请求返回Key Value Store中的键值
 - PUT请求新建/更新KV键值对
 - POST请求添加一个节点
 - DELETE请求删除

REST Server的httpKVAPI类型实现了Handler接口（见后文），并使用这个Handler创建了一个httpServer,当客户端请求进来的时候，ListenAndServer()会调用Handler的ServeHTTP()，根据请求的类型执行不同的操作。 

```go
// Handler for a http based key-value store backed by raft
type httpKVAPI struct {
	store       *kvstore
	confChangeC chan<- raftpb.ConfChange
}

// serveHttpKVAPI starts a key-value server with a GET/PUT API and listens.
func serveHttpKVAPI(kv *kvstore, port int, confChangeC chan<- raftpb.ConfChange, errorC <-chan error) {
	srv := http.Server{
		Addr: ":" + strconv.Itoa(port),
		Handler: &httpKVAPI{
			store:       kv,
			confChangeC: confChangeC,
		},
	}
	go func() {
		if err := srv.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
	}()

	// exit when raft goes down
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```


Handler接口的定义：

```go
type Handler interface {
        ServeHTTP(ResponseWriter, *Request)
}
```

具体的实现摘录如下，其中[引用的链接](https://www.alexedwards.net/blog/a-recap-of-request-handling)解释了http框架如何工作的：

```go
// PUT : propose a key
// GET : get a key
// POST: config change, add a node
// DELETE: config change, delete a node
// see go http framework here: https://www.alexedwards.net/blog/a-recap-of-request-handling

func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// --> here's the key
	key := r.RequestURI
	switch {
	case r.Method == "PUT":
		v, err := ioutil.ReadAll(r.Body)
		if err != nil {
			log.Printf("Failed to read on PUT (%v)\n", err)
			http.Error(w, "Failed on PUT", http.StatusBadRequest)
			return
		}

		h.store.Propose(key, string(v))

		// Optimistic-- no waiting for ack from raft. Value is not yet
		// committed so a subsequent GET on the key may return old value
		w.WriteHeader(http.StatusNoContent)
	case r.Method == "GET":
// ...cut here
}
```

## main 函数

在进入Raft Server之前，我们先来看看raftexample的main函数，理清这几个组件的关系

```go
func main() {
	cluster := flag.String("cluster", "http://127.0.0.1:9021", "comma separated cluster peers")
	id := flag.Int("id", 1, "node ID")
	kvport := flag.Int("port", 9121, "key-value server port")
	join := flag.Bool("join", false, "join an existing cluster")
	flag.Parse()

	proposeC := make(chan string)
	defer close(proposeC)
	confChangeC := make(chan raftpb.ConfChange)
	defer close(confChangeC)

	// raft provides a commit stream for the proposals from the http api
	var kvs *kvstore
	getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }
	commitC, errorC, snapshotterReady := newRaftNode(*id, strings.Split(*cluster, ","), *join, getSnapshot, proposeC, confChangeC)

	kvs = newKVStore(<-snapshotterReady, proposeC, commitC, errorC)

	// the key-value http handler will propose updates to raft
	serveHttpKVAPI(kvs, *kvport, confChangeC, errorC)
}
```
- 首先解析命令行参数，获取集群的配置信息，像是这样`./raftexample --id 1 --cluster http://127.0.0.1:12379,http://127.0.0.1:22379,http://127.0.0.1:32379 --port 12380`,其中包括
  - 逗号分割的集群节点信息，用于Raft协议内部通讯
  - 本节点id
  - 本节点对外监听的端口，用来服务客户端请求
  - join 参数用来加入一个已经存在的集群
- 创建两个proposeC和concChangeC两个管道
  - proposeC用来提交客户端请求
  - confChangeC用来添加/删除节点
- `var kvs *kvstore`为kvstore的指针分配空间
  - 注意这时候kvstore还没有创建，只是分配了一个指针的地址
  - 有了指针之后，使用这个指针创建一个闭包`getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }`，用来获取kvsotre的快照。毫无疑问，如果此时就调用getSnapshot的话会panic的，等kvstore初始化后应该就ok了。还能这么玩？
- `newRaftNode()`创建Raft节点
  - 传入的若干参数包括
    - 从命令行解析的集群配置参数
    - `getSnapshot`闭包，注意kvstore目前仍未初始化
    - procposeC和confChangeC
  - 返回三个管道
    - commitC  用来发布committed的值的管道，让kvstore应用到状态机中
    - errorC  发布错误的管道，让kvsotre退出机制
    - snapshotReady  这个管道用来获取 *snap.Snapshotter对象，这里返回管道而不是对象的原因是，`newRaftNode()`返回的时候还没有初始化这个对象
  - 该函数内部后台会启动若干协程继续完成剩余的工作
- `newKVStore()`正式初始化kvstore，4个参数
  - 第一个参数是`<-snapshotterReady`，看仔细点,这不是一个管道，而是从管道里面读一个值，也就是等待`newRaftNode()`的协程写入snapshot对象之后，再读出来
  - proposeC是main函数之前创建的
  - commitC 和 errorC 是`newRaftNode()`返回的
- 最后`serveHtpKVAPI`启动对客户端的接口，见上文

> 顺便谈谈函数形式参数和实际参数中管端 <-符号的区别
> 
> - 形式参数，也就是定义一个函数的时候，管道中的<-表示数据的方向，仅允许读取或写入，见<https://golang.google.cn/ref/spec#Channel_types> 
>   - `func func1 (ch <-chan int){...}`中ch仅用于读取
>   - `func func2 (ch chan<- int){...}`中ch仅用于读取
> - 实际参数，也就是调用的时候，可能没必要拿出来说，只是`func3 (<- ch)`这种写法很让人迷惑,它等同于`i := <- ch; func3(i)`



## Raft Server 

Raft server是这里的一致性模块，当REST Server（经由Key Value Store）提交一个请求的时候，Raft server把这个请求发送到所有peer。当raft 协议达成共识（多数节点commit了这个请求）的时候，raft server 把所有已经commit的请求发布到一个commited channel，Key Value Store通过读取这个channel 更新自己的值。

注意一点，当raft协议返回一个成功的请求时，我们说这个请求是*committed*，当Key Value Store把这个请求写入自己的map时，我们说这个请求是*applied*

### 数据结构

先来看看它的数据结构，几个比较重要的字段在注释中标注出

```go
// A key-value stream backed by raft
type raftNode struct {
	proposeC    <-chan string            // proposed messages (k,v)
	confChangeC <-chan raftpb.ConfChange // proposed cluster config changes
	commitC     chan<- *string           // entries committed to log (k,v)
	errorC      chan<- error             // errors from raft session

	id          int      // client ID for raft session
	peers       []string // raft peer URLs
	join        bool     // node is joining an existing cluster
	waldir      string   // path to WAL directory
	snapdir     string   // path to snapshot directory
	getSnapshot func() ([]byte, error)

	confState     raftpb.ConfState
	snapshotIndex uint64
	appliedIndex  uint64

	// raft backing for the commit/error channel
	node        raft.Node             //  代表Raft集群的一个节点
	raftStorage *raft.MemoryStorage   //  etcd Raft 自带的存储接口的一个实现
	wal         *wal.WAL              //  etcd 的 Write Ahead Log 

	snapshotter      *snap.Snapshotter
	snapshotterReady chan *snap.Snapshotter // signals when snapshotter is ready

	snapCount uint64
	transport *rafthttp.Transport
	stopc     chan struct{} // signals proposal channel closed
	httpstopc chan struct{} // signals http server to shutdown
	httpdonec chan struct{} // signals http server shutdown complete
}
```

### 初始化

`newRaftNode()`我们已经在main函数中见过，这里

- 创建了作为返回值的三个管道，commitC, errorC，以及snapshotterReady
- 初始化了大部分raftNode的成员
- 但是核心的raft.Node, raft.MemoryStorage以及Snapshot，WAL对象都还没初始化，正如注释所说的`rest of structure populated after WAL replay`,在`rc.startRaft()`这个协程完成

```go
// newRaftNode initiates a raft instance and returns a committed log entry
// channel and error channel. Proposals for log updates are sent over the
// provided the proposal channel. All log entries are replayed over the
// commit channel, followed by a nil message (to indicate the channel is
// current), then new log entries. To shutdown, close proposeC and read errorC.
func newRaftNode(id int, peers []string, join bool, getSnapshot func() ([]byte, error), proposeC <-chan string,
	confChangeC <-chan raftpb.ConfChange) (<-chan *string, <-chan error, <-chan *snap.Snapshotter) {

	commitC := make(chan *string)
	errorC := make(chan error)

	rc := &raftNode{
		proposeC:    proposeC,
		confChangeC: confChangeC,
		commitC:     commitC,
		errorC:      errorC,
		id:          id,
		peers:       peers,
		join:        join,
		waldir:      fmt.Sprintf("raftexample-%d", id),
		snapdir:     fmt.Sprintf("raftexample-%d-snap", id),
		getSnapshot: getSnapshot,
		snapCount:   defaultSnapshotCount,
		stopc:       make(chan struct{}),
		httpstopc:   make(chan struct{}),
		httpdonec:   make(chan struct{}),

		snapshotterReady: make(chan *snap.Snapshotter, 1),
		// rest of structure populated after WAL replay
	}
	go rc.startRaft()
	return commitC, errorC, rc.snapshotterReady
}
```

来看看`rc.startRaft()`这个函数，这里我们标上序号，有几个函数需要单独拎出来看：

1. 如果快照目录不存在则先创建，然后通过`snap.New()`创建快照对象，并写入管道让main函数继续执行`newKVStore`
2. `oldwal := wal.Exist(rc.waldir)`判断是第一次启动还是重启，根据这个区别在后面用不同的方式启动RaftNode
3. `rc.replayWAL()`中读取快照和回放WAL,详细见后文
4. 根据启动参数创建集群的初始成员`rpeers` 
5. 创建`raft.Node`的配置参数`c := &raft.Config{...}`，其中有两个跟超时有关的参数，`ElectionTick`和`HeartbeatTick`，前者是后者的10倍，注意它们并没有设置单位。结合后文的`Node.Tick()`中使用了一个定时器，因此这里实际的超时时间可能是定时器超时时间 * 这两个单位。
6. 根据是否首次启动(即第2步的`oldwal`)分为两个分支
    - 如果是首次启动，则传入配置c和初始成员`rpeers`，调用`raft.StartNode (c, peers)`初始化`rc.node`
    - 如果是重启，则只传入配置c，用`raft.RestartNode(c)`初始化`rc.node`，不需要rpeers，因为最新的集群成员信息已经保存在快照的metadata中或者WAL中(通过回放），并且跟原始成员相比可能已经发生了变化
7. `rc.transport =&rafthttp.Transport{...}`,其中`raft`字段是`transport`定义的接口，把`rc`给传进去了，因为我们实现了这个接口。
8. 以及`rc.transport`分别创建和启动tcd的网络传输框架；下面的for循环中`rc.transport.AddPeer()`把所有其他成员添加到传输列表中。
9. `go rc.serverRaft()` 启动raftx协议的监听端口，用于raft协议的内部通讯，见后文
10. `go rc.serverChannels()` raft.go 真正的业务处理，在协程中监听用户请求、配置等命令


```go
func (rc *raftNode) startRaft() {
	if !fileutil.Exist(rc.snapdir) {
		if err := os.Mkdir(rc.snapdir, 0750); err != nil {
			log.Fatalf("raftexample: cannot create dir for snapshot (%v)", err)
		}
	}
	rc.snapshotter = snap.New(zap.NewExample(), rc.snapdir)
	rc.snapshotterReady <- rc.snapshotter

	oldwal := wal.Exist(rc.waldir)
	rc.wal = rc.replayWAL()

	rpeers := make([]raft.Peer, len(rc.peers))
	for i := range rpeers {
		rpeers[i] = raft.Peer{ID: uint64(i + 1)}
	}
	c := &raft.Config{
		ID:                        uint64(rc.id),
		ElectionTick:              10,
		HeartbeatTick:             1,
		Storage:                   rc.raftStorage,
		MaxSizePerMsg:             1024 * 1024,
		MaxInflightMsgs:           256,
		MaxUncommittedEntriesSize: 1 << 30,
	}

	if oldwal {
		rc.node = raft.RestartNode(c)
	} else {
		startPeers := rpeers
		if rc.join {
			startPeers = nil
		}
		rc.node = raft.StartNode(c, startPeers)
	}

	rc.transport = &rafthttp.Transport{
		Logger:      zap.NewExample(),
		ID:          types.ID(rc.id),
		ClusterID:   0x1000,
		Raft:        rc,
		ServerStats: stats.NewServerStats("", ""),
		LeaderStats: stats.NewLeaderStats(strconv.Itoa(rc.id)),
		ErrorC:      make(chan error),
	}

	rc.transport.Start()
	for i := range rc.peers {
		if i+1 != rc.id {
			rc.transport.AddPeer(types.ID(i+1), []string{rc.peers[i]})
		}
	}

	go rc.serveRaft()
	go rc.serveChannels()
}
```

#### 回放快照和WAL

细看刚才第3步中一笔带过的`rc.replayWAL()`

- `snapshot ：= raftNode.loadSnapshot()`找到最后一个快照，如果有的话
- `w := rc.openWAL(snapshot)`根据快照的记录打开WAL，从快照后面的第一条WAL记录开始读取
- `_, st, ents, err := w.ReadAll()`读取WAL的内容，包括HardState（需要持久化的状态数据），以及WAL条目
- `rc.raftStorage = raft.NewMemoryStorage()`，创建raft存储，然后按照etcd raft的要求的三步进行回放
  - `rc.raftStorage.ApplySnapshot(*snapshot)`加载快照的metadata，如果有的话，但是不加载其内容，raft 存储不需要这个
  - `rc.raftStorage.SetHardState(st)` 回放持久化的状态
  - `rc.raftStorage.Append(ents)` 回放WAL的记录
- `rc.commitC <- nil` 通知KVStore的`s.readCommits()`协程去加载快照，注意这是commitC的第一条记录，因此保证了`s.readCommits()`总是先加载快照（这里），然后才回放WAL记录（在后面的`rc.publishEntries()`中)

> WAL是一系列默认大小为64MB的文件，超过这个长度会进行切割，文件名为`<seq>-<index>.wal`，seq是文件的序号，index是raft协议中已经committed的index。
> TODO快照和WAL的具体实现在另外一篇单独的文章中

```go
// replayWAL replays WAL entries into the raft instance.
func (rc *raftNode) replayWAL() *wal.WAL {
	log.Printf("replaying WAL of member %d", rc.id)
	snapshot := rc.loadSnapshot()
	w := rc.openWAL(snapshot)
	_, st, ents, err := w.ReadAll()
	if err != nil {
		log.Fatalf("raftexample: failed to read WAL (%v)", err)
	}
	rc.raftStorage = raft.NewMemoryStorage()
	if snapshot != nil {
		rc.raftStorage.ApplySnapshot(*snapshot)
	}
	rc.raftStorage.SetHardState(st)

	// append to storage so raft starts at the right place in log
	rc.raftStorage.Append(ents)

	// send nil so client knows commit channel is current
	rc.commitC <- nil
	return w
}
```

#### 启动网络监听端口

第8步的`serveRaft()`启动raft 协议的网络监听端口，主要是两部分
- 创建一个`stoppableListener`对象，它其实是内嵌（embed)了一个`*net.TCPListener`，因此所有能直接调用所有[net.TCPListener](https://golang.google.cn/pkg/net/#TCPListener)的方法，另外加上了一个channel让listner在`Accept()`阻塞时能退出来，这就是它叫做stoppable的原因……
- 创建`http.Server`，并使用刚才创建的stoppableListener监听请求

```go
func (rc *raftNode) serveRaft() {
	url, err := url.Parse(rc.peers[rc.id-1])
	if err != nil {
		log.Fatalf("raftexample: Failed parsing URL (%v)", err)
	}

	ln, err := newStoppableListener(url.Host, rc.httpstopc)
	if err != nil {
		log.Fatalf("raftexample: Failed to listen rafthttp (%v)", err)
	}

	err = (&http.Server{Handler: rc.transport.Handler()}).Serve(ln)
	select {
	case <-rc.httpstopc:
	default:
		log.Fatalf("raftexample: Failed to serve rafthttp (%v)", err)
	}
	close(rc.httpdonec)
}
```

下面是stoppableListener的实现，值得注意的几点

- `stoppableListener`实现了自己的`Accept()`方法，覆盖了`net.TCPListener`自带的`Accept()`，这算是golang的override了吧，这种用法[参考这里](https://medium.com/random-go-tips/method-overriding-680cfd51ce40)。
- `Accept()` 中，`ln.AcceptTCP()`运行在单独的一个协程，新建立的客户端链接通过管道传给主函数的connc，在它阻塞的期间可以写入ln.stopc让主函数返回，此时`ln.AcceptTCP()`则没有做任何处理。TODO：这里会存在什么问题吗？ 
- 新建立的链接默认加上了TCP的KeepAlive选项

```go
// stoppableListener sets TCP keep-alive timeouts on accepted
// connections and waits on stopc message
type stoppableListener struct {
	*net.TCPListener
	stopc <-chan struct{}
}

func newStoppableListener(addr string, stopc <-chan struct{}) (*stoppableListener, error) {
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return nil, err
	}
	return &stoppableListener{ln.(*net.TCPListener), stopc}, nil
}

func (ln stoppableListener) Accept() (c net.Conn, err error) {
	connc := make(chan *net.TCPConn, 1)
	errc := make(chan error, 1)
	go func() {
		tc, err := ln.AcceptTCP()
		if err != nil {
			errc <- err
			return
		}
		connc <- tc
	}()
	select {
	case <-ln.stopc:
		return nil, errors.New("server stopped")
	case err := <-errc:
		return nil, err
	case tc := <-connc:
		tc.SetKeepAlive(true)
		tc.SetKeepAlivePeriod(3 * time.Minute)
		return tc, nil
	}
}
```

`err = (&http.Server{Handler: rc.transport.Handler()}).Serve(ln)`新建了一个`http.Server`对象，其中使用了自定义的Handler`rc.transport.Handler()`，最后在这个`http.Server`对象上监听刚才创建的stoppableListener



### Raft协议处理

第9步正式进行Raft协议处理

- 首先根据快照的metadata更新结构体的`confState`， `snapshotIndex`， `appliedIndex`三个字段
- `ticker := time.NewTicker(100 * time.Millisecond)`创建一个定时器
- 然后是两个核心主循环
  - 第一个主循环在一个单独的协程中，用来处理来自客户端的请求
    - 从proposeC读取客户端的`提交`请求，然后调用`rc.node.Propose(context.TODO(), []byte(prop))`调用raft协议的提交接口，这个调用是阻塞的。
    - 从confChangC读取`配置变更`请求，然后调用`rc.node.ProposeConfChange(context.TODO(), cc)`提交配置修改。
    - TODO： 这两个调用都不返回结果，如果判断请求是否成功？  这里是否会设置context的Error？
    - 如果有任何一个channel被关闭 (在main 函数的 defer里面，serverHttpKVAPI退出后)，对应的channel会被置nil，当两个channel都为nil时退出循环，关闭rc.stopc通知另一个主循环退出。 [nil channel的用法参考](https://lingchao.xin/post/why-are-there-nil-channels-in-go.html)
  - 第二个主循环读取Raft协议的更新， 4个channel对应4个分支
    - 定时器超时的时候调用`rc.node.Tick()`，这是etcd raft实现要求的，估计是用来更新选举和心跳超时时间，用来触发对应的错误处理。
    - `rd := <-rc.node.Ready()`返回一个Ready对象，该分支就是围绕这个对象进行处理,后面再议论
    - `case err := <-rc.transport.ErrorC`接收transport的错误，然后调用`rc.writeError(err)`停止transport和raft.Node，并关闭`rc.httpdonec`(这个channel好像并没有关联任何东西)
    - `case <-rc.stopc`调用`rc.stop()`停止Raft Server
	
```go
func (rc *raftNode) serveChannels() {
	snap, err := rc.raftStorage.Snapshot()
	if err != nil {
		panic(err)
	}
	rc.confState = snap.Metadata.ConfState
	rc.snapshotIndex = snap.Metadata.Index
	rc.appliedIndex = snap.Metadata.Index

	defer rc.wal.Close()

	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	// send proposals over raft
	go func() {
		confChangeCount := uint64(0)

		for rc.proposeC != nil && rc.confChangeC != nil {
			select {
			case prop, ok := <-rc.proposeC:
				if !ok {
					rc.proposeC = nil
				} else {
					// blocks until accepted by raft state machine
					rc.node.Propose(context.TODO(), []byte(prop))
				}

			case cc, ok := <-rc.confChangeC:
				if !ok {
					rc.confChangeC = nil
				} else {
					confChangeCount++
					cc.ID = confChangeCount
					rc.node.ProposeConfChange(context.TODO(), cc)
				}
			}
		}
		// client closed channel; shutdown raft if not already
		close(rc.stopc)
	}()

	// event loop on raft state machine updates
	for {
		select {
		case <-ticker.C:
			rc.node.Tick()

		// store raft entries to wal, then publish over commit channel
		case rd := <-rc.node.Ready():
			rc.wal.Save(rd.HardState, rd.Entries)
			if !raft.IsEmptySnap(rd.Snapshot) {
				rc.saveSnap(rd.Snapshot)
				rc.raftStorage.ApplySnapshot(rd.Snapshot)
				rc.publishSnapshot(rd.Snapshot)
			}
			rc.raftStorage.Append(rd.Entries)
			rc.transport.Send(rd.Messages)
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
			}
			rc.maybeTriggerSnapshot()
			rc.node.Advance()

		case err := <-rc.transport.ErrorC:
			rc.writeError(err)
			return

		case <-rc.stopc:
			rc.stop()
			return
		}
	}
}
```

部分关闭函数如下，`rc.writeError()`和`rc.stop()`都会调用`close(rc.CommitC)`以及`close (rc.errorC)`，前者示意Key Value Store (`readCommits()`中)停止，后者示意 REST server（`serveHttpKVAPI()`中） 停止。

server退出的机制：

- 遇到tranport的错误主动退出：，`rc.writeError()`示意KVStore和REST Server退出，bing退出上文的第二个循环，REST Server退出后main函数的defer机制关闭commitC和proposeC，从而退出第一个循环。
- 第一个循环的commitC和proposeC关闭后，通过rc.stopC停止第二个循环。 //TODO，这是怎么触发的？

```go
func (rc *raftNode) writeError(err error) {
	rc.stopHTTP()
	close(rc.commitC)
	rc.errorC <- err
	close(rc.errorC)
	rc.node.Stop()
}

func (rc *raftNode) stopHTTP() {
	rc.transport.Stop()
	close(rc.httpstopc)
	<-rc.httpdonec
}

// stop closes http, closes all channels, and stops raft.
func (rc *raftNode) stop() {
	rc.stopHTTP()
	close(rc.commitC)
	close(rc.errorC)
	rc.node.Stop()
}
```

最后重点看一下`serveChannel()`第二个循环`case rd := <-rc.node.Ready():`这个分支，`Ready`的数据结构如下:

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

回到`serveChannels()`第二个循环的这个分支：

- 首先`rc.wal.Save(rd.HardState, rd.Entries)`把硬状态和日志写到WAL中
- 如果Ready状态中包含了快照，则直接应用这个快照，一个节点重启后可能缺了很多entry，直接用快照覆盖就完事了
  - 执行`rc.saveSnap()`保存快照，细分为
    - `rc.wal.SaveSnapshot(walSnap)`把快照Metadata的两项(`Metadata.Index`, 和 `Metadata.term`)保存到WAL中,Index和Term用来在重启回放快照和WAL时，确保他们的状态是一致的。这里不保存快照的内容，只是记录一下我在这个时候做了个快照，replay的时候只要找到最后一个快照来加载，回放快照后面的WAL就可以了。
    - `rc.snapshotter.SaveSnap(snap)`把快照实际写入快照文件，文件命名为`<snapshot.Metadata.Term>-<snapshot.Metadata.Index>.snap`
    - `rc.wal.ReleaseLockTo(snap.Metadata.Index)`释放快照Index之前不再使用的WAL文件
  - `rc.raftStorage.ApplySnapshot(rd.Snapshot)`把快照数据应用到Raft 存储中，内部实现中只是把index和term更新为快照对应的值，并丢弃现有的所有log。快照可以认为是已经commit和apply的log，因此之前的东西不需要再保存了。
  - `rc.publishSnapshot(rd.Snapshot)`更新结构体的结构体的`confState`， `snapshotIndex`， `appliedIndex`三个字段，然后往rc.commitC写入通知KV store加载我们刚在`rc.snapshotter.SaveSnap(snap)`保存的快照文件
- `rc.raftStorage.Append(rd.Entries)`把日志应用到raft 存储
- `rc.transport.Send(rd.Messages)`把消息发给peers
- `rc.entriesToApply(rd.CommittedEntries)`找出还没有applied的日志，这里rd.CommittedEntries的index需要小于或等于已经applied index，不然中间就缺失了一些日志，绝对不能应用到状态机里面。而冗余的则通过这个函数去除。把这些还没有applid的日志找出来以后，通过`rc.publishEntries(rc.entriesToApply(rd.CommittedEntries))`发布给kvstore，后面再看这个函数。
- `rc.maybeTriggerSnapshot()`判断是否要作一次本地快照，后面再议。
- `rc.node.Advance()`通知raft我们已经保存了需要的数据，可以开始发下一批数据了。

```go
		case rd := <-rc.node.Ready():
			rc.wal.Save(rd.HardState, rd.Entries)
			if !raft.IsEmptySnap(rd.Snapshot) {
				rc.saveSnap(rd.Snapshot)
				rc.raftStorage.ApplySnapshot(rd.Snapshot)
				rc.publishSnapshot(rd.Snapshot)
			}
			rc.raftStorage.Append(rd.Entries)
			rc.transport.Send(rd.Messages)
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
			}
			rc.maybeTriggerSnapshot()
			rc.node.Advance()
```

`publishEntries`把日志发送给kvstore，注意日志有两种类型：

- 一是`raftpb.EntryNormal` 即客户端提交的，并由raft协议commit的请求。直接通过rc.commitC发给kvstore
- 另一种是`raftpb.EntryConfChange`,即配置（节点）的变更
  - 首先需要`rc.confState = *rc.node.ApplyConfChange(cc)`更新raft node的配置
  - 然后根据是添加还是删除节点分别调用`rc.transport.AddPeer(types.ID(cc.NodeID), []string{string(cc.Context)})`或`rc.transport.RemovePeer(types.ID(cc.NodeID))`
- 发布所有日志之后，`rc.appliedIndex = ents[i].Index`更新applied记录

```go
// publishEntries writes committed log entries to commit channel and returns
// whether all entries could be published.
func (rc *raftNode) publishEntries(ents []raftpb.Entry) bool {
	for i := range ents {
		switch ents[i].Type {
		case raftpb.EntryNormal:
			if len(ents[i].Data) == 0 {
				// ignore empty messages
				break
			}
			s := string(ents[i].Data)
			select {
			case rc.commitC <- &s:
				log.Printf("publishEntries: %v", s)
			case <-rc.stopc:
				return false
			}

		case raftpb.EntryConfChange:
			var cc raftpb.ConfChange
			cc.Unmarshal(ents[i].Data)
			rc.confState = *rc.node.ApplyConfChange(cc)
			switch cc.Type {
			case raftpb.ConfChangeAddNode:
				log.Printf("publishEntries: ConfChangeAddNode")
				if len(cc.Context) > 0 {
					rc.transport.AddPeer(types.ID(cc.NodeID), []string{string(cc.Context)})
				}
			case raftpb.ConfChangeRemoveNode:
				log.Printf("publishEntries: ConfChangeRemoveNode")
				if cc.NodeID == uint64(rc.id) {
					log.Println("I've been removed from the cluster! Shutting down.")
					return false
				}
				rc.transport.RemovePeer(types.ID(cc.NodeID))
			}
		}

		// after commit, update appliedIndex
		rc.appliedIndex = ents[i].Index
	}
	return true
}
```

每次`case rd := <-rc.node.Ready()`的分支都会判断都会看看是否需要在本地创建快照

- 判断的逻辑很简单，看看上一次做快照的日志`rc.snapshotIndex`和最新一条应用的日志`rc.appliedIndex`之间差了多少条日志，如果少于`rc.snapCount`这个常量则跳过
- `data, err := rc.getSnapshot()`拿到应用当前的快照数据
- `snap, err := rc.raftStorage.CreateSnapshot(rc.appliedIndex-1, &rc.confState, data)`这里主要是为了更新storage的快照，同时返回刚更新的快照。这里隐含了把快照的二进制数据封装为`raft snapshot`。不过index为什么是`applidIndex - 1`？
- `rc.saveSnap(snap)`保存这个快照，这个函数前面看过
- 好了，既然快照已经保存了，记再快照中的条目，就可以从raft 存储中`rc.raftStorage.Compact(compactIndex)`一些不再需要的数据 //TODO： compatIndex的计算没看懂.
  - compatcIndex 要么是1，如果rc.appliedIndex <= snapshotCatchUpEntriesN的话，
  - 要么是`compactIndex = rc.appliedIndex - snapshotCatchUpEntriesN`,如果`rc.appliedIndex > snapshotCatchUpEntriesN`的话，目前`snapshotCatchUpEntriesN`跟`rc.snapCount`这个常量的值相同
  - 这里的意思似乎是，至少要在raft 存储中保存有一个快照`rc.snapCount`的数量的日志，也就是说这些日志在快照中有，在raft 内存存储也有，可能是处于可靠性的考虑，如果最新的快照文件出问题了，raft 存储也还有？ 其实理论上把已经在快照中的数据都丢了应该也没问题，这个函数的注释说：
> // Compact discards all log entries prior to compactIndex.
> // It is the application's responsibility to not attempt to compact an index
> // greater than raftLog.applied.

```go
func (rc *raftNode) maybeTriggerSnapshot() {
	if rc.getSnapshot == nil {
		return
	}

	if rc.appliedIndex-rc.snapshotIndex <= rc.snapCount {
		return
	}

	log.Printf("start snapshot [applied index: %d | last snapshot index: %d]", rc.appliedIndex, rc.snapshotIndex)
	data, err := rc.getSnapshot()
	if err != nil {
		log.Panic(err)
	}
	snap, err := rc.raftStorage.CreateSnapshot(rc.appliedIndex-1, &rc.confState, data)
	if err != nil {
		panic(err)
	}
	if err := rc.saveSnap(snap); err != nil {
		panic(err)
	}

	compactIndex := uint64(1)
	if rc.appliedIndex > snapshotCatchUpEntriesN {
		compactIndex = rc.appliedIndex - snapshotCatchUpEntriesN
	}
	if err := rc.raftStorage.Compact(compactIndex); err != nil {
		panic(err)
	}

	log.Printf("compacted log at index %d", compactIndex)
	rc.snapshotIndex = rc.appliedIndex
}
```

### Raft Storage

最后大家可能还有一个疑问，上文中的`rc.raftStorage`（类型为MemoryStorage）到底是干什么的？


其实是这样，`raft.Node`中的`raftlog`中包含了MemoryStorage和unstable两部分：

- MemoryStorage保存的是已经持久化的log；
- unstable顾名思义，就是没持久化的log。
- 回放wal的时候，会把已经持久化的log放到MemoryStroage中，unstable是空的。

- 当propose进来的时候，会先把log entry放到unstable中
- leader广播AppendEntries rpc的时候，每个raft node都会收到这些unstable
  - 应用层通过`node.Ready()`拿到这些unstable entry，通过WAL持久化，然后放到MemoryStorage中
  - `node.Advance()`通知raft把已经持久化的unstable删掉。

- 总之上面的过程完成了log 从 unstable 转移到 MemoryStorage的过程

- 当leader需要获取log entry（比如发给follower）的时候，把MemoryStorage 和 unstable 联合起来做查询.


## Reference

- <https://jin-yang.github.io/post/golang-raft-etcd-example-sourcode-details.html>