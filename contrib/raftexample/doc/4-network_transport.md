# Network Transport

Too Complex, Didn't Write

## Transport



### Tranposrt接口

```go
type Transporter interface {
	// Start starts the given Transporter.
	// Start MUST be called before calling other functions in the interface.
	Start() error
	// Handler returns the HTTP handler of the transporter.
	// A transporter HTTP handler handles the HTTP requests
	// from remote peers.
	// The handler MUST be used to handle RaftPrefix(/raft)
	// endpoint.
	Handler() http.Handler
	// Send sends out the given messages to the remote peers.
	// Each message has a To field, which is an id that maps
	// to an existing peer in the transport.
	// If the id cannot be found in the transport, the message
	// will be ignored.
	Send(m []raftpb.Message)
	// SendSnapshot sends out the given snapshot message to a remote peer.
	// The behavior of SendSnapshot is similar to Send.
	SendSnapshot(m snap.Message)
	// AddRemote adds a remote with given peer urls into the transport.
	// A remote helps newly joined member to catch up the progress of cluster,
	// and will not be used after that.
	// It is the caller's responsibility to ensure the urls are all valid,
	// or it panics.
	AddRemote(id types.ID, urls []string)
	// AddPeer adds a peer with given peer urls into the transport.
	// It is the caller's responsibility to ensure the urls are all valid,
	// or it panics.
	// Peer urls are used to connect to the remote peer.
	AddPeer(id types.ID, urls []string)
	// RemovePeer removes the peer with given id.
	RemovePeer(id types.ID)
	// RemoveAllPeers removes all the existing peers in the transport.
	RemoveAllPeers()
	// UpdatePeer updates the peer urls of the peer with the given id.
	// It is the caller's responsibility to ensure the urls are all valid,
	// or it panics.
	UpdatePeer(id types.ID, urls []string)
	// ActiveSince returns the time that the connection with the peer
	// of the given id becomes active.
	// If the connection is active since peer was added, it returns the adding time.
	// If the connection is currently inactive, it returns zero time.
	ActiveSince(id types.ID) time.Time
	// ActivePeers returns the number of active peers.
	ActivePeers() int
	// Stop closes the connections and stops the transporter.
	Stop()
}

// Transport implements Transporter interface. It provides the functionality
// to send raft messages to peers, and receive raft messages from peers.
// User should call Handler method to get a handler to serve requests
// received from peerURLs.
// User needs to call Start before calling other functions, and call
// Stop when the Transport is no longer used.
type Transport struct {
	Logger *zap.Logger

	DialTimeout time.Duration // maximum duration before timing out dial of the request
	// DialRetryFrequency defines the frequency of streamReader dial retrial attempts;
	// a distinct rate limiter is created per every peer (default value: 10 events/sec)
	DialRetryFrequency rate.Limit

	TLSInfo transport.TLSInfo // TLS information used when creating connection

	ID          types.ID   // local member ID
	URLs        types.URLs // local peer URLs
	ClusterID   types.ID   // raft cluster ID for request validation
	Raft        Raft       // raft state machine, to which the Transport forwards received messages and reports status
	Snapshotter *snap.Snapshotter
	ServerStats *stats.ServerStats // used to record general transportation statistics
	// used to record transportation statistics with followers when
	// performing as leader in raft protocol
	LeaderStats *stats.LeaderStats
	// ErrorC is used to report detected critical errors, e.g.,
	// the member has been permanently removed from the cluster
	// When an error is received from ErrorC, user should stop raft state
	// machine and thus stop the Transport.
	ErrorC chan error

	streamRt   http.RoundTripper // roundTripper used by streams
	pipelineRt http.RoundTripper // roundTripper used by pipelines

	mu      sync.RWMutex         // protect the remote and peer map
	remotes map[types.ID]*remote // remotes map that helps newly joined member to catch up
	peers   map[types.ID]Peer    // peers map

	pipelineProber probing.Prober
	streamProber   probing.Prober
}
```

其中的Raft字段就是raftexample中的raftNode


### Transport handler

这里自定义的Handler是一个`ServeMux`对象——一个路由，它把进来的请求继续分发到不同路径对应的handler去处理。这里的`pipelineHandler`,`streamHandler`,`snapHandler`,`probing.NewHandler()`全都实现了`ServeHTTP()`方法。这一部分留到网络部分再细看。


```go
func (t *Transport) Handler() http.Handler {
	pipelineHandler := newPipelineHandler(t, t.Raft, t.ClusterID)
	streamHandler := newStreamHandler(t, t, t.Raft, t.ID, t.ClusterID)
	snapHandler := newSnapshotHandler(t, t.Raft, t.Snapshotter, t.ClusterID)
	mux := http.NewServeMux()
	mux.Handle(RaftPrefix, pipelineHandler)
	mux.Handle(RaftStreamPrefix+"/", streamHandler)
	mux.Handle(RaftSnapshotPrefix, snapHandler)
	mux.Handle(ProbingPrefix, probing.NewHandler())
	return mux
}
```

