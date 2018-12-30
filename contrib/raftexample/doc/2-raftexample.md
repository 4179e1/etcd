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

## REST Server
REST Server是对外的接口，GET请求返回Key Value Store中的键值，PUT请求新建/更新KV键值对。

## Raft Server
Raft server是这里的一致性模块，当REST Server（经由Key Value Store）提交一个请求的时候，Raft server把这个请求发送到所有peer。当raft 协议达成共识（多数节点commit了这个请求）的时候，raft server 把所有已经commit的请求发布到一个commited channel，Key Value Store通过读取这个channel 更新自己的值。

注意一点，当raft协议返回一个成功的请求时，我们说这个请求是*committed*，当Key Value Store把这个请求写入自己的map时，我们说这个请求时*applied*。