# 补充部分

## Raft 集群成员是怎么保存的

终于，我们能理清Raft node在初始化和重启的过程中是怎么保存的了，这部分是对[](./2-raftexample.md)中raftserver初始化的补充，包括三种情形

### 初始化

`node.StartNode()`的第二个参数中包含了所有peers，这个函数会
1. 为每一个peer创建一个`pb.ConfChangeAddNode`的log entry添加到raft log中，并标记为applied
1. 直接调用`raft.addNode()`把每一个peer添加进去。

等这些raft log被持久化到WAL中，在`node.Ready()`中通过`publishEntries()`发给应用层后，应用层会通过`node.ApplConfChange()`再调用一次`r.addNode()`，`r.addNode()`会忽略这种重复添加

### 重启，但是没有快照

没有快照的情况下，初始化后的peer的信息作为`pb.ConfChangeAddNode`保存在WAL中，（猜测，重启回放WAL时会把这些数据写到Storage暴露给raft log），在`node.Ready()`中通过`publishEntries()`发给应用层后，应用层会通过`node.ApplConfChange()`调用`r.addNode()`


### 重启，有快照

这种情况直接在`raft.newRaft()`中处理，WAL的快照记录中保存了peers，直接在这个函数完成初始化。如果后续还有节点的变更，在回放WAL的过程中通过`r.addNode()`完成，参照上一节