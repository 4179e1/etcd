# Raft 简介及学习资料汇总

## Raft简介

Raft是一个分布式一致性协议。简单的说，就是一个集群里面的所有节点都维护一个复制状态及（Replicated State Machine），要求这个状态机对于每一个给定的输入，总是产生相同的输出；这样，如果所有节点都以相同的顺序执行相同的输入，最终所有节点都会得到相同的输出。Raft协议接收来自不同客户端的请求，以相同的顺序把这些请求存储到所有节点上。每一个请求都以log的形式持久化到磁盘中，当大多数节点都存储了一条log之后，就把这个log提交到状态机执行（这个说法不严谨，需要一个额外的限制来绕过一个corner case）。

![Replicated Log](https://github.com/4179e1/etcd/raw/master/contrib/raftexample/doc/pic/rsm.png)

## Raft和Multi Paxos

Raft和Multi Paxos非常相似，都会选择一个节点作为Leader接受所有客户端的请求，主要的不同点在于：
- Paxos允许任何节点当选为Leader，log之间可能会有缺失的值，需要比较复杂的实现来补齐。
- Raft只允许log最全的节点当选Leader，log 保持连续，中间不存在缺失的值。

## Raft实现

Raft的实现很多，其中Golang的实现主要是两个
- [Etcd Raft](https://github.com/etcd-io/etcd/tree/master/raft) – 使用最广泛的Raft实现，但是它只实现了核心的Raft协议，存储、消息序列化和网络传输都需要用户自己实现。后续的分析基于这个库。
- [Hashicrop Raft](https://github.com/hashicorp/raft) – 主要是consul在用，完整的实现了WAL、SnapShot、存储、序列化。

## 学习资料汇总

- [Raft User Study](https://ramcloud.stanford.edu/~ongaro/userstudy/)
  - MIT的Raft/Paxos 教学PPT
  - TODO : upload vedio
- [In Search of an Understandable Consensus Algorithm (Extended Version)](https://raft.github.io/raft.pdf)
  - Raft的原始论文
- [CONSENSUS: BRIDGING THEORY AND PRACTICE](https://ramcloud.stanford.edu/~ongaro/thesis.pdf)
  - Raft 作者Diego的博士论文，比上文更详细