把常量替换一下，可以看到实际的路由如下

```go
func (t *Transport) Handler() http.Handler {
	pipelineHandler := newPipelineHandler(t, t.Raft, t.ClusterID)
	streamHandler := newStreamHandler(t, t, t.Raft, t.ID, t.ClusterID)
	snapHandler := newSnapshotHandler(t, t.Raft, t.Snapshotter, t.ClusterID)
	mux := http.NewServeMux()
	mux.Handle("/raft", pipelineHandler)
	mux.Handle("/raft/stream"+"/", streamHandler)
	mux.Handle("/raft/snapshot", snapHandler)
	mux.Handle("/raft/probing", probing.NewHandler())
	return mux
}
```

其中/raft/snapshot应该是用来处理快照的，/raft/probing用来检测对方是否存活

举例看看`probing.NewHandler()`的`ServerHTTP()`方法，就是返回一个OK和时间戳

```go
func (h *httpHealth) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	health := Health{OK: true, Now: time.Now()}
	e := json.NewEncoder(w)
	e.Encode(health)
}
```

### Raft接口
`Transport`中定义了一个`Raft`的接口

```go
type Raft interface {
	Process(ctx context.Context, m raftpb.Message) error
	IsIDRemoved(id uint64) bool
	ReportUnreachable(id uint64)
	ReportSnapshot(id uint64, status raft.SnapshotStatus)
}
```

`raftexample` 同样实现了这几个接口，注意`Process()`这个函数，收到消息后要求`rc.node.Step(ctx, m)`，这是在`startRaft()`中设置的：

```go
	rc.transport = &rafthttp.Transport{
		Logger:      zap.NewExample(),
		ID:          types.ID(rc.id),
		ClusterID:   0x1000,
		Raft:        rc,                 #<==== here
		ServerStats: stats.NewServerStats("", ""),
		LeaderStats: stats.NewLeaderStats(strconv.Itoa(rc.id)),
		ErrorC:      make(chan error),
	}
```

```go
func (rc *raftNode) Process(ctx context.Context, m raftpb.Message) error {
	return rc.node.Step(ctx, m)
}
func (rc *raftNode) IsIDRemoved(id uint64) bool                           { return false }
func (rc *raftNode) ReportUnreachable(id uint64)                          {}
func (rc *raftNode) ReportSnapshot(id uint64, status raft.SnapshotStatus) {}
```

`rc.node.Step()`的实现请看[](./7-etcd_raft_node.md)和[](./9-etcd_raft.md)，这是ecd raft消息处理的入口


### PipelineHandler

这里的r就是raftexample中的raftNode，因为它实现了上面说的Raft接口

```go
// newPipelineHandler returns a handler for handling raft messages
// from pipeline for RaftPrefix.
//
// The handler reads out the raft message from request body,
// and forwards it to the given raft state machine for processing.
func newPipelineHandler(t *Transport, r Raft, cid types.ID) http.Handler {
	return &pipelineHandler{
		lg:      t.Logger,
		localID: t.ID,
		tr:      t,
		r:       r,
		cid:     cid,
	}
}
```

```go
func (h *pipelineHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.Method != "POST" {
		w.Header().Set("Allow", "POST")
		http.Error(w, "Method Not Allowed", http.StatusMethodNotAllowed)
		return
	}

	w.Header().Set("X-Etcd-Cluster-ID", h.cid.String())

	if err := checkClusterCompatibilityFromHeader(h.lg, h.localID, r.Header, h.cid); err != nil {
		http.Error(w, err.Error(), http.StatusPreconditionFailed)
		return
	}

	addRemoteFromRequest(h.tr, r)

	// Limit the data size that could be read from the request body, which ensures that read from
	// connection will not time out accidentally due to possible blocking in underlying implementation.
	limitedr := pioutil.NewLimitedBufferReader(r.Body, connReadLimitByte)
	b, err := ioutil.ReadAll(limitedr)
	if err != nil {
		if h.lg != nil {
			h.lg.Warn(
				"failed to read Raft message",
				zap.String("local-member-id", h.localID.String()),
				zap.Error(err),
			)
		} else {
			plog.Errorf("failed to read raft message (%v)", err)
		}
		http.Error(w, "error reading raft message", http.StatusBadRequest)
		recvFailures.WithLabelValues(r.RemoteAddr).Inc()
		return
	}

	var m raftpb.Message
	if err := m.Unmarshal(b); err != nil {
		if h.lg != nil {
			h.lg.Warn(
				"failed to unmarshal Raft message",
				zap.String("local-member-id", h.localID.String()),
				zap.Error(err),
			)
		} else {
			plog.Errorf("failed to unmarshal raft message (%v)", err)
		}
		http.Error(w, "error unmarshaling raft message", http.StatusBadRequest)
		recvFailures.WithLabelValues(r.RemoteAddr).Inc()
		return
	}

	receivedBytes.WithLabelValues(types.ID(m.From).String()).Add(float64(len(b)))

	if err := h.r.Process(context.TODO(), m); err != nil {
		switch v := err.(type) {
		case writerToResponse:
			v.WriteTo(w)
		default:
			if h.lg != nil {
				h.lg.Warn(
					"failed to process Raft message",
					zap.String("local-member-id", h.localID.String()),
					zap.Error(err),
				)
			} else {
				plog.Warningf("failed to process raft message (%v)", err)
			}
			http.Error(w, "error processing raft message", http.StatusInternalServerError)
			w.(http.Flusher).Flush()
			// disconnect the http stream
			panic(err)
		}
		return
	}

	// Write StatusNoContent header after the message has been processed by
	// raft, which facilitates the client to report MsgSnap status.
	w.WriteHeader(http.StatusNoContent)
}
```

这里最重要的就是`h.r.Process(context.TODO(), m)`，把收到的消息发给raft协议去处理

### 其他Handler

略，应该是要处理的消息以及响应不同，没具体看有什么不同

### 发送消息到多节点

在`exampleraft`的`serveChannels()`中，使用了`rc.transport.Send(rd.Messages)`来处理消息发。Message在raft/raftpb/raft.pb.go中定义：
```go
type Message struct {
	Type             MessageType `protobuf:"varint,1,opt,name=type,enum=raftpb.MessageType" json:"type"`
	To               uint64      `protobuf:"varint,2,opt,name=to" json:"to"`
	From             uint64      `protobuf:"varint,3,opt,name=from" json:"from"`
	Term             uint64      `protobuf:"varint,4,opt,name=term" json:"term"`
	LogTerm          uint64      `protobuf:"varint,5,opt,name=logTerm" json:"logTerm"`
	Index            uint64      `protobuf:"varint,6,opt,name=index" json:"index"`
	Entries          []Entry     `protobuf:"bytes,7,rep,name=entries" json:"entries"`
	Commit           uint64      `protobuf:"varint,8,opt,name=commit" json:"commit"`
	Snapshot         Snapshot    `protobuf:"bytes,9,opt,name=snapshot" json:"snapshot"`
	Reject           bool        `protobuf:"varint,10,opt,name=reject" json:"reject"`
	RejectHint       uint64      `protobuf:"varint,11,opt,name=rejectHint" json:"rejectHint"`
	Context          []byte      `protobuf:"bytes,12,opt,name=context" json:"context,omitempty"`
	XXX_unrecognized []byte      `json:"-"`
}
```

可以看到，这里有个`To`字段指定了目标的id。问题来了，有些消息是要同时发给多个节点的，这里是怎么处理的？
好吧，`rd.Messages`是个切片，需要发给多节点的时候，raft协议层会把同一条消息*复制*多份，每一份通过`To`指定不同的接收端。像是这样：

```
09:27:33 raftexample1 | 2019-01-16 09:27:33.800871 I | ---> 2 {MsgApp 2 1 12 12 29 [] 29 {[] {{[] [] []} 0 0 []} []} false 0 [] []}
09:27:33 raftexample1 | 2019-01-16 09:27:33.800890 I | ---> 3 {MsgApp 3 1 12 12 29 [] 29 {[] {{[] [] []} 0 0 []} []} false 0 [] []}
```

`Send()`会循环处理这些消息的发送
//TODO: Transport里面的`peers`和`remotes`都是干嘛的.

```go
func (t *Transport) Send(msgs []raftpb.Message) {
	for _, m := range msgs {
		if m.To == 0 {
			// ignore intentionally dropped message
			continue
		}
		to := types.ID(m.To)

		t.mu.RLock()
		p, pok := t.peers[to]
		g, rok := t.remotes[to]
		t.mu.RUnlock()

		if pok {
			if m.Type == raftpb.MsgApp {
				t.ServerStats.SendAppendReq(m.Size())
			}
			p.send(m)
			continue
		}

		if rok {
			g.send(m)
			continue
		}

		if t.Logger != nil {
			t.Logger.Debug(
				"ignored message send request; unknown remote peer target",
				zap.String("type", m.Type.String()),
				zap.String("unknown-target-peer-id", to.String()),
			)
		} else {
			plog.Debugf("ignored message %s (sent to unknown peer %s)", m.Type, to)
		}
	}
}
```
